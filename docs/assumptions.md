# Assumptions and open questions

Two-way sync of customer and contact records between Salesforce and the Tequity portal (Postgres).

The brief asks for this section at the top of the design document. Each entry has three parts:

- **Question**: what we would have asked the client before locking the design.
- **Assumption**: what we decided in its place.
- **If the answer goes the other way**: one sentence on what would change.

`Dn` labels point to the full reasoning in [decisions.tmp.md](decisions.tmp.md).

---

## 1. The five questions the brief names

### 1.1 Peak and quiet edit rate (D1)
- **Question:** How many edits per second happen at peak, and how many in a quiet hour?
- **Assumption:**
  - Peak: **20 portal edits/s** and **30 Salesforce edits/s**, so 50/s in total.
  - Quiet hour: **1/s** on each side.
  - The outage arithmetic assumes the portal peak lasts the **full 6 hours**, as a worst case.
- **If the answer goes the other way:** at 50 portal edits/s the worst-case drain grows from 122 min to about 600 min, and batching becomes required. At 5/s it falls to about 24 min.

### 1.2 Changed fields or full record in a webhook (D2)
- **Question:** Does a webhook carry only the changed fields, or the whole record?
- **Assumption:** **Changed fields only**, matching the sample. Each field in `changedFields` really was changed by that event, so ordering per field is safe.
- **If the answer goes the other way:** a full record carries stale copies of fields Salesforce hasn't yet received from us. We would then compare it with the last Salesforce values we stored, and treat only the differences as changes. Otherwise an older city would overwrite the newer one finance set.

### 1.3 Changes made during the 6-hour outage (D3)
- **Question:** Can we ask Salesforce for the records that changed during the 6 hours, or are those changes gone unless we already have them?
- **Assumption:**
  - Salesforce keeps a **replayable change log**: "give me every event after `eventId` X".
  - The log is kept for **at least 24 h**.
  - Each request returns a page of **2,000 events**.
  - **Fallback (worst case):** if the log has expired, query records modified since the outage began.
- **If the answer goes the other way:**
  - **With the fallback query only:** we get current values only, so edits in between and their actors are lost. History records them as "reconciled after outage, intermediate edits unknown".
  - **With neither:** we need a full scan of 5M records, at least 2,500 calls.

### 1.4 Two portal users editing the same field (D20)
- **Question:** When two portal users edit the same field, what does the other user see, and what do we send Salesforce?
- **Assumption:**
  - **The check:** the portal save request carries, for each field it changed, the `updatedAt` the form loaded. If the field has changed since, that field gets a `409` with the current value, who changed it and when. The user chooses to keep it or overwrite. Other fields in the same save are accepted.
  - **What the first user sees:** the new value with a "changed by … at …" badge, taken from history. No notification is sent.
  - **What Salesforce receives:** each accepted value, merged if it's still queued.
- **If the answer goes the other way:** if the client wants the last save to win silently, we drop the `409` check. The overwritten value still survives in history.

### 1.5 Ordering a portal edit, given the sample has no time field (D6)
- **Question:** What do we trust for the order of a portal edit?
- **Assumption:**
  - **Time:** the Postgres commit time of the edit. It's one clock, written in the same transaction as the change.
  - **Actor:** the logged-in user. If the body's `editedBy` doesn't match, the save is rejected.
- **If the answer goes the other way:** with a client-supplied time, order could no longer be trusted across browsers with wrong clocks. With the actor taken from the body, edits could be forged.

---

## 2. Salesforce webhooks

### 2.1 Order within a record (D4, D19)
- **Question:** Is there a version or sequence number per record? How precise is `occurredAt`?
- **Assumption:**
  - **Per-record `sequence`:** each webhook carries one (a field we add, for exact order and gap detection). Salesforce applies writes to one record one at a time, because it locks the row.
  - **`occurredAt`** has millisecond precision. It's used for history, and for conflicts between the two sides.
  - **`eventId`** increases over time, and serves only as the cursor for replay.
