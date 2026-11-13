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
*If different:* we recognise our own echoes only by the write ID
(A16), or, without it, by matching the webhook against what we sent
(§9). This still works but is
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
A3, and to settle a conflict when we cannot tell the order (§7). We
also assume the read returns the record's last-modified time, so the
values we apply from it get an `sf_time` (without it they stay NULL,
which is safe but costs later reads). Each read counts against the
100 req/s cap.
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
Send at most **80 req/s** (20 below the cap, for safety). Outbox work
has three priorities: **live** (user edits made while Salesforce is
up), **backlog** (user edits made during an outage) and **recovery**
(reads, sweep, bulk changes). Live is served first. **Backlog is
guaranteed at least 50% and recovery at least 20% of the *current*
rate** (so 40 and 16 req/s at 80; 20 and 8 req/s after a 429 halves it
to 40). Capacity one priority does not use is borrowed by the others.
Each
Salesforce call has a **5-second timeout**. On HTTP 429 the limit is
**halved** (80 → 40 req/s) and then raised by **5 req/s per minute**
without another 429, back up to 80. A `Retry-After` header is always
respected. Inbound events are tried **5 times** before they are marked
failed. These are configuration values, not constants in code.
*If different:* the budget in §3 changes; the rules do not.

**A17. A Salesforce update is all-or-nothing per record.**
If one field fails validation, none of the fields in that request are
saved, and the error usually names the field.
*If different (partial saves):* a rejection response would also have
to tell us which fields were saved; we would mark those `synced` and
treat the rest as below (§6).

**A18. `occurredAt` orders changes to the same record, but may tie.**
It comes from Salesforce's single clock, and for one record a later
change has a later or equal `occurredAt`. We do **not** assume it is
more precise than one second (the sample ends in `.000`), that it is
exactly the save time, or that it can be compared with our clock
(A6). When it cannot decide (a tie with different values, or
`sf_time` NULL) we read the record (§8).
*If different (changes to one record can be stamped out of order):*
the age rule would sometimes keep the wrong value; the periodic sweep
(A3) comparing against Salesforce's current values becomes the safety
net. Change Data Capture (A3, fallback 1) carries a commit sequence
number and would make ordering exact.

**A19. We may be able to read many records in one call.**
*Not in the brief.* Real Salesforce documents an sObject Collections
retrieve (up to 2,000 records of one type per POST, about 800 per GET
because of URL length), and counts it as one call toward its API
limits. We do **not** know whether the client's 100 req/s cap counts
it the same way, or what batch size is practical for them. §10 shows
the outage both with and without it, using a batch size of **200 for
illustration only**.
*If different:* the outage estimate without batch reads applies.

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

**A16. A write ID on every update we send.**
*Assumption beyond the brief:* Customer and Contact have a custom
field `Portal_Write_Id__c`. Every update or upsert we send sets it to
a new unique ID. Because the field changes on every write we make,
Salesforce includes it in `changedFields` of the webhook for that
write. This is a field we add to the webhook payload, and the reason
is exact matching: the webhook for our own write tells us *which* of
our writes it is, even when several writes to one record happen close
together or the same value is sent twice. Sales users never change
this field, so their webhooks do not carry it. It is never shown in
the portal or treated as a synced field.
*If different (Option B):* we match our own webhook by integration
user (A7) and value. If a field is sent again before the webhook for
its previous write arrived, the match is ambiguous, so we leave that
field's `sf_time` NULL. NULL is always safe: a later different value
then costs one read (§7 rule 3).

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
16. Can the client add a `Portal_Write_Id__c` field to Customer and
    Contact, and will its changes appear in webhooks? (A16)
17. When Salesforce resends a webhook, does it keep the same
    `eventId`? How precise is `occurredAt`, and is it the time the
    change was saved? (§8, A18)
18. Can we read many records in one call, and does the client's
    100 req/s cap count such a call as one request? (A19)

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
2. Verify the signature (A10); check the required keys. Fields we do
   not sync are ignored, not refused (§5).
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
tried again, up to 5 times; then it is marked failed and kept for
replay (§5).

### 2.3 Outbound: portal save → Salesforce

API process, in **one transaction**:
1. Actor from the login session (A5).
2. Check known Salesforce rules (A15); a failure returns `422` and
   nothing is saved.
