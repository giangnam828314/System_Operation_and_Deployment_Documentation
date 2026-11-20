# 13 — KẾ HOẠCH PHÁT TRIỂN VÀ TRIỂN KHAI

## 1. Workstreams

| Workstream | Outputs | Owner |
|---|---|---|
| Product/BA | BRD, SRS, workflow, backlog, traceability | Product/BA |
| UI/UX | Research, design system, Figma, prototype | UX Lead |
| Architecture | ADR, deployment, NFR, integration design | Architect |
| Backend | Booking, dispatch, trip, fare, partner, APIs | Backend Lead |
| Web | Call Center, Control Tower, Admin, Finance | Frontend Lead |
| Mobile | Driver App, offline sync, GPS, push | Mobile Lead |
| Data | Migration, analytics, reconciliation | Data Lead |
| QA | Test plan, automation, UAT | QA Lead |
| IT Ops | CI/CD, monitoring, backup, DR, runbooks | IT Ops |

## 2. Triển khai theo giai đoạn

| Phase | Deliverable | Exit gate |
|---|---|---|
| 0 Discovery | Workshops, AS-IS/TO-BE, decision closure | Business scope signed |
| 1 Foundation | IAM, master data, audit, API standards | Architecture/security gate |
| 2 Booking & Dispatch | Call Center, queue, candidates, Control Tower MVP | Dispatch UAT |
| 3 Driver & Trip | Driver App, trip lifecycle, GPS | Field pilot |
| 4 Fare & Finance | Fare, payment, ledger, partner settlement | Finance reconciliation |
| 5 CS & Reporting | Cases, KPI, reports | Business UAT |
| 6 Rollout | Migration, cutover, training, support | Go-live approval |

Timeline and staffing require sizing; phases are not calendar commitments.

## 3. Các môi trường

| Environment | Purpose | Data |
|---|---|---|
| DEV | Integration development | Synthetic |
| TEST/SIT | Functional/integration tests | Synthetic/masked |
| UAT | Business validation | Masked/approved |
| STAGING | Release rehearsal | Production-like, masked |
| PROD | Live operation | Controlled production |

## 4. Kiểm soát quy trình CI/CD

Reproducible builds; dependency pinning; unit/lint/static/secret/dependency scans; artifact promotion between environments; production approval; backward-compatible DB migration; rollback plan; release notes; secrets outside repository.

## 5. Chuyển đổi dữ liệu

Inventory sources; profile quality; map fields; resolve duplicate identities; normalize phone/plate/timezone; map DIRECT/PARTNER and contract periods; dry run; exception report; final delta migration; count/balance reconciliation; sampling; business sign-off; rollback snapshot.

## 6. Danh sách kiểm tra chuyển hệ thống

- Change freeze and stakeholder communication.
- Backup and restore point verified.
- DNS/certificates/firewall/secrets checked.
- Migration executed and reconciled.
- Integrations enabled in controlled order.
- Smoke test login, booking, dispatch, driver event, trip, fare and payment.
- Pilot by approved branch/zone/fleet cohort.
- Monitor dashboard and incident channel.
- Go/no-go by sponsor, Operations, Finance and IT.
- Rollback triggers/authority confirmed.

## 7. Điều kiện kích hoạt quay lui

Examples: booking/dispatch unavailable beyond tolerance; duplicate/lost trip events; financial variance; security/privacy incident; GPS/Driver App degradation unsafe for operations. Thresholds must be approved. Rollback must account for external payment/events, not database restore alone.