- **If the answer goes the other way:** without a sequence, order within a record falls back to `occurredAt` plus `eventId` to break ties. We could no longer detect a missed webhook from a gap.

### 2.2 Clock skew (D4, D16)
- **Question:** How far apart can Salesforce's clock and ours be?
- **Assumption:** **≤ 1 s** (both sides use NTP). On an exact tie, Salesforce wins, so the result is always the same.
- **If the answer goes the other way:** with larger skew, cross-side conflicts less than that far apart could be decided wrongly. We would then need a version number from Salesforce, or ownership per field.

### 2.3 Duplicate delivery (D5, D18)
- **Question:** When Salesforce delivers the same webhook twice, is the `eventId` the same?
- **Assumption:**
  - Yes. Duplicates come from a lost 2xx reply, our process crashing after the commit, or our reply arriving after Salesforce's timeout.
  - **(recordId, sequence)** is the second key, in case Salesforce re-issues an event under a new ID.
  - **`eventId`s are kept 7 days.** After that, the per-record sequence check still rejects an old duplicate.
- **If the answer goes the other way:** with no stable `eventId`, we'd rely only on (recordId, sequence) to remove duplicates.

### 2.4 Webhook retries (D12)
- **Question:** Does Salesforce retry webhooks, and for how long?
- **Assumption:**
  - **Retried:** whenever Salesforce gets no 2xx (our endpoint down, a 5xx, a timeout, or our deliberate 429/503), with backoff, for an unknown window.
  - **Not retried:** a 401, or events never sent because Salesforce itself was down.
  - **Never relied on:** we return 2xx only after the event is safely in Postgres.
- **If the answer goes the other way:** with no retries at all, replay and gap detection (2.1, 1.3) become the only recovery path. The design already allows for that.

### 2.5 Authenticity (D7)
- **Question:** How do we know a webhook really came from Salesforce?
- **Assumption:** an **HMAC signature** over the body plus a timestamp header, using a shared secret, with a 5-minute tolerance. A failed check gets a `401`, and nothing is stored.
- **If the answer goes the other way:** if Salesforce can't sign, we fall back to mutual TLS, or an IP allowlist plus a bearer token.

### 2.6 Webhooks during an API outage (D11)
- **Question:** While the Salesforce API is down, can sales users still edit, and do webhooks still arrive?
- **Assumption:** it varies. Some webhooks may arrive and some won't (*"may never arrive"*). Our webhook endpoint never depends on the Salesforce API being up. Anything that arrives is applied, gaps are found by `sequence`, and replay fills them after recovery.
- **If the answer goes the other way:** if Salesforce is always fully down, UI included, the replay is simply smaller. No rule changes.

---

## 3. Salesforce record API

### 3.1 Rate limit (D11)
- **Question:** How does the 100 req/s cap behave, and does a batch call count as one request?
- **Assumption:**
  - The cap counts **every call we make, retries included**.
  - A `429` with `Retry-After` means pause everything for that long and keep the work.
  - We run at **80 req/s**, keeping **20 req/s** free for retries, replay, customer fetches and gap replays.
  - **One record per request** is the baseline for all the arithmetic. Batching is not assumed.
- **If the answer goes the other way:** if batch calls (up to 200 records) count as one request, the drain shrinks by up to about 100×. It's listed as a possible speed-up only.

### 3.2 Safe retries for creates and updates (D8)
- **Question:** If the network drops halfway through a create, how do we avoid creating it twice?
- **Assumption:** the client adds two custom fields:
  - **`Portal_Id__c`** (an External ID). The portal generates it before the first call, and every create is an **upsert** on it.
  - **`Portal_Change_Id__c`**. Every write carries that outbox row's **idempotency key**, and it comes back in the webhook's `changedFields`.
- **If the answer goes the other way:**
  - **Without External ID upsert:** a retry after a lost reply can create a duplicate record. We'd have to search Salesforce before each retry.
  - **Without `Portal_Change_Id__c`:** we'd recognise our own echo by `changedBy` plus matching values, which is less exact.

