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

## Phase 2 - The systems section (outline item 2)

```
Round 1 (prerequisites already settled by D1-D26)
  S1  Object and field mapping (Account/Contact, full name, city, status, owner) .. DECIDED
  S2  Webhook endpoint contract (path, headers, body, response codes) ............. DECIDED
  S3  Update webhook for a record we do not have yet ............................. DECIDED (a)
  S4  Portal edit interface and who writes the outbox ............................ DECIDED (a)
  S5  How the worker authenticates to Salesforce ................................. DECIDED (a)
  S6  Network boundary (what is public, egress, database exposure) ............... DECIDED (egress a)

Round 2
  S3b History rows for a fetched record, and the fetch's sequence ............... DECIDED (via column; GET returns sequence)
  S6b Receiver instances behind the load balancer ............................... DECIDED (1 receiver, auto-restart; 1 worker + standby)

Frontier empty - all Phase 2 decisions made.

Then: write the systems section (diagram + interface on each side) from S1-S6.
```

### S1 - Object and field mapping (DECIDED)
- customer = `Account`, contact = `Contact`.
- customer name -> `Account.Name`; status -> `Account.Status__c` (custom); owner -> `Account.OwnerId`.
- contact full name -> **stored as two portal fields** `first_name` / `last_name` -> `Contact.FirstName` /
  `Contact.LastName`, shown together as the full name. Reason: `Contact.Name` is read-only in Salesforce.
  They are two separate fields for conflicts, ordering and history.
- phone / email -> `Contact.Phone` / `Contact.Email`; city -> `Contact.MailingCity`;
  owner -> `Contact.OwnerId`; customer -> `Contact.AccountId`.
- The design doc must say why the brief's "full name" is two fields.
- Client question: the story says "billing city" (`Account.BillingCity`) but the field list puts city
  on the contact.
- Rejected: split on the last space (silent damage to names like "Mary Ann Van Dyke");
  custom `FullName__c` (drifts from First/Last edited in Salesforce).

### S4 - Portal edit interface and who writes the outbox (DECIDED: a)
- `POST /api/{customers|contacts}` (portal generates `Portal_Id__c`);
  `PATCH /api/{customers|contacts}/{id}` with `changedFields` + per-field `baseUpdatedAt`;
  responses `200`, `409` per field, `400`.
- A shared `saveEdit()` module in the portal backend writes record + history + outbox in one
  Postgres transaction.
- Accepted costs and their mitigations (also written into the project CLAUDE.md so future work
  keeps them): sync tables in their own `sync` schema; same language (TypeScript/Node, assumed);
  every portal write goes through `saveEdit()` with a test that fails on a record change without an
  outbox row; capped connection pool per service; a failed outbox insert fails the save (deliberate);
  one Postgres role per service with least privilege.
- Rejected: internal HTTP to the sync service (two writes, breaks D23); Postgres trigger (fires on
  webhook writes, breaks D17 unless hidden session-variable logic).

### S6 - Network boundary (DECIDED)
- Public: only `POST /webhooks/salesforce`, HTTPS 443, TLS at the load balancer, HMAC in the receiver.
  No inbound IP allowlist (Salesforce ranges are broad and change); HMAC is the check.
- Outgoing (a): the worker reaches the internet only through a NAT gateway with a **fixed public IP**
  (one per availability zone), which the client can add to the integration user's Login IP Ranges;
  plus an **outbound allowlist** to the Salesforce instance and login hosts only.
- Postgres: private network only. Portal backend: reached by portal users through its own login,
  never talks to Salesforce.
- Rejected: fixed IP without allowlist (worker can reach the whole internet); whatever cloud IP
  (outbound sync stops if the client restricts login IPs).

### S6b - Instances (DECIDED)
- Worker: one active copy plus a standby via the Postgres advisory lock (D11). More workers would not
  send faster; the cap is Salesforce's 100 req/s.
- Receiver (round 3, a): **one copy**, restarted by the platform's health check on a crash (about
  30-60 s), with **rolling deploys** (the new copy passes its health check before the old one stops).
  Stateless, so scaling out later is a setting.
