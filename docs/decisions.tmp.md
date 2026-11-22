# Decisions log (temporary, from the grilling session)

Status key: OPEN = asked, waiting for answer · DECIDED = settled · BLOCKED = waits on another decision

Source of the method: mattpocock/skills `grill-me` -> `grilling` (design tree, asked in rounds;
each round = every question whose prerequisites are already settled).

## Design tree

```
Round 1 (no prerequisites)
  D1  Peak and quiet edit rates ................................ DECIDED (b)
  D2  Webhook payload: changed fields vs full record ............ DECIDED (a, changed fields only)
  D3  Catch-up after outage ..................................... DECIDED (a' replay log, a fallback)
  D4  Ordering signal on a webhook .............................. DECIDED (b, per-record sequence)
  D5  Duplicate detection keys .................................. DECIDED (eventId + recordId/sequence)
  D6  Time and actor of a portal edit ........................... DECIDED (c + logged-in user)
  D7  Webhook authenticity ...................................... DECIDED (a, HMAC)
  D8  Idempotent create / idempotency key ....................... DECIDED (c, Portal_Id__c + Portal_Change_Id__c)
  D9  Returned ID vs webhook recordId ........................... DECIDED (follows from D8 c)
  D10 Field writability, owner mapping, rejected updates ........ DECIDED (c, revert-if-unchanged + show)
  D11 Rate limit, outage behaviour, merging ..................... DECIDED (breaker + banner + ramp + merging)
  D12 Salesforce webhook retry behaviour ........................ DECIDED
  D13 Initial load of 5M records ................................ DECIDED (bulk file API, assumption)
  D14 Contact arriving before its customer ...................... DECIDED (c, fetch at 5 s / 15 s / 25 s)
  D15 History retention ......................................... DECIDED (b, 13 months hot + archive)

Round 2
  D16 Conflict rule, same field, opposite sides ......... DECIDED (a + guard; badge from history, no notifications)
  D17 Loop-stopping rule ................................ DECIDED (webhook path never writes outbox + echo by Portal_Change_Id__c)
  D18 Duplicate-webhook rule ............................ DECIDED (inbox, 7-day eventId retention)
  D19 Out-of-order webhook rule ......................... DECIDED (a, per-field sequence + gap detection)
  D20 Two portal users, same field (UX + what we send) .. DECIDED (b, per-field updatedAt check, 409, accept per field)
  D21 Outage drain arithmetic ........................... DECIDED (replay first; 122 min worst, ~62 min w/ merging; 50/50 lanes)

Later (architecture, after assumptions are locked)
  D22 One service or two; what crosses the boundary ..... DECIDED (b, receiver + worker, Postgres boundary)
  D23 Outbox / durable queue on disk .................... DECIDED (same-txn outbox, merge on save, 1 active worker)
  D24 Three metrics and alert thresholds ................ DECIDED (red line = the brief's 30 s)
  D25 Subproblems in build order ........................ DECIDED (a, riskiest rules first; metrics + bulk load removed)
  D26 What we leave out this weekend .................... DECIDED
```

## Decisions

### D1 - Peak and quiet edit rates - DECIDED
- Peak: 20 portal edits/s + 30 Salesforce edits/s (50/s total). Quiet hour: 1/s each side.
- Outage arithmetic assumes the portal peak is sustained for all 6 h (worst case).
- Baseline: 6 h x 3600 s x 20/s = 432,000 queued calls; drain at 80 req/s - 20/s new = 60/s net
  -> 432,000 / 60 = 7,200 s = 120 min (before merging; final number in D21).
- Why b: user judged it the most realistic for a few hundred sales + finance users on 5M records.
- If wrong: at 50/s the drain is 600 min and batching becomes mandatory; at 5/s it is 24 min.

### D2 - Webhook payload - DECIDED
- Assume the current API structure: a webhook carries changed fields only (matches the sample).
- Each field in `changedFields` was really changed at that event, so per-field ordering is safe.
- Other case (full record arrives): diff against the last Salesforce values we stored; only differing
  fields count as changes. Reason: a full record carries stale copies of fields Salesforce has not yet
  received from us (city-overwrite example).

### D3 - Catch-up after the 6-hour outage - DECIDED
- Assumption a': Salesforce keeps a replayable change log; after the outage we ask for every event after
  the last eventId we applied. Keeps every intermediate edit with actor and time. Log retention assumed >= 24 h.
- Fallback a (worst case): if the log has expired, query records modified since T; we get current values
  only, and history records "reconciled after outage, intermediate edits unknown".
- Rejected: "Salesforce queues our failed calls" - our calls fail at their edge; the brief puts pending
  calls on us (our outbox in Postgres).

