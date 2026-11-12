# Design: Two-way sync of customer and contact records

## 1. Assumptions and open questions
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