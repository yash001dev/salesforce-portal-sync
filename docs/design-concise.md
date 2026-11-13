# Two-way sync of customer and contact records: concise design

This is the primary design document. Every decision here is taken from
the full design, [design.md](design.md), which has the reasoning, the
complete assumption statements and the step-by-step arithmetic.
Worked examples are in [examples.md](examples.md). Section numbers
(§) refer to design.md.

## 1. Assumptions and open questions

| ID | Assumption | If it is wrong |
|---|---|---|
| A1 | Peak 10 portal edits/s (1/s quiet), 20 Salesforce edits/s. Only portal edits use the 100 req/s cap | outage drain time scales with the rates |
| A2 | A webhook carries changed fields only | full records: values show *whether* a field changed; age still decides *whether* it applies |
| A3 | We can list records changed since a time (Salesforce Get Updated; not in the brief) | catch-up falls back to Change Data Capture, then retries, then a nightly compare, or we say the outage loses Salesforce edits |
| A4 | Two portal users, same field: the stale save is rejected (`409`) | silent last-writer-wins; history still keeps both |
| A5 | Portal actor from the session, time from our server (display only); order by a per-field version | nothing orders by our clock |
| A6 | We never compare our clock with Salesforce's | sides could disagree within the skew |
| A7 | A dedicated integration user writes to Salesforce | own writes recognised by write ID or value only |
| A8 | Salesforce retries webhooks for a limited time | catch-up runs more often |
| A9 | Creates are upserts by an external ID (not in the brief) | a timed-out create can leave a duplicate |
| A10 | Webhooks are HMAC-signed | at least an IP allowlist |
| A11 | We can read a record by ID, with its last-modified time (not in the brief) | unclear conflicts cannot be settled by asking Salesforce |
| A12 | Salesforce field history is not available to us | we could recover more lost values |
| A13 | Webhooks arrive within a few seconds, 20 s at worst | inbound 30 s cannot be met by us |
| A14 | 80 req/s; live first, backlog ≥ 50%, recovery ≥ 20% of the current rate; 5 s timeout; 10 s lease; 429 halves the rate, +5 req/s per minute back, `Retry-After` respected; 5 inbound attempts. All configurable | budget numbers change, rules do not |
| A15 | We can get Salesforce's validation rules; the portal returns `422` for known ones | more `rejected` fields, nothing lost |
| A16 | A `Portal_Write_Id__c` field is set on every write and appears in its webhook (not in the brief) | Option B: match own writes by value, leave `sf_time` NULL when unsure |
| A17 | A Salesforce update is all-or-nothing per record | rejection must say which fields were saved |
| A18 | `occurredAt` orders one record's changes but may tie (maybe 1 s precision) | the sweep becomes the safety net |
| A19 | Batch reads may exist (real Salesforce: up to 2,000 per POST; brief: not given) | use the outage estimate without them |

**Questions for the client, most important first:** Can we list
changed records (A3), read by ID (A11) and read in batches, and does a
batch count as one request (A19)? Can we add `Portal_Write_Id__c`
(A16) and an external-ID field (A9)? Can Change Data Capture be
enabled? Is there a conditional update ("only if not modified
since…")? Are the 5 million records already linked on both sides (if
not, loading them through the API at 80 req/s would take 5,000,000 ÷
80 = 62,500 s ≈ 17 h, so a bulk import is needed)? What are the real
edit rates, webhook delivery times, and does a resend keep its
`eventId`? All 18 questions are in §1.2.

## 2. Systems, who calls whom, interfaces

One service, two processes, one Postgres database. The processes share
nothing but Postgres; `NOTIFY` only wakes the worker early, and a 1 s
poll is the backup.

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

| Caller → callee | Path | Public internet |
|---|---|---|
| Salesforce → API process | webhook, HTTPS | yes |
| Portal user → API process | portal save, HTTPS | yes |
| Worker → Salesforce | record API, HTTPS, rate-limited | yes |
| API process ↔ worker | Postgres `inbox` and `outbox` tables | no |

**Interfaces.**
- **Webhook** `POST /webhooks/salesforce` (signature header): `200`
  stored or duplicate; `401` bad signature; `400` malformed only
  (invalid JSON or a missing required key; unknown fields are
  ignored, not refused); `503` database down, so Salesforce resends.
