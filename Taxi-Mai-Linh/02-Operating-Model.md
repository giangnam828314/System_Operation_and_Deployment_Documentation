# 02 — MÔ HÌNH VẬN HÀNH: DIRECT VÀ PARTNER

## 1. Nguyên tắc

Một nền tảng điều phối, hai mô hình khai thác. Phân loại DIRECT/PARTNER phải xuyên suốt Vehicle → Driver → Assignment → Trip → Fare/Ledger → Settlement/Reporting.

## 2. So sánh mô hình

| Thuộc tính | DIRECT | PARTNER | System control |
|---|---|---|---|
| Nguồn lực | Mai Linh quản lý trực tiếp | Đối tác hợp tác | owner/organization reference |
| Điều kiện dispatch | Hồ sơ và trạng thái nội bộ | Hồ sơ + hợp đồng hiệu lực + phạm vi | Eligibility service |
| Policy | Chính sách nội bộ | Hợp đồng/điều khoản phiên bản | Effective-dated policy |
| Doanh thu | Theo quy trình nội bộ | Theo cơ chế phân chia hợp đồng | Ledger dimensions |
| Đối soát | Nội bộ nếu cần | Theo partner/kỳ | Settlement batch |
| Báo cáo | DIRECT | PARTNER | fleet_type dimension |

Chi tiết pháp lý về quyền sở hữu xe, lao động, bảo hiểm và thuế phải theo hồ sơ đã được Mai Linh xác minh.

## 3. Phân biệt booking_channel và fleet_type

Ví dụ: HOTLINE booking có thể được gán xe DIRECT hoặc PARTNER. CUSTOMER_APP booking cũng vậy. Corporate API không mặc nhiên là PARTNER.

- booking_channel: HOTLINE, CUSTOMER_APP, WEB, CORPORATE_API, PARTNER_PORTAL, OTHER.
- fleet_type: DIRECT hoặc PARTNER.
- assignment lưu snapshot fleet_type được gán.

## 4. Eligibility matrix

| Điều kiện | DIRECT | PARTNER |
|---|---|---|
| Vehicle active | Required | Required |
| Driver active | Required | Required |
| Document valid | Internal policy | Policy + contract |
| Service type match | Required | Required |
| Operating area | Internal zone policy | Contract/service zone |
| Available/shift valid | Required | Required |
| Contract valid | Internal policy | Required |
| Partner suspended | N/A | Block |

## 5. Vai trò và quyền

| Role | Allowed | Restriction |
|---|---|---|
| Call Center Agent | CRUD booking trong scope | Không chỉnh fare/settlement |
| Dispatcher | View map/queue, assign/reassign | Chỉ vùng được cấp |
| Supervisor | Override, xử lý exception | Reason; approval threshold TBD |
| Driver | Chuyến/offer được giao | Không truy cập driver khác |
| Fleet Manager | Vehicle/driver eligibility | Theo đơn vị |
| Partner Manager | Partner/contract | Theo portfolio |
| Finance | Payment, ledger, settlement | Không điều phối |
| CS Agent | Case và thông tin cần thiết | PII tối thiểu |
| System Admin | IAM/config/integration | Business data cấp riêng |
| Auditor | Read-only/audit | Export controlled |

## 6. Reporting dimensions

Date/time, branch, region, shift, channel, service type, fleet_type, partner_id, contract_id, driver_id, vehicle_id, booking/trip status, cancellation reason, incident category, payment method/status.

## 7. Policy snapshot

Khi assignment/trip được tạo, lưu policy_version_id, contract_id (nếu PARTNER), tariff_version_id và timestamp. Thay đổi hợp đồng hoặc bảng giá mới không được âm thầm thay đổi kết quả nghiệp vụ đã phát sinh.
