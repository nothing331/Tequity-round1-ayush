# 10. The 6-hour outage: what is on disk, how we drain it, and the minutes of arithmetic

Labels like `D21` point to [decisions.tmp.md](decisions.tmp.md). Tables are in [data.md](data.md); the send
path is in [outbound.md](outbound.md).

The brief: the Salesforce API is unreachable for **6 hours**. Webhooks Salesforce tried to send in that window
may never arrive. Pending calls must survive our own restarts. The 30 s target applies only while both sides are
up; after the outage we **drain as fast as the rate limit allows**.

## 10.1 During the outage

| Part | Behaviour |
|---|---|
| **Detection** | After **5 failed calls in a row**, the circuit breaker opens. From then on, one test call every 30 s. Only a successful test closes it (D11) |
| **Portal** | Stays fully writable. A banner says "Salesforce sync paused: your changes are saved", and each changed record shows "pending sync" (D11) |
| **Outbox** | Every save still writes its row in the same transaction (outbound.md 6.1). Rows merge per record while `pending` |
| **Rows that were mid-send when it started** | Stay `sending` with the same key and content. Whether they landed is decided by evidence after recovery (outbound.md 6.6) |
| **Webhooks** | The receiver doesn't depend on the Salesforce API. Any webhook that does arrive is applied normally; the missing ones come back through the replay |
| **Sweep** | Fails like every other call, so its watermarks stop advancing. That frozen point is where the replay starts (10.3) |
| **Our restarts** | Nothing is lost: every pending call is a row in Postgres, and a `sending` row is picked up again when its lease has expired (outbound.md 6.3) |
| **Alert** | The breaker open for more than 5 minutes pages (D24) |

## 10.2 What is on disk after 6 hours

Worst case: the portal peak of **20 edits/s** lasts the full 6 h, and no two edits touch the same record, so
nothing merges.

| Where | Rows | Size |
|---|---|---|
| `sync.outbox` (pending, plus at most one `sending` row per record) | 20 × 21,600 = **432,000** | × ~500 B ≈ **216 MB** |
| `history`, one row per portal edit (already counted in the history budget, D15) | 432,000 | × ~200 B ≈ 86 MB |
| `sync.inbox`, only for webhooks that did arrive | up to 30 × 21,600 = 648,000 | × ~150 B ≈ 97 MB |
| `sync.watermarks` | the two sweep watermarks: the replay start | tiny |

Everything is in Postgres, written in the same transaction as the edit, so it survives any restart of our
processes. With merging, the outbox holds one row per edited record instead of one per edit.

## 10.3 Where the replay starts (Q10a)

