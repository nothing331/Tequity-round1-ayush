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
| 4 | Find the record by `sf_id` and lock its row (`withRecordLock`). If it's unknown, see 5.3 | – | G2, S3, V8 |
| 5 | Is it **our own echo**? Does `changedFields.Portal_Change_Id__c` match an outbox `change_id`? | Mark that outbox row confirmed, and use the echo rules in 5.2 | D17, I1 |
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
    withRecordLock(event.recordId):         -- creates the row first if unknown (5.3)
      state   = loadRecord(event.recordId)
      echoRow = findOutboxByChangeId(event.changedFields.Portal_Change_Id__c)
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
| b | **Echo**, and the value is exactly what we sent (I1) | **unchanged**: the field keeps the portal save's time and actor | moved forward to this sequence and value **only if this sequence is newer** | none: the portal save already wrote one |
| c | **Echo**, but Salesforce changed our value, e.g. reformatted a phone (I1, D17) | judged by rows d–f like any Salesforce change | as rows d–f | as rows d–f, noted "normalised by Salesforce" |
| d | `event.sequence <= sfSeq` (an **older or repeated** event, D19) | unchanged | unchanged | yes, `superseded = true` |
| e | `occurredAt < updatedAt`: the portal value is **newer** (D16) | unchanged: the portal value stays and its outbox row will send it | set: Salesforce does hold this value right now | yes, `superseded = true` |
| f | Otherwise, including an exact time tie, which Salesforce wins (D16) | value = new value, `updatedAt = occurredAt`, `source = salesforce`, `actor = changedBy` | set | yes, `superseded = false` |

- **Echo first, then sequence, then time.** An echo kept as sent (row b) never writes a history row, even when it arrives late. Row d runs before rows e and f. An older event can never win on time, even
  with a skewed clock.
- **"Set" always means "move forward only".** `sfSeq` and `sfValue` change only when this event's sequence is
  higher than the stored one. The worker's `2xx` reply follows the same rule, so neither a late echo nor a
  late reply can move anything backwards (I1).
- **Other fields in the same event** are judged separately. An old event's change to a field nobody has
  touched since still lands (D19).
- **The `Portal_Id__c` and `Portal_Change_Id__c` keys** are used only to recognise an echo. They aren't
  stored as fields.

## 5.3 A record we don't have yet

| Case | What happens |
|---|---|
| Unknown `recordId`, `sequence = 1` | A **create** carrying every field. Insert the row with a new UUID `id` and `sf_id = recordId`, then apply the rule. No fetch is needed (V8) |
| Unknown `recordId`, `sequence > 1` | We've missed the create. Insert a row holding only the fields received, and add a `fetch_record` job at 5 s, 15 s and 25 s (S3). The fetch stamps fields with the record's sequence (`via = fetch`) |
| A contact whose `customer` (`AccountId`) we don't have | Store `customer_sf_id` and leave `customer_id` NULL, shown as "customer pending". Add a `fetch_customer` job (D14) |
| A contact whose `customer` **changed** | Relink `customer_id` to the new customer, or mark it pending and fetch the customer as above |
| A customer arrives that waiting contacts point at | Link its pending contacts (`customer_sf_id` matches) in batches of 500, each its own transaction (G2b) |

## 5.4 Record-level updates and jobs

| After applying the fields | Effect |
|---|---|
| `sequence = sf_seq_contiguous + 1` | `sf_seq_contiguous` advances, then keeps advancing while the inbox already holds the next sequence. `sf_seq_max` is raised |
| `sequence > sf_seq_contiguous + 1` (a **gap**) | `sf_seq_max` is raised, and `sf_seq_contiguous` stays. A `replay_record` job runs in 10 s if the gap is still open (D19). Any outbox send for this record meanwhile gets a `412`, never a stale write (V1) |
| An echo | Also sets the outbox row to confirmed (`sent`), whatever happened to the fields |
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
| Webhooks lost during the 6-hour outage | 5.7 replay from the heartbeat minus 10 minutes (G4) |
