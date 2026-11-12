# Design: Two-way sync of customer and contact records

## 1. Assumptions and open questions

The brief is silent on these points. Each one has the assumption we
build on, and one sentence on what changes if the answer goes the other
way. Section 1.2 lists the questions we would ask the client before we
lock the design.

### 1.1 Assumptions

**A1. Peak edit rate.**
Portal users make **10 edits/s at peak** and 1 edit/s in a quiet hour.
Salesforce users make 20 edits/s at peak. One "edit" is one form save
on one record, possibly touching several fields.
Only portal edits consume our 100 req/s Salesforce cap; inbound
webhooks do not.
*If different:* the outage drain time in §10 scales with the portal
rate. If portal edits ever approach our sending budget (~80/s), normal
running can no longer stay inside 30 s without batching several records
per call.

**A2. A webhook carries only the changed fields.**
`changedFields` holds exactly the fields the Salesforce user changed,
as in the sample.
*If different (full record):* the value comparison only tells us
*whether* a field changed. *Whether we apply it* is still decided by
age: each field stores the Salesforce time of the value we last
applied, and an incoming field is applied only if its `occurredAt` is
newer (§8). So an old full-record webhook cannot overwrite newer data.

**A3. We can ask Salesforce what changed since a point in time.**
*This is an API the brief does not give us.* Real Salesforce has one
(sObject Get Updated: `/sobjects/Contact/updated/?start=…&end=…`).
Known limits: it returns record IDs only, ignores the seconds in the
window (so we overlap windows by a minute), returns at most 600,000
IDs per call, and looks back at most 30 days. To get the values we
also need a read call (A11). We use it for outage catch-up and a
periodic sweep for missed webhooks.
It returns only the *current* state: if a field changed twice while we
were not listening, the middle value and its author are not visible.
*If different, fallbacks in order:*
1. Change Data Capture, if the client enables it: Salesforce keeps
   change events for 72 hours and lets us replay from a stored
   position, including intermediate values.
2. Salesforce's own webhook retries, if they outlast 6 hours (A8).
3. A nightly full comparison of all records (slow safety net).
4. Otherwise we tell the client plainly: Salesforce edits made during
   the outage are lost to us, and draining our backlog may overwrite
   them without us knowing.

**A4. Two portal users editing the same field: the stale save is rejected.**
Each portal save carries the version of each field the user loaded.
If another save changed that field in between, the later save is
rejected for that field. The user sees the current value and who set
it, and can save again on purpose. Only the accepted save goes to
Salesforce. Saves that touch *different* fields of the same record
both succeed.
*If different (silent last-writer-wins):* simpler UI, but one user's
value is replaced without them knowing. History still keeps both
values, so nothing is lost on record, only in the user's awareness.

**A5. The time and actor of a portal edit come from our server.**
The actor is the logged-in user from the session, not the `editedBy`
field in the request body (the body is only checked to match). The
time is `clock_timestamp()` on our server, never a time sent by the
browser.
That time is **for display in history only; it never decides which
edit wins.** (`now()` would be the transaction *start* time, and true
commit time needs `track_commit_timestamp`; we need neither.)
Order among portal edits comes from a **version number per field**,
incremented while holding the row lock (`SELECT … FOR UPDATE`). A4's
stale-save check compares these versions.
*If different:* nothing in the ordering depends on our clock, so clock
drift on our side only affects the displayed time.

**A6. We never compare our clock with Salesforce's clock.**
Two clocks cannot reliably order edits made a second apart. Order
between the two sides is decided by Salesforce's own ordering: what
Salesforce has accepted, and `occurredAt` on webhooks, which comes
from a single clock. The detailed rule is in §7.
*If different (we had to compare clocks):* edits within the skew
window would be ordered arbitrarily, and the two sides could end up
showing different values.

**A7. We get a dedicated Salesforce integration user.**
Every write we make to Salesforce is made as `integration@…`, so the
webhook Salesforce sends back for our own write has
`changedBy = integration@…`.
*If different:* we recognise our own echoes only by matching the
webhook against the log of what we sent (§9). This still works but is
harder to make airtight.

**A8. Salesforce retries failed webhooks for a limited time, then gives up.**
We do not rely on the retries for correctness. A 6-hour outage may
outlast them, which is why A3 is needed.
*If different (no retries at all):* even a short blip in our own
endpoint loses events, so the A3 catch-up query runs more often, not
only after outages.

**A9. Records created in the portal get their Salesforce ID from Salesforce.**
The portal creates the record with its own ID and stores the
Salesforce ID once the create call succeeds. Before retrying a create,
we must be able to tell whether the first attempt actually succeeded.
We assume we can set an external-ID field on the Salesforce record
(our portal ID) and look it up, so a retried create never makes a
duplicate.
*If different:* a timeout on a create could leave a duplicate record
in Salesforce, and we would need a manual de-duplication step.

**A10. Webhooks are authenticated.**
Each webhook carries a signature (shared secret, HMAC over the body)
that we verify before storing anything. We may add a field for it.
*If different:* anyone who finds the endpoint could write records, so
we would at least need an IP allowlist.

**A11. We can read one record from Salesforce by ID.**
*Another API the brief does not give us* (it names only create and
update). We use it in two places: to fetch values for IDs returned by
A3, and to settle a conflict when we cannot tell the order (§7). Each
read counts against the 100 req/s cap.
*If different:* we cannot settle unclear conflicts by asking
Salesforce, and outage catch-up depends entirely on the A3 fallbacks.

