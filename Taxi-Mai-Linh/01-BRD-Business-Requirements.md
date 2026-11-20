# 01 — TÀI LIỆU YÊU CẦU NGHIỆP VỤ (BRD)

## 1. Bối cảnh và vấn đề

Nền tảng mới phải hỗ trợ nhiều kênh tiếp nhận và hai nguồn lực DIRECT/PARTNER. Cần tránh phân mảnh dữ liệu booking, xe, trip, fare, payment và settlement; các số liệu điều hành/tài chính phải truy ngược đến giao dịch gốc.

## 2. Business objectives

| ID | Mục tiêu | Kết quả mong muốn | KPI |
|---|---|---|---|
| BO-01 | Chuẩn hóa tiếp nhận | Booking đúng và đủ dữ liệu | Booking validity |
| BO-02 | Tối ưu điều phối | Gán xe phù hợp, phản hồi nhanh | Dispatch success, time-to-assign |
| BO-03 | Minh bạch vận hành | Theo dõi trạng thái và timeline | Completion, exception rate |
| BO-04 | Quản lý nguồn lực | Biết xe/tài xế đủ điều kiện | Eligibility, utilization |
| BO-05 | Minh bạch tài chính | Fare/payment/ledger truy vết | Reconciliation variance |
| BO-06 | Quản trị đối tác | Đối soát đúng hợp đồng | Settlement accuracy |
| BO-07 | Chất lượng dịch vụ | Xử lý khiếu nại có SLA | Complaint rate, case SLA |

Mục tiêu số và cách loại trừ chuyến cần owner xác nhận.

## 3. Scope

### In scope
- Booking đa kênh, booking tức thời/đặt trước, địa chỉ và multi-stop theo policy.
- Dispatch tự động/thủ công, offer, accept/reject/timeout, reassign.
- Control Tower/Maps, GPS freshness, hàng đợi ngoại lệ.
- Driver App và trip lifecycle.
- Fleet, Driver, Partner, Contract và eligibility.
- Fare snapshot, payment record, ledger.
- Partner settlement, CSKH/incident, báo cáo, IAM/audit.
- API và tích hợp map/GPS/payment/notification sau khi chọn nhà cung cấp.

### Out of scope / TBD
- Loyalty, định giá động, AI demand forecasting.
- Bảo dưỡng xe chuyên sâu.
- ERP/e-invoice cụ thể khi chưa khảo sát.
- Tích hợp contact center/voice theo hệ thống hiện hữu.
- Customer App và Partner Portal nếu chưa được duyệt ở giai đoạn đầu.

## 4. Business capabilities

| Capability | Sub-capability | Owner đề xuất |
|---|---|---|
| Demand capture | Hotline, app/web, corporate API | Call Center/Product |
| Booking | Validation, address, schedule, modification | Operations |
| Dispatch | Eligibility, ranking, offer, assignment | Operations |
| Fleet operations | Vehicle, driver, shift, documents | Fleet |
| Trip execution | Pickup, trip, completion, cancellation | Operations |
| Commercial | Tariff, surcharge, discount, payment | Business/Finance |
| Partner operations | Contract, policy, performance | Partner Management |
| Settlement | Ledger, batch, approval, payout | Finance |
| Customer service | Complaint, lost item, incident | CSKH |
| Platform | IAM, audit, integrations, observability | IT |

## 5. Stakeholders

Executive Sponsor; Business Owner; Operations Owner; Call Center Lead; Fleet Owner; Partner Owner; Finance Owner; Customer Service Owner; IT Architect; Product/BA; UX/UI; Development; QA; IT Operations.

Mỗi capability cần một accountable owner. Người tham gia và người phê duyệt phải được ghi vào RACI dự án.

## 6. Business constraints

- Chưa giả định giá cước, tỷ lệ chia, phí hủy hoặc ưu tiên fleet type.
- PII khách/tài xế cần phân quyền, masking, retention, audit.
- GPS có thể mất/chậm; UI phải thể hiện data freshness.
- PARTNER chỉ được nhận chuyến nếu hợp đồng và điều kiện khai thác hợp lệ.
- Không sửa ngược số liệu đã đối soát/khóa kỳ.

## 7. KPI catalogue

| KPI | Công thức khởi tạo | Owner xác nhận |
|---|---|---|
| Booking volume | Count booking theo thời gian/kênh | Operations |
| Dispatch success | Assigned / eligible bookings | Operations |
| Time to assign | assigned_at − created_at | Operations |
| Pickup punctuality | Pickup trong SLA / trip đủ điều kiện | Operations |
| Trip completion | Completed / trip đủ điều kiện | Operations |
| Cancellation | Cancelled phân theo actor/reason | Operations |
| Fare revenue | Sum fare theo định nghĩa kế toán | Finance |
| Partner net | Gross − deductions ± adjustment | Finance |
| Complaint rate | Cases / completed trips | CSKH |

## 8. BRD acceptance

BRD được Approved khi owner cho capability, scope, KPI, mô hình DIRECT/PARTNER và các quyết định Critical đều được ghi nhận.
