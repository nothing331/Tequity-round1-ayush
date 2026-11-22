# 6. The outbound path, from a portal edit to a Salesforce API call

The outbound path is everything that happens when **a portal user saves a change**, until Salesforce
confirms it, rejects it, or we prove it landed. Table and column names are defined in [data.md](data.md).
Labels like `D23` and `O1′` point to [decisions.tmp.md](decisions.tmp.md).

```
Finance user saves
      |
      v
Portal backend: saveEdit()  --one transaction-->  Postgres: record + history + outbox row
                                                        |
Sync worker:  claim (tx)  ->  HTTP call (no tx)  ->  result (tx)       <= 80 req/s
                                                        |
                                           Salesforce record API
```

Two components take part. The **portal backend** writes the outbox row and never calls Salesforce. The
**sync worker** is the only caller of Salesforce, and never receives HTTP (D22, S4).

## 6.1 The save (portal backend)

`PATCH /api/{customers|contacts}/{id}` with `changedFields` and, for each field, the `baseUpdatedAt` the
form loaded. `POST` for a create; the portal generates the UUID `id`, which is also `Portal_Id__c` (S4, T2).

One transaction inside `saveEdit()`:

| # | Step | If it stops here | Rule |
|---|---|---|---|
| 1 | Validate the body. An owner must be an active row in `sync.sf_users` | `400` | S4, V3 |
| 2 | Lock the record row (`withRecordLock`) | Lock wait over 2 s: `503`, the portal retries once, then "please try again" | G2, G2b |
| 3 | For each field, compare `baseUpdatedAt` with `field_meta.updatedAt` | That field gets a `409`: current value, who set it, when. Other fields go ahead | D20 |
| 4 | Insert one **history row** per accepted field (old → new, `via portal_save`, actor = the logged-in user's email). Its `id` is that edit's **`editId`** | – | D15, O1′ |
| 5 | Write the value and `field_meta`: `updatedAt` = clock time taken now, under the lock; `source portal`; `actor` email; **`editId`**. `sfSeq` and `sfValue` don't change | – | D16, V4, O1′ |
| 6 | Merge into the record's pending outbox row: `fields[f] = {value, editId}` | – | D23, T3 |
| 7 | `COMMIT`, then reply `200`. The field shows "pending sync" | – | D11 |

```sql
INSERT INTO sync.outbox (record_type, record_id, fields, status, content_gen, next_attempt_at, created_at)
VALUES ($type, $id, $fields, 'pending', 1, now(), now())
ON CONFLICT (record_id) WHERE status = 'pending'
DO UPDATE SET fields = sync.outbox.fields || EXCLUDED.fields;   -- a newer edit to a field replaces its entry
```

- **Only `saveEdit()` writes to the outbox.** The webhook path never does. That's the loop rule (D17).
- A record can have **one `pending` row and one `sending` row** at most. An edit made while a row is being
  sent goes into the pending row and waits behind it (6.3).

## 6.2 The life of an outbox row

```
             saveEdit (merge)                    claim                   2xx / echo / replay proof
 (none) ─────────────────────▶ pending ───────────────────────▶ sending ─────────────────────────▶ sent
                                  ▲                              │   ▲
                                  │ guard empties it, or         │   │ retry due (timeout, 5xx, 412,
                                  │ nothing left to send         │   │ resend after a rejection):
                                  └──────── (closed)             │   │ same row, lease taken again
                                                                 │   └──────────────┘
                                                                 └──────── 4xx, twice at most ──▶ rejected
```

| Column | Meaning while the row moves |
|---|---|
| `status` | `pending` (mergeable, not sent yet), `sending` (sent at least once; **content frozen**), `sent`, `rejected` |
| `content_gen` | The content version. Starts at 1. Goes up **only** when the content changes after a "not landed" verdict (6.6) or a resend after a rejection (6.7) |
| Send key | `Portal_Change_Id__c = "ob_<id>.<content_gen>"`, e.g. `ob_1042.1`. The same content always travels with the same key |
| `lease_until` | Set when a call starts (now + 30 s: the 10 s timeout plus margin). While it's in the future, the row is **mid-call** |
| `next_attempt_at` | When a `pending` row may be claimed, or a `sending` row retried |
| `base_version` | `sf_seq_contiguous` when the content was built: the precondition (V1) |

**Indexes:** `UNIQUE (record_id) WHERE status = 'pending'`, `UNIQUE (record_id) WHERE status = 'sending'`,
and `(next_attempt_at) WHERE status IN ('pending', 'sending')`.

## 6.3 The claim (worker, one transaction)

```sql
BEGIN;
SELECT o.* FROM sync.outbox o
 WHERE ( o.status = 'pending' AND o.next_attempt_at <= now()
         AND NOT EXISTS (SELECT 1 FROM sync.outbox s
                          WHERE s.record_id = o.record_id AND s.status = 'sending') )
    OR ( o.status = 'sending' AND o.next_attempt_at <= now()
         AND (o.lease_until IS NULL OR o.lease_until < now()) )
 ORDER BY o.created_at
 LIMIT 50
 FOR UPDATE SKIP LOCKED;
-- for each row, under withRecordLock(record_id):
--   pending row  -> guard, build payload, base_version = sf_seq_contiguous, status = 'sending'
--   sending row  -> same payload, same key (a retry)
--   both         -> lease_until = now() + 30 s, attempts + 1
COMMIT;
```

- **No two sends for one record.** A pending row isn't picked while its record has a `sending` row, and the
  `UNIQUE … WHERE status = 'sending'` index makes a second claim fail even if two workers ever ran at once.
  A record's changes reach Salesforce in the order they were saved (O1′).
- **The guard** (D16, O2′): a field is kept only if `field_meta[f].editId` still equals the row's
  `fields[f].editId`. If a newer portal save or a newer Salesforce value replaced it, the field is
  dropped (and the newer value has its own path). A row the guard empties is closed without a call.
- **Building the payload** (V7, O3′):

  | Record state | Call | Payload |
  |---|---|---|
  | Has an `sf_id` | Conditional update, `if version = base_version` (G1, V1) | Only the row's fields, plus the key |
  | No `sf_id` | Upsert on `Portal_Id__c` | **The full current record**: every synced field, each with its `editId` written into the row so the row names exactly which edits it carries. Plus the key |

- **A contact needs its customer** (V2, O3′). If the payload needs an `AccountId` and the customer has no
  `sf_id` yet, the row is **parked**: `next_attempt_at = now() + 5 min`, a safety re-check. `AccountId` is read
  from `customers.sf_id` at claim time, never copied in advance.
- After an outage, the claim splits capacity 50/50 between fresh rows and backlog rows, each lane borrowing the
  other's unused share. The lane comes from `created_at` against the recovery time (D21).

**Restart.** A new worker takes the advisory lock and runs the same query. A `sending` row whose retry is due
and whose lease has expired is resent with **the same key and the same content**. The lease covers the dead
process's last call, so that call is over.

## 6.4 The call

- **Rate:** an in-memory token bucket at **80 req/s** for outbox sends. **20 req/s** of headroom covers retries,
  fetches, replays and the sweep (D11). About 50 calls in flight at once.
- **`429` + `Retry-After`:** pause **all** sending for that long and keep the work.
- **Circuit breaker:** 5 failures in a row → stop, and send one test call every 30 s. Only a successful test
  closes it. Then ramp 20 → 40 → 80 req/s (D11).
- **No transaction is open during the call.** The Salesforce client throws if called inside `withRecordLock`
  (G2b).

## 6.5 The result (worker, one transaction)

| Reply | What the worker does |
|---|---|
| **`2xx`** | Row → `sent`. On a create, set `sf_id` **only if it's still NULL** (the webhook may have adopted the row already, I2), and queue `link_contacts` (6.8). Move the sent fields' `sfSeq` and `sfValue` forward **only if newer** (I1). The fields keep their portal actor |
| **Timeout or `5xx`** | Row **stays `sending`**: same content, same key. It may have landed. `lease_until = NULL`, `next_attempt_at` = now + backoff with jitter. 5 failures in a row open the breaker |
| **`412`** | Row stays `sending`. The attempt is **uncertain** if an earlier copy might have landed (6.6). `next_attempt_at` = now + 5 s for the echo, then a `replay_record` job if no evidence has arrived. After 5 `412`s in a row: slower retry, counted in M3 (G1b) |
| **`4xx` validation** | See 6.7 |
| **`401`** | Refresh the token once, then retry (S5) |
| **`429`** | Pause everything (6.4). The row is retried with the same key |

## 6.6 Deciding whether an uncertain attempt landed (O1′)

After a timeout or a `412`, the worker never guesses. It waits for **evidence**:

| Evidence | Verdict | Then |
|---|---|---|
| An event (webhook or replay) carrying **this row's key** (`ob_1042.1`) | **Landed** | Row → `sent`. Fields follow the echo rules (inbound.md 5.2, row b or c) |
| A replay of **every** event after `base_version`, none carrying the key | **Not landed** | The record moved because of someone else's change. Re-run the guard. If the content changes, `content_gen` + 1 and a new key; `base_version` = the new `sf_seq_contiguous`; resend |
| Neither yet | **Unknown** | Keep waiting: 5 s for the echo, then `replay_record` |

- **A copy of the old attempt can't land later.** Every copy carries "if version = `base_version`", and the
  record has moved past it. That's why the "not landed" verdict is final.
- **No `sequence` from Salesforce (G3 fallback):** GET the record's `Portal_Change_Id__c`. Only we write that
  field, and the record's next row isn't sent until this one resolves, so the value equal to this row's key
  means it landed.
- **The newer pending row waits** until this row is `sent` or `rejected`.
- **History stays linked** across content versions: the row's `fields[f].editId` points at the history row of
  each save it carries, whatever the key became.

## 6.7 Rejections (O2′, O3′)

Salesforce rejects a whole update when one field fails validation.

| Case | What happens |
|---|---|
| **Update rejected, the error names fields** | For each named field: **revert to `sfValue` only if `field_meta[f].editId` still equals the row's `editId`** (no newer save, even one with the same value). Add a history note row: "edit h18 rejected by Salesforce: reason". Then **resend once** with the remaining fields that pass the guard: `content_gen` + 1, new key, `base_version` = current `sf_seq_contiguous` |
| **Update rejected, no field named** | Same editId check and revert for **every** field in the call. No resend |
| **The resend is rejected too** | Revert what remains (same editId check). Row → `rejected`. **No third attempt** |
| **Create rejected** | **No revert**: `sfValue` is NULL for a record Salesforce never had, and reverting would wipe the user's fields. Row → `rejected`. The portal shows "not in Salesforce: reason". The next save creates a new pending row, and its upsert carries the **full current record** (6.3) |

The reverted value is written as a history row noted "reverted: rejected by Salesforce", with
`source = salesforce`, `actor` = the Salesforce value's original actor, and a new `editId`.

## 6.8 A contact and its customer (V2, O3′)

| Moment | What happens |
|---|---|
| A contact's row needs an `AccountId` and the customer has no `sf_id` | The row is parked (6.3). The portal shows "waiting for customer" (or "… rejected by Salesforce" if the customer's create was rejected) |
| **Anything sets the customer's `sf_id`**: the create reply, the webhook adopting the row (inbound.md 5.3), or a fetch | That transaction queues a `link_contacts` job |
| `link_contacts` runs | Copies the `sf_id` into `customer_sf_id` on the customer's contacts, and sets `next_attempt_at = now()` on their parked rows. Batches of 500, each its own transaction (G2b) |
| The contact's row is claimed | `AccountId` = `customers.sf_id`, read at claim time |

