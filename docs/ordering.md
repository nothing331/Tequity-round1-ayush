# 8. The rule for a duplicate webhook, and for an older webhook that arrives after a newer one

Labels like `D18` point to [decisions.tmp.md](decisions.tmp.md). The steps are in [inbound.md](inbound.md) 5.1
and 5.2; tables in [data.md](data.md).

We add one field to the brief's webhook: a per-record **`sequence`** (D4). Salesforce writes one record at a
time, so each change to a record gets the next number. The brief's `eventId` identifies a delivery; `sequence`
gives exact order and shows when one is missing.

## 8.1 Duplicate webhook

**Rule: a webhook whose `eventId` we have already stored is a duplicate. We reply `200` and change nothing.**

| Layer | Check | Covers |
|---|---|---|
| 1. `sync.inbox` | `INSERT (event_id, received_day, record_id, sequence) ON CONFLICT DO NOTHING`. Zero rows inserted means a duplicate. A second unique key, (`record_id`, `sequence`, `received_day`), catches the same change re-sent under a new `eventId` | Duplicates within the same day, including two copies arriving at the same moment (the second insert waits for the first to commit, then does nothing) |
| 2. Per-field `sfSeq` | A field is applied only if the event's `sequence` is **higher** than `field_meta[f].sfSeq` | Duplicates on a later day (the inbox is partitioned by day and the oldest day is dropped after 7 days), and anything else that slips past layer 1 |

- **Why `200`, not an error:** Salesforce did nothing wrong. A non-2xx reply would only make it retry.
- **Why the inbox is checked first, inside the same transaction as the apply:** a crash after the insert but
  before the commit rolls both back, so the retry is treated as new, not as a duplicate.
- **Replayed events go through the same inbox,** so the overlap between a replay and the webhooks we already
  have is dropped the same way (inbound.md 5.7).
- **A duplicate past the inbox adds no history** (Q8). When **every** field of an event loses the sequence check,
  the apply step first looks in that record's history for a row with the **same `event_id` or the same
  `sf_seq`**. If one exists, the event is a duplicate and nothing is written. A true duplicate always loses every
  field, so this check runs only on that rare path.

**The two cases side by side.** The record already holds `evt_1001` (phone, seq 3), applied yesterday as history
row h31 (`event_id evt_1001`, `sf_seq 3`):

| | Same event replayed the next day | A genuinely older event, arriving for the first time |
|---|---|---|
| Arrives | `evt_1001`, seq 3, phone `+919876543210` | `evt_0999`, seq 2, phone `+91 98100 22222` (made at 11:25, delayed) |
| Inbox (today's partition) | Inserted: yesterday's row is in another partition | Inserted |
| Sequence check (phone `sfSeq` 3) | 3 ≤ 3, loses | 2 ≤ 3, loses |
| Every field lost? | Yes | Yes |
| History check, same `event_id` or `sf_seq` for this record | **Found** (h31: `evt_1001`, seq 3) | **Not found** (no row with `evt_0999` or seq 2) |
| History rows written | **0** | **1**: phone `+91 98100 22222`, `superseded = true`, `event_id evt_0999`, `sf_seq 2` |
| Field values | Unchanged | Unchanged |

**If a replay supplies a different `event_id` for the same record and sequence** (say `rep_77`, seq 3): the same
day, the inbox's second key (`record_id`, `sequence`, `received_day`) rejects it. On a later day, the history
check matches on `sf_seq` 3 instead, so it's still a duplicate and writes nothing. One record and one sequence are
one change, whatever ID it arrives under.

**Example.** `evt_1001` (phone, seq 3) is delivered twice. The first is applied (inbound.md 5.5). The second's
inbox insert finds `(evt_1001, today)` and does nothing: `200`, no field, history or job is written.

## 8.2 Older webhook after a newer one

**Rule: for each field, apply an incoming value only if the event's `sequence` is higher than the `sequence`
that last set that field (`field_meta[f].sfSeq`). Judge each field separately. Always keep the event in history.**

| Situation | Result |
|---|---|
| Seq 2 (phone A) arrives after seq 3 (phone B) | Phone stays B, because 2 ≤ 3. Phone A goes to history as `superseded` |
| Seq 2 changed **phone and email**, seq 3 changed only phone | Phone stays B. **Email from seq 2 lands**, because nothing newer has touched email |
| Seq 2 arrives and our portal value of that field is newer by time | Sequence check passes, time check keeps the portal value (item 7). `sfSeq` and `sfValue` still move to 2 |

- **Sequence first, time second.** The sequence check (inbound.md 5.2 row d) runs before the time comparison,
  so an older Salesforce event can never win on time, even with a skewed clock.
- **`sfSeq` and `sfValue` only move forward**, whichever path sets them: webhook, echo, reply, replay, fetch or
  sweep (I1).

## 8.3 A missing webhook (a gap)

| Moment | What happens |
|---|---|
| Seq 5 arrives, the record's `sf_seq_contiguous` is 3 | Seq 5 is applied field by field. `sf_seq_max` = 5; `sf_seq_contiguous` stays 3. A `replay_record` job is scheduled for 10 s later |
| Seq 4 arrives within 10 s | Applied by the same rule (its fields lose to seq 5 where both touched them). `sf_seq_contiguous` moves to 5, and the job finds nothing to do |
| Seq 4 hasn't arrived after 10 s | The worker replays that record's events after seq 3 from Salesforce's change log, and applies them through the same steps (`via replay`) |
| Still unfilled after 10 min | M2 pages (D24) |

While a gap is open, any outbox send for that record carries `if version = 3`, so it gets `412` instead of
overwriting a change we haven't seen (V1).

## 8.4 If Salesforce can't send `sequence` (G3)

| Rule | Falls back to |
|---|---|
| Duplicate | `eventId` alone in the inbox |
| Older after newer | Apply a field only if the event's `occurredAt` is later than the stored one. An exact tie keeps the existing value and is logged |
| Gap detection | Not possible. The 5-minute reconciliation sweep catches a missed change, without its intermediate history (`via reconcile`) |
| Precondition toward Salesforce | `sf_last_modified` instead of `sf_seq_contiguous` |