- **Portal save** `PATCH /records/:type/:id`, with `changedFields`
  and the version of each field the user loaded: `200` new versions;
  `409` stale field, with its current value and who set it; `422`
  known Salesforce rule broken.
- **Salesforce record API:** update (changed fields only), create as
  upsert by external ID (A9), read (A11), changed-since (A3).
  `2xx` done; `429` slow down and keep the work; timeout or `5xx`
  retry; other `4xx` rejected or blocked, no retry.

**No Kafka or Redis.** Peak is about 30 events/s and an outage leaves
about 216,000 outbox rows. The outbox only works because the queue
entry and the data are written in one Postgres transaction.
`FOR UPDATE SKIP LOCKED`, `LISTEN/NOTIFY` and unique keys give us the
queue. The rate limiter lives in the single worker's memory.

## 3. Subproblems, in build order

Riskiest first: what is hardest to change later, what we do not
control, then what spends the scarce resource. Each step adds its own
measurement when it is built.

0. **Initial load** (open question): are the 5 million records already
   linked?
1. **Per-field state and append-only history.** Every later rule
   compares against it; history added later would mean lost changes.
2. **Inbound:** verify, store in `inbox` by `eventId`, reply `200` only
   after the commit, then apply by the age rule. Applying never writes
   to the outbox (the loop rule, from day one).
3. **Outbound:** a portal save writes the change, history and an outbox
   row in one transaction; the worker sends. The only way pending calls
   survive a restart.
4. **Rate limiter and retries:** the first thing that can harm
   Salesforce.
5. **Recognising our own write** when its webhook comes back.
6. **Conflict rules** (§7 below).
7. **Recovery:** sweep, outage catch-up, backlog drain, all through
   the same apply path as webhooks.
8. **Alert thresholds.**

### The 30-second target

The brief's 30 s worst case holds **under these conditions**: C1 load
at or below A1 (in normal running live edits have ≥ 64 req/s, so at
most ~320 are ahead); C2 Salesforce answers within the 5 s timeout and
an edit is accepted within the three attempts that carry it; C3 no
429 in progress; C4 no worker crash; C5 webhooks delivered within
20 s. Outside them we detect a miss rather than promise 30 s.

| Outbound step | Worst case | Inbound step | Worst case |
|---|---|---|---|
| Save + history + outbox commit | 0.1 s | Salesforce delivers (C5, not ours) | 20 s |
| Worker notices | 1 s | Verify, store, reply | 0.2 s |
| In-flight attempt on the record finishes | 5 s | Worker notices and applies | 1 s |
| Queue wait (C1) | 5 s | Read if order unclear | 5 s |
| Attempts: 5 + 1 + 5 + 2 + 5 (C2) | 18 s | | |
| **Total** | **29.1 s** | **Total** | **26.2 s** |

Typical case: 2–3 s outbound; delivery plus about 1 s inbound. What
breaks it: a burst of more than ~320 live edits, slow answers or more
failures, a 429, a worker crash (the 10 s lease adds up to about 39 s
in total), or late webhooks.

## 4. The data we store

Seven Postgres tables.

| Table | Holds |
|---|---|
| `customer` | `id` (ours), `sf_id`, `name`, `status`, `owner`, `is_placeholder` |
| `contact` | `id`, `sf_id`, `full_name`, `phone`, `email`, `city`, `owner`, `customer_id` (our customer ID) |
| `field_state` | per (record, field): `version`, `sf_time` (NULL allowed), `sync_state` (`pending` / `synced` / `rejected`), `base_sf_time`, `conflict`, `sent_version`, `sent_write_id` |
| `field_history` | append-only: `old_value`, `new_value`, `actor`, `source`, `sf_time`, `recorded_at`, `version`, `event_id`, `outcome` |
| `inbox` | `event_id` (primary key), `payload`, `received_at`, `applied_at`, `attempts`, `last_error`, `failed_at` |
| `outbox` | one row per record and kind (`write` / `read`): `priority` (`live` / `backlog` / `recovery`), `state` (`pending` / `in_flight` / `blocked`), `again`, `sent` (versions, values, write ID), `attempts`, `next_attempt_at`, `lease_until`, `last_error` |
| `sync_cursor` | how far the sweep and catch-up have read |