3. Lock the record row; version check per field (A4); a stale field
   returns `409`.
4. Set the fields to `pending`, write history rows, and make sure the
   record has an outbox row (create it, or set `again` if it is in
   flight). The outbox row holds no values.
5. Placeholder customers (§4) cannot be edited until they are
   complete; the API returns `409`.

Commit, then `NOTIFY`.

Worker:
1. **Claim (short transaction).** Pick a pending outbox row; user
   priorities and shares as in A14. Skip a contact whose
   customer has no Salesforce ID yet (it waits for the customer's own
   create). Take a rate-limit slot. Read the record's **current**
   pending fields and store exactly what will be sent on the row:
   `sent = {field: {version, value}}` and a new write ID (A16). Mark
   the row in flight with a lease ("mine until now + 30 s"). Commit.
2. **Call Salesforce outside any transaction.** We never hold a
   database transaction open across an HTTP call. Re-run §7 rule 1
   first. The call sets the sent fields and `Portal_Write_Id__c`.
3. **Record the result (new transaction).**
   - Success: each sent field whose version still equals its sent
     version becomes `synced`, with `sf_time = NULL` and the write ID
     stored. A field edited during the call stays `pending`. If
     `again` is set, the row goes back to pending; otherwise it is
     deleted.
   - Timeout, `5xx`, `429`: attempt count + 1, next attempt time set,
     lease released. The next attempt takes a fresh snapshot and a
     new write ID.
   - Other `4xx`: named fields become `rejected` and the rest are
     sent again; if no field is named, the row becomes `blocked`
     (§6).

The webhook for our own write later arrives on the inbound path and,
if its write ID matches, fills in `sf_time` (§3 step 5).

### 2.4 Why inbox and outbox are separate

| | inbox | outbox |
|---|---|---|
| Holds | what Salesforce told us | what we must tell Salesforce |
| Key | `eventId` (unique: catches duplicates) | record: one row per record, holding no values (merges edits) |
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
signature · `400` malformed (invalid JSON, or a required key such as
`eventId` or `recordId` missing) · `503` database down, please retry.
Unknown fields are not a reason for `400`.

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
`5xx` retry · other `4xx` rejected or blocked, no retry (§6).
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
ordering); placeholder customers older than 5 minutes (§4).

### Step 3: outbound (portal save → outbox → worker)
**What:** a portal save writes the field change, the history row and
an `outbox` row in **one Postgres transaction**. A worker process
sends outbox rows to Salesforce.
- At most one call per record in flight, so two edits to one record
  cannot reach Salesforce in the wrong order.
- One outbox row per record. It holds no values: the worker reads the
  record's current pending fields when it sends, so several saves
  merge into one call automatically. A save during an in-flight call
  sets `again`, and the row goes back to pending once the call ends.
  The worker records exactly what it sent (§2.3). Merging never
  touches history: each save writes its own history rows (§4).
- Three priorities: **live**, **backlog** and **recovery**, with
  guaranteed shares (A14, §10).
- Creates are sent as upserts by external ID (A9).
**Why here:** it is the only way pending calls survive a restart, and
the rate limiter (step 4) needs something to control.
**If skipped:** writing to Postgres and calling Salesforce directly
means a crash between the two loses the edit, or sends it without
saving it; during an outage pending calls live only in memory.
**Measures:** age of the oldest pending live edit (the 30-second
number); outbox size per priority.

### Step 4: rate limiter and retries
**What:** one shared limit of 80 req/s for every call we make,
including retries, reads (A11) and catch-up queries (A3). Recovery is
guaranteed at least 16 req/s when it has work (A14).
On timeout or failure: up to **2 quick retries** (after 1 s, then 2 s).
After that the row stays in the outbox and retries with a growing wait
(4 s, 8 s … up to 60 s), marked late. Nothing is dropped. On HTTP 429
we halve the rate, respect `Retry-After`, and keep the work (A14).
**Permanent rejection.** A `4xx` other than 429 (for example a
Salesforce validation rule refusing a phone format) will not succeed
on retry. If the error names the field, that field becomes `rejected`,
the portal shows the user Salesforce's error so they can correct it,
and the other fields are sent again without it. If the error names no
field, we do not guess: the outbox row becomes **`blocked`**, all its
fields stay `pending`, and we alert for investigation (§6). We never
undo a portal value automatically.
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
as permissions or a rule that depends on other data); blocked outbox
rows (alert on any).

