# 16 — MA TRẬN TRUY VẾT YÊU CẦU

## 1. Traceability principle

Business Objective → BRD → Requirement → Workflow → UI → API/Data → Test Case → Acceptance Evidence.

## 2. Initial mapping

| Requirement group | Workflow | UI | API/Data | Test |
|---|---|---|---|---|
| BKG-001..007 | WF-01 booking | UI-01 | Booking, Customer, BookingStop | TC-001..003 |
| DSP-001..007 | WF-01/WF-03 | UI-02/UI-03 | Candidate, Assignment | TC-004..006, TC-019 |
| MAP-001..005 | WF-03 | UI-02 | GPS/Realtime | TC-008 |
| DRV-001..005 | WF-04 | UI-04/UI-05 | Driver Event, TripEvent | TC-007, TC-010 |
| TRP-001..004 | WF-04 | UI-06 | Trip, TripEvent | TC-009, TC-020 |
| FAR/PAY/LED | WF-05 | UI-07 | Fare, PaymentTransaction, LedgerEntry | TC-011..013, TC-016 |
| PRT/SET | WF-06 | UI-08/UI-09 | PartnerContract, SettlementBatch/Line | TC-005, TC-014 |
| IAM | All | UI-13 | User, Role, Permission, AuditLog | TC-015 |
| CS/RPT | WF-07 | UI-11/UI-12 | Incident/Case, Reporting | TC-017, TC-018 |

## 3. Change impact procedure

1. Identify impacted business objective and rule.
2. Identify workflow nodes/transitions.
3. Identify UI screens and states.
4. Identify API/schema/event contract.
5. Identify tests and acceptance evidence.
6. Update linked docs and version history.
7. Obtain required approval.

## 4. Coverage tracker template

| Mã yêu cầu | Mã luồng | Mã giao diện | Mã API/Dữ liệu | Mã kiểm thử | Trạng thái | Bằng chứng |
|---|---|---|---|---|---|---|
| BKG-001 | WF-01 | UI-01 | Booking | TC-001 | Draft | TBD |
| DSP-002 | WF-03 | UI-02 | CandidateService | TC-004/005 | Draft | TBD |
| DRV-003 | WF-04 | UI-04/05 | TripEvent | TC-009/010 | Draft | TBD |
| FAR-001 | WF-05 | UI-07 | Fare | TC-011 | Draft | TBD |
| SET-001 | WF-06 | UI-09 | SettlementBatch | TC-014 | Draft | TBD |