**History rules.** Each portal save writes its own history rows in the
same transaction. The outbox holds no values, so merging several saves
into one call cannot drop any history. The service's database role
can only `INSERT` and `SELECT` history. Each row's `outcome` is fixed
when written: `applied`, `superseded` (older Salesforce edit, order
reliable), `order_unknown`, `conflict` (a Salesforce value that
replaced a pending portal edit), or `rejected`.

**Placeholders.** A contact naming a customer we do not have creates a
placeholder customer (`is_placeholder`; required fields may be NULL
only on it), with a read queued. It cannot be edited or sent until the
read fills it. A contact whose customer has no `sf_id` yet waits for
the customer's create.

## 5. Inbound path: webhook to our database

Store first, then apply, in one transaction per event, under a row
lock. Per field:
- **our own write** (write ID and value match): fill in `sf_time`, no
  history row;
- **newer than `sf_time`:** applied;
- **older, `sf_time` known:** `superseded`, current value kept;
- **tie, or `sf_time` NULL, with a different value:** read the record;
- **field pending in the portal:** conflict rule 1 (§7 below).

A record we do not have is created. An event that fails 5 times gets
`failed_at` and an alert, and stays in the inbox; other events carry
on, because the age rule does not need events in order. Once fixed,
`npm run replay -- <eventId>` (or `--all-failed`) applies it safely.

## 6. Outbound path: portal edit to Salesforce

The save checks known Salesforce rules (`422`) and versions (`409`),
sets fields `pending`, writes history and ensures one outbox row. The
worker claims a row by priority, takes a rate slot, snapshots the
current pending values with versions and a new write ID, and leases
the row for 10 s. It re-checks conflict rule 1, and reads first if the
row is marked read-before-write. It calls Salesforce outside any
database transaction.

| Response | What we do |
|---|---|
| `2xx` | fields still at the sent version become `synced` with `sf_time` NULL; row deleted, or pending again if `again` |
| `429` | keep the row; halve the rate, respect `Retry-After` |
| timeout, `5xx` | 2 quick retries (1 s, 2 s), then 4, 8 … 60 s; each a fresh snapshot and write ID |
| `4xx` naming a field | that field `rejected`, user shown the message; other fields sent again at once |
| `4xx` naming no field | row `blocked`, fields stay `pending`, alert; an operator unblocks it. Never split field by field, never undo the portal value |

Retries are not fully safe: if a timed-out update actually landed and a
Salesforce user changes the field before our retry and before their
webhook arrives, the retry overwrites it.

## 7. Both sides change the same record

- **Different fields both survive:** we sync per field.
- **Which edit is current:** on a webhook, `occurredAt` against
  `field_state.sf_time` (both Salesforce times); on a portal save,
  `field_state.version`.
- **Rule 1, don't send a stale edit.** If a Salesforce value newer
  than the edit's `base_sf_time` arrives for a pending field, the
  portal edit was made without seeing it: it is not sent, the
  Salesforce value becomes current (`conflict` row), and the user is
  notified. Checked again before every send.
- **Rule 2, the API response means synced.** `sf_time` stays NULL
  until our own webhook fills it in.
- **Rule 3, unclear order means read.** A different value for a field
  whose `sf_time` is NULL: read the record and take Salesforce's
  current value.

**Limitation.** A read restores agreement, not history: it shows
Salesforce's current value, not the order or values in between. If a
Salesforce edit's webhook never arrives and our write overwrites it,
that value is not in our history. A conditional update, Change Data
Capture or field history would close this; the brief gives none of
them.

## 8. Duplicate and out-of-order webhooks

- **Duplicate:** the inbox key is `eventId`; a second delivery inserts
  nothing and gets `200`. (If a resend ever had a new `eventId`, the
  values stay right; only a duplicate history row is the cost.)
- **Out of order:** per field, newer `occurredAt` applies; older with
  a known `sf_time` is `superseded`; equal with the same value changes
  nothing; equal with a different value, or `sf_time` NULL, means a
  read (A18). We never pick a winner when the order is unclear.

## 9. Stopping the loop

1. **No path from inbound to outbound:** applying a webhook never
   writes to the outbox, and nothing else saves records in reaction to
   a change. This alone stops loops.
2. **Our own write** counts only if the write ID **and** value match.
   A mistake here costs a read, never a loop.
3. **Salesforce automation** that changes a field in our write's save
   arrives with our write ID but a different value, so it is a
   Salesforce change and goes through rule 1.

## 10. The 6-hour outage

