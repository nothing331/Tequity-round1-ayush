# 5. The inbound path, from webhook to our database

The inbound path is everything that happens when **Salesforce tells us a record changed**. It starts when
the webhook reaches our endpoint and ends when the change is committed in Postgres and we reply `200`.
Table and column names are defined in [data.md](data.md). Labels like `D18` point to
[decisions.tmp.md](decisions.tmp.md).

```
Sales user edits in Salesforce
        |
        v
Salesforce --HTTPS POST /webhooks/salesforce--> Load balancer (TLS) --> Webhook receiver --> Postgres
                                                                         one transaction, then 200
```

Only the **webhook receiver** takes part. It never calls Salesforce (D22). When it needs something from
Salesforce, such as a missing customer or a missing event, it writes a row to `sync.jobs` for the worker.

## 5.1 The steps

| # | Step | Outcome if it stops here | Rule |
|---|---|---|---|
| 1 | Check `X-Timestamp` is within 5 minutes and that `X-Signature` is the HMAC-SHA256 of timestamp + body | `401`, nothing stored, Salesforce doesn't retry | D7 |
| 2 | Check the body has `eventId`, `recordId`, a known `objectType`, `sequence`, `occurredAt`, `changedBy` and `changedFields` | `400`, nothing stored, logged and counted in M3 | S2 |
| 3 | `BEGIN`. `INSERT INTO sync.inbox (event_id, received_day, record_id, sequence) ON CONFLICT DO NOTHING` | 0 rows inserted means a duplicate: `COMMIT`, `200`, stop | D18, T4 |
| 4 | Find our row and lock it (`withRecordLock`), in this order: by `sf_id`, then by `id = Portal_Id__c`, then create. See 5.3 | A `Portal_Id__c` row that already holds a **different** `sf_id`: not applied, logged, counted in M3 | G2, I2, S3, V8 |
| 5 | Is it **our own echo**? `changedFields.Portal_Change_Id__c` has the form `ob_<id>.<content_gen>`: read the outbox row by `id`, and it's the echo of that row's content version | Mark that outbox row confirmed (`sent`), and use the echo rules in 5.2 | D17, I1, O1′ |
| 6–7 | For **each field**, run the apply rule in 5.2: sequence first, then time | – | D19, D16 |
| 8 | Write values, `field_meta`, `sf_seq_max`, `sf_seq_contiguous`, history rows, and any jobs (5.4) | – | data.md |
| 9 | `COMMIT`, then reply `200` | Crash before the commit means no `200`, and Salesforce retries. Database down, or a lock wait over 2 s, gives `503` and Salesforce retries | D12, G2b |

Everything from step 3 to step 9 is **one transaction**. A `200` always means the change and its history are
on disk.

```
handleWebhook(req):
  verifySignature(req)                      -> 401
  event = parseAndValidate(req.body)        -> 400
  withTransaction:
    if not insertInbox(event): return 200   -- duplicate
    row = resolveRecord(event)              -- sf_id -> Portal_Id__c -> create (5.3)
    withRecordLock(row.id):
      state   = loadRecord(row.id)
      echoRow = findOutboxByKey(event.changedFields.Portal_Change_Id__c)   -- "ob_<id>.<gen>" -> row id
      result  = applyEvent(state, event, echoRow)    -- pure function, no I/O (subproblem 2)
      write(result.record, result.historyRows, result.jobs, result.outboxConfirm)
  return 200
```

## 5.2 The apply rule, field by field

`applyEvent` looks at each key in `changedFields` against that field's `field_meta`. The first matching row
decides.

| # | Condition | Field value and `updatedAt` / `source` / `actor` | `sfSeq` / `sfValue` | History row |
|---|---|---|---|---|
| a | Not a field we sync (e.g. `Description`) | unchanged; logged | unchanged | none |
| b | **Echo**, and the value is exactly what that row's content version sent (`fields[f].value`) (I1) | **unchanged**: the field keeps the portal save's time and actor | moved forward to this sequence and value **only if this sequence is newer** | none: the portal save already wrote one |
| c | **Echo**, but Salesforce changed our value, e.g. reformatted a phone (I1, D17) | judged by rows d–f like any Salesforce change | as rows d–f | as rows d–f, noted "normalised by Salesforce" |
| d | `event.sequence <= sfSeq` (an **older or repeated** event, D19) | unchanged | unchanged | yes, `superseded = true`, **unless every field of the event lost here and this record's history already has its `event_id` or `sf_seq`**: then it's a duplicate and nothing is written (Q8, ordering.md 8.1) |
| e | `occurredAt < updatedAt`: the portal value is **newer** (D16) | unchanged: the portal value stays and its outbox row will send it | set: Salesforce does hold this value right now | yes, `superseded = true` |
| f | Otherwise, including an exact time tie, which Salesforce wins (D16) | value = new value, `updatedAt = occurredAt`, `source = salesforce`, `actor = changedBy`, `editId` = this history row's `id` (so a queued portal edit of this field fails the outbound guard, O1′) | set | yes, `superseded = false` |

