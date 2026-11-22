# 11. Three numbers we watch, and when we call someone

Labels like `D24` and `R1` point to [decisions.tmp.md](decisions.tmp.md). Tables are in [data.md](data.md).
Metrics and alerts are **described, not built** this weekend (D26).

The three numbers cover the three ways the sync fails a user: a portal change doesn't reach Salesforce, a
Salesforce change doesn't reach the portal, or Salesforce starts refusing us. The red line follows the brief's
30-second worst case (D24).

## 11.1 Summary

| # | Number | Page when |
|---|---|---|
| **1** | **Oldest unconfirmed portal edit** | > **30 s**, while the breaker is closed. Plus: the breaker open > **5 min**; during recovery, the fresh-lane re-arm rule and the drain-stall page |
| **2** | **Salesforce-to-portal delay** | p95 > **30 s** over 5 min; a sequence gap unfilled after **10 min**; or **two successful sweeps in a row** each find a change the webhooks missed |
| **3** | **Salesforce API failure rate** | > **5%** of attempts fail over 5 min, or > **1%** are validation or permission rejections |

## 11.2 Number 1: oldest unconfirmed portal edit

**Definition:** `now − age_from` of the oldest outbox row whose status is `pending` or `sending`.

| Detail | Rule |
|---|---|
| **What counts** | `pending` rows and `sending` rows, including a row whose send is **uncertain** and still waiting for evidence that it landed (outbound.md 6.6). "Unconfirmed" catches a send stuck in that state; "unsent" would miss it |
| **`age_from`** | The row's `created_at`, except as below (R1) |
| **Excluded: a contact blocked by a rejected customer create** (R1) | While a contact's row is parked because its customer's create was **rejected** by Salesforce, it's left out (`blocked_by = 'rejected_customer'`). The rejection stays visible in the portal ("waiting for customer: rejected by Salesforce") and is already counted by number 3 |
| **When the block ends** | When the customer gets its `sf_id`, the row becomes eligible to send again: `link_contacts` clears `blocked_by` and sets **`age_from = now()`**, so its 30-second timer starts then, not from the original save |
| **A contact waiting for a customer that's merely in flight** | Not excluded. It should be sent within 30 s like any other edit |
| **Page** | Oldest > 30 s while the breaker is closed |

**Example (R1).**

```
09:00  Finance creates customer Acme (no name) and contact Ravi -> Ravi's row parked behind Acme
09:00  Acme's create is rejected: Name required
       -> Ravi's row: blocked_by = 'rejected_customer'; excluded from number 1
       -> portal shows "waiting for customer: rejected by Salesforce"; number 3 counts the 4xx
14:00  Finance adds the name; Acme's new upsert is sent (Acme's own row is counted normally)
14:00:01  Acme gets its sf_id -> link_contacts wakes Ravi's row: blocked_by = NULL, age_from = 14:00:01
14:00:02  Ravi is sent -> age 1 s, well under 30 s. Five hours parked never paged
```

**During an outage and recovery** (outage.md 10.6):
- **Breaker open:** the 30-second rule doesn't apply (the brief's target is "while both sides are up"). The breaker
  open for more than **5 minutes** pages instead.
- **Catch-up:** only **fresh-lane** rows (created after recovery) count. The 30-second rule re-arms when the
  fresh lane's oldest row first drops below 30 s, or 15 minutes after recovery, whichever is first (about 11.5
  minutes, predicted).
- **Drain stall:** every minute, page if the backlog shrank by less than 75% of (allowed sends − fresh-lane
  sends) over the last 5 minutes. A backlog that stops shrinking 10 minutes into the drain pages at minute 12.
- **Sustained throttling** (`429`s): in normal operation, rows age past 30 s and number 1 pages; during
  recovery, the drain-stall rule pages.

## 11.3 Number 2: Salesforce-to-portal delay

**Definition:** for each webhook applied, **our commit time − `occurredAt`**.

| Condition | Page when | Why |
|---|---|---|
| **Delay** | p95 > 30 s over 5 minutes | The brief's 30-second target for Salesforce → portal |
| **Gap** | A sequence gap still unfilled after 10 minutes | A specific webhook is lost and the per-record replay (D19) couldn't fill it |
| **Sweep fallback** (R2) | **Two consecutive successful sweeps each find at least one Salesforce change the webhook path missed** | Webhooks stopping altogether gives number 2 no samples, so it can't page on delay. The sweep compares Salesforce with us every 5 minutes |

- **What the sweep counts as missed:** a record whose Salesforce values are newer than ours and different, last
  changed **more than 60 s before the sweep read it**. A change younger than that is left for the next run, so a
  webhook still in flight isn't counted as a miss.
- **One miss doesn't page.** The sweep repairs it and records it in history (`via reconcile`). Two runs in a row
  with misses means webhooks are being lost repeatedly, and that pages.
- **A failed sweep run** neither counts nor breaks the streak; only successful runs are compared.
- **It's a fallback**, with a detection window of about **10 minutes** (two 5-minute runs), not 30 s. It catches
  a broken webhook subscription and steady partial loss that the delay and gap rules can't see.
- **Replayed events don't count in the delay.** After an outage, replayed events have hours of delay by design;
  only webhooks are measured.

## 11.4 Number 3: Salesforce API failure rate

**Definition:** over 5 minutes, failed attempts ÷ all attempts to the Salesforce API, **retries included** in the
total.

| Reply | In the failure rate? | Notes |
|---|---|---|
| Timeout | **Yes** | |
| `5xx` | **Yes** | |
| `4xx` other than `412` and `429` | **Yes** | Includes validation (`400`) and permission (`401` after one token refresh, `403`) |
| `412` | No, tracked separately | Normal when both sides edit the same record (G1). Back-off after 5 in a row is recorded (G1b) |
| `429` | No, tracked separately | Throttling, not failure. Sustained throttling is caught by number 1 or the drain-stall rule |
| `2xx` | No (it's in the total) | |

| Page when | Catches |
|---|---|
| **> 5%** of attempts fail | Salesforce degrading, network trouble, our credentials breaking |
| **> 1%** of attempts are **validation or permission rejections** | A new validation rule, a field-level permission change, or a rejected create (the R1 case). Users' edits are being refused |

- **Not evaluated while the breaker is open.** Every test call fails then; the breaker's own 5-minute page covers
  the outage, so one outage gives one page.
- **Outage probes are excluded** from the rate after the breaker closes, so the test calls made during the outage
  don't keep the rate high once Salesforce is back.

## 11.5 Pages that sit beside the three numbers

These aren't separate numbers; they're conditions on the same data, already decided elsewhere.

| Page | From |
|---|---|
| Breaker open > 5 min | Number 1 during an outage (D24) |
| Drain stall | Number 1 during recovery (Q10b) |
| Sweep watermark missing | outage.md 10.3 (Q10a) |
| `Portal_Id__c` row holding a different `sf_id` | inbound.md 5.3 (I2); counted in number 3 |
| Unrecoverable replay window, full scan needed | outage.md 10.7 (G4) |

## 11.6 What the three numbers don't catch

| Blind spot | Where it shows instead |
|---|---|
| A forgotten customer whose create was rejected | The portal badge, and number 3's rejection rate when it happened |
| A storm of `412`s on a hot record | The separate `412` count and the G1b back-off |
| Clock skew between Salesforce and us beyond the assumed 1 s | Number 2 drifting high or negative; no page of its own |
| A single lost webhook on a record never edited again | Repaired by the next sweep; pages only if misses repeat (R2) |