### Step 5: recognising our own write when it comes back
**What:** the webhook for our own write is matched by its write ID
(A16). For each field still at the version we sent with that ID and
holding the value we sent, it fills in `sf_time` (NULL since the API response, §7 rule 2). A webhook
with an older write ID changes no state. It is kept in the inbox and
never sent back. Without the write ID we use Option B (A16): match by
value, and leave `sf_time` NULL when unsure.
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
**What:** the drain is the step 3 worker at the step 4 limit, with the
backlog priority and the read-before-write check (§10). Catch-up and
the sweep use A3 and A11 and feed results through the **same apply
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
| Queue wait | ≤ 5 s | live edits are served first; at ≥ 64 req/s up to ~320 live edits ahead |
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
| More user edits queued than we can send in time | oldest pending live edit: warn at 20 s, page at 30 s |
| Salesforce answers slowly | call duration p95: warn above 2 s |
| Timeouts or HTTP 429 push past the quick retries | timeouts and 429s per minute |
| Salesforce delivers webhooks late | webhook delivery lag: warn above 10 s |
| Draining the backlog after an outage | separate target (§10); backlog size over time |

After an outage, edits made during it go to the **backlog** priority;
new edits go to **live** and are served first, so they return to the
30-second target almost at once (§10). A new edit to a record already
in the backlog moves that record to live. If the record is one
Salesforce changed during the outage, it still gets its one read
before the write (§10).
## 4. Data we store

Seven tables, all in Postgres.

```
customer ─┐                      ┌─ field_state    sync state, one row per (record, field)
contact  ─┼── record_id (uuid) ──┼─ field_history  append-only, one row per change
          │                      └─ outbox         one row per record with something to send
          └─ contact.customer_id → customer.id
inbox        raw webhooks, keyed by eventId
sync_cursor  how far the sweep / catch-up has read
```

### 4.1 Tables

**`customer`**: current values the portal reads.
`id` (uuid, ours), `sf_id` (unique, NULL until created in
Salesforce), `name`, `status`, `owner`, `is_placeholder` (bool).

**`contact`**: current values.
`id`, `sf_id`, `full_name`, `phone`, `email`, `city`, `owner`,
`customer_id` (foreign key to `customer.id`).
The link uses *our* customer ID, which always exists. The worker sends
it to Salesforce as the customer's `sf_id`; a contact whose customer
has no `sf_id` yet waits until the customer's create succeeds (§2.3).

**`field_state`**: the per-field state from §7, one row per
(record, field). Key `(record_id, field)`.
`version`, `sf_time` (NULL allowed), `sync_state` (`pending` /
`synced` / `rejected`), `base_sf_time`, `conflict` (bool),
`sent_version`, `sent_write_id`.
This is a table rather than jsonb on the record row so that the
measurements in §3 are plain indexed queries. At about 7 fields per
record that is about 35 million small rows. The record's row lock
still covers it.

**`field_history`**: every change, never updated or deleted.
`id`, `record_id`, `field`, `old_value`, `new_value`, `actor`,
`source` (`portal` / `salesforce` / `read`), `sf_time`, `recorded_at`
(`clock_timestamp()`, display only), `version`, `event_id`, `outcome`.
Index `(record_id, field, id)`.

**`inbox`**: what Salesforce sent us.
`event_id` (primary key: catches duplicates), `payload` (jsonb),
`received_at`, `applied_at`, `attempts`, `last_error`, `failed_at`
(set after 5 failed attempts; the row is kept and can be replayed,
§5).

**`outbox`**: "this record has something to send". Key
`(record_id, kind)`, where `kind` is `write` or `read`.
`priority` (`live` / `backlog` / `recovery`), `state` (`pending` / `in_flight` /
`blocked`),
`again` (bool), `sent` (jsonb: `{field: {version, value}}` and the
write ID, set at claim), `attempts`, `next_attempt_at`, `lease_until`,
`last_error`. It holds no values while pending.

**`sync_cursor`**: `name`, `value`. Where the sweep and outage
catch-up have read up to, so a restart resumes there.

