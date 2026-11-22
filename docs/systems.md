# 2. The systems, who calls whom, the network boundary, and the interface on each side

Labels like `D22` and `S4` point to the reasoning in [decisions.tmp.md](decisions.tmp.md). Assumptions are
in [assumptions.md](assumptions.md).

## 2.1 The parts

There are **three external parties**: Salesforce, the portal, and the public internet between them. On our
side there are **three components that share one Postgres database**:

| Component | What it does | Copies | Receives HTTP? | Calls Salesforce? |
|---|---|---|---|---|
| **Portal backend** (exists; we add the `saveEdit()` module) | Serves portal users. Every create or edit writes the record, history and an outbox row in one transaction (S4, D23) | as today | yes, from portal users | **never** |
| **Webhook receiver** (new) | Accepts Salesforce webhooks: checks, removes duplicates, applies and records them (D22) | 1, restarted on failure, rolling deploys (S6b) | yes, from Salesforce | **never** |
| **Sync worker** (new) | The **only** caller of Salesforce: sends the outbox with a version precondition, runs fetch and replay jobs, replays the change log after any outage, and runs the 5-minute reconciliation sweep (G3) | 1 active + 1 standby (advisory lock) | **never** | yes, at most 80 req/s |
| **Postgres** (exists; we add tables) | Records with per-field state, history, and the `sync` schema (`inbox`, `outbox`, `jobs`) | 1 primary | private only | no |

**The brief asks: one service or two?** Two, the receiver and the worker. **Postgres is the only boundary
between them:** no HTTP between them and no message broker.

- The receiver must answer Salesforce quickly, so it never waits on Salesforce. If the worker is busy
  draining a backlog or crashes, webhooks are still accepted.
- The rate limit is enforced in exactly one place, the worker.

## 2.2 Who calls whom

```
   Sales user                                              Finance / customer-success user
       |                                                                 |
       v                                                                 v
 +-------------+                                              +----------------------+
 |  Salesforce |                                              |    Portal backend    |
 |  (client's) |                                              |  + saveEdit() module |
 +-------------+                                              +----------------------+
    |      ^                                                        |   (one transaction:
    |      |                                                        |    record + history
    | (1)  | (2) record API: upsert / update /                      |    + outbox row)
    |      |     GET one record / change-log replay                  v
    |      |     <= 80 req/s, HTTPS, OAuth JWT bearer       +----------------------+
    |      |                                                |       Postgres       |
    |      |      ===== public internet (HTTPS only) =====  |  records, history,   |
    |      |                                                |  sync.inbox,         |
    |      |  NAT gateway, fixed IP, Salesforce-only egress  |  sync.outbox,        |
    |      +-----------------------------+                  |  sync.jobs           |
    |                                    |                  +----------------------+
    | HTTPS webhook                +-------------+             ^              ^
    | POST /webhooks/salesforce    | Sync worker |-------------+              |
    v                              | (1 active,  |  reads outbox + jobs,      |
 +----------------+                |  1 standby) |  writes results, history   |
 | Load balancer  |                +-------------+                            |
 | (TLS, port 443)|                                                           |
 +----------------+                                                           |
    |                                                                         |
    v                                                                         |
 +------------------+   one transaction: inbox + apply + history (+ job row)  |
 | Webhook receiver |---------------------------------------------------------+
 +------------------+
```

| # | Caller | Callee | Why | Path |
|---|---|---|---|---|
| 1 | Salesforce | Webhook receiver | A record changed in Salesforce | public internet → load balancer → receiver |
| 2 | Sync worker | Salesforce record API | A portal change to send, or a record to fetch or replay | worker → NAT gateway → public internet |
| 3 | Portal user | Portal backend | Create or edit in the portal | the portal's existing route and login |
| 4 | Receiver, worker, portal backend | Postgres | All state and every handoff | private network |

**Who never calls whom:**
- A sales user never reaches the portal, and a portal user never reaches Salesforce (brief).
- The receiver never calls Salesforce. The worker never receives HTTP. The portal backend never calls
  Salesforce or the worker (D22, S4).
- **The webhook path never writes the outbox.** That is the loop rule, D17.

## 2.3 What crosses the boundary between receiver and worker

Only rows in Postgres:

| Row | Written by | Read by | Purpose |
|---|---|---|---|
| `sync.outbox` | portal backend (`saveEdit()`) | worker | A portal change to send. At most one pending row per record, merged on save (D23) |
| `sync.jobs` | receiver | worker | A Salesforce call the receiver needs but may not make: fetch a missing customer (D14), fetch an unknown record (S3), replay one record after a sequence gap (D19) |
| per-field state on records | receiver, worker, portal backend | all three | The worker's guard drops a queued field that a newer Salesforce value has already won (D16) |
| history | all three | portal UI | Who changed what, when, and how it reached us (`via`, S3b) |
| `sync.inbox` | receiver | receiver | Duplicate removal by `eventId` and (recordId, sequence), kept 7 days (D18) |
| `sync.heartbeat` | receiver | worker | Last time the receiver could write. After a database outage, replay starts here minus 10 minutes (G4) |
| `sync.watermarks` | worker | worker | The reconciliation sweep's "modified since" point (G3) |
| `sync.sf_users` | worker | portal backend, receiver | Salesforce users who can be an owner, so the portal checks an owner before saving (V3) |

Full columns, indexes and sizes for every table are in [data.md](data.md).

### Three writers, one record: the lock rule (G2, G2b)

The portal backend, the receiver and the worker all write per-field state. Each write runs in one
transaction of the same shape, opened only through `withRecordLock(recordId, fn)`:

```
BEGIN -> SELECT ... FOR UPDATE on the record row -> read per-field state
      -> pure apply function -> write state + history (+ outbox / job row) -> COMMIT
```

- **Lock order:** customer before contact, then by ID. A deadlock victim retries.
- **One record per transaction** for replay, drain and sweep. Linking waiting contacts to a newly arrived
  customer (D14) runs in batches of 500.
- **No Salesforce call inside a transaction.** The worker sends in three steps:

  ```
  claim:   BEGIN; lock; guard; read sf_seq_contiguous; outbox row -> 'sending'; COMMIT
  call:    HTTP to Salesforce, no transaction open
  result:  BEGIN; lock; store new version / handle 412 / 4xx revert; COMMIT
  ```

  The Salesforce client throws if it is called while `withRecordLock` is active (unit-tested).
- **Postgres safety net, per role:** `idle_in_transaction_session_timeout = 5s`, `lock_timeout = 2s`,
  `statement_timeout = 5s`.
- **A crash between steps leaves nothing locked.** A `sending` row is retried, with the same key and content,
  once its retry is due and its `lease_until` has passed, including after a restart. At most one `sending`
  row per record (outbound.md 6.3, O1′).

## 2.4 The network boundary (S6)

| Zone | What is in it | Reachable from |
|---|---|---|
| **Public** | Load balancer, HTTPS on 443 only. Forwards only `POST /webhooks/salesforce` to the receiver | the internet |
| **Private: apps** | Receiver, worker, portal backend. No public IPs | the load balancer (receiver only); the portal's own route (portal backend) |
| **Private: data** | Postgres | the three app components only, one Postgres role each, least privilege |
| **Egress** | NAT gateway with a **fixed public IP** (one per availability zone). Allowlist: the Salesforce instance and login hosts only | the worker only |

- **Inbound trust:** the HMAC signature and timestamp, checked in the receiver (D7). There is no IP allowlist,
  because Salesforce's ranges are broad and change.
- **Outbound trust:** OAuth 2.0 JWT bearer as a dedicated integration user (S5). The client can add our
  fixed IP to that user's login IP ranges.
- **Secrets:** the HMAC secret is held by the receiver. The JWT private key is held by the worker. The
  database credentials are one role per component.

## 2.5 Interface on the Salesforce → us side: the webhook (S2)

`POST https://<our-host>/webhooks/salesforce`

| Header | Meaning |
|---|---|
| `X-Timestamp` | When Salesforce signed the request. Rejected if more than 5 minutes off |
| `X-Signature` | HMAC-SHA256 of `timestamp + "." + body` with the shared secret |

Body: the brief's sample, plus **`sequence`**. We add `sequence` because it gives exact order within a record
and lets us detect a missed webhook (D4, D19).

```json
{
  "eventId": "evt_9f2c...",
  "objectType": "Contact",
  "recordId": "0031A00000Abc",
  "sequence": 3,
  "occurredAt": "2026-11-02T11:30:04.000Z",
  "changedBy": "sales.user@client.example",
  "changedFields": {
    "phone": "+919876543210"
  }
}
```

- **Changed fields only** (D2). If a full record arrives instead, each field still goes through the same
  per-field `sequence` check, so unchanged fields are no-ops.
- Field keys are the logical names the brief's sample uses (`phone`, `city`). `changedBy` is an email, and it
  is stored as the history `actor` (data.md, V4).
- An unknown `recordId` with `sequence` 1 is a create carrying every field, so no fetch is needed. With
  `sequence` above 1 it goes to S3: store the fields we have, and fetch the rest (V8).