Which path sets the `sf_id` first doesn't matter. The late one finds the same value and changes nothing (I2).

## 6.9 Worked example: finance changes the city

| Step | Rows |
|---|---|
| Save at 11:20:00 | History `h17`: city Mumbai → Noida, finance.user@client.example. `field_meta.city = {updatedAt 11:20:00, source portal, actor finance.user@…, editId h17, sfSeq 1, sfValue Mumbai}`. Outbox `1042`: `pending`, `fields {city: {"Noida", h17}}`, `content_gen 1` |
| Claim | Guard: `editId` is still h17. `base_version 1`, `sending`, `lease_until` 11:20:31 |
| Call | Conditional update `MailingCity = Noida`, `Portal_Change_Id__c = ob_1042.1`, if version = 1 |
| `2xx`, version 2 | `1042 → sent`. `sfSeq 2`, `sfValue Noida`. The city keeps finance as its actor |
| Echo (seq 2, key `ob_1042.1`) | Already `sent`. Moves nothing backwards, writes no history (inbound.md 5.2 row b) |

## 6.10 Other cases

| Case | Outcome |
|---|---|
| r1 times out, the retry gets `412`, and the echo hasn't arrived | r1 stays `sending`. The echo or a replay carrying `ob_1042.1` → `sent`. A replay without it → resend with a new `base_version`. r2 waits throughout (6.6) |
| City and phone sent together, phone rejected, the phone saved again (same value) while in flight | Phone's `editId` has moved on, so it isn't reverted; a note row records the rejection. City still has its `editId`, so it's resent once as `content_gen 2`. The new phone save goes out in r2 (6.7) |
| A customer create is rejected for a missing name, then only the name is saved | The next upsert carries name, status and owner with their `editId`s. Whichever of the webhook or the reply sets `sf_id` first queues `link_contacts`, which wakes the contacts (6.8) |
| A create's reply is lost | The webhook adopts the row by `Portal_Id__c` (inbound.md 5.3). The resend, if any, upserts on the same External ID. One record on both sides (I2) |
| Sales edits the same field while our row is in flight | The call gets `412` (or passes, if sales's change is later in Salesforce). 6.6 decides, and the guard keeps whichever edit is later (G1) |
| Our worker restarts mid-call | The row stays `sending`. After the lease expires it's resent with the same key and content (6.3) |
| A 6-hour Salesforce outage | Rows stay `pending` or `sending` in Postgres across restarts. After recovery: replay first, then drain at 20 → 40 → 80 req/s in two lanes (D21, outline item 10) |

## 6.11 Where the brief's required rules live on this path

| Brief | Here |
|---|---|
| A sync loop | Only `saveEdit()` writes the outbox (6.1). Our echo is recognised by its key and never re-sent (inbound.md 5.2) |
| A pending call still there after a restart while Salesforce is down | The outbox row is in Postgres, written with the edit (6.1). A restart re-claims `sending` rows whose lease has expired (6.3) |
| The 100 req/s cap, including retries | One token bucket at 80, with retries from the 20 req/s headroom (6.4) |
| Two people editing the same record | Per-field `409` on the portal (6.1); `editId` guard and precondition toward Salesforce (6.3, 6.6) |
