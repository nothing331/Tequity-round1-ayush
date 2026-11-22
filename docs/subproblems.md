# Subproblems, in build order

Two-way sync of customer and contact records between Salesforce and the Tequity portal (Postgres).

The order is **riskiest rule first** (inside out). The most dangerous failure in this system is a change
that is **silently lost**: a field overwritten by an older value, or an edit dropped without a trace. That
risk lives in the per-field data model and the inbound apply rule, so those come first. They are built as
plain functions that can be tested before any HTTP or Salesforce code exists. Durability and the outage come
next, and standard plumbing comes last.

`Dn` labels point to the reasoning in [decisions.tmp.md](decisions.tmp.md). Assumptions are in
[assumptions.md](assumptions.md).

```
 1 Per-field state + history
          |
          v
 2 Inbound apply rule (pure) ---------------------+
          |                                       |
          v                                       |
 3 Outbox + idempotent send + loop rule           |
          |                                       |
          v                                       |
 4 Rate limiter, 429, breaker, restart            |
          |                                       |
          v                                       v
 5 Outage catch-up (replay first, lanes)    6 Webhook endpoint wrapper
          |                                       |
          +-------------------+-------------------+
                              v
                   7 Edge features
```

---

## 1. Per-field state and history

- **What:**
  - **Records:** customers and contacts.
  - **For every field:** value, `updatedAt`, source (portal / Salesforce), actor, and the last applied Salesforce `sequence`.
  - **History:** an append-only table with one row per field change: record, field, old value, new value, actor, source, time, `eventId` or change ID, `via` (`portal_save` / `webhook` / `fetch` / `replay` / `reconcile`, which says how exact the actor is), and a "superseded" flag.
- **Why first:** every rule reads or writes this. If state is kept per record instead of per field, the brief's own scenario fails: finance's city and sales's phone can't both survive.
- **Decisions:** D15 (history, retention), D16 (per-field `updatedAt`), D19 (per-field `sequence`), S1 (first/last name as two fields), S3b (`via`).
- **Locking (G2):** every write goes through `withRecordLock(recordId, fn)`: lock the record row, read, apply, write, commit. Lock order is customer before contact.
- **Done when:** the brief's scenario can be stored with both actors and both times visible. A Salesforce create, then a portal city edit, then a Salesforce phone edit all show in history, and the current state has the new phone and the finance city.

## 2. Inbound apply rule (a plain function with no I/O)

- **What:**
  - **Input:** a function receives the current per-field state and one incoming change: source, fields, time, `sequence`, `eventId`.
  - **Output:** which fields to write, which history rows to add, and which are marked superseded.
  - **Order of checks:** duplicate, then out of order, then conflict.
- **Why second:** **the highest risk.** Three of the brief's four rules live here (duplicate, out of order, and conflict, which the loop rule builds on), and this is where a change can be silently lost. As a plain function, every edge case is a test that runs in milliseconds.
- **Rules:**
  - **Duplicate (D18):** a seen `eventId`, or a seen (recordId, sequence), means no change.
  - **Out of order (D19):** a field is applied only if its `sequence` is higher than the field's stored one. An older event's changes to *other* fields still land. The event always goes to history.
  - **Conflict between sides (D16):** per field, the later time wins (`occurredAt` against Postgres commit time). On a tie, Salesforce wins. The loser stays in history.
- **Done when these tests pass:**
  - the same event twice changes nothing
  - sequence 5 arriving after 7 skips the field 7 set, but applies the field only 5 touched
  - Salesforce edits phone while the portal edits city, and both survive
  - both sides edit city, and the later one wins while the other stays in history
  - on an exact tie, Salesforce wins
  - **our echo (seq 4) arriving after a newer sales edit (seq 5) leaves the sales value and its actor, and writes no history row** (I1)
  - our echo kept as sent leaves the field's actor as the portal user, not the integration user (I1)

## 3. Outbox, idempotent send, and the loop rule

