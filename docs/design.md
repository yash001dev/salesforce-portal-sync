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
Salesforce ID once the create call succeeds.
*Assumption beyond the brief* (which names only create and update):
Customer and Contact have an external-ID field holding our portal ID,
and we can **upsert by that external ID**. Every create is sent as an
upsert, so if a timed-out create actually landed, the retry updates
that same record instead of making a second one.
*If different:* before retrying a create we would have to look the
record up by external ID (another assumed API), or accept that a
timeout can leave a duplicate that needs manual de-duplication.

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

**A13. Salesforce delivers webhooks within a few seconds of the edit.**
Webhook delivery time is outside our control, but it is part of the
30-second target on the inbound path. We measure it (§3, step 2).
*If different:* the inbound 30-second target cannot be met by anything
we build; we can only detect and report it.

**A14. Starting numbers, to be tuned in production.**
Send at most **80 req/s** (20 below the cap, for safety). **Recovery
work is guaranteed at least 20% of that (16 req/s)** whenever it has
work waiting; capacity one class does not use goes to the other. Each
Salesforce call has a **5-second timeout**. These are configuration
values, not constants in code.
*If different:* the budget in §3 changes; the rules do not.

**A15. We can get the client's Salesforce validation rules and allowed
values.**
The Portal API checks the rules we know about before saving
(phone and email format, required fields, allowed `status` values,
`owner` is a known Salesforce user, a contact's customer exists) and
returns `422` with an error per field. We keep these rules in a
config file. They reduce rejections; they do not remove them, because
some Salesforce rejections depend on things we cannot check
(permissions, rules that look at other data, rules changed without
notice).
*If different:* more edits reach Salesforce and come back `rejected`
(§3 step 4). Nothing is lost; users just see the error later.

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
13. Are the 5 million records already linked on both sides (each
    portal record knows its Salesforce ID)? If not, the first load
    must be a bulk import: through the record API at 80 req/s it
    would take 5,000,000 / 80 = 62,500 s, about 17 hours.
14. How quickly does Salesforce normally deliver a webhook? (A13)
15. Can we get the current Salesforce validation rules and allowed
    values, and who tells us when they change? (A15)

## 2. Systems, who calls whom, network boundary, interfaces

### 2.1 Shape

One service, two processes, one database. **The two processes share
nothing but Postgres.** Nothing passes between them in memory; a
Postgres `NOTIFY` only wakes the worker early, and a 1-second poll
finds the work if a notify is missed.

```
                 ┌──────────────── our service ─────────────────┐
 Salesforce ───► │ API process                                   │
  (webhook)      │  POST /webhooks/salesforce ──► inbox          │
 Portal user ──► │  PATCH /records/:type/:id  ──► record,        │
  (save)         │                               history, outbox │
                 │                         Postgres              │
                 │ Worker process             ▲                  │
                 │  applies inbox ────────────┤                  │
                 │  sends outbox ─────────────┴── HTTPS ─────────┼──► Salesforce record API
                 └───────────────────────────────────────────────┘
```

| Caller → callee | Path | Crosses the public internet |
|---|---|---|
| Salesforce → API process | webhook over HTTPS | yes |
| Portal user → API process | portal save over HTTPS | yes |
| Worker → Salesforce | record API over HTTPS, rate-limited (§3 step 4) | yes |
| API process ↔ worker | Postgres tables (`inbox`, `outbox`) | no |

A sales user talks only to Salesforce; a portal user talks only to the
portal.

**No Kafka or Redis.** Peak traffic is about 30 events/s, and a
6-hour outage leaves about 216,000 outbox rows; Postgres handles both.
The outbox works *because* the queue entry and the data are written in
one transaction; a separate queue service would bring back the
dual-write problem. Postgres already gives us what a queue needs:
`FOR UPDATE SKIP LOCKED` to claim rows, `LISTEN/NOTIFY` to wake the
worker, unique constraints for duplicates. The rate limiter lives in
the single worker's memory. If we later run several workers, the
shared limiter is a small token table in Postgres first.

### 2.2 Inbound: Salesforce change → portal

API process:
1. Salesforce calls `POST /webhooks/salesforce`.
2. Verify the signature (A10); check object type and field names.
3. Insert into `inbox` keyed by `eventId` (`ON CONFLICT DO NOTHING`);
   commit.
4. Reply `200`, also for a duplicate, then `NOTIFY`.

Worker, in **one transaction**:
1. Claim one unapplied inbox row (`FOR UPDATE SKIP LOCKED`).
2. Lock the record row (`SELECT … FOR UPDATE`).
3. Per field: age rule (§8), own-write match (§3 step 5, §9), conflict
   rules (§7).
4. Write history rows; update field state; mark the inbox row
   applied.
5. **Never write to the outbox.**

Any failure rolls the whole transaction back and the inbox row is
tried again.

### 2.3 Outbound: portal save → Salesforce

API process, in **one transaction**:
1. Actor from the login session (A5).
2. Check known Salesforce rules (A15); a failure returns `422` and
   nothing is saved.
3. Lock the record row; version check per field (A4); a stale field
   returns `409`.
4. Set the fields to `pending`, write history rows, write or merge the
   record's pending outbox row.

Commit, then `NOTIFY`.

Worker:
1. **Claim (short transaction).** Pick an outbox row whose record has
   no call in flight; user edits first, recovery guaranteed 20%
   (A14). Take a rate-limit slot. Mark the row in flight with a lease
   ("mine until now + 30 s"). Commit.
2. **Call Salesforce outside any transaction.** We never hold a
   database transaction open across an HTTP call. Re-run §7 rule 1
   first.
3. **Record the result (new transaction).**
   - Success: fields still holding the sent value become `synced`
     with `sf_time = NULL`; the outbox row is done.
   - Timeout, `5xx`, `429`: attempt count + 1, next attempt time set,
     lease released.
   - Other `4xx`: fields become `rejected`; the row stops (§3 step 4).

The webhook for our own write later arrives on the inbound path and
fills in `sf_time` (§3 step 5).

### 2.4 Why inbox and outbox are separate

| | inbox | outbox |
|---|---|---|
| Holds | what Salesforce told us | what we must tell Salesforce |
| Key | `eventId` (unique: catches duplicates) | record: one in flight + one pending (merges edits) |
| Written by | the webhook endpoint | a portal save, in the same transaction as the data |
| Limited by | nothing (webhooks are outside the cap) | 80 req/s, recovery share (A14) |
| Retry means | re-run a local transaction | growing wait, lease, re-check §7 rule 1 |
| Afterwards | kept as a raw record of what Salesforce sent | removed or archived |

Without the inbox we would have to apply before replying: a bug or a
slow lock becomes an error, Salesforce resends, and eventually gives
up. Without the outbox, "saved in Postgres" and "will be sent" cannot
happen together.

### 2.5 What a crash does

| Crash point | What happens | Change lost? |
|---|---|---|
| API, before the inbox commit | no `200`; Salesforce resends (A8); the sweep catches it if Salesforce gives up | no |
| API, after the commit, before the `200` | Salesforce resends; duplicate `eventId` ignored; `200` | no |
| API, during a portal save | transaction rolls back; the user sees an error | no |
| Worker, while applying | rolls back; inbox row still unapplied; tried again | no |
| Worker, before the Salesforce call | lease runs out; row retried | no |
| Worker, after sending the call, before recording the result | lease runs out; call retried, as for a timeout | no, but the retry can overwrite an unseen Salesforce edit (§3 step 4, §7) |
| Postgres down | webhooks get `503` (Salesforce resends); portal saves fail visibly; worker waits | no, while Salesforce keeps resending |

Postgres is the one part everything depends on. Managed Postgres with
automatic failover covers that; it is outside this weekend's scope.

### 2.6 Interfaces

**Webhook endpoint** (Salesforce calls us)
`POST /webhooks/salesforce`, a signature header (A10), body as in the
brief's sample. Responses: `200` stored or duplicate · `401` bad
signature · `400` malformed · `503` database down, please retry.

**Portal save** (portal user calls us)
`PATCH /records/:type/:id`, body `changedFields` plus, per field, the
version the user loaded. The actor comes from the session; a body
`editedBy` is only checked to match. Responses: `200` with new
versions · `409` per stale field, with its current value and actor ·
`422` per field breaking a known Salesforce rule (A15).

**Salesforce record API** (we call Salesforce)
- update: changed fields only;
- create: upsert by external ID (A9);
- read one record by ID (A11);
- records modified since a time (A3).

Responses: `2xx` done · `429` slow down, keep the work · timeout or
`5xx` retry · other `4xx` rejected, no retry.
## 3. Subproblems, in build order

The order follows three questions: what is hardest to change later
(the data model, and durable writes), what we do not control (inbound
webhooks), and what spends the scarce resource (calls to Salesforce,
capped at 100 req/s). Each step adds its own measurement when it is
built, so we can see each part working before the next one depends on
it. Measurements are SQL queries on tables we already have (inbox,
outbox, history), exposed on a `/metrics` endpoint.

### Step 0 (open question): initial load
Are the 5 million records already linked on both sides? (§1.2 q13.)
If not, the first load is a one-off bulk import, not the record API,
which would take about 17 hours. Not part of the build order below.

### Step 1: per-field state and history
**What:** the record table with the current values and, per field, the
state from §7 (`sf_time`, version number, `sync_state` =
`pending` / `synced` / `rejected`, `base_sf_time`); the Salesforce-ID ↔ portal-ID mapping; and an
append-only history table, one row per field change (value, actor,
time, source).
**Why first:** every later rule compares an incoming change with this
state.
**If skipped:** we would sync whole records, so concurrent edits to
different fields overwrite each other; and history added later means
changing every write path. Any history dropped before then counts as
a lost change.

### Step 2: inbound (webhook → inbox → apply)
**What:** verify the signature, insert the raw webhook into `inbox`
with a unique `eventId`, and return 2xx **only after that insert
commits**. A separate apply step updates each field by the age rule
(§8), under a row lock (`SELECT … FOR UPDATE`).
**Rule from day one: applying a webhook never queues a Salesforce
update.** This is the core of the loop rule (§9).
**Why before outbound:** it is the input we do not control, and the
conflict and loop rules depend on it.
**If skipped:** acknowledging before the commit loses events on a
crash; without the unique `eventId` a duplicate is applied twice;
without the age rule an older webhook overwrites newer data.
**Measures:** age of the oldest unapplied webhook; rejected
signatures; duplicates dropped; webhook delivery lag (`occurredAt` to
received time — a clock comparison used only for alerting, never for
ordering).

### Step 3: outbound (portal save → outbox → worker)
**What:** a portal save writes the field change, the history row and
an `outbox` row in **one Postgres transaction**. A worker process
sends outbox rows to Salesforce.
- At most one call per record in flight, so two edits to one record
  cannot reach Salesforce in the wrong order.
- Pending edits to the same record are merged into one call. A row
  already in flight is never changed; a new save adds one pending row
  behind it. So each record has at most one row in flight and one
  pending.
- Two priorities: **user edits**, and **recovery work** (catch-up
  reads, sweep, bulk changes).
- Creates are sent as upserts by external ID (A9).
**Why here:** it is the only way pending calls survive a restart, and
the rate limiter (step 4) needs something to control.
**If skipped:** writing to Postgres and calling Salesforce directly
means a crash between the two loses the edit, or sends it without
saving it; during an outage pending calls live only in memory.
**Measures:** age of the oldest pending user edit (the 30-second
number); outbox size per priority.

### Step 4: rate limiter and retries
**What:** one shared limit of 80 req/s for every call we make,
including retries, reads (A11) and catch-up queries (A3). Recovery is
guaranteed at least 16 req/s when it has work (A14).
On timeout or failure: up to **2 quick retries** (after 1 s, then 2 s).
After that the row stays in the outbox and retries with a growing wait
(4 s, 8 s … up to 60 s), marked late. Nothing is dropped. On HTTP 429
we slow down and keep the work.
**Permanent rejection.** A `4xx` other than 429 (for example a
Salesforce validation rule refusing a phone format) will not succeed
on retry. The field becomes `rejected`, the outbox row stops, and the
portal shows the user Salesforce's error so they can correct it; their
correction is a normal new edit. We never undo the portal value
automatically. Until it is corrected, the two sides differ on that
field, and the field is visibly marked.
**Retry safety.** Updates set fields to fixed values, so sending the
same update twice leaves the same state. Before every retry we re-run
§7 rule 1 (cancel if a newer Salesforce value has arrived), and we
send the *current* pending values, not the original request.
*Limitation:* if our update landed but the response timed out, and a
Salesforce user changes the same field before our retry and before
their webhook reaches us, the retry overwrites their value. This is
the same blind overwrite described in §7; retries are not completely
safe.
**Why here:** with steps 2 and 3 in place, this is the first thing
that can harm Salesforce itself.
**If skipped:** retries pile up, we exceed the cap, Salesforce
throttles us, and throttling causes more retries.
**Measures:** calls per second against the cap; HTTP 429s and
timeouts per minute; Salesforce call duration (p95); age of the oldest
waiting recovery item; rejections per hour (each one is investigated:
it may be an outdated rule copy, A15, or a cause we cannot check, such
as permissions or a rule that depends on other data).

### Step 5: recognising our own write when it comes back
**What:** the webhook for our own write is matched by the integration
user (A7) and the values we sent. It fills in the field's `sf_time`
(NULL since the API response, §7 rule 2), is recorded, and is never
sent back.
**Why after 2 and 3:** it is about how those two paths interact.
**If skipped:** step 2's rule already prevents loops, but `sf_time`
stays NULL, so every later different value costs a read (§7 rule 3).
**Measures:** synced fields whose `sf_time` has been NULL for more
than 5 minutes.

### Step 6: conflict rules (§7)
**What:** don't send a stale edit; the API response means synced;
when the order is unclear, read the record. A few checks inside the
paths built in steps 2–5.
**If skipped:** the two sides can show different values, or a portal
edit silently overwrites a Salesforce edit.
**Measures:** conflicts per hour.

### Step 7: recovery (sweep, outage catch-up, backlog drain)
**What:** the drain is the step 3 worker at the step 4 limit. Catch-up
and the sweep use A3 and A11 and feed results through the **same apply
path as webhooks** (step 2).
**Why last:** it reuses everything above; building it first would
create a second apply path that drifts from the first.
**If skipped:** Salesforce edits whose webhooks were dropped never
reach us.
**Measures:** how far behind the sweep is.

### Step 8: alert thresholds
Choose, from the measurements above, the three we page on (§11).

### The 30-second target
30 seconds is a **design target** while both systems are up, not a
guarantee. Our own outbound steps are budgeted as follows:

| Step | Target | Held by |
|---|---|---|
| Save + history + outbox commit | < 0.1 s | one transaction |
| Worker notices the row | ≤ 1 s | `LISTEN/NOTIFY`, 1 s poll as backup |
| Earlier call on the same record finishes | ≤ 5 s | one call per record, 5 s timeout |
| Queue wait | ≤ 5 s | ≥ 64 req/s for user edits, so up to ~320 edits ahead |
| Call plus 2 quick retries | ≤ 18 s | 5 + 1 + 5 + 2 + 5 |
| **Total** | **≈ 29 s** | |

At the assumed 10 portal edits/s the queue is nearly always empty, so
the realistic worst case for our steps is about 24 s. Inbound adds
Salesforce's webhook delivery time (A13), which we do not control;
our part (verify, store, apply) is about 1 s, plus up to 5 s if §7
rule 3 needs a read.

**When the target can be missed, and how we see it:**

| Case | Measured by |
|---|---|
| More user edits queued than we can send in time | oldest pending user edit: warn at 20 s, page at 30 s |
| Salesforce answers slowly | call duration p95: warn above 2 s |
| Timeouts or HTTP 429 push past the quick retries | timeouts and 429s per minute |
| Salesforce delivers webhooks late | webhook delivery lag: warn above 10 s |
| Draining the backlog after an outage | separate target (§10); backlog size over time |

During a drain, queued portal edits are sent oldest first; a new edit
to a record already queued merges into that item, and a new edit to
any other record waits its turn. The 30-second target is suspended
until the backlog is gone.
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
| `sf_time` | Salesforce time (`occurredAt`) of the current value. **NULL** after our own write is accepted, until the webhook for that write tells us its Salesforce time |
| `sync_state` | `pending` (not yet accepted by Salesforce), `synced` (accepted), or `rejected` (Salesforce refused it permanently, §3 step 4) |
| `base_sf_time` | for a pending edit: the `sf_time` the user saw when they edited |

Three rules:

1. **Don't send a stale edit.** If a Salesforce value arrives for a
   field that is `pending`, with `occurredAt` newer than the edit's
   `base_sf_time`, the portal edit was made without seeing it. We
   cancel the send. The Salesforce value becomes current, the portal
   value goes into history, and the field is flagged as a conflict
   for the portal user. The same check runs again just before the
   call is sent.
2. **The API response means synced.** When the update call succeeds,
   the field becomes `synced` with `sf_time = NULL`. When the webhook
   for our own write arrives (§9), we fill in `sf_time` from its
   `occurredAt` and never send it back. Nothing else changes; the
   portal shows the edit as synced from the API response onward. If
   that webhook never arrives, `sf_time` simply stays NULL; the only
   cost is that a later different value for that field triggers
   rule 3.
3. **When the order is unclear, read the record.** If a webhook with a
   *different* value arrives for a field whose `sf_time` is NULL, we
   cannot tell whether it came before or after our write. We read the
   record (A11) and take Salesforce's current value. Both values go
   into history, with their relative order recorded as **unknown**.

Rule 3 makes both sides converge: the portal ends up with the value
Salesforce holds.

**What a read does not give us.** A read returns Salesforce's *current*
value. It does not tell us the order in which the competing edits
happened, and it does not return values that were set and overwritten
in between. So rule 3 restores agreement between the two sides, not
complete history: values we received a webhook for are in history
(order possibly unknown); values we never received are missing (see
the limitation below).

**Worked example.** A finance user sets city to Noida. A sales user
sets city to Delhi at about the same time.

- *Delhi's webhook arrives before we send Noida.* Rule 1: Noida is not
  sent. Delhi is current, Noida is in history, and the finance user
  sees a conflict.
- *Delhi is saved in Salesforce, but its webhook arrives after Noida
  is sent.* Salesforce now holds Noida (last write wins there). Delhi's
  webhook arrives while the field's `sf_time` is NULL, so rule 3 reads
  the record: Noida. Delhi is recorded in history, the order of Delhi
  and Noida is recorded as unknown, and the field is flagged as a
  conflict.

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

The same limitation applies to **retries** (§3, step 4). If our update
lands but the response times out, and a Salesforce user changes the
same field before our retry and before their webhook reaches us, the
retry overwrites their value. Rule 1 runs before every retry, so it
catches this only if their webhook has already arrived.

## 8. Rule: duplicate and out-of-order webhooks
## 9. Rule: stopping the sync loop
## 10. The 6-hour outage and the arithmetic
## 11. Three numbers to watch and alert thresholds
## 12. Out of scope this weekend