# 4. The data you store

Everything lives in one Postgres database (S4). The record tables are the portal's own, extended with sync
metadata. Everything that exists only for the sync lives in a separate `sync` schema owned by the sync code.
Labels like `D15` and `T1` point to [decisions.tmp.md](decisions.tmp.md).

## 4.1 Overview

| Table | What it holds | Written by | Rows | Size |
|---|---|---|---|---|
| `customers`, `contacts` | Current values, plus per-field metadata in `field_meta` | portal backend, receiver, worker | 5M together | about 5 GB, plus about 1 GB of indexes |
| `history` | One row per field change, append-only, partitioned by month | all three | about 60 GB/year at a realistic 10 edits/s | 13 months kept, then archived (D15) |
| `sync.inbox` | Webhooks received in the last 7 days, partitioned by day | receiver | about 18M at peak | about 2.7 GB (T4) |
| `sync.outbox` | Portal changes waiting for, or recently sent to, Salesforce | portal backend, worker | one pending row per record, plus 7 days of sent rows | about 216 MB pending after a 6-hour outage (D21) |
| `sync.jobs` | Salesforce calls the receiver needs but may not make | receiver, worker | hundreds | tiny |
| `sync.heartbeat` | Last time each receiver could write | receiver | one per receiver | tiny |
| `sync.watermarks` | The reconciliation sweep's "modified since" point | worker | a few | tiny |
| `sync.sf_users` | Salesforce users who can be an owner (V3). Described, not built this weekend | worker | thousands | tiny |

## 4.2 Records: `customers` and `contacts` (T1, T2)

```
customers
  id               uuid  PK            -- portal's own ID; also sent as Portal_Id__c (D8)
  sf_id            text NULL           -- Salesforce Account ID once known (D9); unique when set (V5)
  name             text
  status           text                -- Account.Status__c
  owner_sf_id      text                -- Account.OwnerId, a Salesforce user ID (D10)
  field_meta       jsonb               -- per-field metadata, below
  sf_seq_max        bigint NULL        -- highest Salesforce sequence seen; a webhook above
                                       --   sf_seq_contiguous + 1 means a gap (D19)
  sf_seq_contiguous bigint NULL        -- highest sequence with NO gap below it: every event up to here
                                       --   is applied. The G1 precondition uses THIS (V1)
  sf_last_modified timestamptz NULL    -- Salesforce LastModifiedDate (precondition if no sequence, G3)
  created_at, updated_at timestamptz

contacts
  id, sf_id, field_meta, sf_seq_max, sf_seq_contiguous, sf_last_modified,
  created_at, updated_at                                                        -- as above
  first_name       text                -- Contact.FirstName  (brief's "full name" is two fields, S1)
  last_name        text                -- Contact.LastName
  phone            text                -- Contact.Phone
  email            text                -- Contact.Email
  city             text                -- Contact.MailingCity
  owner_sf_id      text                -- Contact.OwnerId
  customer_id      uuid NULL FK -> customers.id   -- NULL while the customer is pending (D14)
  customer_sf_id   text NULL           -- Contact.AccountId. From Salesforce for its contacts; for a portal
                                       --   contact, copied from the customer once its sf_id is known (V2)
```

The two link columns together give three states:

| `customer_id` | `customer_sf_id` | Meaning |
|---|---|---|
| set | set | Linked on both sides |
| NULL | set | Salesforce contact whose customer hasn't reached us yet: "customer pending" (D14) |
| set | NULL | Portal contact whose customer isn't in Salesforce yet. Its outbox row waits (V2) |

**How the two sequence columns move** (V1):

| Event | `sf_seq_max` | `sf_seq_contiguous` |
|---|---|---|
| Webhook with seq = contiguous + 1 | max(max, seq) | advances to seq, then keeps advancing while the inbox already holds the next sequence |
| Webhook with seq > contiguous + 1 (a gap) | raised | unchanged. A `replay_record` job is scheduled in 10 s (D19) |
| Reply to our own write, new seq N | max(max, N) | N, if N = contiguous + 1. The sent fields' `sfSeq` and `sfValue` move to N **only if N is newer** than what they hold, so a reply arriving after a newer webhook changes nothing (I1) |
| Fetch or sweep returning the record at seq N | max(max, N) | N: a full read covers everything up to N (history for any missed events is `via fetch` / `reconcile`) |

