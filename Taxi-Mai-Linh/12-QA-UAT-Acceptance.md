# 12 — KIỂM THỬ, NGHIỆM THU NGƯỜI DÙNG VÀ CHẤP NHẬN

## 1. Chiến lược kiểm thử

| Layer | Coverage | Evidence |
|---|---|---|
| Unit | State rules, eligibility, fare, settlement | Unit report |
| API/Contract | Validation, permission, idempotency, webhook | API report |
| Integration | Maps, GPS, push, payment, SMS/voice, ERP | Integration evidence |
| E2E | Booking → dispatch → trip → fare → payment → settlement | E2E report |
| Security | RBAC, scope, PII, OWASP, audit | Security report |
| Performance | Peak booking, concurrent dispatch, GPS events | Load report |
| UAT | Role-based DIRECT/PARTNER scenarios | Signed UAT |
| Operations | Restore, failover, alerts, runbooks | Ops sign-off |

## 2. Sample test cases

| TC | Module | Scenario | Expected result |
|---|---|---|---|
| TC-001 | Booking | Create valid booking | Unique ID and correct state |
| TC-002 | Booking | Missing pickup | Reject, field error, preserve input |
| TC-003 | Booking | Scheduled booking | Dispatch only at configured lead time |
| TC-004 | Dispatch | Eligible DIRECT vehicle | Assignment created |
| TC-005 | Dispatch | Expired PARTNER contract | Candidate blocked with reason |
| TC-006 | Dispatch | Concurrent assign | Only one active assignment |
| TC-007 | Driver | Reject offer | Reason logged; next candidate attempted |
| TC-008 | GPS | Stale location | Warning; no auto-offline mutation |
| TC-009 | Trip | Invalid transition | Server rejects; no state mutation |
| TC-010 | Driver | Offline event sync | Idempotent sync; timeline reconciled |
| TC-011 | Fare | Historical tariff | Saved tariff version used |
| TC-012 | Payment | Duplicate webhook | Single effective transaction |
| TC-013 | Payment | Invalid signature | Rejected; no SUCCESS |
| TC-014 | Settlement | Lock approved batch | Immutable; adjustment batch required |
| TC-015 | IAM | Agent calls finance API | Denied at UI and API |
| TC-016 | Audit | Fare adjustment | Before/after/reason/actor/approver recorded |
| TC-017 | CS | Create complaint | Owner/category/SLA/timeline present |
| TC-018 | Reporting | DIRECT/PARTNER filter | Totals reconcile to source |
| TC-019 | Exception | No eligible vehicle | Booking remains in exception queue |
| TC-020 | Reassign | Change assigned vehicle | Old assignment retained and audited |

## 3. UAT entry criteria

Approved requirements; prepared test environment/accounts; approved reference tariff and contract data; test-mode integrations; monitoring available; synthetic or approved masked data.

## 4. Exit criteria

- 100% MUST requirements have PASS evidence or approved waiver.
- No open Sev-1/blocker.
- Sev-2 has approved mitigation and owner.
- Finance reconciles fare/payment/ledger/settlement against approved sample.
- Operations approves dispatch, trip, exception and driver flows.
- Security/data-scope tests pass.
- Backup restore and operational readiness pass.
- Formal UAT signatures from Business, Operations, Finance, IT and QA.

## 5. Defect severity

| Severity | Definition | Release |
|---|---|---|
| Sev-1 | Safety/security breach, financial corruption, unavailable core system | Block |
| Sev-2 | Major function broken, no workable workaround | Normally block |
| Sev-3 | Partial failure with workaround | Risk acceptance |
| Sev-4 | Cosmetic/low impact | Backlog |

## 6. Sign-off record

| Role | Name | Decision | Date | Evidence |
|---|---|---|---|---|
| Business Owner | TBD | Pending | TBD | BRD/requirements |
| Operations | TBD | Pending | TBD | Dispatch/trip UAT |
| Finance | TBD | Pending | TBD | Fare/payment/settlement |
| IT | TBD | Pending | TBD | Security/operations |
| QA Lead | TBD | Pending | TBD | Test report |