- **What:**
  - **The outbox:** a portal edit writes the record, history and an outbox row **in one transaction**.
  - **Merging:** each record has at most one pending row, and a new edit updates it. A row being sent is locked, so edits arriving meanwhile go to a new row.
  - **Sending:** the worker sends to a **fake Salesforce**. Creates are upserts on `Portal_Id__c`, and every write carries `Portal_Change_Id__c`.
  - **Confirmation:** the API reply is the confirmation, and it returns the record's new version.
  - **Conditional update (G1, V1):** every update carries `sf_seq_contiguous`, the highest sequence with no gap below it. A `412` means Salesforce has events we haven't applied. The row waits 5 s for in-flight webhooks, runs a `replay_record` job if it's still behind, then re-runs the guard on exact per-field times and resends only the winning fields. There's no GET. The row stays `sending` until evidence decides: an event carrying its key `ob_<id>.<gen>` means it landed; a full replay without it means it didn't (O1′).
  - **Create or update at claim time (V7):** an upsert on `Portal_Id__c` carrying the full current record if there's no `sf_id`, otherwise a conditional update. A contact whose customer has no `sf_id` is parked until `link_contacts` wakes it (V2, O3′).
  - **One row in flight per record (O1′):** `UNIQUE (record_id) WHERE status = 'sending'`, `lease_until` for "mid-call", and the claim skips a record with a `sending` row.
  - **Edit identity (O1′, O2′):** each outbox field carries the `editId` of its save; the guard and the revert check compare it with `field_meta.editId`.
  - **Three-step send (G2b):** claim (transaction), HTTP call (no transaction), store result (transaction). The Salesforce client throws if it is called inside `withRecordLock`.
- **Why third:** the next biggest risk is duplicates in Salesforce and loops. Every outbound feature depends on it.
- **Rules:**
  - **Safe retries (D8, D9):** a retry after a lost reply resends the same content with the same key. The upsert finds the existing record.
  - **Loop (D17):** only a portal save writes to the outbox. **The webhook path never does.** A webhook carrying our `Portal_Change_Id__c` is an echo: it confirms the outbox row, moves `sfSeq` and `sfValue` forward only if newer, keeps the portal actor, and writes no duplicate history row. If Salesforce changed our value (e.g. reformatted the phone), the difference is applied as a Salesforce change and never sent back (I1, inbound.md 5.2).
  - **Outbox guard (D16):** before sending, drop any field that a newer Salesforce value has already won.
- **Done when:**
  - a create whose reply is lost, then retried, leaves **one** record in the fake Salesforce **and one row in Postgres**, whether the create webhook or the worker's resend arrives first (I2)
  - a webhook received for our own write adds no outbox row and no extra history row
  - a webhook from Salesforce is never sent back to Salesforce
  - a queued older portal value doesn't overwrite a newer Salesforce value
  - **a sales edit that lands in Salesforce after our guard check but before our send gets a `412`, and both sides end on the later value** (G1)
  - calling the Salesforce client inside `withRecordLock` throws (G2b)
  - **with seq 2 missing and seq 3 received, our send carries version 1 and gets a `412`, not a stale write** (V1)
  - a contact created with its new customer is sent only after the customer's create is confirmed, including when the webhook adopts the customer before the reply (V2, O3′)
  - r1 times out, r2 waits, r1's retry gets `412`: r1 ends `sent` on the echo or replay carrying `ob_1042.1`, and only then is r2 sent (O1′)
  - phone rejected while the same value was saved again: the phone isn't reverted, the city is resent once (O2′)
  - after a worker restart, a `sending` row with an expired lease is resent with the same key and content (O1′)

## 4. Rate limiter, 429 handling, circuit breaker, surviving a restart

- **What:**
  - **One active worker,** with a standby (Postgres advisory lock).
  - **Rate limiter:** an in-memory limiter enforcing the **configured** cap: 80 req/s in production, lower in a demo.
  - **429:** pause all sending for `Retry-After`.
  - **5xx or timeout:** retry with exponential backoff and jitter.
  - **4xx:** no retry.
  - **Circuit breaker:** after 5 failures in a row, stop and send one test call every 30 s. Only a successful test closes it.
- **Why fourth:** the 6-hour outage rule depends on it, and durability is hard to add later.
- **Decisions:** D11, D23.
- **Done when:**
  - the worker **never exceeds the configured cap**, retries included
  - a 429 pauses sending and loses no work
  - the breaker opens and sends test calls every 30 s
  - **pending calls are still there after the process restarts while the fake Salesforce is down,** and are sent once it's back

## 5. Outage catch-up

- **What:** after the breaker closes, or after our own database comes back:
  1. **Replay Salesforce's change log first:** from the oldest completed sweep watermark (customers, contacts) minus a 10-minute overlap (Q10a, outage.md 10.3), falling back to "modified since T". A completed replay advances both watermarks. Not from "the last `eventId` we applied", because webhooks arrive out of order. The overlap's duplicates are dropped by the inbox and `sequence`.
  2. **Then drain** with a ramp of 20 → 40 → 80 req/s, split **50/50 between fresh edits and the backlog, each borrowing the other's unused share**. The lane comes from `created_at` against the recovery time.
