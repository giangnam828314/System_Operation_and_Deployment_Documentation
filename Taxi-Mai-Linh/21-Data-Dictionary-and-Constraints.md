# 21 — TỪ ĐIỂN DỮ LIỆU VÀ RÀNG BUỘC TOÀN VẸN
Tài liệu xác định các trường dữ liệu ở mức logic. Hệ quản trị CSDL vật lý, phân vùng và kiểu SQL chính xác là quyết định kiến trúc.

## 1. Từ điển thực thể cốt lõi
| Thực thể | Trường bắt buộc | Toàn vẹn / ghi chú |
|---|---|---|
| Khách hàng | id, status, created_at | hạn chế truy cập dữ liệu cá nhân; chính sách xóa mềm TBD |
| Booking | id, public_reference, channel, customer_id, state, version, requested_at | enum channel độc lập fleet_type |
| Điểm dừng booking | id, booking_id, sequence, type, address, lat, lng | sequence duy nhất trong booking |
| Tài xế | id, status, fleet_type, partner_id có thể rỗng | PARTNER bắt buộc partner_id; DIRECT để rỗng |
| Xe | id, plate_normalized, status, fleet_type, partner_id có thể rỗng | phạm vi duy nhất biển số TBD |
| Phân công tài xế–xe | id, driver_id, vehicle_id, valid_from, valid_to | không cho khoảng hiệu lực đang hoạt động chồng lấn nếu chưa duyệt |
| Lần điều phối | id, booking_id, attempt_no, policy_version, state | duy nhất theo booking_id + attempt_no |
| Lời mời tài xế | id, dispatch_attempt_id, driver_id, vehicle_id, expires_at, state | chống trùng lời mời còn hiệu lực theo lần điều phối/tài xế |
| Phân xe | id, booking_id, offer_id, driver_id, vehicle_id, assigned_at, state | mỗi booking tối đa một phân xe đang hiệu lực |
| Chuyến đi | id, booking_id, assignment_id, state, started_at, completed_at | quan hệ assignment duy nhất |
| Phiên bản biểu giá | id, code, version, valid_from, valid_to, status | bất biến sau khi kích hoạt |
| Báo giá | id, booking_id, tariff_version_id, amount_minor, currency, expires_at | lưu bản chụp đầu vào bất biến |
| Chi tiết cước | id, quote/trip, component_code, amount_minor, metadata | tổng chi tiết phải khớp tổng tiền |
| Thanh toán | id, trip_id, method, amount_minor, currency, status | mã giao dịch nhà cung cấp duy nhất nếu áp dụng |
| Bút toán sổ cái | id, trip_id, account_code, debit_minor, credit_minor, currency, entry_type | chỉ ghi nối tiếp; bút toán phải cân |
| Phiên bản hợp đồng đối tác | id, partner_id, version, valid_from, valid_to, terms_snapshot | bất biến khi đã được sử dụng |
| Kỳ đối soát | id, partner_id, period_start, period_end, state, version | chính sách duy nhất đối tác + kỳ TBD |
| Dòng đối soát | id, settlement_id, ledger_entry_id, amount_minor, line_type | không đưa trùng bút toán vào cùng bảng kê |
| Sự kiện kiểm toán | id, actor_id, action, resource_type, resource_id, occurred_at, correlation_id | chỉ ghi nối tiếp, có biện pháp chống sửa đổi |

## 2. Chỉ mục cần có ở mức logic
- Booking: public_reference duy nhất; (state, requested_at); (customer_id, created_at).
- BookingStop: duy nhất (booking_id, sequence).
- Driver/Vehicle: (status, fleet_type, partner_id).
- Kho vị trí GPS mới nhất: chỉ mục địa lý + observed_at; thời hạn lưu TBD.
- DispatchAttempt: duy nhất (booking_id, attempt_no).
- Offer: (driver_id, state, expires_at), (dispatch_attempt_id, state).
- Assignment: chỉ mục duy nhất có điều kiện cho booking và tài xế/xe đang hoạt động, tùy CSDL.
- Trip: (state, started_at), assignment_id duy nhất.
- LedgerEntry: (trip_id), (account_code, posted_at), duy nhất source_event_id + entry_type.
- Settlement: (partner_id, period_start, period_end, state).

## 3. Tiền tệ và độ chính xác
Lưu số nguyên theo đơn vị tiền nhỏ nhất và mã tiền tệ. Số chữ số thập phân phải lấy từ metadata tiền tệ; không mặc định mọi loại tiền đều có hai chữ số. Quy ước dấu và Nợ/Có phải được ghi trong hệ thống tài khoản kế toán do Tài chính phê duyệt.

## 4. Vòng đời dữ liệu
Xác định thời hạn lưu theo nhóm: dữ liệu cá nhân khách hàng, GPS, ghi âm cuộc gọi, bằng chứng chuyến, sổ cái tài chính, nhật ký bảo mật. Thời hạn lưu, yêu cầu giữ dữ liệu pháp lý, xóa/ẩn danh và hết hạn bản sao lưu phải được Pháp chế/Tuân thủ duyệt. Không xóa dấu vết tài chính để đáp ứng yêu cầu xóa dữ liệu; áp dụng quy trình ẩn danh/lưu trữ đã duyệt.

## 5. Quy tắc toàn vẹn
- Khóa ngoại cho quan hệ sở hữu cốt lõi.
- CHECK constraint cho số tiền không âm khi phù hợp.
- Dịch vụ nghiệp vụ kiểm tra chuyển trạng thái và ghi trong một giao dịch nguyên tử.
- Dùng cột version/khóa lạc quan cho thực thể có thể thay đổi.
- Bảng kiểm toán và sổ cái chỉ ghi nối tiếp; điều chỉnh bằng bút toán đảo/điều chỉnh.
- Giữ liên kết đến bản chụp biểu giá và hợp đồng đã dùng cho chuyến hoàn tất.
- Khoảng thời gian hiệu lực dùng quy ước nửa mở [valid_from, valid_to).

## 6. Chuyển đổi dữ liệu
Ánh xạ ID nguồn sang ID đích bằng bảng đối chiếu. Kiểm tra số bản ghi, tổng tài chính, toàn vẹn tham chiếu, trùng lặp, mã hóa, múi giờ, ánh xạ trạng thái và xử lý dữ liệu cá nhân. Chủ sở hữu dữ liệu phải ký xác nhận đối soát.