If Salesforce sends no `sequence` (G3), both columns stay NULL and the precondition uses `sf_last_modified`.

**`field_meta`**: one entry for each synced field, holding what the apply rule needs:

```json
{
  "city":  { "updatedAt": "2026-11-02T11:20:00.120Z", "source": "portal",
             "actor": "finance.user@client.example", "editId": 17, "sfSeq": 1, "sfValue": "Mumbai" },
  "phone": { "updatedAt": "2026-11-02T11:30:04.000Z", "source": "salesforce",
             "actor": "sales.user@client.example", "editId": 31, "sfSeq": 3, "sfValue": "+919876543210" }
}
```

| Key | Meaning | Used by |
|---|---|---|
| `updatedAt` | The time used to decide conflicts: Salesforce `occurredAt`, or the portal's time inside the save transaction, taken after the record lock | D16 |
| `source` | `portal` or `salesforce` | D16 tie-break, badge |
| `actor` | The person's email, as the brief's samples give it: `changedBy` on a webhook, the logged-in user's email on a portal save (the brief's `editedBy`). For a fetch or sweep, the record's last modifier's email (assumption 3.8) (V4) | history, badge |
| `editId` | The `id` of the history row that set the **current value**. Changes on every portal save, every Salesforce value that wins, and every revert. Unchanged by an echo or a Salesforce value that loses on time | Outbound guard and revert check: "is this still the edit we sent?" (O1′, O2′) |
| `sfSeq` | The Salesforce `sequence` that last set this field | D19 out-of-order rule |
| `sfValue` | The last value Salesforce is known to hold. Moves forward with `sfSeq` on any webhook, echo or confirmed send, even when the field keeps a newer portal value (inbound.md 5.2) | D10 revert after a rejection |

**Indexes:**
- `customers(sf_id)` and `contacts(sf_id)`: unique where not NULL. Webhooks look records up by this.
- `contacts(customer_id)`.
- `contacts(customer_sf_id) WHERE customer_id IS NULL`: finds waiting contacts when their customer arrives.

**Why a JSONB column, not more columns or a separate table** (T1):
- The portal reads and filters plain typed columns.
- The metadata is one row, so one lock (G2) and one write, and it loads straight into the apply function.
- Postgres doesn't enforce the JSONB shape, so the apply function validates it.

## 4.3 `history` (D15, S3b)

Append-only. Rows are inserted and never updated.

```
history                                   -- PARTITION BY RANGE (recorded_at), one partition per month
  id            bigint identity           -- also the edit's editId (O1′); one sequence across partitions
  record_type   text                      -- customer | contact
  record_id     uuid
  field         text
  old_value     jsonb
  new_value     jsonb
  actor         text                      -- email: sales.user@client.example (V4)
  source        text                      -- portal | salesforce
  via           text                      -- portal_save | webhook | fetch | replay | reconcile (S3b)
  occurred_at   timestamptz               -- the time the conflict rule compared
  recorded_at   timestamptz               -- when we wrote the row
  event_id      text NULL                 -- webhook / replay event
  sf_seq        bigint NULL
  superseded    boolean                   -- this value lost at the moment it arrived (older sequence or earlier time)
  note          text NULL                 -- e.g. "rejected by Salesforce: <reason>", "normalised by Salesforce"
```

- **Index:** `(record_id, recorded_at)`. History is read one record at a time (D15).
- **`superseded` is decided when the row is inserted**, because the apply rule knows at that moment whether
  the value won. A value that was current and is later replaced is ordinary history, not superseded. That
  is what keeps the table append-only.
- **Retention:** 13 monthly partitions in Postgres. Older ones are detached and archived to object storage,
  never deleted. About 200 B per row, about 0.9 GB per day at peak, and about 65–80 GB in Postgres in total.
- **Duplicate check on the rare path** (Q8): when every field of an event loses the sequence check, the apply
  step looks for a row of this record with the same `event_id` or `sf_seq`; if found, nothing is written. It uses
  the `(record_id, recorded_at)` index; one record's history is small.
- **The UI badge** (D16, G1b) is a query on this table: the latest rows for a field, compared by
  `occurred_at`.

## 4.4 `sync.inbox` (D18, T4)

```
sync.inbox                               -- PARTITION BY RANGE (received_day), one partition per day
  event_id      text
  received_day  date
  record_id     text    
  sequence      bigint
  received_at   timestamptz
  PRIMARY KEY (event_id, received_day)
  UNIQUE (record_id, sequence, received_day)
```