**A12. Salesforce field history (who changed a field to what, when) is
not available to us.**
We assume we cannot read it, so the design does not depend on it.
*If different:* we could recover intermediate values that we never
received a webhook for (see the limitation in §7).

### 1.2 Questions we would ask before locking the design

1. What are the real peak and quiet-hour edit rates on each side? (A1)
2. Does the webhook carry changed fields only, or the whole record? (A2)
3. Can we query Salesforce for records modified since a time, and does
   that query count against the 100 req/s cap? (A3)
4. When two portal users edit the same field, is rejecting the stale
   save acceptable to finance and customer-success staff? (A4)
5. Can we read a single record by ID, and read Salesforce field
   history? (A11, A12)
6. Can the client enable Change Data Capture on Customer and Contact?
   (A3 fallback)
7. Does the record API support a conditional update ("only if not
   modified since…")? That would close the blind overwrite in §7.
8. Can we have a dedicated integration user and an external-ID field
   on Customer and Contact? (A7, A9)
9. How long and how often does Salesforce retry a failed webhook? (A8)
10. If both sides change the same field within a second or two, does
    the client want a rule ("Salesforce wins for owner, portal wins for
    billing city"), or is "latest accepted by Salesforce wins, history
    kept" enough?
11. Does the Salesforce API support updating several records in one
    request? If so, how many records does a batch count as against the
    cap?
12. How long must history be kept, and who may read it?

## 2. Systems, who calls whom, network boundary, interfaces
## 3. Subproblems, in build order
## 4. Data we store
## 5. Inbound path: webhook → Postgres
## 6. Outbound path: portal edit → Salesforce API
## 7. Rule: both sides change the same record

**Different fields: both survive.** We sync per field, never whole
records. A webhook updates only the fields in `changedFields`; an
outbound call sends only the fields the portal user changed. So a
sales edit to `phone` and a finance edit to `city` never touch each
other.

**Same field, two portal users:** the stale save is rejected (A4),
using the per-field version number (A5).

**Same field, one portal user and one Salesforce user:** we never
compare our clock with Salesforce's (A6). Instead, each field keeps:

| Field state | Meaning |
|---|---|
| `value` | current value in the portal |
| `sf_time` | Salesforce `occurredAt` of the last Salesforce value we applied |
| `sync_state` | `synced`, `pending` (not yet accepted by Salesforce) or `awaiting_echo` (accepted, our own webhook not seen yet) |
| `base_sf_time` | for a pending edit: the `sf_time` the user saw when they edited |

Three rules:

1. **Don't send a stale edit.** If a Salesforce value arrives for a
   field that is `pending`, with `occurredAt` newer than the edit's
   `base_sf_time`, the portal edit was made without seeing it. We
   cancel the send. The Salesforce value becomes current, the portal
   value goes into history, and the field is flagged as a conflict
   for the portal user. The same check runs again just before the
   call is sent.
2. **The API response means accepted.** When the update call succeeds,
   the field moves to `awaiting_echo`. When the webhook for our own
   write arrives (§9), we record its `occurredAt` as `sf_time`, move
   to `synced`, and never send it back. The portal does not show the
   edit as pending while it waits for that webhook.
3. **When the order is unclear, ask Salesforce.** If a webhook with a
   *different* value arrives for a field in `awaiting_echo`, we cannot
   tell whether it came before or after our write. We read the record
   (A11) and take Salesforce's current value. Both values go into
   history. If our own webhook never arrives, the periodic sweep (A3)
   moves the field to `synced` once Salesforce's value matches ours.

Rule 3 makes both sides converge: the portal always ends up with the
value Salesforce holds.

**Worked example.** A finance user sets city to Noida. A sales user
sets city to Delhi at about the same time.

- *Delhi's webhook arrives before we send Noida.* Rule 1: Noida is not
  sent. Delhi is current, Noida is in history, and the finance user
  sees a conflict.
- *Delhi is saved in Salesforce, but its webhook arrives after Noida
  is sent.* Salesforce now holds Noida (last write wins there). Delhi's
  webhook arrives while the field is `awaiting_echo`, so rule 3 reads
  the record: Noida. Delhi is recorded in history, and the field is
  flagged as a conflict.

**Limitation: a blind overwrite can lose a value from our history.**
In the second case above, Delhi reaches our history *only because its
webhook arrived*, even if late. If that webhook never arrives (for
example during the 6-hour outage, when undelivered webhooks may be
dropped), then Noida overwrites Delhi in Salesforce, the sweep reads
Noida, and we never learn that Delhi existed. We cannot guarantee
Delhi is in our history in that case.

The design does not hide this. It narrows it to a few seconds in
normal running, and to the outage window. Three things would close it,
none of which the brief gives us:
- a conditional update in the record API ("only if not modified
  since…"; question 7 in §1.2), so Noida would be refused instead;
- Change Data Capture (A3 fallback 1), which replays every change,
  including Delhi;
- Salesforce field history (A12), which keeps every value of the
  field.
## 8. Rule: duplicate and out-of-order webhooks
## 9. Rule: stopping the sync loop
## 10. The 6-hour outage and the arithmetic
## 11. Three numbers to watch and alert thresholds
## 12. Out of scope this weekend