# Worked examples

These examples belong to [design.md](design.md). Each one shows a rule
from the design applied to one record, step by step.

## E1. One contact through the data model

From [§4 Data we store](design.md#4-data-we-store).

Contact `c-42`, Salesforce ID `0031A00000Abc`. Start: city Mumbai
(version 3, `sf_time` 11:00:00, synced); phone with `sf_time`
10:00:00.

| # | Event | What is saved |
|---|---|---|
| 1 | Finance user sets city to **Noida** (11:30:02) | `contact.city` = Noida. `field_state` city: version 4, `pending`, `base_sf_time` 11:00:00. History #101: Mumbai → Noida, finance user, portal, `applied`. Outbox row created (write, live, pending) |
| 2 | Same user sets city to **Gurgaon** before anything is sent | version 5. History #102: Noida → Gurgaon, `applied`. Outbox row already pending: unchanged. *One future call, two history rows.* |
| 3 | Worker claims the row | Outbox: `in_flight`, `sent` = {city: v5 Gurgaon}, write ID W1, lease. Sends city = Gurgaon, `Portal_Write_Id__c` = W1 |
| 4 | Call times out | Outbox: attempts 1, next attempt in 1 s. *No history row: a retry is not an edit.* |
| 5 | Retry | Rule 1 re-checked. Fresh snapshot: still v5 Gurgaon; write ID W2. **Success.** city: `synced`, `sf_time` NULL, `sent_version` 5, `sent_write_id` W2. Outbox row deleted |
| 6 | Webhook `evt_1001`: sales user sets phone (11:30:04) | Inbox row; `200`. Applied: phone's `sf_time` 10:00 is older, so phone updates. History #103: old → new phone, sales user, salesforce, `sf_time` 11:30:04, `evt_1001`, `applied`. *City untouched.* |
| 7 | `evt_1001` again | Inbox insert does nothing; `200`. Nothing else written |
| 8 | Webhook for attempt 1 (write ID W1, 11:30:08) | W1 ≠ `sent_write_id` W2: no state change, no history row. Kept in inbox |
| 9 | Webhook for attempt 2 (W2, 11:30:09) | W2 matches, city is still version 5 and holds the value sent: `sf_time` = 11:30:09. No history row |
| 10 | Late webhook `evt_0999`: sales user set city to Pune (11:29:50) | `sf_time` 11:30:09 is known and later: History #104: Pune, sales user, `sf_time` 11:29:50, **`superseded`**, `old_value` NULL. Current value stays Gurgaon |

End state: the contact shows Gurgaon and the new phone; history has
Noida, Gurgaon, the phone change and Pune, each with who and when; the
outbox is empty; the inbox has four rows (the duplicate added none).

## E2. Inbound cases

From [§5 Inbound path](design.md#5-inbound-path-webhook--postgres).
Contact `c-77`.

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

## E3. Outbound responses

From [§6 Outbound path](design.md#6-outbound-path-portal-edit--salesforce-api).
Contact `c-42`.

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

## E4. Same field on both sides

From [§7 both sides change the same record](design.md#7-rule-both-sides-change-the-same-record).

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
