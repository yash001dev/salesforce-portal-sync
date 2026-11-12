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
*If different (full record):* we diff the incoming record against our
stored copy and treat only fields whose value differs as changed. The
per-field ordering rule (§8) then still stops an old full-record
webhook from overwriting fields it never meant to touch.

**A3. We can ask Salesforce what changed since a point in time.**
*This is an API the brief does not give us.* We assume a query of the
form "records of type X modified since T", returning the current
values and Salesforce's own last-modified time per record. We use it
only to recover after an outage or a webhook gap.
*If different:* Salesforce edits made while webhooks were failing are
lost unless Salesforce itself retries them. We would have to say so to
the client and ask for a nightly full export to compare against.

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
The time is the Postgres transaction commit time on our server, never
a time sent by the browser. The actor is the logged-in user from the
session, not the `editedBy` field in the request body (the body is
only checked to match).
*If different:* if we had to trust a client time, a wrong browser
clock could make an old edit win.

**A6. Our clock and Salesforce's clock agree within 1 second.**
We compare our commit times with Salesforce's `occurredAt` when the
same field changes on both sides. Both are assumed NTP-synced.
*If different:* with larger skew, the conflict rule for opposite-side
edits within that window is effectively arbitrary. We would then need
to order by Salesforce's own modified time, returned on each of our
writes, rather than by our clock.

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

### 1.2 Questions we would ask before locking the design

1. What are the real peak and quiet-hour edit rates on each side? (A1)
2. Does the webhook carry changed fields only, or the whole record? (A2)
3. Can we query Salesforce for records modified since a time, and does
   that query count against the 100 req/s cap? (A3)
4. When two portal users edit the same field, is rejecting the stale
   save acceptable to finance and customer-success staff? (A4)
5. Who is the source of truth for clock time, and how much skew should
   we tolerate? (A5, A6)
6. Can we have a dedicated integration user and an external-ID field
   on Customer and Contact? (A7, A9)
7. How long and how often does Salesforce retry a failed webhook? (A8)
8. If both sides change the same field within a second or two, does
   the client want a rule ("Salesforce wins for owner, portal wins for
   billing city"), or is "latest wins, history kept" enough?
9. Does the Salesforce API support updating several records in one
   request? If so, how many records does a batch count as against the
   cap?
10. How long must history be kept, and who may read it?
## 2. Systems, who calls whom, network boundary, interfaces
## 3. Subproblems, in build order
## 4. Data we store
## 5. Inbound path: webhook → Postgres
## 6. Outbound path: portal edit → Salesforce API
## 7. Rule: both sides change the same record
## 8. Rule: duplicate and out-of-order webhooks
## 9. Rule: stopping the sync loop
## 10. The 6-hour outage and the arithmetic
## 11. Three numbers to watch and alert thresholds
## 12. Out of scope this weekend