Not stored: a log of each retry attempt (the count is on the outbox
row); rate-limiter state (in the worker's memory); validation rules (a
config file, A15).

### 4.2 History rules

1. **Each portal save writes its own history rows**, one per changed
   field, in the same transaction as the save. The worker never
   writes portal history.
2. **The outbox holds no values, so merging has nothing to drop.** It
   only decides how many calls reach Salesforce.
3. **`field_history` is append-only.** The service's database role has
   `INSERT` and `SELECT` on it, never `UPDATE` or `DELETE`.
4. **`outcome` is decided when the row is written and never changes.**
   A later event adds a new row.

| `outcome` | Meaning | `old_value` |
|---|---|---|
| `applied` | became the current value | the value it replaced |
| `superseded` | a real Salesforce edit that arrived after a newer one; order reliable (§7) | NULL |
| `order_unknown` | a Salesforce edit whose order against ours we cannot establish (§7 rule 3) | NULL |
| `conflict` | a Salesforce value that replaced a pending portal edit (§7 rule 1) | the cancelled portal value |
| `rejected` | a value Salesforce refused; `new_value` is exactly what we sent (from `outbox.sent`) | NULL |

Our own write coming back from Salesforce adds **no** history row: it
is not a new edit. The raw webhook stays in the inbox.

One consequence for the client: when a user saves twice before the
first is sent, Salesforce receives only the latest value. Our history
has both; Salesforce's own history would show only the last.

### 4.3 Placeholder customers

A contact's webhook can name a customer we do not have yet. We create
a **placeholder** customer with only its `sf_id`, link the contact to
it, and queue a recovery read (A11) for the real values.

- `is_placeholder = true`. Required fields may be NULL only on a
  placeholder:
  `CHECK (is_placeholder OR (name IS NOT NULL AND status IS NOT NULL))`
  (and any other fields A15 makes required).
- A placeholder has no `field_state` rows and never gets an outbox
  write: there is nothing of ours to send.
- The portal shows it as "loading from Salesforce" and refuses edits
  to it (`409`) until it is complete.
- The read's values go through the normal apply path (source `read`),
  then `is_placeholder` becomes false.
- Measured: placeholders older than 5 minutes (§3 step 2).

### 4.4 Example: one contact

Contact `c-42`, Salesforce ID `0031A00000Abc`. Start: city Mumbai
(version 3, `sf_time` 11:00:00, synced); phone with `sf_time`
10:00:00.

| # | Event | What is saved |
|---|---|---|
| 1 | Finance user sets city to **Noida** (11:30:02) | `contact.city` = Noida. `field_state` city: version 4, `pending`, `base_sf_time` 11:00:00. History #101: Mumbai → Noida, finance user, portal, `applied`. Outbox row created (write, user, pending) |
| 2 | Same user sets city to **Gurgaon** before anything is sent | version 5. History #102: Noida → Gurgaon, `applied`. Outbox row already pending: unchanged. *One future call, two history rows.* |
| 3 | Worker claims the row | Outbox: `in_flight`, `sent` = {city: v5 Gurgaon}, write ID W1, lease. Sends city = Gurgaon, `Portal_Write_Id__c` = W1 |
| 4 | Call times out | Outbox: attempts 1, next attempt in 1 s. *No history row: a retry is not an edit.* |
| 5 | Retry | Rule 1 re-checked. Fresh snapshot: still v5 Gurgaon; write ID W2. **Success.** city: `synced`, `sf_time` NULL, `sent_version` 5, `sent_write_id` W2. Outbox row deleted |
| 6 | Webhook `evt_1001`: sales user sets phone (11:30:04) | Inbox row; `200`. Applied: phone's `sf_time` 10:00 is older, so phone updates. History #103: old → new phone, sales user, salesforce, `sf_time` 11:30:04, `evt_1001`, `applied`. *City untouched.* |
| 7 | `evt_1001` again | Inbox insert does nothing; `200`. Nothing else written |
| 8 | Webhook for attempt 1 (write ID W1, 11:30:08) | W1 ≠ `sent_write_id` W2: no state change, no history row. Kept in inbox |
| 9 | Webhook for attempt 2 (W2, 11:30:09) | W2 matches and city is still version 5: `sf_time` = 11:30:09. No history row |
| 10 | Late webhook `evt_0999`: sales user set city to Pune (11:29:50) | `sf_time` 11:30:09 is known and later: History #104: Pune, sales user, `sf_time` 11:29:50, **`superseded`**, `old_value` NULL. Current value stays Gurgaon |