### 3.3 What a write returns (D9)
- **Question:** What does Salesforce send back on create or update? Is it the ID later webhooks carry?
- **Assumption:**
  - **The reply:** the record ID, success or an error code, and the new `LastModifiedDate`.
  - **The ID:** the one returned on create is the permanent ID, and every later webhook carries it. We store only the **18-character** form.
  - **A lost create reply:** we match the create's webhook by `Portal_Id__c`.
- **If the answer goes the other way:** if Salesforce returns only success with no ID, we always match by `Portal_Id__c`.

### 3.4 Writable fields and owner (D10)
- **Question:** Which fields may we change? Are there validation rules, read-only fields, or fields owned by one team?
- **Assumption:**
  - The integration user can write all nine fields: customer name, status and owner; contact full name, phone, email, city, owner and customer.
  - **`owner`** is a Salesforce user ID. Portal users can only choose users synced from Salesforce.
- **If the answer goes the other way:** read-only fields become read-only in the portal too, and are never sent.

### 3.5 Rejected updates (D10)
- **Question:** If Salesforce rejects one of our updates, should we retry, flag it, tell the user, or put the old value back?
- **Assumption:**
  - **A `4xx` rejection:** never retried. The portal field goes back to the last value Salesforce confirmed, but **only if it still holds the rejected value**, so another user's later edit isn't wiped. The portal shows "rejected by Salesforce: <reason>", and history keeps the attempt.
  - **`429`, `5xx` and timeouts:** retried.
- **If the answer goes the other way:** if the client prefers to keep the portal value and flag it, we skip the revert and show the record as "out of sync" until someone fixes it.

### 3.6 Initial load (D13)
- **Question:** How do the first 5 million records get into the portal?
- **Assumption:** Salesforce provides a **bulk file export/upload API**. The initial load runs through it before go-live, outside the 100 req/s budget. It's out of scope this weekend.
- **If the answer goes the other way:** through the record API, 5M ÷ 80 req/s ≈ 17.4 h.

### 3.7 Contact arriving before its customer (D14)
- **Question:** Can a contact reach us before its customer? How should we handle it?
- **Assumption:**
  - **Yes, it can.** We store the contact at once, with the raw `customer_sf_id`, shown as "customer pending".
  - **If the customer hasn't arrived,** we fetch it from Salesforce at **5 s**, then again at **15 s** and **25 s**, all inside the 30 s target. The calls come from the 20 req/s headroom.
  - **If all three fail** (Salesforce down), the contact stays pending until catch-up.
- **If the answer goes the other way:** if Salesforce guarantees customer-before-contact, the fetch never fires, and the rule is harmless.

---

## 4. Portal, conflicts and history

### 4.1 Same field changed on both sides (D16)
- **Question:** When finance and sales change the same field, which value is current?
- **Assumption:**
  - **Per field, the latest time wins:** Salesforce `occurredAt` against our Postgres commit time. The losing value stays in history.
  - **Outbox guard:** before each send, the worker drops any field that a newer Salesforce value has already won.
  - **What the losing user sees:** a badge from history ("replaced by <actor> in Salesforce at <time>"). No notifications table.
- **If the answer goes the other way:** if the client wants **ownership per field** (e.g. city belongs to the portal, phone to Salesforce), conflicts disappear, and the owner's side always wins for that field.

### 4.2 Portal login (D6, D26)
- **Assumption:** the portal already authenticates its users. We read the actor from the session.
- **If the answer goes the other way:** login would have to be built first, since the actor in history depends on it.

### 4.3 History retention and access (D15)
- **Question:** How long must change history be kept, who reads it, and is it searched across all records?
- **Assumption:**
  - **Storage:** an append-only table, one row per field change.
  - **Retention:** 13 months in Postgres (one partition per month). Older months are **archived** to object storage, never deleted.
  - **Access:** read one record at a time.
  - **Size:** about 200 B per row. That's about 0.9 GB/day at peak, and about 60 GB/year at a realistic average of 10/s.
