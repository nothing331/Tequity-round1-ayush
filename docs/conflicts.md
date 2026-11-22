# 7. The rule when both sides change the same record

Labels like `D16` point to [decisions.tmp.md](decisions.tmp.md). Tables and columns are in [data.md](data.md);
the steps that run these rules are in [inbound.md](inbound.md) and [outbound.md](outbound.md).

## 7.1 The rule

**Each field is decided on its own. The later edit wins, judged by `field_meta[f].updatedAt`.**

| Where the edit comes from | Its time (`updatedAt`) |
|---|---|
| A Salesforce webhook | The webhook's **`occurredAt`** |
| A portal save | The **Postgres clock time taken inside the save transaction, after the record lock** (`saveEdit()` step 5). The brief's portal sample has no time field, so we never trust a client clock |

- **Different fields of one record:** both edits survive, because each field has its own `updatedAt`. The
  record is never compared as a whole.
- **The same field, edited on both sides:** the later `updatedAt` becomes the current value. **On an exact tie,
  Salesforce wins**, so the result never depends on arrival order.
- **The loser is never thrown away.** It stays in `history`. If it lost at the moment it arrived it's marked
  `superseded`, and the UI badge explains it (7.4).
- **Clock skew** between Salesforce and us is assumed to be **≤ 1 s** (assumption 2.2). Two cross-side edits to
  the same field less than a second apart could be decided either way; anything further apart is decided
  correctly.

## 7.2 How the rule is enforced on each path

| Path | Where | Check |
|---|---|---|
| **Webhook into Postgres** | inbound.md 5.2, rows e and f | `occurredAt < updatedAt`: the portal value is newer and stays. Otherwise, including a tie, the Salesforce value is applied. Always **after** the sequence check, so an older Salesforce event never wins on time |
| **Portal save into Postgres** | outbound.md 6.1, step 3 | Per-field `baseUpdatedAt` check. A field changed since the form loaded gets a `409` instead of a silent overwrite (D20) |
| **Our queued value toward Salesforce** | outbound.md 6.3, the guard | A field is sent only if `field_meta[f].editId` still equals the outbox field's `editId`. If a newer Salesforce value won in the meantime, its `editId` replaced ours, and the field is dropped |
| **Our call landing in Salesforce** | outbound.md 6.5–6.6 | Every update is conditional on `sf_seq_contiguous`. If Salesforce has a change we haven't seen, the call gets `412` and nothing lands. The worker waits for the webhook or replays, then the guard re-decides on exact per-field times (G1, V1) |

The last row is what closes the race the guard alone can't: a sales edit that reaches Salesforce after our
guard check but before our call (G1).

## 7.3 The cases the brief names

| Case | What happens |
|---|---|
| **Same record, different fields, opposite sides** (the brief's scenario: finance city, sales phone) | Both survive. The webhook only touches `phone`; the outbox only sends `city` |
| **Same field, opposite sides** | Per-field `updatedAt`, tie to Salesforce, loser kept in history |
| **Same field, two portal users** | The second save gets `409` with the current value, who set it and when ("A changed City to Noida 3 s ago; keep or overwrite?"). Overwrite means saving again with the new `baseUpdatedAt`. **Salesforce receives only accepted values**, merged into the record's pending outbox row (D20) |
| **Same field, two sales users** | Salesforce orders them itself; each webhook carries a higher `sequence`, and the per-field sequence rule keeps the later one (D19) |
| **Same record, two portal users, different fields** | Both saves are accepted. The record lock serialises them, and the pending outbox row merges both fields into one call |

## 7.4 Worked example: a race in flight

```
11:20:00.0  Finance saves city = Noida        portal updatedAt 11:20:00.0, editId h17, outbox row 1042
11:20:00.5  Sales saves city = Delhi           Salesforce seq 3, occurredAt 11:20:00.5 (webhook in flight)
11:20:01    Worker claims 1042                 guard passes (editId still h17); call: city = Noida, if version 2
            Salesforce is at version 3         -> 412, nothing lands
11:20:01.2  Delhi webhook arrives              5.2 row f: 11:20:00.5 > 11:20:00.0 -> city = Delhi,
                                               actor sales.user@…, editId h19
11:20:06    Worker re-judges 1042              guard: editId is now h19, not h17 -> city dropped,
                                               row closed with nothing to send
Result      Both sides: Delhi. History: Noida (finance, 11:20:00.0), Delhi (sales, 11:20:00.5)
```

If finance's save had been the later one, the webhook would have hit row e (portal stays), the `editId`
would still be h17, and the resend after the `412` would land Noida on both sides.

## 7.5 What the losing user sees (D16, G1b)

- While their value is in flight: **"pending sync"**.
- If it lost: a badge on the field, built from history: **"Replaced by sales.user@client.example in Salesforce
  at 11:20:00.5. Your value 'Noida' (saved 11:20:00.0) is in history."**
- **"Use my value again"** sends a normal new save with `baseUpdatedAt` = the current value's time. It wins
  because it's now the later edit, and goes out with the precondition. If the field changed yet again, the user
  gets the usual `409`.
- **Nothing retries on the user's behalf.** Two teams overwriting each other on purpose is visible in history;
  it's a business question (field ownership, client question 7), not a sync bug.
- No notifications table: the badge shows when the record or list is opened.
