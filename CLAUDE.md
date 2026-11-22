# Tequity Round 1 - Salesforce <-> portal sync

Design docs live in `docs/`: decisions in `decisions.tmp.md`, assumptions in `assumptions.md`,
build order in `subproblems.md`. Decision labels (`Dn`, `Sn`) point into `decisions.tmp.md`.

## Shared database and `saveEdit()` module (decision S4) - keep these rules when changing code

The portal backend, the webhook receiver and the sync worker share one Postgres database. A portal
edit writes the record, its history and the outbox row in **one transaction** through a shared
`saveEdit()` module. That transaction is what stops a crash from losing an edit or sending an
unsaved one (D23). The costs we accepted, and the rules that contain them:

1. **Every portal write goes through `saveEdit()`.** A direct `UPDATE` on a records table skips the
   outbox, and that edit silently never reaches Salesforce. A test must fail if a record row changes
   without a matching outbox row. Never bypass it, even for scripts or admin fixes.
2. **The webhook path never writes to the outbox** (loop rule, D17). Do not add a database trigger
   that writes the outbox on every update: it would also fire on webhook writes.
3. **Sync tables live in their own `sync` schema** (`inbox`, `outbox`, `jobs`). The sync code owns
   their migrations; the portal only calls the module.
4. **Same language:** the module is TypeScript on Node (20+), the portal backend's language.
5. **Each service has its own capped connection pool** (e.g. receiver 10, worker 5), so a webhook
   burst or an outage drain cannot starve portal users.
6. **A failed outbox insert fails the user's save.** This is deliberate: a save that cannot be
   synced must not look saved.
7. **One Postgres role per service, least privilege.** For example, the receiver cannot touch
   `outbox`; the worker never receives HTTP; the receiver never calls Salesforce (D22).

## Locking and the Salesforce precondition (decisions G1, G2, G2b) - keep these rules

1. **Every write to per-field state goes through `withRecordLock(recordId, fn)`**: lock the record row
   (`SELECT ... FOR UPDATE`), read, run the pure apply function, write, commit. No other way to open a
   transaction on records.
2. **Never call Salesforce inside a transaction.** The Salesforce client throws when called inside
   `withRecordLock`; keep that check and its test. The worker sends in three steps: claim (tx) -> HTTP
   call (no tx) -> store result (tx).
3. **Lock order:** customer before contact, then by ID. Deadlock victims retry.
4. **One record per transaction** for replay, drain and sweep. Bulk work (e.g. linking contacts to a
   newly arrived customer) runs in batches of 500, each its own transaction.
5. **Keep transactions to database writes only.** Emails, HTTP calls and other side effects happen
   after the commit.
6. **Postgres timeouts per role:** `idle_in_transaction_session_timeout = 5s`, `lock_timeout = 2s`,
   `statement_timeout = 5s`. Migrations use their own role with a `lock_timeout`.
7. **Every update to Salesforce is conditional on `sf_seq_contiguous`** (the highest sequence with no
   gap below it), never on `sf_seq_max`: a missing webhook below our version must cause a `412`, not a
   stale write. On `412`: no GET. Wait 5 s for in-flight webhooks; if `sf_seq_contiguous` has not
   passed `base_version`, run a `replay_record` job; then re-run the guard on exact per-field times and
   resend only the winners. Our own landed write is recognised by its echo's `change_id`. Never retry a
   lost field automatically; the user overrides with "Use my value again".
8. **Replay restarts from a heartbeat minus an overlap**, never from "the last `eventId` we applied"
   (webhooks arrive out of order).
9. **A contact is never sent before its customer has an `sf_id`**; the claim pushes it back 2 s.

## Other fixed rules

- Only one sync worker is active at a time (Postgres advisory lock, standby takes over). The rate
  limiter is in memory; do not scale the worker horizontally without a shared limiter.
- Contact full name is stored as `first_name` / `last_name` (Salesforce `Contact.Name` is read-only).
- **`history` is append-only:** insert, never update or delete (old partitions are archived, not
  deleted). `superseded` is set at insert time by the apply rule.
- **`field_meta` (JSONB) has no database-enforced shape:** only the apply function writes it, and it
  validates `{updatedAt, source, actor, sfSeq, sfValue}` for every field. Table layouts are in
  `docs/data.md`.
- **`sfSeq` and `sfValue` only move forward:** webhooks, echoes, replies, fetches and sweeps update them
  only when their sequence is newer. An echo kept as sent never changes the field's `actor` or
  `updatedAt` and writes no history row (I1, `docs/inbound.md` 5.2).
- **At most one pending outbox row per record**, enforced by a unique partial index; merge with
  `INSERT ... ON CONFLICT ... DO UPDATE`, never a second pending row.