- **If the answer goes the other way:** a search across all records needs a warehouse. A legal hold or deletion rule changes the archive policy.

### 4.4 How often outage edits hit the same record (D21)
- **Question:** During 6 hours, how many portal edits land on a record already waiting to be sent?
- **Assumption:** unknown. We present the **worst case with no merging (122 min)** as the headline. A **50% guess (about 62 min)** shows what merging is worth.
- **If the answer goes the other way:** with real logs we'd replace the guess with the actual ratio. The worst case stays the same.

---

## 5. Numbers used in the arithmetic

| Quantity | Value | Source |
|---|---|---|
| Portal edits at peak | 20/s | assumption 1.1 |
| Salesforce edits at peak | 30/s | assumption 1.1 |
| Quiet-hour edits | 1/s each side | assumption 1.1 |
| Salesforce API cap | 100 req/s | brief |
| Our normal send rate | 80 req/s (20 req/s headroom) | assumption 3.1 |
| Rate during the outage | ~0 (one test call every 30 s) | D11 |
| Ramp after recovery | 20 req/s (minute 1) → 40 (minute 2) → 80 | D11 |
| Outage length | 6 h = 21,600 s | brief |
| Portal backlog after the outage | 20 × 21,600 = **432,000 calls** | derived |
| Salesforce events to replay | 30 × 21,600 = 648,000 → ÷ 2,000 per page = **324 calls** | assumption 1.3 |
| Drain time, worst case | 60T = 438,324 → T ≈ 7,305 s ≈ **122 min** | D21 |
| Drain time, 50% merged (guess) | 60T = 222,000 → T ≈ 3,700 s ≈ **62 min** | D21 |
| Outbox on disk after the outage | ~432,000 × 500 B ≈ **216 MB** (less with merging) | D21 |
| Sync target while both sides are up | 30 s worst case | brief |

How the drain time is computed:

```
sent(T)   = 1,200 + 2,400 + 80 x (T - 120) = 80T - 6,000   (ramp, then 80 req/s)
needed(T) = 432,000 + 20T                                    (backlog + new edits)
80T - 6,000 = 432,000 + 20T  ->  60T = 438,000  ->  T = 7,300 s ~ 122 min
```

Replaying first takes 324 of minute 1's 1,200 calls, so the sum becomes 60T = 438,324, T ≈ 7,305 s, still about 122 min.

---

## 6. Scope assumptions (D26)

**Excluded by the brief:**
- deletes
- Salesforce login screens
- a visual UI
- a second region
- a load test against the real Salesforce

**Excluded by our own decisions:**
- **The initial bulk load:** described only.
- **Batching:** a possible speed-up, not used in the arithmetic.
- **A notification feed:** can be added later as a query over history.
- **The history archive job:** designed, not built.
- **Several active workers:** one active worker with a standby is enough at 80 req/s.
- **Field ownership:** kept as a question for the client.
- **Rotating the HMAC secret, and wiring up the alerts:** described, not built.
- **Portal login:** assumed to exist.

---

## 7. Questions still open for the client

1. Real peak and quiet edit rates (1.1).
2. Does Salesforce keep a replayable change log, and for how long (1.3)?
3. Can webhooks carry a per-record `sequence` (2.1)?
4. Can we add `Portal_Id__c` (External ID) and `Portal_Change_Id__c` to Contact and Account (3.2)?
5. Validation rules, read-only fields, and fields owned by one team (3.4)?
6. On rejection, revert or keep and flag (3.5)?
7. Should any field be owned by one side (4.1)?
8. How long to keep history, may the archive ever be deleted, and is it searched across all records (4.3)?
9. How often do outage-period edits hit the same record (4.4)?
10. Can Salesforce sign webhooks with HMAC (2.5)?