- The receiver does `INSERT … ON CONFLICT DO NOTHING`. Zero rows inserted means a duplicate: reply `200`
  and stop.
- **Each night the oldest partition is dropped.** That is instant and leaves no dead space.
- A duplicate that arrives on a **later day** lands in a different partition, so it passes this check. The
  per-field `sequence` rule (D19) then rejects its fields. The worst case is an extra history row marked
  superseded, never a wrong value.
- We store only the keys, not the body, so a webhook for a record is never applied twice and the table
  stays small.

## 4.5 `sync.outbox` (D23, T3, G1, G2b, O1′)

```
sync.outbox
  id               bigint identity PK
  record_type      text
  record_id        uuid                 -- our record (Portal_Id__c)
  fields           jsonb                -- {"city": {"value": "Noida", "editId": 17}, ...}: each value with the
                                        --   history row of the save it came from (O1′)
  content_gen      int                  -- content version; send key = "ob_<id>.<content_gen>" (O1′)
  base_version     bigint NULL          -- sf_seq_contiguous when the content was built: the precondition (G1, V1)
  status           text                 -- pending | sending | sent | rejected
  lease_until      timestamptz NULL     -- in the future = a call is in flight right now (O1′)
  attempts         int
  consecutive_412  int                  -- back off after 5 (G1b)
  next_attempt_at  timestamptz          -- when a pending row may be claimed, or a sending row retried
  created_at       timestamptz          -- picks the lane after an outage (D21)
  age_from         timestamptz          -- start of the 30 s clock for metric 1; = created_at, reset when a
                                        --   blocked contact becomes eligible to send (R1)
  blocked_by       text NULL            -- 'rejected_customer' while parked behind a rejected customer
                                        --   create; excluded from metric 1 (R1)
  last_error       text NULL
```

- **One pending row per record:** `UNIQUE (record_id) WHERE status = 'pending'`. A save does
  `INSERT … ON CONFLICT (record_id) WHERE status = 'pending' DO UPDATE SET fields = outbox.fields || EXCLUDED.fields`,
  so a newer edit to a field replaces that field's `{value, editId}`.
- **One row in flight per record:** `UNIQUE (record_id) WHERE status = 'sending'`. The claim skips a pending row
  while its record has a `sending` row, so a record's changes reach Salesforce in save order (O1′).
- **The send key** is `Portal_Change_Id__c = "ob_<id>.<content_gen>"`. A retry reuses it. `content_gen` goes up only
  when the content changes (outbound.md 6.6, 6.7). An echo finds its row by the `id` inside the key, so no
  separate key column or index is needed.
- **History is linked by `editId`,** not by key: `fields[f].editId` points at the history row of each save the
  row carries, in every content version.
- **Create or update is decided at claim time** (V7). With an `sf_id`: a conditional update with `base_version`,
  carrying only the row's fields. Without one: an upsert on `Portal_Id__c` carrying **the full current record**,
  each field's `editId` written into the row (O3′).
- **A contact waits for its customer** (V2, O3′). If the payload needs an `AccountId` and the customer has no
  `sf_id`, the row is parked: `next_attempt_at = now() + 5 min`. Whatever sets the customer's `sf_id` (reply,
  webhook adopt, fetch) queues `link_contacts`, which wakes it.
- **Timeout, `5xx` or `412`:** the row **stays `sending`** with the same content and key; `lease_until` is cleared
  and `next_attempt_at` set. It never goes back to `pending`. Whether an uncertain attempt landed is decided only by
  evidence (outbound.md 6.6).
- **Claiming and restart:** index on `(next_attempt_at) WHERE status IN ('pending', 'sending')`. A `sending` row is
  retried when it's due and `lease_until` has passed, including after a worker restart (outbound.md 6.3).
- **Sent rows** are kept 7 days, for echo matching and support, then deleted in small batches every minute.
  That keeps the delete load steady. This table can't be partitioned the way the inbox is, because the
  one-pending-row index must cover the whole table.
- **Size:** about 500 B per row. After a 6-hour outage: 432,000 pending rows, about 216 MB, less with
  merging (D21). Seven days of sent rows at a sustained portal peak of 20/s would be about 12M rows, about
  6 GB. At realistic rates it's well under that.

## 4.6 `sync.jobs` (D14, S3, D19)

