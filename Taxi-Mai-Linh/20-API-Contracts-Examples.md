# 20 — HỢP ĐỒNG API VÀ VÍ DỤ TÍCH HỢP
**Kiểu hợp đồng:** REST/JSON cho lệnh và truy vấn đồng bộ; sự kiện nghiệp vụ bất đồng bộ cho tích hợp. Địa chỉ API gốc, nhà cung cấp xác thực và cổng API đang TBD.

## 1. Quy ước chung
- Tiền tố: `/api/v1`.
- Kiểu nội dung: `application/json; UTF-8`.
- Xác thực: access token OAuth2/OIDC (nhà cung cấp TBD); định danh dịch vụ-dịch vụ cho lời gọi nội bộ.
- Mọi yêu cầu làm thay đổi dữ liệu phải có `Idempotency-Key`.
- Mọi yêu cầu truyền tiếp `X-Correlation-ID`.
- Thời gian dùng ISO 8601 UTC; lưu UTC, hiển thị theo múi giờ địa phương đã cấu hình.
- Tiền lưu bằng số nguyên đơn vị nhỏ nhất và mã tiền tệ ISO; không dùng số thực nhị phân.
- Tọa độ dùng hệ WGS84 latitude/longitude và phải kiểm tra hợp lệ.
- Phân trang bằng cursor; thứ tự sắp xếp phải ổn định.
- Cấu trúc lỗi gồm: code, message, details, correlation_id, retryable.

## 2. Cấu trúc phản hồi lỗi
```json
{
  "error": {
    "code": "BOOKING_STATE_CONFLICT",
    "message": "Booking is no longer dispatchable",
    "details": {"booking_id": "uuid", "current_state": "ASSIGNED"},
    "correlation_id": "uuid",
    "retryable": false
  }
}
```
Không trả về stack trace, lỗi SQL, bí mật hoặc tên máy chủ nội bộ. Các giá trị kỹ thuật trong JSON giữ nguyên để bảo đảm tương thích máy-máy.

## 3. Tạo booking
`POST /api/v1/bookings`
```json
{
  "channel": "HOTLINE",
  "customer_id": "uuid",
  "pickup": {"address": "string", "lat": 10.77, "lng": 106.69, "place_id": null},
  "dropoff": {"address": "string", "lat": 10.80, "lng": 106.70},
  "service_class": "STANDARD",
  "passenger_count": 2,
  "requested_pickup_at": null,
  "notes": null
}
```
Phản hồi: HTTP 201 cùng booking_id, public_reference, state, created_at, version. Kênh đặt không truyền loại đội xe; loại đội xe được xác định bởi điều kiện nguồn xe/chính sách điều phối.

## 4. Tính báo giá
`POST /api/v1/quotes`
Yêu cầu: booking_id, pickup, dropoff, service_class; route_distance/duration chỉ được dùng nếu lấy từ dịch vụ định tuyến đáng tin cậy. Phản hồi: quote_id, currency, estimated_amount_minor, breakdown[], tariff_version_id, valid_until, assumptions[].
Báo giá chỉ là ước tính, trừ khi chính sách quy định rõ là giá bảo đảm.

## 5. Bắt đầu điều phối
`POST /api/v1/bookings/{booking_id}/dispatch`
Yêu cầu: policy_version, mode (AUTO|MANUAL), operator_note không bắt buộc. Phản hồi: dispatch_id, state, attempt_no, candidate_count, created_at.

## 6. Tài xế phản hồi lời mời
`POST /api/v1/offers/{offer_id}/response`
Yêu cầu: response (ACCEPT|DECLINE), reason_code. Phản hồi: trạng thái lời mời, trạng thái booking, assignment_id nếu chấp nhận. Máy chủ kiểm tra đồng thời hạn lời mời, điều kiện và xung đột phân xe.

## 7. Cập nhật mốc chuyến
`POST /api/v1/trips/{trip_id}/milestones`
Yêu cầu: milestone (EN_ROUTE|ARRIVED|STARTED|COMPLETED), occurred_at, location, odometer tùy chọn, tham chiếu bằng chứng tùy chọn. Máy chủ lưu riêng received_at và occurred_at; không mặc nhiên tin thời gian trên thiết bị.

## 8. Ghi nhận thanh toán
`POST /api/v1/payments`
Yêu cầu: trip_id, method, amount_minor, currency, provider_reference tùy chọn, idempotency_key. Phản hồi: payment_id, status, amount_minor, currency. Callback của nhà cung cấp đi qua endpoint riêng có xác thực.

## 9. Duyệt đối soát đối tác
`POST /api/v1/partner-settlements/{settlement_id}/approve`
Yêu cầu: approval_note, statement_version. Chỉ vai trò Tài chính được cấp quyền mới thực hiện; khi duyệt phải kiểm tra tổng bảng kê, tranh chấp và việc duyệt trùng.

## 10. Cấu trúc sự kiện/webhook
```json
{
  "event_id": "uuid",
  "event_type": "booking.assigned.v1",
  "occurred_at": "2026-11-14T04:00:00Z",
  "aggregate_id": "uuid",
  "aggregate_version": 7,
  "correlation_id": "uuid",
  "producer": "dispatch-service",
  "data": {}
}
```
Bên nhận chống trùng theo event_id và từ chối phiên bản aggregate cũ nếu thứ tự sự kiện có ý nghĩa.

## 11. Danh mục endpoint ban đầu
| Miền | Endpoint | Quyền |
|---|---|---|
| Đặt xe | POST/GET /bookings | booking:create/read |
| Đặt xe | POST /bookings/{id}/cancel | booking:cancel |
| Định giá | POST /quotes | quote:create |
| Điều phối | POST /bookings/{id}/dispatch | dispatch:execute |
| Điều phối | GET /dispatch/queue | dispatch:read |
| Điều phối | POST /offers/{id}/response | driver:offer-response |
| Chuyến đi | POST /trips/{id}/milestones | trip:update |
| Thanh toán | POST /payments | payment:create |
| Tài chính | GET /ledger/entries | finance:read |
| Đối soát | GET /partner-settlements | settlement:read |
| Đối soát | POST /partner-settlements/{id}/approve | settlement:approve |

## 12. Danh sách cần hoàn thiện trước khi khóa hợp đồng
Tạo OpenAPI 3.1, lược đồ dữ liệu, danh mục enum, phạm vi quyền xác thực, giới hạn tần suất, phân loại dữ liệu cá nhân, danh mục lỗi, phân trang, thời hạn lưu khóa chống trùng, chữ ký/chống phát lại webhook, quản lý phiên bản và kiểm thử tương thích ngược.