- **Why fifth:** it builds on 3 and 4. Replaying first lets the guard drop an older queued portal value before any call; the conditional update would also catch it, so replay-first now saves a flood of `412`s (V1).
- **Decisions:** D3, D11, D21, G4, Q10a, Q10b.
- **Arithmetic it must reproduce (scaled down in a demo):**
  - **Backlog:** 432,000 calls.
  - **Replay:** 338 calls: 30 events/s × (21,600 + 900) s = 675,000 events ÷ 2,000. (The six-hour window alone, before adding the overlap, is 324.)
  - **Drain:** 60T = 438,338, so about **122 min** at the rate limit in the worst case, or about 62 min if 50% merged (60T = 222,338). End to end, with the ~270 s it takes to apply the replay first, about 128 min.
- **Done when:**
  - after a simulated outage, replay finishes before the first outbound call
  - a sweep that fails halfway leaves its watermark unchanged, and the replay starts at the older watermark minus 10 minutes
  - a missing watermark makes the replay start at the retention limit and raises a page
  - fresh edits are back under 30 s (M1 re-armed) by about 11.5 minutes after recovery, scaled to the demo's cap
  - a backlog that stops shrinking triggers the drain-stall page within the 5-minute window rule
  - the backlog drains in the time the formula predicts for the configured cap

## 6. Webhook endpoint wrapper

- **What:** an HTTP endpoint that, **in one transaction**:
  1. verifies the HMAC signature (failure → `401`, nothing stored)
  2. inserts into the inbox (`ON CONFLICT DO NOTHING`; a duplicate → `200` and stop)
  3. runs the apply rule from subproblem 2
  4. writes history
  5. returns `200` only after the commit

  When a Salesforce call is needed (a customer fetch or gap replay), it writes a **job row** for the worker. **It never calls Salesforce.**
- **Why sixth:** standard plumbing, low risk. The logic it wraps is already tested.
- **Decisions:** D7, D12, D18, D22.
- **Done when:**
  - a bad signature gets `401` and stores nothing
  - a duplicate gets `200` and changes nothing
  - a crash before the commit returns no `200`, so Salesforce retries

## 7. Edge features

| Feature | Rule | Decision |
|---|---|---|
| Contact before its customer | Store with `customer_sf_id`, shown as "customer pending". Fetch the customer at 5 s, 15 s, 25 s. Stay pending if all three fail | D14 |
| Rejected update (4xx) | Revert to `sfValue` **only if the field's `editId` still equals the rejected attempt's** (a newer save of the same value is kept). If the error names fields, resend the rest once. Never revert a rejected create. Show "rejected by Salesforce: <reason>" | D10, O2′, O3′ |
| Two portal users, same field | Each changed field carries the `updatedAt` the form loaded. A mismatch gets `409` with the current value. Other fields are accepted | D20 |
| Gap in the sequence | Apply the newer event and log the gap. If the missing one hasn't arrived within 10 s, replay that record from the change log | D19 |
| Overwritten finance edit | Badge on the field, built from history: "replaced by <actor> in Salesforce at <time>", with a "Use my value again" button that sends a normal PATCH with `baseUpdatedAt` | D16, G1b |
| Reconciliation sweep | Every 5 min, "records modified since the watermark minus an overlap", each record through the normal apply rule (`via = reconcile`). Catches webhooks missed by any path | G3 |
| Database outage recovery | After Postgres returns, replay from the oldest completed sweep watermark minus 10 minutes. Beyond the log's retention, "modified since". Beyond that, page "unrecoverable window" | G4, Q10a |
| Duplicate past the inbox | When every field loses the sequence check and history already has the `event_id` or `sf_seq`, write nothing | Q8 |
| Full-record webhooks | No echo rules from a snapshot; equal values are no-ops; a snapshot never confirms an uncertain send | Q9 |

- **Why last:** each is small and depends on 1–5.

---

## The brief's four required checks, by subproblem

| Check from the brief | Where it's built | Where it's tested |
|---|---|---|
| A duplicate webhook | 2 (rule), 6 (inbox) | 2, 6 |
| An older event after a newer one | 2 | 2 |
| A webhook that must not be sent back to Salesforce | 3 | 3 |
| A pending call still there after a restart while the fake Salesforce is down | 3 (outbox), 4 (breaker, restart) | 4 |

## Not in the build list

- **The Salesforce users table (V3):** described in data.md 4.8. The demo seeds a few users.

- **Metrics and alerts (D24):** described in the design document (three numbers, with the red line at the brief's 30 s). Not built this weekend.
- **Initial 5M-record bulk load (D13):** assumed to run through Salesforce's bulk file API before go-live. Out of scope.