- Accepted risk: a crash can delay webhooks past 30 s. Nothing is lost (Salesforce retries, D19 gap
  check). **M2 (webhook delay p95 > 30 s) is the trigger to move to two running copies.**
- Rejected: a cold spare (same recovery time as an auto-restart, more setup); two running copies
  now (user chose to start small; a running "emergency copy" costs the same as two copies).

### S3b - Fetched record history and order (DECIDED)
- Part 2: assume the record API's GET reply includes the record's current `sequence`; fetched fields
  are stamped with it so a late older webhook is rejected by D19. Fallback if not: compare
  `LastModifiedDate` with `occurredAt` (weaker).
- Part 1 (round 3, a): one history row per fetched field, and history gains a **`via`** column:
  `portal_save` | `webhook` | `fetch` | `replay` | `reconcile`. For `fetch`, the actor is the record's
  `LastModifiedById` and the UI says "fetched from Salesforce, last modified by X". `reconcile`
  replaces the free-text "reconciled after outage" note from D3.
- Rejected: a yes/no `actor_exact` column (says unsure but not why); reading Salesforce field history
  objects (extra calls from the headroom, needs field history tracking on).

### S2 - Webhook endpoint contract (DECIDED)
- One endpoint `POST /webhooks/salesforce`, split by `objectType`. Headers `X-Signature`
  (HMAC-SHA256 of timestamp + body) and `X-Timestamp`. Body = the sample plus `sequence`;
  `Portal_Id__c` / `Portal_Change_Id__c` arrive inside `changedFields`.
- Codes: `200` applied or duplicate; `401` bad signature or stale timestamp (no retry);
  `503` our database is down (Salesforce retries).
- **Malformed body -> `400`, nothing stored, no retry.** User's reason: a broken body is the
  sender's problem, and retrying it can never succeed. We still write an application log line and
  count it toward the M3 failure rate, so a broken Salesforce payload is noticed.
- Rejected: store as `invalid` in the inbox and return `200` (hides the error from Salesforce).

### S3 - Update webhook for an unknown record (DECIDED: a)
- Store a partial record with the fields received, and write a job row so the worker fetches the
  full record (5 s / 15 s / 25 s, from the 20 req/s headroom), same pattern as D14.
- **Refined by V8:** an unknown record with `sequence` 1 is a create carrying every field - insert it, no
  fetch. The partial-plus-fetch path applies only to `sequence` > 1.
- Rejected: ignore and wait for replay (replay may never run outside an outage); hold the webhook
  (the create may never come).
- Follow-up S3b: what history rows the fetch writes, and how fetched values are ordered against
  late webhooks.

### S5 - Salesforce authentication (DECIDED: a)
- OAuth 2.0 JWT bearer flow for a dedicated integration user; private key held by the worker,
  certificate uploaded to a Salesforce connected app.
- Token cached until expiry, refreshed on `401`. Token calls go to the login endpoint, assumed
  outside the 100 req/s record-API cap (about 1 call/hour even if counted).
