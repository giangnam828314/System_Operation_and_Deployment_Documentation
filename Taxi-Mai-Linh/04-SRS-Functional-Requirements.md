# 04 — ĐẶC TẢ YÊU CẦU PHẦN MỀM (SRS)

## 1. Quy ước về yêu cầu

MUST = baseline bắt buộc; SHOULD = ưu tiên cao; MAY = tùy chọn; TBD = cần owner xác nhận. Mọi requirement phải liên kết workflow, screen/API và test case.

## 2. Yêu cầu chức năng

| ID | Module | Requirement | Priority |
|---|---|---|---|
| IAM-001 | IAM | Authenticate users before internal access | MUST |
| IAM-002 | IAM | RBAC by module/action and data scope by branch/zone/partner | MUST |
| IAM-003 | IAM | Audit role changes and sensitive operations | MUST |
| BKG-001 | Booking | Create booking with channel, customer, pickup, destination, schedule, service | MUST |
| BKG-002 | Booking | Support immediate and scheduled booking | MUST |
| BKG-003 | Booking | Geocode, pin confirmation, landmark and pickup instruction | SHOULD |
| BKG-004 | Booking | Multi-stop as configured | SHOULD |
| BKG-005 | Booking | Duplicate warning without automatic deletion | SHOULD |
| BKG-006 | Booking | Update/cancel guarded by state, permission, reason and audit | MUST |
| BKG-007 | Booking | Keep booking_channel separate from fleet_type | MUST |
| DSP-001 | Dispatch | Queue by status, SLA age, pickup time and priority | MUST |
| DSP-002 | Dispatch | Find candidates by zone, vehicle, driver, state and contract | MUST |
| DSP-003 | Dispatch | Support auto and manual dispatch | MUST |
| DSP-004 | Dispatch | Offer expiry, accept/reject, retry | MUST |
| DSP-005 | Dispatch | Maximum one active assignment per booking | MUST |
| DSP-006 | Dispatch | Reassign/withdraw with permission, reason and audit | MUST |
| DSP-007 | Dispatch | No candidate moves booking to exception queue | MUST |
| MAP-001 | Control Tower | Show vehicles by latest GPS and operating state | MUST |
| MAP-002 | Control Tower | Separate GPS freshness from online/offline state | MUST |
| MAP-003 | Control Tower | Filter branch, zone, vehicle type, fleet and partner | MUST |
| MAP-004 | Control Tower | Marker opens vehicle/driver/current-trip panel | MUST |
| MAP-005 | Control Tower | Booking click opens detail and eligible candidates | MUST |
| DRV-001 | Driver App | Driver accesses only valid assigned trips/offers | MUST |
| DRV-002 | Driver App | Accept/reject offer; record response | MUST |
| DRV-003 | Driver App | Submit trip events under state machine | MUST |
| DRV-004 | Driver App | Offline queue, retry and idempotent sync | MUST |
| DRV-005 | Driver App | GPS/network status and SOS/incident | SHOULD |
| TRP-001 | Trip | Link trip to booking, driver, vehicle, assignment and fleet type | MUST |
| TRP-002 | Trip | Reject invalid state transitions | MUST |
| TRP-003 | Trip | Timeline includes actor/time/source/reason | MUST |
| TRP-004 | Trip | Reassign/cancel/no-show/incident follow policy | MUST |
| FAR-001 | Fare | Calculate using tariff version effective at trip time | MUST |
| FAR-002 | Fare | Persist fare breakdown and calculation snapshot | MUST |
| PAY-001 | Payment | Store method, amount, status, provider ref and timestamp | MUST |
| PAY-002 | Payment | Validate webhook signature, idempotency and replay | MUST |
| LED-001 | Ledger | Financial adjustment is a new entry with reason/approval | MUST |
| FLT-001 | Fleet | Manage vehicle/driver profile and document validity | MUST |
| FLT-002 | Fleet | Enforce availability and eligibility | MUST |
| PRT-001 | Partner | Manage partner, contract, validity and policy version | MUST |
| PRT-002 | Partner | Block ineligible/expired partner resources from new trips | MUST |
| SET-001 | Settlement | Create batch by partner/contract/period | MUST |
| SET-002 | Settlement | Drill down each trip and calculation | MUST |
| SET-003 | Settlement | Approve/lock/pay; post-lock adjustment via new batch | MUST |
| CS-001 | CSKH | Case links booking/trip, category, owner, SLA and timeline | MUST |
| RPT-001 | Reporting | Filter by period, branch, channel, fleet and partner | MUST |
| RPT-002 | Reporting | Drill down KPI to source booking/trip/ledger | MUST |

## 3. Quy tắc nghiệp vụ

| ID | Rule |
|---|---|
| BR-001 | Booking channel does not determine fleet type. |
| BR-002 | Assignment stores resource, fleet type, policy version, source and timestamps. |
| BR-003 | Do not assign vehicle/driver with conflicting active trip. |
| BR-004 | Eligibility checks active status, documents, shift, zone, vehicle type and contract. |
| BR-005 | Pickup/destination change after assignment triggers ETA and fare estimate review. |
| BR-006 | Override requires reason and audit; sensitive override may require supervisor approval. |
| BR-007 | Historical trip fare uses saved tariff version. |
| BR-008 | Partner settlement uses contract/policy snapshot at trip time. |
| BR-009 | Payment callback is idempotent and replay-protected. |
| BR-010 | GPS stale does not automatically mean vehicle offline. |
| BR-011 | No candidate does not delete/cancel booking. |
| BR-012 | Locked financial records are not overwritten/deleted. |

## 4. State model

- Booking: DRAFT → CONFIRMED → READY_FOR_DISPATCH → SEARCHING → ASSIGNED → IN_SERVICE → COMPLETED; branches SCHEDULED/CANCELLED/EXCEPTION.
- Assignment: CREATED → OFFERED → ACCEPTED → ACTIVE; branches REJECTED/EXPIRED/WITHDRAWN/REASSIGNED.
- Vehicle: OFFLINE / AVAILABLE / RESERVED / TO_PICKUP / ON_TRIP / PAUSED / MAINTENANCE.
- Trip: ASSIGNED → EN_ROUTE_TO_PICKUP → ARRIVED → IN_TRIP → COMPLETED; branches CANCELLED/NO_SHOW/INCIDENT.
- Payment: PENDING → INITIATED → SUCCESS/FAILED/EXPIRED/REFUNDED/PARTIALLY_REFUNDED.
- Settlement: DRAFT → CALCULATED → REVIEWED → APPROVED → LOCKED → PAID.

Final state names and transition guards require business sign-off before API implementation.

## 5. Exception behavior

| Exception | Minimum behavior |
|---|---|
| No candidate | Keep booking, show reason, retry/escalate |
| Offer reject/timeout | Close offer, log event, try next candidate |
| GPS stale | Show freshness warning; no silent vehicle state mutation |
| Driver offline | Queue event and show sync pending |
| Invalid transition | Reject server-side; no state mutation |
| Duplicate event/webhook | Idempotent result; no duplicate record |
| Partner contract expired | Block new assignment and explain |
| Payment failed | Trip remains completed; payment exception separate |
| Settlement variance | Flag discrepancy; dispute/adjustment workflow |
