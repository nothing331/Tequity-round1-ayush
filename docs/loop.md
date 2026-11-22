# 9. The rule that stops a loop

Labels like `D17` point to [decisions.tmp.md](decisions.tmp.md). The steps are in [inbound.md](inbound.md) 5.2
and [outbound.md](outbound.md) 6.1.

## 9.1 The rule

**Only a portal user's save writes to the outbox. Nothing on the webhook path ever does.**

A change that came from Salesforce can never be sent back to Salesforce, so a loop is impossible by
construction, not by detection. Our own update coming back (an echo) is recognised so it doesn't create a
duplicate history row, but even if it weren't recognised it still couldn't be sent back.

| Writes to `sync.outbox` | Never writes to `sync.outbox` |
|---|---|
| `saveEdit()` in the portal backend, on a portal user's save (outbound.md 6.1) | The webhook receiver |
| The worker, only to change a row's status, key version or lease (never to add a row) | The worker applying a replay, a fetch or the sweep |
| | Any database trigger (there is none, by rule) |

## 9.2 How it's enforced

- **In code:** `saveEdit()` is the only function that inserts outbox rows. The apply function used by the
  receiver and the worker (`applyEvent`) has no outbox write in it.
- **In the database:** the `receiver` role has **no access** to `sync.outbox` at all (data.md 4.9).
- **By rule (CLAUDE.md):** no trigger writes the outbox, and no portal automation may call `saveEdit()` in
  reaction to a value a webhook applied. Either would turn a Salesforce change into an outgoing call.

## 9.3 Recognising our own echo

Every write we send carries `Portal_Change_Id__c = "ob_<outbox id>.<content_gen>"` (O1′). When Salesforce sends
the webhook for that write, the key is in `changedFields`. The receiver reads the outbox row from the key, and:

| Field in the echo | What happens (inbound.md 5.2) |
|---|---|
| **Kept exactly as we sent it** (row b) | The field keeps the portal save's time, actor and `editId`. `sfSeq` and `sfValue` move forward only if newer. **No history row**, because the portal save already wrote one |
| **Changed by Salesforce**, e.g. a phone reformatted (row c) | Applied as a Salesforce change, noted "normalised by Salesforce". **Not sent back**, by the rule in 9.1 |
| Any field, when a **newer** sales edit already reached us | Nothing moves backwards: `sfSeq` is already higher (I1) |

The echo also marks the outbox row `sent`. For an attempt whose reply was lost, that's the evidence that it
landed (outbound.md 6.6).

**Rejected ways to recognise an echo:**
- **"`changedBy` is the integration user"** on its own: it would also swallow genuine changes made by Salesforce
  automation running as that user.
- **"The value equals what we sent"**: it would swallow a real A → B → A change by a sales user.

## 9.4 Cases

| Case | Outcome |
|---|---|
| Finance's city write comes back as seq 4 | Echo, row b. The outbox row is `sent`; the field still shows finance; no history row; nothing is sent |
| Salesforce reformats our phone `9876543210` → `+91 98765 43210` | Row c. The portal shows the reformatted phone, noted "normalised by Salesforce". Nothing is sent |
| A Salesforce flow changes another field because of our write (a separate event, no key) | A normal Salesforce change, applied by the usual rules. Nothing is sent |
| The echo never arrives | The `2xx` reply already marked the row `sent`. A later replay that contains the event is dropped as a duplicate or treated as the echo |
| Our echo arrives after a newer sales edit to the same field | Row b changes nothing on that field; the sales value and its actor stay (I1) |
| Finance and sales keep overwriting each other's value | Each overwrite is a **human** save, each recorded in history. It's not a sync loop, and nothing repeats on its own (item 7) |

## 9.5 If a webhook carries the full record (the brief's "other case", D2)

With full-record webhooks, `Portal_Change_Id__c` is present on **every** event after our write, because it's a
field on the record, not just on our event.

**Why a snapshot can't identify our echo (Q9).** Take our write with key X, then a Salesforce automation that
runs **as the integration user** and changes the phone. Its full snapshot still carries X, and `changedBy` is
the integration user. Both checks once proposed ("the key matches a row" and "`changedBy` is the integration
user") pass, yet this event isn't our echo. Nothing in a full snapshot proves that **this** event set X; X may
simply have been on the record already.

**So in full-snapshot mode:**

| Rule | Detail |
|---|---|
| **Echo rules are never applied from a snapshot** | Rows b and c of inbound.md 5.2 need proof that the event is ours. A snapshot can't give it |
| **A field equal to our current value is a no-op** | Only `sfSeq` and `sfValue` move forward. No history row; the field keeps its actor and `editId`. This is what stops finance's city flipping to the integration user every time a snapshot repeats it |
| **A field with a different value is a normal Salesforce change** | Judged by rows d–f, with `actor = changedBy`. The automation's phone change is recorded as made by the integration user, which is what happened |
| **A snapshot never confirms an uncertain send** | An outbox row waiting for evidence (outbound.md 6.6) is resolved only by the `2xx` reply, a change-log event that lists `Portal_Change_Id__c` among **its own** changed fields, or, as the last fallback, the G3 GET |

Observation: the G3 GET is the same kind of evidence as a snapshot. X on the record proves an attempt with key X
landed at some point (only our calls write that field, and each key belongs to one content version), but not
which event delivered it. It's kept only as the last fallback, when Salesforce provides neither `sequence` nor
per-event changed fields.

The loop rule itself doesn't change: whatever the payload looks like, the webhook path never writes the outbox.