**Replay start = the oldest completed sweep watermark, across customers and contacts, minus 10 minutes.** This
holds for any outage, Salesforce's or our database's: the sweep can't complete in either case, so its watermark
always sits before the outage began. (A heartbeat keeps beating during a Salesforce outage, so it can't mark
the start of one; G4's heartbeat rule is superseded.)

**When a watermark advances:**

| Event | `sweep:customers` / `sweep:contacts` |
|---|---|
| A sweep run starts at time S | Unchanged. The run reads "modified since watermark − 2 min", 2,000 records per page |
| Each page is applied | Unchanged |
| **The run's last page is applied and committed** | That object's watermark → **S** (the run's start, so changes made during the run are read again next time) |
| A page fails, or the worker stops mid-run | **Unchanged.** The next run starts again from the same point; re-applying is harmless (the apply rule only moves values forward) |
| **A change-log replay completes** | **Both** watermarks → the replay's start time, because the replay covered everything up to then |
| First deploy | Both set to the time the initial bulk load finished (D13) |

**Walk-through: a sweep that fails halfway.**

```
10:00:00  Sweep run: customers completes, contacts completes   -> customers 10:00:00, contacts 10:00:00
10:05:00  Sweep run starts (S = 10:05:00)
          customers: last page applied                          -> customers 10:05:00
          contacts: page 1 applied, page 2 call fails at 10:05:20 (the outage has begun)
                                                                -> contacts stays 10:00:00
10:10 ... 16:05  every sweep run fails                          -> both frozen
16:05:20  Breaker closes. Replay start = min(10:05:00, 10:00:00) - 10 min = 09:50:00
          Window 09:50:00 -> 16:05:20 = 22,520 s; 30 events/s x 22,520 = 675,600 events = 338 calls
16:09:50  Replay (started 16:05:20) completes                   -> customers 16:05:20, contacts 16:05:20
16:10:00  Normal sweeps resume from there
```

**Walk-through: a restart with one watermark missing.**

```
Before:   customers 10:05:00, contacts row missing (e.g. lost in a bad migration)
Restart:  the worker can't take a min() over a missing value, and must not treat it as "now"
          -> replay start = the change log's retention limit, now - 24 h
          -> page "sweep watermark missing: contacts"
          -> the next contacts sweep reads from the retention limit, then writes its watermark
Cost:     24 h x 30 events/s = 2,592,000 events = 1,296 calls; applying them ~17 min at 2,500 events/s.
          Rare, and safe: replayed events we already have are dropped by the normal rules.
```

## 10.4 Recovery order

1. **The breaker closes** on a successful test call.
2. **Replay Salesforce's change log** from the start point in 10.3. Each event goes through the inbox and the
   normal apply rule (`via replay`), so events we already have are dropped (item 8).
3. **Then drain the outbox** with a ramp of **20 → 40 → 80 req/s** (minute 1, minute 2, then on), in two lanes:
   **fresh** rows (created after recovery) and **backlog** rows, 50/50, each lane borrowing the other's unused
   share (D21).
4. Normal sweeps resume from the watermarks the replay set.

**Why replay first:** a queued 10:00 finance value must not overwrite an 11:00 sales edit from the outage. With
the replay applied first, the sales value's `editId` has replaced finance's, so the guard drops the stale field
before any call. The conditional update would also catch it (`412`), so replay-first isn't what makes this
correct any more; it avoids hundreds of thousands of `412`s at the start of the drain (V1).

## 10.5 The arithmetic

**Inputs**

| Quantity | Value | Source |
|---|---|---|
| Outage | 6 h = 21,600 s | brief |
| Portal edits at peak | 20/s, for the whole outage and the recovery | assumption 1.1 |
| Salesforce edits at peak | 30/s | assumption 1.1 |
| Send rate | 80 req/s; 20 req/s of the 100 cap stays free for retries, `412` evidence, fetches and gap replays | assumption 3.1 |
| Ramp | 20 req/s in minute 1, 40 in minute 2, then 80 | D11 |
| Replay page | 2,000 events per call | assumption 1.3 |
| Sweep interval | 5 minutes, so the watermark is at most 5 minutes before the outage | G3 |
| Applying events | ~2,500 events/s (4 appliers split by record, ~1.6 ms per event each). To be measured | assumption 4.5 |

**Backlog:** 20 edits/s × 21,600 s = **432,000 calls**.

**Replay:** the six-hour window alone would be 30 × 21,600 = 648,000 events = 324 calls. The replay also covers
up to 15 minutes before it (up to 5 minutes of sweep staleness plus the 10-minute overlap): 30 events/s ×
(21,600 + 900) s = 675,000 events, and 675,000 ÷ 2,000 = **338 calls**.

**Drain, as fast as the rate limit allows.** Find T, in seconds after recovery, when everything sent equals
everything needed:

```
sent(T)   = 20 x 60  +  40 x 60  +  80 x (T - 120)  =  80T - 6,000      (ramp, then 80 req/s)
needed(T) = 432,000  +  338  +  20T                                      (backlog + replay calls + new portal edits)

80T - 6,000 = 432,338 + 20T
        60T = 438,338
          T = 438,338 / 60 = 7,305.6 s
          T = 7,305.6 / 60 = 121.8 min   ~ 122 minutes
```

**If half the outage edits hit a record already queued** (a guess; the real ratio is client question 9): the
backlog is 216,000 calls, so 60T = 216,000 + 338 + 6,000 = 222,338, and T = 222,338 ÷ 60 = 3,705.6 s ≈ **62
minutes**.

**What the 122 minutes includes, and what it doesn't.**

| Cost | Size | In the ~122 min? |
|---|---|---|
| The 338 replay **calls**, as a share of the rate limit | 338 calls | **Yes**: they're in `needed(T)` |
| The ramp's lost capacity | 6,000 calls | **Yes** |
| New portal edits during recovery | 20/s | **Yes** |
| **Fetching** the replay: pages follow a cursor, so they're sequential | 338 × ~200 ms ≈ **68 s** (at least 338 ÷ 20 = 17 s at minute 1's rate) | **No**: it overlaps with applying |
| **Applying** the 675,000 replayed events | 675,000 ÷ 2,500/s ≈ **270 s ≈ 4.5 min** | **No**: the formula assumes the drain isn't held up by it |

**End to end, with the replay applied strictly before the first outbox call:** the drain can only start after
R ≈ 270 s, and the 5,400 portal edits made meanwhile join it. With the replay calls already spent during R:

```
80t - 6,000 = 432,000 + 20 x (270 + t)
        60t = 443,400
          t = 7,390 s
total = 270 + 7,390 = 7,660 s = 127.7 min   ~ 128 minutes
```

So **122 minutes is the rate-limit answer the brief asks for**; **about 128 minutes is the planned end-to-end
time**, the difference being the time to apply the replay. Faster applying (more appliers) moves it towards 122.

**The order doesn't change the drain time.** The total to send and the rate are fixed, so the lanes decide only
*which* calls wait, not how long the drain takes.

## 10.6 When fresh edits are back under 30 s, and catching a stalled drain (Q10b)

**M1 during catch-up counts only fresh-lane rows** (created after recovery). Predicted fresh-lane backlog:

| Phase (time after recovery) | Fresh rows arriving | Fresh-lane send rate | Fresh rows waiting at the end |
|---|---|---|---|
| Replay, 0–270 s | 20/s | 0 (no outbox calls yet) | 5,400 |
| Ramp minute 1, 270–330 s | 20/s | 10/s (half of 20; the backlog lane is always busy, so there's nothing to borrow) | 6,000 |
| Ramp minute 2, 330–390 s | 20/s | 20/s (half of 40) | 6,000 |
| From 390 s | 20/s | 40/s (half of 80) | falls by 20/s: 6,000 ÷ 20 = 300 s |

The fresh lane is clear at about 390 + 300 = **690 s ≈ 11.5 minutes after recovery**. Edits made in the first
minutes of recovery can wait several minutes; that's the catch-up period the brief treats separately.

**Rule: M1 is re-armed when the fresh lane's oldest row first drops below 30 s after recovery, or 15 minutes after
recovery, whichever comes first. Once armed, it pages at > 30 s as usual.** The 15-minute cap stops a stuck
fresh lane from never being watched.

**Drain-stall rule.** Every minute, while the breaker is closed: if the backlog shrank by **less than 75% of
(allowed sends − fresh-lane sends) over the last 5 minutes**, page. "Shrank" counts backlog rows that left the
queue: sent, rejected, or emptied by the guard. A `429` pause counts as slowness, which is worth knowing.

**Example: the backlog stops shrinking 10 minutes into the drain.** By then the fresh lane is clear, so the
backlog runs at about 60/s, 3,600 rows a minute, and the 5-minute expectation is 18,000 rows; 75% of that is
13,500.

| Drain minute | Backlog rows cleared in the last 5 minutes | Page? |
|---|---|---|
| 10 | 5 × 3,600 = 18,000 | no |
| 11 | 4 × 3,600 + 0 = 14,400 | no (≥ 13,500) |
| **12** | 3 × 3,600 = 10,800 | **yes** (< 13,500) |

So we page **2 minutes after the stall**, about 110 minutes before the predicted finish.

## 10.7 Longer or different outages

| Case | What happens |
|---|---|
| The outage is longer than the change log's retention (assumed ≥ 24 h) | Replay what's still there, then "records modified since" the start point gives current values (`via reconcile`, intermediate edits and actors lost). Beyond that, page "unrecoverable window T1–T2, full scan needed" (G4) |
| **Our Postgres** is down instead | The portal is down too, so no portal edits are lost. Webhooks get `503`. On recovery, the same replay start applies: the oldest completed sweep watermark minus 10 minutes (10.3) |
| Our worker restarts during the outage | Nothing changes; the rows and watermarks wait in Postgres |
| Salesforce comes back but returns `429` | Pause all sending for `Retry-After`, keep the work (D11). The drain-stall rule pages if it lasts |