- `Portal_Id__c` and `Portal_Change_Id__c` appear in `changedFields` when our own write set them. That is how
  we recognise our own echo (D8, D17).

| Case | Response | What Salesforce does next |
|---|---|---|
| Applied, after the commit | `200` | stops |
| Duplicate (`eventId` or (recordId, sequence) already seen) | `200` | stops |
| Bad or missing signature, or stale timestamp | `401`, nothing stored | doesn't retry |
| Malformed body (missing `recordId`, unknown `objectType`) | `400`, nothing stored, logged, counted in M3 | doesn't retry |
| Our database unavailable, or a lock or statement timeout | `503` | retries with backoff (D12). If it stops retrying, the replay after the outage recovers the event (G4) |

A `200` is sent only after the transaction commits. A crash before that sends nothing back, so Salesforce
retries the webhook.

## 2.6 Interface on the us → Salesforce side: the record API (worker only)

All calls are HTTPS from the fixed egress IP, carry the OAuth bearer token, and count toward the cap. The
worker runs at **80 req/s**, leaving **20 req/s** for retries, fetches and replays (D11). There is one record
per request. Batching is not assumed.

| Call | Used for | Key detail |
|---|---|---|
| **Upsert by External ID** on `Portal_Id__c` | A record with no `sf_id` yet (decided at claim time, V7) | A retry after a lost reply finds the same record, so there is no duplicate (D8). The reply gives the record ID (D9). A contact whose customer has no `sf_id` yet is held 2 s at a time until it does (V2) |
| **Conditional update by record ID** | A record with an `sf_id` | Sends only the changed fields plus `Portal_Change_Id__c`, **only if the record is still at `sf_seq_contiguous`**, the highest sequence with no gap below it (G1, V1). The reply is the confirmation and returns the new version (D9) |
| **GET one record** | Missing customer (D14), unknown record with sequence > 1 (S3, V8) | The reply includes `sequence`, `LastModifiedDate` and the last modifier's email (assumption 3.8). **Not used after a `412`** (V1) |
| **List active users** | Hourly refresh of `sync.sf_users`, plus unknown owner IDs (V3) | About 1 call an hour from the headroom (assumption 3.14). Described, not built this weekend |
| **Change-log replay**: events from a point in time, 2,000 per page | After any outage, Salesforce's or ours: from the last heartbeat minus 10 minutes (G4). Also a per-record gap (D19) | Duplicates from the overlap are dropped by the inbox and `sequence` |
| **Records modified since T**: the 10 fields, 2,000 per page | The 5-minute reconciliation sweep (G3); the replay fallback (D3) | About 5 calls per object every 5 minutes at peak. Applied through the normal rule (`via = reconcile`) |
| **OAuth token** (login endpoint) | JWT bearer exchange | About 1 call/hour, assumed outside the cap (S5) |