```
sync.jobs
  id           bigint identity PK
  type         text          -- fetch_customer | fetch_record | replay_record | fetch_user | link_contacts
  record_type  text
  sf_id        text    
  run_at       timestamptz   -- 5 s / 15 s / 25 s schedule for fetches; +10 s for a gap replay or after a 412
  attempts     int
  status       text          -- pending | running | done | failed
  created_at   timestamptz
  UNIQUE (type, sf_id) WHERE status = 'pending'     -- one pending job of each kind per record
```

| Job | Created when |
|---|---|
| `fetch_customer` | A contact arrives whose customer we don't have (D14) |
| `fetch_record` | A webhook for an unknown record with **sequence > 1**, so we only have some of its fields (S3). An unknown record with **sequence 1** is a create carrying every field: we insert it, and no fetch is needed (V8) |
| `replay_record` | A sequence gap is still unfilled after 10 s (D19), or an outbox row got a `412` and `sf_seq_contiguous` hasn't caught up (V1) |
| `fetch_user` | A webhook carries an owner ID not in `sync.sf_users` (V3) |
| `link_contacts` | A customer's `sf_id` is set, by any path. Copies it into its contacts' `customer_sf_id` and wakes their parked outbox rows, in batches of 500 (O3′): `next_attempt_at = now()`, and for rows with `blocked_by` set, `blocked_by = NULL`, `age_from = now()` (R1). Makes no Salesforce call |

Finished jobs are deleted after 1 day. They use the 20 req/s headroom, never the 80 req/s outbox share.

## 4.7 `sync.heartbeat` and `sync.watermarks` (G3, G4, Q10a)

```
sync.heartbeat    receiver_id text PK, beat_at timestamptz        -- updated every 5 s by each receiver
sync.watermarks   name text PK, value timestamptz                 -- 'sweep:customers', 'sweep:contacts'
```

- **The replay start after any outage** (Salesforce's or our database's) is the **oldest** of the two sweep
  watermarks minus 10 minutes (Q10a, outage.md 10.3). A missing watermark means "replay from the change log's
  retention limit" and pages.
- **A sweep watermark advances only** when a sweep run has applied its last page (to that run's start time), or
  when a change-log replay completes (both, to the replay's start time). A failed or interrupted run leaves it
  unchanged. The run reads "modified since `value` − 2 min" (G3).
- **The heartbeat is a diagnostic only** (when each receiver last could write). It no longer sets the replay
  start: it keeps beating during a Salesforce outage (Q10a supersedes G4 here).
- A heartbeat is a timer write, not a write per webhook, so 30 webhooks/s never queue up on one row.

## 4.8 `sync.sf_users` (V3, assumption 3.4)

```
sync.sf_users
  sf_user_id  text PK        -- the value stored in owner_sf_id
  email       text
  name        text
  active      boolean
  synced_at   timestamptz
```

- **Refreshed every hour** by the worker with one "list active users" query, about 2,000 users per call
  (assumption 3.14). Users missing from the result are marked `active = false`, never deleted, because old
  records still point at them.
- **An owner ID we don't know** that arrives in a webhook is stored as received, and a `fetch_user` job looks
  it up. It uses the headroom.
- **The portal checks a new owner against `active = true` rows before saving**, so a bad owner is a `400` in
  the portal instead of a Salesforce rejection (D10).
- **Described, not built this weekend.** The demo seeds a few users. None of the brief's four checks needs it.

## 4.9 Who may touch what (S4, G2b)

| Postgres role | Records | `history` | `inbox` | `outbox` | `jobs` | `heartbeat` / `watermarks` / `sf_users` |
|---|---|---|---|---|---|---|
| `portal_app` | read / write through `saveEdit()` | insert, select | – | insert, merge pending | – | `sf_users`: read |
| `receiver` | read / write | insert | insert | **none** (loop rule, D17) | insert | heartbeat: write; `sf_users`: read |
| `worker` | read / write | insert | – | read / write | read / write | read / write |

Each role has `lock_timeout = 2s`, `statement_timeout = 5s` and `idle_in_transaction_session_timeout = 5s`
(G2b). Migrations run under their own role.

## 4.10 What is on disk after the 6-hour outage

The outage leaves about **432,000 pending outbox rows (about 216 MB)** in `sync.outbox`. They survive any
restart because they sit in Postgres and are written in the same transaction as the edit. Webhooks that
did arrive are already applied, and the rest come back through replay. The drain is covered in
section 10.