### D4 - Ordering signal - DECIDED
- Assumption: each webhook carries a per-record `sequence` (added field; reason: exact per-record order
  and gap detection). Salesforce applies writes to one record one at a time (row lock).
- `eventId` is used as an opaque increasing cursor for replay only (D3).
- `occurredAt` kept for history and for cross-side conflicts (D16). Clock skew assumed <= 1 s (NTP).
- Dropped: global sequence; "Salesforce takes one API call at a time".

### D5 - Duplicate detection keys - DECIDED
- Primary key: `eventId`. Second key: (recordId, sequence) - catches a re-issued event with a new eventId.
- Duplicates happen when our 2xx is lost / we crash after commit / we exceed Salesforce's timeout.

### D6 - Time and actor of a portal edit - DECIDED
- Time = Postgres commit time of the edit (single clock, same transaction as the change).
- Actor = authenticated session user; body `editedBy` must match it or the save is rejected.
- If wrong (e.g. client time required): ordering becomes untrustworthy across browsers.

### D7 - Webhook authenticity - DECIDED
- HMAC signature over body + timestamp header, shared secret, 5-minute tolerance.
- Failure -> 401, nothing stored. Kept simple on purpose.
- If wrong (Salesforce can't sign): fall back to mTLS or IP allowlist + bearer token.

### D8 - Idempotent writes to Salesforce - DECIDED
- Assumption: two custom fields on the client's Salesforce objects:
  - `Portal_Id__c` (External ID): portal generates it before the first call; creates are upserts on it,
    so a retry after a lost reply updates the same record instead of creating a duplicate.
  - `Portal_Change_Id__c`: every write we send carries the idempotency key of that outbox row; it comes
    back in the webhook's `changedFields`, which identifies our own echo exactly (used by D17).
- Confirmation = the API reply. The key stays in the outbox until the reply says success or failure.
  The webhook only marks the echo; we never wait on a webhook that may not arrive.
- Why Q5 (inbound dedupe) does not cover this: the duplicate is created inside Salesforce by our retry;
  it then produces two webhooks with different recordIds and eventIds.

### D9 - Returned ID vs webhook recordId - DECIDED (follows from D8)
- The ID Salesforce returns on create is the record's permanent ID; every later webhook carries it.
- Store only the 18-character form (normalise any 15-character ID).
- If the create reply is lost, the create's webhook carries `Portal_Id__c`; we match on it and store the
  Salesforce ID then.

### D10 - Rejected update from Salesforce - DECIDED
- All nine fields assumed writable by the integration user; `owner` stored as Salesforce user ID.
- 4xx validation rejection -> no retry; show "rejected by Salesforce: <reason>" in the portal;
  rejected attempt stays in history.
- Revert guard: revert to the last Salesforce-confirmed value only if the field still holds the rejected
  value; if another user has since changed it, leave it and only mark the attempt rejected.
- 429 / 5xx / timeout -> retry (counts against the cap).

### D16 - Same field changed on both sides - DECIDED (notify mechanism pending)
- Per-field last-writer-wins: compare Salesforce `occurredAt` with Postgres commit time; skew assumed <= 1 s;
  exact tie -> Salesforce wins (deterministic). Losing value always kept in history.
- Outbox guard: before each send, drop any field whose current winner is a newer Salesforce value, so an
  older queued portal value never overwrites a newer Salesforce edit.
- Overwritten finance edits: no notifications table. The UI derives a field badge ("replaced by
  <actor> in Salesforce at <time>, later edit wins") from the history rows and their times; the reason is
  visible when the user opens the record. A notification feed can be added later as a query over history.
- Field ownership (option c) goes on the client question list.

### D11 - Rate limit and outage behaviour - DECIDED
- One record per request (documented record API) as the arithmetic baseline; batching listed as upside only.
- 429 / Retry-After pauses the whole sender for that long; work stays in the outbox.
- Merging: queued edits to the same record are merged into one call; history keeps every edit.
- Circuit breaker on the OUTBOUND path only: after 5 consecutive failures, stop sending and send one probe
  every 30 s. Only a successful probe closes it (an arriving webhook does not).
- Webhook endpoint never depends on the Salesforce API: during the outage some webhooks may still arrive
  (API down, UI up) and are applied normally; missing ones are found by sequence gaps and replayed (D3, D19).
- Portal stays writable; UI banner "Salesforce sync paused - your changes are saved" + per-record
  "pending sync".
- Ramp-up after recovery: 20 req/s (minute 1) -> 40 (minute 2) -> 80.

### D17 - Loop-stopping rule - DECIDED
- Core rule: only a portal user's save writes to the outbox; the webhook path never does. A change that
  came from Salesforce can never be sent back, so a loop is impossible by construction.
- Echo: a webhook whose `changedFields.Portal_Change_Id__c` matches a key we sent -> record its sequence,
  mark the outbox row "echo seen", no duplicate history row.
- If Salesforce altered our value (e.g. phone reformatted), apply the difference as a Salesforce change
  labelled "normalised by Salesforce"; nothing is sent back (core rule).
- Rejected: ignore by `changedBy == integration user` (drops real automation changes); value-match echo
  (swallows A->B->A).

### D18 - Duplicate webhook - DECIDED
- `inbox` table, unique `eventId`, unique (recordId, sequence). INSERT ... ON CONFLICT DO NOTHING.
- Nothing inserted -> duplicate -> return 200 (stops Salesforce retrying), apply nothing.
- Keep eventIds 7 days (~2.6M rows/day at 30/s); after that the per-record sequence check (D19) still
  rejects an old duplicate.

### D19 - Out-of-order webhook - DECIDED
- Per field, store the Salesforce sequence that last set it. Apply an incoming field only if its sequence
  is higher. The older event's changes to other fields still land; the event always goes to history,
  marked "superseded" where it lost.
- Gap detection: if sequence 7 arrives after 5, apply 7 and log the gap; if 6 has not arrived within 10 s,
  replay that record's events from the change log (D3).

### D20 - Two portal users, same field - DECIDED
- Optimistic check per field: the portal save request carries, for each field it changed, the `updatedAt`
  the form loaded. Server compares with the stored per-field `updatedAt` (already kept for D16).
- Mismatch -> that field rejected with 409 + current value, actor and time ("A changed City to Noida 3 s ago;
  keep or overwrite?"). Matching fields in the same save are accepted (per-field, brief: different fields survive).
- Overwrite = resend with the new `updatedAt`. No notifications table; A sees the history badge (D16).
- Salesforce receives each accepted value via the outbox (merged if still queued), each with its own
  `Portal_Change_Id__c`.

### D21 - Outage arithmetic - DECIDED
- Inputs: 20 portal edits/s, 6 h = 21,600 s -> backlog 20 x 21,600 = 432,000 calls; send rate 80 req/s
  (100 cap - 20 headroom); ramp 20 req/s (min 1), 40 (min 2), then 80.
- Drain time is independent of order: sent(T) = 1,200 + 2,400 + 80(T - 120) = 80T - 6,000;
  needed(T) = 432,000 + 20T -> 60T = 438,000 -> T = 7,300 s = 121.7 min ~ 122 min (worst case, no merging).
- Merging is implemented (decided). Illustration, labelled as a guess: if 50% of edits hit an already-queued
  record -> 216,000 calls -> 60T = 222,000 -> T = 3,700 s ~ 62 min. Real ratio = client question.
- Salesforce change-log replay: 30/s x 21,600 = 648,000 events / 2,000 per page (assumed) = 324 calls.
- On disk: ~432,000 outbox rows x ~500 B ~ 216 MB (less with merging), in Postgres, survives restarts.
- Lanes after recovery: 50/50 split of the current send rate between fresh edits (created after recovery)
  and backlog, each lane may borrow the other's unused share. At 20/s fresh: fresh 20, backlog 60 ->
  still 122 min; fresh edits back under 30 s after the ramp; backlog never starves (>= half the rate).
  Lane derived from `created_at` vs recovery time - no new field.
- Rejected: strict 50/50 without borrowing (backlog 432,000 / 40 = 180 min); fresh-always-first (backlog can stall).
- Order after recovery: replay Salesforce's change log FIRST (324 calls, ~16 s of minute 1), then drain.
  Reason: otherwise an older queued portal value (10:00) overwrites a newer sales edit (11:00) still in the
  log; the portal ends on 11:00 and Salesforce on 10:00, and nothing repairs it (D17 never sends back).
  With replay first, the D16 guard drops the older value. Recomputed: 60T = 438,324 -> T = 7,305 s ~ 122 min.
- Rate by phase: normal <= 80 req/s (20 req/s headroom always reserved); outage ~0 (one probe / 30 s);
  recovery 20 -> 40 -> 80 req/s; drain at 80 split 40/40 with borrowing.

### D22 - Services and boundary - DECIDED
- Two services, Postgres is the only boundary (no HTTP between them, no broker).
- Webhook receiver: verify HMAC (D7) -> dedupe (D18) -> apply by D16/D19 -> write history -> 200,
  all in one transaction. Never calls Salesforce. Needs a Salesforce call (customer fetch D14, gap replay D19)
  -> writes a job row.
- Sync worker: the only component that calls Salesforce. Drains outbox, runs change-log replay, executes job rows.
- Crosses the boundary: outbox rows, job rows, per-field state (read by the D16 guard), history.
- Rule: the receiver never calls Salesforce; the worker never receives HTTP. Cap enforced in one place.
- Rejected: one service (worker load/crash takes the endpoint down -> Salesforce retries); broker (extra system,
  dual-write problem, unnecessary at 50/s).

### D23 - Outbox - DECIDED
- Transactional outbox: record update + history row + outbox row commit in one Postgres transaction.
- Merge on save: at most one pending row per record; a new edit updates its field values. A row being sent is
  locked; edits arriving meanwhile go to a new pending row, so a retry resends identical content with the same
  `Portal_Change_Id__c`.
- One active worker + standby via Postgres advisory lock; in-memory token bucket at 80 req/s, ~50 calls in flight.
- Retries: 5xx/timeout -> exponential backoff with jitter; 429 -> pause all (D11); 4xx -> no retry, D10 revert.
  Never give up during an outage (breaker holds the work, D24 alerts).

### D25 - Subproblems in build order - DECIDED
Riskiest rules first (inside out):
1. Per-field state + history (value, updatedAt, source, Salesforce sequence; append-only history) - D15, D16, D19
2. Inbound apply rule as a pure function: duplicate -> out of order -> conflict (highest risk: silent loss) - D16, D18, D19
3. Outbox + idempotent send to fake Salesforce + loop rule - D8, D9, D17, D23
4. Rate limiter, 429, circuit breaker, restart durability - D11, D23
5. Outage catch-up: replay first, then 50/50 lanes - D3, D21
6. Webhook endpoint wrapper: HMAC, HTTP, 2xx-after-commit - D7, D12, D22
7. Edge features: contact before customer, rejection revert, portal 409, gap replay - D10, D14, D19, D20
- Removed from the build list by user: metrics (still described in the design doc, outline item 11) and
  initial bulk load (D13, out of scope).
- Rejected: outside-in (risky rules last); outbound-first (silent-loss rule later).

### D24 - Three numbers to watch - DECIDED
- User decision: the red line follows the brief's 30 s worst case, not a looser 120 s.
- M1 Oldest unsent portal change (outbox age): page when > 30 s while Salesforce is reachable.
  Separate outage alert: circuit breaker open > 5 min (30 s does not apply during an outage, per the brief).
- M2 Salesforce -> portal delay (our commit time - occurredAt): page when 95 of 100 webhooks are > 30 s late,
  or a missing sequence is still unfilled after 10 min.
- M3 Share of failed calls to Salesforce over 5 min: page when > 5% fail, or > 1% are 4xx rejections
  (usually a changed validation rule or permission in Salesforce).
- Trade-off accepted: no hold period on the 30 s line, so a short burst of 429s may page; a short hold can be
  added later if it proves noisy.

### D26 - Left out this weekend - DECIDED
- From the brief: deletes, Salesforce login screens, visual UI, second region, load test against real Salesforce.
- Ours: initial bulk load (D13, described only); batching (D11, upside only); notification feed (D16/D20);
  history archive job (D15, designed only); multiple active workers (D23); field ownership (client question);
  HMAC secret rotation and alert wiring (described only); portal user login (assumed to exist, D6).

### D12 - Salesforce webhook retries - DECIDED
- Salesforce retries when it gets no 2xx: endpoint down, 5xx, timeout, or our deliberate 429/503.
- Not retried: 401 (bad signature); events never sent because Salesforce itself was down (covered by D3).
- We return 2xx only after the event is durably in Postgres; we never rely on Salesforce's retry.

### D13 - Initial load - DECIDED
- Assumption: Salesforce offers a bulk file export/upload API; the 5M-record initial load goes through it
  before go-live, outside the 100 req/s record-API budget. Out of scope this weekend.
- If wrong: 5M / 80 req/s ~ 17.4 h through the record API.

### D14 - Contact before its customer - DECIDED
- Store the contact immediately with raw `customer_sf_id`, shown as "customer pending".
- If the customer has not arrived: fetch it at 5 s; on failure retry at 15 s and 25 s (10 s apart),
  all inside the 30 s sync target. Calls come from the 20 req/s headroom.
- If all three fail (Salesforce down), the contact stays pending until outage catch-up (D3).

### D15 - History retention - DECIDED
- Append-only field-change table, monthly partitions, 13 months in Postgres, older months archived to
  object storage (not deleted). Read per record, indexed (record_id, changed_at).
- Size: ~200 B/row; peak 50/s ~ 0.9 GB/day; realistic avg ~10/s ~ 60 GB/year -> Postgres bounded ~65-80 GB.
- Open question for client: real retention period; may the archive ever be deleted.