| Reply | Worker does |
|---|---|
| `2xx` | Marks the outbox row sent and stores the returned ID and new version. On a create, it sets `sf_id` only if it's still NULL. If the create webhook already adopted the row with the same ID (inbound.md 5.3), nothing changes; a different ID is logged and counted in M3 (I2). The sent fields' `sfSeq` and `sfValue` move forward only if the new version is newer, so a late reply can't undo a newer webhook (I1) |
| `412` (Salesforce has events we haven't applied) | No GET (V1). The row **stays `sending`**. It waits 5 s for in-flight webhooks, then runs a `replay_record` job if no evidence has arrived. An event carrying the row's key (`ob_<id>.<gen>`) = landed → `sent`. A replay of every event after `base_version` without it = not landed → re-run the guard and resend (new `content_gen` if the content changed) (outbound.md 6.6, O1′). Fields that lost show the badge (G1b). After 5 `412`s in a row: slower retry, counted in M3 |
| `429` + `Retry-After` | Pauses **all** sending for that long and keeps the work (D11) |
| `5xx` or timeout | Retries with exponential backoff and jitter. 5 failures in a row open the breaker, with a test call every 30 s |
| `4xx` (validation) | Reverts a field only if its `editId` still equals the rejected attempt's, and shows "rejected by Salesforce: reason". If the error names fields, the rest are resent **once**; a second rejection ends the row. A rejected create is never reverted (outbound.md 6.7, O2′, O3′) |
| Timeout or `5xx` on any call | The row stays `sending` with the same key and content; it may have landed (O1′) |
| `401` | Refreshes the token once, then retries |

## 2.7 Interface on the portal side (S4)

| Endpoint | Body | Responses |
|---|---|---|
| `POST /api/customers`, `POST /api/contacts` | The new record's fields. The portal generates `Portal_Id__c` | `201` |
| `PATCH /api/customers/{id}`, `PATCH /api/contacts/{id}` | `changedFields`, plus for each field the `baseUpdatedAt` the form loaded (D20) | `200`; `409` per field, with the current value, who changed it and when; `400` |

- **Actor and time** (the brief's sample has neither): the actor comes from the portal login session. The time
  is the Postgres commit time (D6).
- **Name:** contact full name is `first_name` + `last_name`, mapped to `Contact.FirstName` and
  `Contact.LastName`, because `Contact.Name` is read-only in Salesforce (S1).
- **Lost a race with Salesforce** (G1b): the field shows a badge, "Replaced by <actor> in Salesforce at
  <time>. Your value '<v>' is in history", with a **"Use my value again"** button. The button sends a normal
  PATCH with `baseUpdatedAt` set to the current value's time, so a further change in the meantime gives a
  `409`. Nothing retries on the user's behalf.

### Field mapping (S1)

| Portal | Salesforce |
|---|---|
| customer name / status / owner | `Account.Name` / `Account.Status__c` (custom) / `Account.OwnerId` |
| contact first name / last name | `Contact.FirstName` / `Contact.LastName` |
| contact phone / email / city | `Contact.Phone` / `Contact.Email` / `Contact.MailingCity` |
| contact owner / customer | `Contact.OwnerId` / `Contact.AccountId` |
| (ours, on both objects) | `Portal_Id__c` (External ID), `Portal_Change_Id__c` |

Open question for the client: the story's "billing city" would be `Account.BillingCity`, but the field list
puts city on the contact (assumptions, question 11).

## 2.8 Scaling and failure, per component

| Component | If it fails | How it scales |
|---|---|---|
| Webhook receiver | Restarted by the health check in about 30–60 s. Salesforce retries in the meantime and nothing is lost. **M2 (webhook delay p95 > 30 s) is the trigger to run two copies** | Stateless: add copies behind the load balancer |
| Sync worker | The standby takes the advisory lock. If the active worker hangs, its heartbeat stops and the health check kills it, freeing the lock. Pending work is in the outbox, so nothing is lost | Not needed: Salesforce's cap, not our CPU, is the limit. Several workers would need a shared rate limiter |
| Portal backend | As today. Edits are saved with their outbox row, or not saved at all | As today |
| Postgres | Everything stops accepting writes, and the portal is down too, so no portal edits are made. Webhooks get `503`. On recovery, the worker replays from the last heartbeat minus 10 minutes (G4). Beyond the log's retention: "modified since". Beyond that: a page, "unrecoverable window T1–T2, full scan needed" | One primary is enough at 50 edits/s. A replica for failover is out of scope this weekend |
| Salesforce API | Breaker opens and the outbox grows (about 216 MB over 6 h). Replay first, then drain (D11, D21) | — |

## 2.9 What we assume from Salesforce, and the fallback if it is missing (G3)

The brief gives only `eventId`, `occurredAt`, and the create and update operations. The design adds the
items below. Each one is an assumption, and the table says which rules break and what we fall back to.

| Assumed | If missing, these rules break | Fallback |
|---|---|---|
| **`sequence`** per record on webhooks and in replies | D19 ordering and gap check; the (recordId, sequence) duplicate key; S3b stamp on fetched fields; G1 version | Order by **`occurredAt`**: apply a field only if its `occurredAt` is newer than the stored one, and on an exact tie keep the existing value and log it. Remove duplicates by `eventId` alone. G1 version = `LastModifiedDate`. Missed webhooks become invisible, so the sweep is the only safety net |
| **Retry on `503`** | D12; S6b's "a crash only delays webhooks" | A receiver crash or database outage loses webhooks. With `sequence`, the next edit shows the gap. The sweep catches records never edited again. S6b moves to 2 receiver copies |
| **Change-log replay** | D3 option a′; D19 per-record replay; G4 | "Records modified since T": current values only, with intermediate edits and actors lost (`via = reconcile`). Without that either: a full scan of 5M records, at least 2,500 calls |
| **Conditional update** (`412`) | G1 | No safe way to stop an in-flight write overwriting a newer sales edit. Fall back to repairing after the echo (G1 option b): Salesforce is briefly wrong, then corrected |
| **"Modified since" query** | G3 sweep; the D3 fallback | No sweep: a missed webhook without a later `sequence` gap stays wrong until the record is edited again. This would be a blocker question for the client |

**Worst combination:** no `sequence` and no retries. Every missed webhook is then caught only by the
5-minute sweep, so a missed change lands within about 5 minutes instead of 30 s, without its intermediate
history.