End state: the contact shows Gurgaon and the new phone; history has
Noida, Gurgaon, the phone change and Pune, each with who and when; the
outbox is empty; the inbox has four rows (the duplicate added none).
## 5. Inbound path: webhook → Postgres

The steps are in §2.2. This section covers what each case does.

**Request checks.** Bad signature: `401`, nothing stored. Malformed
(invalid JSON, missing `eventId` / `recordId` / `objectType` /
`occurredAt`): `400`. Postgres down: `503`, so Salesforce resends.
**Fields we do not sync are ignored, not refused**: the raw webhook is
stored, the unknown field is counted, and the rest is applied.
Refusing would make Salesforce resend or drop the whole event.

**Applying, per field:**
- **Duplicate** `eventId`: the inbox insert does nothing; reply `200`.
- **Our own write** (write ID *and* value match, A16): fills in
  `sf_time`, no history row (§9).
- **Newer than the field's `sf_time`:** applied; history `applied`.
- **Older, order reliable:** history `superseded`; current value kept
  (§8).
- **Order unclear** (`sf_time` NULL, different value): read the record
  (§7 rule 3).
- **Field pending in the portal:** §7 rule 1.
- Applying never writes to the outbox (§9).

**Unknown records.** A webhook for a record we do not have creates it,
with its `sf_id`. A customer it names that we do not have becomes a
placeholder, with a recovery read queued (§4.3).

**Failed events are kept and can be replayed.** If applying an event
keeps failing (a bug, an unexpected value), it is retried with a
growing wait. After **5 attempts** it gets `failed_at`, an alert goes
out, and it stays in the inbox with its payload and `last_error`.
Failed events are never deleted. Other events keep being applied:
the age rule does not need events in order, so skipping one does not
corrupt the others.
**Replay:** once the cause is fixed, a command (`npm run replay --
<eventId>`, or `--all-failed`) clears `failed_at` and resets
`attempts`, and the worker applies the event normally. Replaying is
safe at any time: the age rule and the write-ID match decide the
outcome as if the event had just arrived late.

**Example** (contact `c-77`):

| # | Event | Result |
|---|---|---|
| 1 | `evt_2001` phone → X1, 10:00:00 | applied |
| 2 | `evt_2001` again | ignored, `200` |
| 3 | `evt_2003` phone → X3, 10:00:20 (arrives first) | applied, `sf_time` 10:00:20 |
| 4 | `evt_2002` phone → X2, 10:00:10 (arrives later) | history `superseded`; phone stays X3 |
| 5 | bad signature | `401`, nothing stored |
| 6 | `evt_2004` while Postgres is down | `503`; stored when Salesforce resends |
| 7 | `evt_2005` fails 5 times | `failed_at` set, alert; kept; replayed after the fix |
| 8 | `evt_2006` for an unknown contact and customer | contact created; placeholder customer; read queued |

## 6. Outbound path: portal edit → Salesforce API

The steps are in §2.3. This section covers what each response does.

**Before saving.** Known Salesforce rules (A15) are checked; a failure
returns `422` and nothing is saved. A stale field returns `409` (A4).

**Each Salesforce response:**