**Scenario** (illustrative, not a worst-case guarantee): 6 h =
21,600 s at peak, every edit to a different record, 5% of calls
retried once, independent records on the two sides. Webhooks sent in
the window are lost.

**On disk:** 10 × 21,600 = **216,000** portal edits with history,
≤ 216,000 outbox rows, pending fields with the value each user saw,
and `sync_cursor`. A restart loses nothing. After 1 minute of failed
calls the worker stops and tests every 30 s.

**Recovery, in order.**
- **A. Find what Salesforce changed first.** Get Updated from
  `sync_cursor` − 1 min, 10-minute windows: 36 × 2 types = **72
  calls**, about 2 minutes. Result C, up to 20 × 21,600 = **432,000**
  records.
- **B. Drain the backlog.** Records not in C are sent. Records in C
  are read first: a field whose Salesforce value still equals what the
  user saw is sent; one that differs means Salesforce wins (`conflict`
  row, user notified).
- **C. Read and apply the rest of C.**
- **Live edits** go first. Until step A finishes every live edit is
  read before it is written; afterwards only records in C are. Get
  Updated reruns every 5 minutes with a 1-minute overlap. This reduces
  the blind window, but does not remove it.

**Calls:** 72 + 216,000 writes + 18,662 overlap reads (216,000 ×
432,000 ÷ 5,000,000) + 413,338 other reads = **648,072**.

**Rate.** Useful calls = rate ÷ 1.05. Live needs (10 + 10) × 1.05 =
21 req/s during step A (backlog and recovery are idle then), and
(10 + 0.86) × 1.05 ≈ 11.4 req/s after it, inside its guaranteed 30%
(24 at 80). Below about 11.4 ÷ 0.3 ≈ 38 req/s live edits queue and
miss 30 s. Then backlog = (80 − 11.4 − 16) ÷ 1.05 = **50.09/s**,
recovery = 16 ÷ 1.05 = **15.24/s**, and either alone gets
(80 − 11.4) ÷ 1.05 = **65.33/s**.

> time = calls ÷ ((current rate − live use) ÷ (1 + retry share))

**Without batch reads:** backlog 234,662 ÷ 50.09 ≈ 4,685 s ≈ **78
min**; meanwhile recovery does 15.24 × 4,685 ≈ 71,390 reads; the
remaining 341,948 ÷ 65.33 ≈ 5,234 s ≈ **87 min**; drain 78 + 87 ≈
**165 min**.

**With batch reads** (A19, 200 per call, illustration only): 2,160
read calls; recovery 2,067 ÷ 15.24 ≈ 136 s; backlog 6,794 done in that
time, then 209,299 ÷ 65.33 ≈ 3,204 s; drain ≈ 3,340 s ≈ **56 min**.

| | Without batch reads | With batch reads |
|---|---|---|
| Initial reconciliation | ~2 min | ~2 min |
| Portal backlog in Salesforce | ~78 min | ~56 min |
| Salesforce changes in the portal | ~165 min | ~2–3 min |
| **Total elapsed** | **~167 min** | **~58 min** |

These are estimates that scale with edit rates and outage length, and
roughly double if Salesforce throttles us to half rate.

## 11. Three numbers to watch

| Metric | Warning (team channel, ticket) | Critical (page on-call) |
|---|---|---|
| Oldest pending live portal edit | > 20 s for 1 min | > 30 s |
| Oldest unapplied inbound event | > 10 s for 1 min | > 30 s, or any `failed` event |
| Salesforce error and throttle rate (429, 5xx, timeouts, last 5 min) | > 5% | > 25%, or outage mode |

Critical age alerts have no hold time, because the age already shows
how long something has waited. In a known outage: one incident, no
repeated pages, an hourly backlog and drain estimate, and a warning if
the backlog stops shrinking for 10 minutes once recovery starts.
Blocked rows, rejected fields, failed events, conflicts and old
placeholders stay visible in the portal and on `GET /ops/sync-issues`.

## 12. Out of scope this weekend

As the brief allows: deletes, Salesforce login screens, a visual UI, a
second region, a load test against real Salesforce, a shippable
service. Our choice, described but not built: the initial 5-million
record load, Change Data Capture, batch writes, several worker
machines, Postgres high availability, history retention, secret
rotation, new fields or objects, and an operator screen. A running
program is optional; if built, it follows §3 against a fake
Salesforce.