- **Echo first, then sequence, then time.** An echo kept as sent (row b) never writes a history row, even when it arrives late. Row d runs before rows e and f. An older event can never win on time, even
  with a skewed clock.
- **"Set" always means "move forward only".** `sfSeq` and `sfValue` change only when this event's sequence is
  higher than the stored one. The worker's `2xx` reply follows the same rule, so neither a late echo nor a
  late reply can move anything backwards (I1).
- **Full-record webhooks** (the other case, Q9): echo rows b and c are never used, because a snapshot can't prove
  our key belongs to this event. A field equal to our current value is a no-op (only `sfSeq` and `sfValue` move
  forward); a different value goes through rows d–f with `actor = changedBy`. A snapshot never confirms an
  uncertain send (loop.md 9.5).
- **Other fields in the same event** are judged separately. An old event's change to a field nobody has
  touched since still lands (D19).
- **The `Portal_Id__c` and `Portal_Change_Id__c` keys** are used to find our own row (5.3) and to recognise
  an echo. They aren't stored as fields.

## 5.3 Finding our row, and records we don't have yet

**Lookup order (I2).** `resolveRecord` stops at the first match:

| # | Lookup | If found | Why |
|---|---|---|---|
| 1 | `sf_id = recordId` | Use it | The normal case: we already know this record |
| 2 | `id = changedFields.Portal_Id__c`, locked (`FOR UPDATE`) | `sf_id` NULL: **adopt the row**, setting `sf_id = recordId`; for a customer, also queue `link_contacts` (outbound.md 6.8). `sf_id` already equal to `recordId` (the worker's reply got there first): use it. `sf_id` **different**: don't apply; log, count in M3 | Our own create whose API reply was lost. The webhook names our row by its External ID (D8, D9) |
| 3 | Not found, `sequence = 1` | `INSERT … ON CONFLICT (sf_id) DO NOTHING` with `id = Portal_Id__c` if present (else a new UUID) and `sf_id = recordId`, then lock whichever row holds that `sf_id`. It's a **create** carrying every field, so no fetch is needed (V8) | A record created in Salesforce |
| 4 | Not found, `sequence > 1` | Same insert, holding only the fields received, plus a `fetch_record` job at 5 s, 15 s and 25 s (S3). The fetch stamps fields with the record's sequence (`via = fetch`) | We missed the create |

- **Why this gives exactly one row:** the webhook and the worker both lock **the same portal row** (`id = u-1`)
  and both write the same `sf_id`. Whoever comes second finds it already set to that value. The unique index on
  `sf_id`, plus `ON CONFLICT DO NOTHING` in lookups 3–4, means two webhooks racing to create the same
  Salesforce record also end on one row.
- A `Portal_Id__c` holding a different `sf_id` would mean Salesforce has two records with our External ID.
  Assumption 3.2 makes it unique, so this is an alert, never a silent overwrite.

**Other cases:**

| Case | What happens |
|---|---|
| A contact whose `customer` (`AccountId`) we don't have | Store `customer_sf_id` and leave `customer_id` NULL, shown as "customer pending". Add a `fetch_customer` job (D14) |
| A contact whose `customer` **changed** | Relink `customer_id` to the new customer, or mark it pending and fetch the customer as above |
| A customer arrives that waiting contacts point at | Link its pending contacts (`customer_sf_id` matches) in batches of 500, each its own transaction (G2b) |
| A portal customer's `sf_id` gets set here (lookup 2 adopts it) | Queue `link_contacts`: it copies the `sf_id` to the customer's contacts and wakes their parked outbox rows (O3′, outbound.md 6.8) |

## 5.4 Record-level updates and jobs

| After applying the fields | Effect |
|---|---|
| `sequence = sf_seq_contiguous + 1` | `sf_seq_contiguous` advances, then keeps advancing while the inbox already holds the next sequence. `sf_seq_max` is raised |
| `sequence > sf_seq_contiguous + 1` (a **gap**) | `sf_seq_max` is raised, and `sf_seq_contiguous` stays. A `replay_record` job runs in 10 s if the gap is still open (D19). Any outbox send for this record meanwhile gets a `412`, never a stale write (V1) |
| An echo | Also sets the outbox row to confirmed (`sent`), whatever happened to the fields. This is the "landed" evidence for an uncertain attempt (outbound.md 6.6) |
| An owner ID not in `sync.sf_users` | A `fetch_user` job (V3) |

## 5.5 Worked example: the brief's sample webhook

**Before.** Finance has changed the city:

| Column | Value |
|---|---|
| `sf_id` | `0031A00000Abc` |
| `phone` | `+91 98100 11111`. `field_meta.phone = {updatedAt 11:00:00, source salesforce, actor sales.user@client.example, sfSeq 1, sfValue +91 98100 11111}` |
| `city` | `Noida`. `field_meta.city = {updatedAt 11:20:00, source portal, actor finance.user@client.example, sfSeq 2, sfValue Noida}` |
| `sf_seq_max` / `sf_seq_contiguous` | 2 / 2 |

**Webhook** (the brief's sample, plus `sequence`):

```json
{ "eventId": "evt_1001", "recordId": "0031A00000Abc", "objectType": "Contact",
  "sequence": 3, "occurredAt": "2026-11-02T11:30:04.000Z",
  "changedBy": "sales.user@client.example",
  "changedFields": { "phone": "+919876543210" } }
```

**Steps:**
1–2. The signature and body are valid.
3. Inbox row (`evt_1001`, today, `0031A00000Abc`, 3) is inserted, so it isn't a duplicate.
4. The contact is found by `sf_id` and locked.
5. There's no `Portal_Change_Id__c`, so it isn't an echo.
6–7. Phone: not an echo, 3 is above `sfSeq` 1, and 11:30:04 is later than 11:00:00, so row f applies.
8. `phone = +919876543210` and `field_meta.phone = {updatedAt 11:30:04, source salesforce, actor sales.user@client.example, sfSeq 3, sfValue +919876543210}`. Both sequence columns become 3. One history row: phone `+91 98100 11111` → `+919876543210`, `via webhook`, `event_id evt_1001`, `superseded false`.
9. `COMMIT`, then `200`.

**After:** the new phone, with the city still **Noida** from finance. History answers both "who changed the
phone" (sales.user) and "who changed the city" (finance.user). This is the brief's scenario.

## 5.6 The same path in the other cases

| What arrives | Where it's decided | Outcome |
|---|---|---|
| `evt_1001` **again** | Step 3 | `200`. Nothing changes |
| `evt_1001` again on a **later day** (the inbox partition has rolled over) | Row d | The phone isn't touched. One extra history row marked superseded (T4) |
| An **older** event, seq 2 with an old phone, after seq 3 | Row d | The phone stays. The old value goes to history as superseded |
| **Seq 5** while seq 4 is missing | 5.4 gap | Seq 5 is applied. `sf_seq_contiguous` stays at 3, and `replay_record` runs in 10 s |
| **Our own** city write coming back (seq 4, Noida) | Row b | The outbox row is confirmed. The field keeps finance as the actor, and there's no history row. **Nothing is sent back** (D17) |
| Our echo arriving **after** a newer sales edit (seq 5, Delhi) | Row b | City stays Delhi with its own actor. The echo only confirms the outbox row (I1) |
| Salesforce reformatted the phone we sent | Row c | The reformatted phone is applied, noted "normalised by Salesforce", and never sent back |
| Sales sets the city at 11:19:59, the webhook arrives after finance's 11:20:00 save | Row e | Portal keeps Noida. `sfValue` becomes Salesforce's value. The pending outbox row sends Noida with a precondition (G1) |
| A **new** contact (seq 1) whose customer we don't have | 5.3 | Created and shown "customer pending". `fetch_customer` runs |
| Our portal create's **API reply was lost**, and the create webhook arrives (seq 1, `Portal_Id__c u-1`, our key `ob_1042.1`) | 5.3 lookup 2, then row b | Row `u-1` is adopted (`sf_id` set) and the outbox row is confirmed `sent`. The fields keep the portal actor, with no history rows. **One contact** (I2) |
| The same, but the worker's 60 s resend gets there **before** the webhook | 5.3 lookup 1 | The resend's upsert finds the record by External ID, and its reply sets `u-1.sf_id`. The webhook then finds `u-1` by `sf_id`. Still **one contact** |

## 5.7 Who else uses this path

The worker runs **steps 3–9** (the same `applyEvent`) for events it gets from Salesforce itself:

| Source | `via` | Notes |
|---|---|---|
| Change-log replay, after an outage or for a gap (G4, D19) | `replay` | Each event also goes into the inbox, so the overlap with webhooks we already have is dropped |
| Single-record fetch (D14, S3) | `fetch` | One "event" carrying the record's current sequence and every field |
| Reconciliation sweep (G3) | `reconcile` | Only fields that are newer and different get written |

Steps 1–2 (signature and body checks) belong to the receiver alone.

## 5.8 Where the brief's required rules live on this path

| Brief | Here |
|---|---|
| A duplicate webhook | Step 3 (inbox), backed up by row d |
| An older webhook after a newer one | Row d (per-field `sequence`), plus the 5.4 gap replay |
| Our own update coming back (sync loop) | Rows b and c. The webhook path **never writes to the outbox** (D17) |
| Webhooks lost during the 6-hour outage | 5.7 replay from the oldest completed sweep watermark minus 10 minutes (Q10a, outage.md 10.3) |