| Response | What we do |
|---|---|
| `2xx` | Fields still at the sent version become `synced` (`sf_time` NULL). Row deleted, or back to pending if `again` |
| `429` | Keep the row. Halve the rate, respect `Retry-After`, raise again slowly (A14) |
| Timeout, `5xx`, unreachable | Keep the row; 2 quick retries, then a growing wait up to 60 s. Each attempt takes a fresh snapshot and a new write ID; §7 rule 1 is re-checked first |
| `4xx` naming field(s) | Those fields become `rejected` (history row with exactly what we sent; the user sees Salesforce's message on the field). The **other fields are sent again at once** without them |
| `4xx` naming no field | **Blocked** (below) |

**One invalid field in a multi-field update.** A Salesforce update is
all-or-nothing per record (A17), so when it is rejected nothing in it
was saved. Nothing is lost on our side either: every value is in the
portal and in history.
- If the error names the field, only that field is `rejected`; the
  rest go through in the next call.
- If the error names no field (a whole-record rule, for example "a
  closed customer needs an owner"), **we do not guess** and do not
  send fields one by one, which could leave Salesforce holding a mix
  of the user's changes. The outbox row becomes **`blocked`**: no
  more retries, all its fields stay `pending` with their values and
  history, and an alert goes out. The portal shows the record as
  "not synced: Salesforce refused, being investigated".
- New portal edits to a blocked record are saved as usual and stay
  pending with it. An operator unblocks the row once the cause is
  understood; the next attempt sends the current pending values.
  Salesforce edits to those fields still follow §7 rule 1.

**Waiting on a customer.** A contact whose customer has no `sf_id` yet
is skipped at claim and tried again shortly; the customer's create is
its own outbox row.

**Example** (contact `c-42`):

| # | Event | Result |
|---|---|---|
| 1 | save email = `bad@` | `422`, nothing saved |
| 2 | save phone and city | `200`; both pending; one outbox row |
| 3 | send phone, city (W1) → `429` | row kept; rate 80 → 40 req/s |
| 4 | user also changes owner | owner pending; same outbox row |
| 5 | retry phone, city, owner (W2) → timeout | attempts 2; if W2 landed, its webhook changes nothing |
| 6 | newer Salesforce phone arrives | phone cancelled (§7 rule 1, `conflict`) |
| 7 | retry city, owner (W3) → `400` naming `City` | city `rejected`; owner sent alone (W4) |
| 8 | worker crashes after sending W4, before recording the result | lease runs out; W5 sends owner again (same value, harmless) → owner `synced`; only W5's webhook updates `sf_time` |
| 9 | Salesforce unreachable | waits up to 60 s; oldest-pending alert; if it lasts, §10 |
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
   cancel the send. The Salesforce value becomes current; its history
   row has outcome `conflict` and `old_value` = the cancelled portal
   value (whose own history row, written at save time, stays as it
   is). The field is flagged as a conflict for the portal user. The
   same check runs again just before the call is sent. If
   `base_sf_time` is NULL (the user edited after our own unconfirmed
   write), any Salesforce value for that field that the user had not
   seen counts as newer.
2. **The API response means synced.** When the update call succeeds,
   the field becomes `synced` with `sf_time = NULL`. When the webhook
   for our own write arrives (§9), matched by its write ID (A16), we
   fill in `sf_time` from its `occurredAt` and never send it back. Nothing else changes; the
   portal shows the edit as synced from the API response onward. If
   that webhook never arrives, `sf_time` simply stays NULL; the only
   cost is that a later different value for that field triggers
   rule 3.
3. **When the order is unclear, read the record.** If a webhook with a
   *different* value arrives for a field whose `sf_time` is NULL, we
   cannot tell whether it came before or after our write. We read the
   record (A11) and take Salesforce's current value. If the read
   shows the webhook's value, it is recorded as `applied`; otherwise
   it is recorded as `order_unknown` (§4).

**Late webhooks.** A webhook older than the field's current value is
still a real edit, so it goes into history. It is recorded as
`superseded` only when the order is reliable: the field's `sf_time` is
known (not NULL) and later than the webhook's `occurredAt`, both
Salesforce times. Otherwise rule 3 applies.

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

**Duplicate.** The inbox key is `eventId`. A second delivery inserts
nothing, applies nothing, writes no history, and gets `200`. This
assumes a resend keeps its `eventId` (§1.2 q17). If it did not, the
age rule below still prevents a wrong value (equal time, equal value
changes nothing); the only cost would be a duplicate history row.

**Out of order.** Each field of a webhook is decided on its own, by
comparing `occurredAt` with the field's `sf_time` (both Salesforce
times, A18):

| Case | Result |
|---|---|
| `occurredAt` newer | applied |
| `occurredAt` older, `sf_time` known | history `superseded`; current value kept |
| `occurredAt` equal, same value | nothing changes |
| `occurredAt` equal, different value | order unclear → read the record |
| `sf_time` NULL, different value | order unclear → read the record |
| field `pending` in the portal | §7 rule 1 first |

Updates that arrive before the create, for a record we do not have,
create it; the later create webhook's fields are `superseded`.

**When the order is unclear we do not pick a winner.** We read the
record (A11) and take Salesforce's current value. The webhook's value
goes into history as `applied` if the read shows it, otherwise
`order_unknown`. **A read gives us Salesforce's current value, not the
missing edit history:** it cannot tell us the true order of the
competing edits, or any value set and overwritten in between (§7).

## 9. Rule: stopping the sync loop

Three layers. The first is enough on its own.

1. **No path from inbound to outbound.** Applying a webhook never
   writes to the outbox. Only a portal save through the Portal API
   does, and there are no database triggers or portal automations
   that save records in reaction to a change. So no webhook, ours or
   anyone's, can cause a call to Salesforce.
2. **Recognising our own write.** A field counts as our own write only
   if the webhook carries its write ID (A16) **and** its value equals
   what we sent. It then fills in `sf_time` and writes no history.
   Getting this wrong never causes a loop; at worst it costs a read.
3. **Salesforce automation.** A Flow or trigger may change a field in
   the same save as our write (for example reformatting a phone
   number). Its change arrives with our write ID but a different
   value, so by layer 2 it is **not** our own write: it is a
   Salesforce change and goes through the normal rules (§8, §7). In
   particular, if a newer portal edit for that field is `pending`,
   §7 rule 1 applies; the write ID does not let it overwrite the
   pending edit blindly.

**Measures:** synced fields whose `sf_time` stays NULL for more than
5 minutes (our own webhooks not matching), and calls to Salesforce
per portal save, not counting retries (should stay at or below 1; a
rise would show a loop we did not foresee).
## 10. The 6-hour outage and the arithmetic

### 10.1 The scenario

The Salesforce API is unreachable for **6 h = 21,600 s**. Webhooks
Salesforce tries to send in that window are lost. The numbers below
are an **illustrative scenario, not a worst-case guarantee**:

- peak rates for the whole 6 hours: **10 portal edits/s, 20 Salesforce
  edits/s** (A1);
- every edit is to a different record (merging would only help);
- **5% of calls fail once and are retried** (assumption);
- portal and Salesforce users edit independent records (assumption,
  used only to estimate the overlap).

### 10.2 What is on disk during the outage

| In Postgres | After 6 h |
|---|---|
| Portal edits saved, with history rows | 10 × 21,600 = **216,000** |
| Outbox rows (one per record, no values) | ≤ **216,000** |
| Pending fields in `field_state`, with the value the user saw (the `old_value` of the edit's history row) | one per edited field |
| `sync_cursor` | the last time Salesforce was fully read (about the outage start) |

Every row was written in the same transaction as the user's save, so
restarting our processes loses nothing. The portal keeps working;
saves show "pending sync".

**Outage mode.** After 1 minute of failed calls the worker stops
sending and makes one test call every 30 s. When a test call succeeds,
all user rows in the outbox are tagged **`backlog`**, and recovery
starts.

### 10.3 Recovery, in order

**Step A: find what Salesforce changed (before sending anything).**
Get Updated (A3) from `sync_cursor` − 1 minute to now, for Customer
and Contact, in 10-minute windows: 36 windows × 2 types = **72
calls**. Result: the set **C** of records changed in Salesforce,
worst case 20 × 21,600 = **432,000**. Every outbox row whose record is
in C is marked **read before write**.

**Step B: drain the portal backlog safely.**
- Record **not in C**: Salesforce did not change it, so nothing can be
  overwritten. Send it.
- Record **in C**: read it first (A11 or A19), then per pending field:
  - Salesforce value = the value the user saw → Salesforce did not
    change this field → send ours;
  - different → Salesforce changed it unseen → **Salesforce wins**
    (§7 rule 1): a `conflict` history row, the user is notified, our
    edit is not sent and stays in history.

**Step C: bring in the rest of C.** Records changed in Salesforce but
not in the outbox are read and applied through the normal apply path
(recovery priority).

**Throughout: new edits and late changes.**
- New portal edits go to **live**, which is served first, so they get
  back to the 30-second target almost at once. A new edit to a backlog
  record moves that record to live. **If the record is in C, its
  read-before-write check still runs first.**
- Shares (A14) apply to the **current** rate: backlog is guaranteed
  50%, recovery 20%; unused capacity is borrowed by the others. So
  after a 429 halves the rate to 40 req/s, backlog keeps ≥ 20 and
  recovery ≥ 8.
- Salesforce webhooks flow again and are applied as usual; §7 rule 1
  runs before every send.
- **Get Updated runs every 5 minutes** during recovery, over the
  window since the last run with a 1-minute overlap (2 calls per run).
  New IDs join C. This **reduces** the risk of missing a Salesforce
  change made during the drain whose webhook is late or lost; it does
  **not** eliminate the blind window: a change made in the seconds
  before we send can still be overwritten (§7 limitation).

**What reconciliation cannot recover.** A read shows Salesforce's
current value, not the edits in between. If Salesforce changed a field
and changed it back (A → B → A), the read looks unchanged, we send our
value, and B is not in our history.

### 10.4 The calls needed

| Work | Calls |
|---|---|
| Get Updated (step A) | 72 |
| Portal writes from the outage | **216,000** |
| Reads for records in both the outbox and C | 216,000 × 432,000 / 5,000,000 ≈ **18,662** |
| Reads for the rest of C | 432,000 − 18,662 = **413,338** |
| **Total, one read per record** | **648,072** |

(If the overlap were larger, the total would not change; those reads
would just move from recovery to backlog.)

### 10.5 Rate available

- Budget 80 req/s; with 5% retries, useful = rate ÷ 1.05.
- Live edits keep arriving at 10/s → 10 × 1.05 = **10.5 req/s**.
- Left: 80 − 10.5 = 69.5 req/s. Recovery gets its guaranteed 16;
  backlog gets the rest, 53.5 (above its 40 guarantee).
- Useful: **backlog 53.5 ÷ 1.05 = 50.95/s**, **recovery 16 ÷ 1.05 =
  15.24/s**. When one finishes, the other gets all 69.5 ÷ 1.05 =
  **66.2/s**.

General form, for other numbers:

> **time = calls ÷ (current rate ÷ (1 + retry share) − live edits/s)**

### 10.6 Drain times

**Without batch reads (one read per record):**
- Backlog: (216,000 + 18,662) ÷ 50.95 = 234,662 ÷ 50.95 = 4,606 s
  ≈ **77 min**.
- Recovery while the backlog runs: 15.24 × 4,606 ≈ 70,179 reads done;
  413,338 − 70,179 = 343,159 left ÷ 66.2 ≈ 5,184 s ≈ **86 min** more.
- Salesforce's outage changes are all in the portal after
  77 + 86 ≈ **163 min**.

**With batch reads (A19, 200 per call, illustration):**
- Reads: 432,000 ÷ 200 = **2,160 calls** (≈ 93 of them for the
  overlap, in backlog).
- Recovery: ≈ 2,067 calls ÷ 15.24 ≈ 136 s ≈ **2.3 min**.
- Backlog: 136 s × 50.95 ≈ 6,910 calls done; (216,000 + 93 − 6,910)
  = 209,183 left ÷ 66.2 ≈ 3,160 s; total 136 + 3,160 ≈ 3,296 s
  ≈ **55 min**.

### 10.7 Elapsed recovery time

| | Without batch reads | With batch reads (illustrative) |
|---|---|---|
| (a) Initial reconciliation (step A: 72 calls, run in sequence, plus marking rows) | ~2 min | ~2 min |
| (b) Portal backlog drained into Salesforce | ~77 min | ~55 min |
| Salesforce's outage changes in the portal | ~163 min | ~2–3 min |
| **(c) Total elapsed, from Salesforce coming back** | **~2 + 163 ≈ 165 min (≈ 2 h 45 min)** | **~2 + 55 ≈ 57 min** |
| New edits after recovery starts | ~30 s target (+1 read if the record is in C) | same |

These are **illustrative estimates, not guarantees**. They grow in
proportion to edit rates and outage length, roughly double if
Salesforce throttles us to half rate during the drain, and fall to
about a tenth at quiet-hour rates. Without batch reads, recovery time
is driven by **reads**; with them, by **our portal writes**. Batching
writes (§1.2 q11) would shorten it further, but would change the
per-record error handling in A17, so we do not count on it.
## 11. Three numbers to watch and alert thresholds
## 12. Out of scope this weekend