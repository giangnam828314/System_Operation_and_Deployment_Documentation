# 29 — MA TRẬN TÍCH HỢP VÀ NHÀ CUNG CẤP DỊCH VỤ BÊN NGOÀI

| Tích hợp | Mục đích | Chiều dữ liệu | Cách xử lý khi lỗi | Tiêu chí lựa chọn | Nhà cung cấp |
|---|---|---|---|---|---|
| Bản đồ/định tuyến | mã hóa địa chỉ, lộ trình, ETA | yêu cầu/phản hồi | bộ nhớ đệm, dự phòng/thủ công | độ phủ Việt Nam, giao thông, chi phí, SLA | TBD |
| GPS tài xế | vị trí thiết bị | di động → nền tảng | đánh dấu vị trí cũ, xếp hàng telemetry an toàn | độ chính xác, pin, quyền riêng tư | TBD |
| SMS/Voice/PBX | OTP/thông báo khách-tài xế/tổng đài | hai chiều | thử lại, kênh thay thế, tổng đài dự phòng | tỷ lệ gửi, độ trễ, báo cáo, SLA | TBD |
| Thông báo đẩy | lời mời/trạng thái tài xế | nền tảng → di động | thử lại, dự phòng bằng polling | khả năng gửi Android/iOS | TBD |
| Cổng thanh toán | trạng thái thu/hoàn tiền | hai chiều | trạng thái chờ/hàng đợi đối soát | giấy phép, bảo mật, phí, thanh toán | TBD |
| ERP/kế toán | ghi nhận tài chính | hai chiều/theo lô | Outbox và đối soát | API, hệ thống tài khoản, kiểm toán | TBD |
| Hóa đơn điện tử | phát hành hóa đơn | nền tảng → nhà cung cấp | thử lại/ngoại lệ thủ công | phù hợp pháp lý, API, lưu trữ | TBD |
| Nhà cung cấp định danh | xác thực/MFA | hai chiều | từ chối truy cập đặc quyền khi lỗi | OIDC/SAML, vòng đời tài khoản | TBD |
| Phân tích dữ liệu/BI | báo cáo vận hành và tài chính | sự kiện/theo lô | pipeline có thể phát lại | phân quyền, độ mới | TBD |

## Yêu cầu hợp đồng tích hợp
Với mỗi nhà cung cấp được chọn, lưu: pháp nhân, sản phẩm/gói, URL môi trường, phương thức xác thực, phạm vi quyền, giới hạn tần suất, timeout, chính sách thử lại, chống xử lý lặp, chữ ký webhook, IP allowlist nếu có, nơi lưu dữ liệu, xử lý dữ liệu cá nhân, SLA, liên hệ hỗ trợ, chuyển cấp sự cố, giá, ngày hết hạn hợp đồng và phương án rút/chuyển nhà cung cấp.

## Kiểm thử tiếp nhận nhà cung cấp
Kết nối → xác thực → luồng thành công → yêu cầu sai → timeout → thử lại → callback trùng → nhà cung cấp ngừng hoạt động → đối soát → xoay vòng thông tin xác thực → ký xác nhận sẵn sàng sản xuất.