- Rejected: username-password (stored password, expiry breaks sync); client credentials
  (depends on the client's org setup).

## Phase 3 - Review questions from the user (gaps found)

```
  G1  Same field, both sides, while our write is in flight (also: guard-then-send race) .. DECIDED (a)
      Found: with D16 + D17 as written, portal ends "Delhi" and Salesforce "Noida" forever,
      or the reverse, depending on which webhook arrives first. The outbox guard only knows
      what has already reached us, so it cannot stop this.
  G2  Concurrent writers of per-field state (portal backend, receiver, worker) ............ DECIDED (a)
      Found: no lock or compare-and-set is specified; two transactions can read the same
      state and the later commit silently discards the earlier decision.
  G3  Fallbacks if Salesforce has no sequence / no retry on 503 / no change-log replay ... DECIDED (table + sweep a)
  G4  Postgres down longer than Salesforce keeps retrying ............................... DECIDED (a)
      Also found: "replay after the last eventId we applied" is not a safe cursor when
      webhooks arrive out of order (we may hold 105 but miss 103).

Round 2
  G1b What the portal user is told after losing a G1 race, and what they can do ......... DECIDED (a)
  G2b How "never hold a lock while calling Salesforce" is enforced, and other ways a
      lock can be held too long ......................................................... DECIDED (a)

Frontier empty - all Phase 3 decisions made.
```

### G1b - What the user sees after a 412 (DECIDED: a)
- Their edit still later: worker resends automatically; user sees "pending sync" then synced.
- Sales's edit later: field takes sales's value, finance's goes to history as superseded; badge (D16)
  "Replaced by <actor> in Salesforce at <time>. Your value '<v>' (saved <time>) is in history."
- Badge has a **"Use my value again"** button: a normal new PATCH with `baseUpdatedAt` = the current
  value's time, so it wins by time and goes out with a precondition; if the field changed again meanwhile,
  the user gets the usual 409 (D20). Overriding is always a deliberate human action.
- No notifications table: the badge shows when the record or list is opened.
- After 5 consecutive 412s on one outbox row: slower retry with jitter, counted in M3; never dropped.
- Rejected: badge only (retyping, typos); automatic retry until it sticks (endless fight = loop).

### G2b - Enforcing short locks (DECIDED: a)
- Send in three steps: claim (tx: lock, guard, read version, row -> `sending`, `claimed_at`) -> HTTP call
  with no transaction open -> result (tx: lock, store version / handle 412 / 4xx revert).
- Crash between steps: nothing locked; rows in `sending` > 60 s are resent with the same
  `Portal_Change_Id__c`. A 412 whose GET shows our own change ID on the record = our write landed ->
  mark sent, no resend.
  **SUPERSEDED by V1:** no GET; our landed write is recognised when its echo arrives with our change_id.
- `withRecordLock(recordId, fn)` is the only way to open a transaction; it sets an AsyncLocalStorage
  flag and the Salesforce client throws if called while it is set (unit-tested).
- Per-role Postgres settings: `idle_in_transaction_session_timeout = 5s`, `lock_timeout = 2s`,
  `statement_timeout = 5s`. On timeout: receiver 503 (Salesforce retries), portal retries once then
  "please try again", worker retries next pass.
- Scenario rules: one record per transaction for replay / drain / sweep (or small sorted batches);
  `saveEdit()` does DB writes only, side effects after commit; D14 contact linking in batches of 500;
  lock order customer then contacts by ID (deadlock victim retries); TCP keepalives; worker heartbeat +
  health-check kill so a hung leader frees the advisory lock (M1 also pages); migrations with
  `lock_timeout` in quiet hours; history reports on a replica later.
- Rejected: convention + review only (fails under load in production); timeouts only (kills but never
  prevents).

### G1 - Same field on both sides while our write is in flight (DECIDED: a)
- Conditional update: the worker sends the record version it last saw; if Salesforce's record has
  changed, Salesforce returns `412` and changes nothing. Worker GETs the record (headroom), re-runs D16
  per field, resends only the fields that still win, with the new version.
- **SUPERSEDED IN PART by V1 (Phase 5):** the version is `sf_seq_contiguous`, not the highest seen; on
  `412` there is no GET - wait 5 s for webhooks, `replay_record` if still behind, then the guard on exact
  per-field times. Reason: a GET has no per-field times, and max-seen lets a stale write pass a gap.
- The write's reply must return the record's new version (assumption), else every follow-up send 412s.
- The outbox guard stays as a cheap pre-filter; correctness rests on the precondition.
- Assumption: the brief does not give conditional updates; real Salesforce REST has
  `If-Unmodified-Since` / `If-Match` -> `412`. The fake Salesforce implements it.
- Cost: record-level, so a change to another field also 412s (one GET + one resend); rare at
  30 edits/s over 5M records.
- Rejected: repair after echo (Salesforce briefly wrong, breaks D17's simple rule); Salesforce arrival
  order wins (a slow queue picks the winner; old drain values beat fresh sales edits).

### G2 - Concurrent writers of per-field state (DECIDED: a)
- Every writer: lock the record row (`SELECT ... FOR UPDATE`) -> read -> pure apply function -> write
  state + history -> commit. Lock order: customer before contact.
- The worker never holds a lock while calling Salesforce (enforcement in G2b).
- Rejected: per-field compare-and-set (retry loops in three components); SERIALIZABLE (retries
  everywhere, false aborts).

### G3 - Fallbacks when Salesforce lacks sequence / retries / replay (DECIDED)
- The fallback table goes into the design doc: no `sequence` -> order by `occurredAt`, dedupe by
  `eventId` only, G1 version = `LastModifiedDate`, missed webhooks invisible; no retry -> crashes lose
  webhooks, S6b moves to 2 copies; no replay -> "modified since T" (`via = reconcile`), else a full scan.
- Reconciliation sweep (a): every 5 min, "records modified since last sweep minus overlap", the 9 fields
  returned directly, each record through the normal apply rule. ~5 calls per object per 5 min at peak
  (about 0.03 req/s). Watermark stored in Postgres. "Modified since" is itself an assumption.
- Rejected: no sweep (silent loss if any assumption is wrong); sweep only after outages (misses a crash
  when Salesforce does not retry).

### G4 - Postgres down longer than Salesforce retries (DECIDED: a)
- At risk: only Salesforce changes (the portal is down too, so no portal edits are made).
- The receiver writes a heartbeat row. After any database outage the worker replays the change log
  from the last heartbeat before the outage minus a 10-minute overlap; inbox + sequence drop what we
  already have. This replaces D3's unsafe "after the last eventId we applied" cursor everywhere.
- Beyond log retention (>= 24 h assumed): "modified since" gives current values (`via = reconcile`).
  Beyond that: page "unrecoverable window T1-T2, full scan needed". Never silent.
- Rejected: durable buffer in front of Postgres (second store, 200 before applied); accept the loss.

## Phase 4 - The data you store (outline item 4)

```
Already settled: per-field value/updatedAt/source/actor/sequence (D16, D19); history append-only,
monthly partitions, 13 months then archive (D15); via + superseded (S3b); inbox 7 days (D18);
outbox one pending row per record, ~500 B/row, ~216 MB after 6 h (D21, D23); sync schema (S4);
first/last name (S1); heartbeat, watermarks (G3, G4); record version for G1.

Round 1
  T1  Layout of per-field state (columns / field_state table / JSONB meta) .............. DECIDED (c)
  T2  Record identity and the contact->customer link .................................... DECIDED (a)
  T3  Outbox row shape and the "one pending row per record" constraint .................. DECIDED (a)
  T4  Cleaning up the inbox after 7 days (delete vs daily partitions) ................... DECIDED (b)

Frontier empty. Data section written to docs/data.md.
```

### T1 - Per-field state layout (DECIDED: c)
- Typed value columns on the record table, plus one `field_meta` JSONB column keyed by field:
  `{updatedAt, source, actor, sfSeq, sfValue}`. One row = one lock = one write; loads straight into the
  apply function's map. The apply function validates the shape (Postgres cannot).
- `sfValue` (added while writing the section): the last value Salesforce is known to hold for the field,
  set on webhook apply and on a confirmed send; it is what a D10 revert goes back to.
- Rejected: 5 columns per field (35 columns on contacts, mapping code, migration per field); a
  `field_state` table (~30M rows, values as text, pivot on every read, state in two places).

### T2 - Identity and contact->customer link (DECIDED: a)
- Primary key = the portal UUID, which is also `Portal_Id__c` (D8). `sf_id` (18-char, D9) nullable,
  unique when set; webhooks look up by it.
- Contacts carry `customer_id` (FK, NULL while pending) and `customer_sf_id` (always from Salesforce).
  "Customer pending" = `customer_id IS NULL`; linking runs in batches of 500 (G2b).
- Rejected: `customer_sf_id` only (no FK guarantee; portal-created customers have no `sf_id` yet).

### T3 - Outbox row (DECIDED: a)
- One row per record, pending fields merged in `fields jsonb`; columns for `change_id`, `base_version`,
  `status`, `attempts`, `consecutive_412`, `next_attempt_at`, `created_at`, `claimed_at`, `last_error`.
- Unique partial index on `record_id` WHERE `status = 'pending'`; saves use INSERT ... ON CONFLICT DO
  UPDATE to merge. `sending` rows don't count, so an edit during a send starts a new pending row.
- `sent` rows deleted after 7 days (history keeps every edit).
- Rejected: one row per field change (3-5x the rows, grouping at send time, breaks the D21 arithmetic).

### T4 - Inbox clean-up (DECIDED: b)
- One partition per day; drop the oldest nightly. Unique keys include the day:
  (`event_id`, `received_day`) and (`record_id`, `sequence`, `received_day`).
- A duplicate arriving on a later day passes the inbox, but the per-field `sequence` rule (D19) rejects its
  fields; worst case an extra history row marked superseded, never a wrong value.
- Size: ~18M rows, ~2.7 GB at peak.
- Rejected: daily DELETE of ~2.6M rows (dead space, vacuum load); keep everything (~140 GB/year).

## Phase 5 - Review of docs/data.md against the brief (scenario walk-through)

```
Needs a decision (asked one at a time)
  V1  Record version for the G1 precondition uses max-seen sequence, and the 412 path
      needs per-field times a GET does not give ........................................ DECIDED (a)
      Found: with seq 2 (sales city, later) missing and seq 3 (phone) received,
      sf_version = 3, our older Noida passes the precondition and beats Delhi; the
      late seq 2 is then rejected by D19. Wrong winner, silently.
  V2  Portal-created contact under a portal-created customer: AccountId unknown
      until the customer's create is confirmed .......................................... DECIDED (b)
  V3  No table of Salesforce users for choosing an owner (assumption 3.4 says
      "users synced from Salesforce") ................................................... DECIDED (a)

V1 decision (a): store sf_seq_contiguous (highest sequence with no gap below it) next to
  sf_seq_max; the G1 precondition uses sf_seq_contiguous. On 412: no GET; wait 5 s for in-flight
  webhooks, then a replay_record job if still behind, then re-run the guard with exact per-field
  occurredAt and resend. Own write detected by its echo (our change_id). Replaces G1's
  "GET and re-run D16" step. Rejected: keep max-seen + GET (gap bug stays, LastModifiedDate
  picks wrong winners); GET + replay on every 412 (2 calls, gap bug stays).

V2 decision (b): at claim, a contact whose customer has no sf_id is pushed back 2 s. When the
  customer's create reply arrives, its waiting contacts get customer_sf_id (batches of 500). If the
  customer is rejected, the contact waits and the portal shows "waiting for customer: rejected by
  Salesforce". Rejected: reference by Account.Portal_Id__c (new assumption, still needs a retry on
  "parent not found"); send unlinked then link (2 calls, orphan contact visible to sales).

V3 decision (a): sync.sf_users (sf_user_id, email, name, active, synced_at), refreshed hourly by one
  "list active users" query (assumption 3.14), unknown owner IDs fetched by a fetch_user job; the portal
  validates owner against active users before saving. Described, not built this weekend; demo seeds a
  few users. Rejected: no table (bad owner becomes a Salesforce rejection, raw IDs shown, assumption 3.4
  false); owner read-only in the portal (brief lets both sides update owner).

Frontier empty. All other sections aligned with data.md in the same pass.

Small fixes, no decision needed (APPLIED to data.md with V1)
  V4  actor: the brief gives emails (changedBy / editedBy), not IDs; store the email
  V5  sf_id char(18) pads the brief's sample IDs; use text, unique when set
  V6  outbox change_id set when the pending row is created (merges keep it, frozen on
      claim), so history.change_id is known at save; unique index for echo lookup
  V7  drop outbox.op; choose at claim: sf_id known -> conditional update, else upsert
  V8  unknown recordId with sequence 1 is a create carrying all fields: no fetch;
      unknown with sequence > 1 is S3 (partial + fetch)

Other sections to align after V1-V8: systems.md 2.5 (sample keys/changedBy), 2.6 (412),
CLAUDE.md rule 7, subproblems 3, assumptions 3.3/3.8, decisions G1 and S3.
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
