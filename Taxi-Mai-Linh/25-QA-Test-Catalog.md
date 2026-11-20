# 25 — DANH MỤC KIỂM THỬ VÀ KỊCH BẢN NGHIỆM THU
Dữ liệu kiểm thử có thể chạy và kết quả mong đợi chi tiết phải được quản lý trong công cụ kiểm thử do Mai Linh lựa chọn.

## 1. Kiểm thử đầu-cuối trọng yếu
| Mã | Kịch bản | Kết quả mong đợi |
|---|---|---|
| E2E-001 | Tổng đài tạo booking | lưu booking với channel HOTLINE; chưa quyết định loại đội xe |
| E2E-002 | Ứng dụng tạo booking | cùng vòng đời booking; ghi nhận channel APP |
| E2E-003 | Có ứng viên DIRECT đủ điều kiện | đánh giá theo chính sách đã duyệt |
| E2E-004 | Có ứng viên PARTNER đủ điều kiện | xác thực và ghi nhận điều kiện hợp đồng |
| E2E-005 | DIRECT và PARTNER cùng đủ điều kiện | chọn theo chính sách đã cấu hình/duyệt; truy vết được lý do |
| E2E-006 | Hai tài xế nhận đồng thời | chỉ một phân xe thành công |
| E2E-007 | Lời mời hết hạn trước khi nhận | từ chối nhận; thử lại/chuyển thủ công theo chính sách |
| E2E-008 | Tài xế mất điều kiện sau khi nhận lời mời | từ chối thao tác; ghi lý do |
| E2E-009 | Điều phối viên phân xe thủ công | vẫn áp dụng điều kiện bắt buộc; kiểm toán lý do ghi đè |
| E2E-010 | Đổi điểm đón sau khi báo giá | báo giá cũ bị vô hiệu hoặc được xác nhận tính lại |
| E2E-011 | Hoàn tất chuyến nhiều lần do gửi lại | chỉ ghi nhận một lần hoàn tất và một lần hạch toán |
| E2E-012 | Callback thanh toán gửi hai lần | bỏ qua bản trùng; chỉ một lần đổi trạng thái |
| E2E-013 | Số tiền thanh toán không khớp | đưa vào ngoại lệ đối soát; không đánh dấu thành công giả |
| E2E-014 | Lập đối soát PARTNER | bản chụp hợp đồng và dòng sổ cái khớp |
| E2E-015 | Duyệt đối soát hai lần | lần hai bị từ chối an toàn hoặc trả kết quả cũ |
| E2E-016 | Đối tác truy cập dữ liệu đối tác khác | bị từ chối và ghi sự kiện bảo mật |
| E2E-017 | GPS cũ/ngoại tuyến | giao diện đánh dấu dữ liệu cũ; áp dụng chính sách đủ điều kiện |
| E2E-018 | Bộ môi giới thông điệp ngừng hoạt động | Outbox lưu sự kiện; gửi tiếp sau khi phục hồi |
| E2E-019 | Chuyển đổi dự phòng CSDL | toàn vẹn giao dịch trong RTO/RPO đã duyệt |
| E2E-020 | Khôi phục bản sao lưu | khôi phục đạt kiểm tra toàn vẹn và chạy thử ứng dụng |

## 2. Kiểm thử bảo mật
- IDOR/BOLA trên mã booking, chuyến, tài xế, đối tác và đối soát.
- Nâng quyền trái phép và vượt phạm vi vai trò.
- Chiếm phiên, hết hạn/thu hồi token, bắt buộc MFA.
- Injection, XSS, CSRF nếu áp dụng, SSRF, tải tệp không an toàn.
- Giả mạo chữ ký webhook và phát lại thông điệp.
- Vượt giới hạn tần suất và dò tìm dữ liệu.
- Rò rỉ dữ liệu cá nhân trong log, tệp xuất, lỗi và phân tích.
- Bí mật xuất hiện trong mã nguồn/tệp build.

## 3. Kiểm thử đối soát tài chính
- Tổng các thành phần cước bằng tổng cước chuyến.
- Thu/hoàn tiền khớp sao kê nhà cung cấp.
- Tổng Nợ bằng tổng Có theo bút toán và loại tiền.
- Tổng gộp, hoa hồng/khấu trừ, điều chỉnh và số ròng PARTNER khớp phiên bản hợp đồng.
- Bút toán đảo tham chiếu bút toán gốc.
- Kiểm thử làm tròn và độ chính xác tiền tệ.
- Khoản tranh chấp/tạm giữ được loại theo chính sách đã duyệt.

## 4. Cổng nghiệm thu
- 100% ca kiểm thử trọng yếu đã duyệt đạt.
- Không còn lỗi bảo mật nghiêm trọng/cao chưa xử lý.
- Không có phân xe trùng hoặc hạch toán tài chính trùng.
- Đối soát tài chính có bằng chứng Tài chính ký xác nhận.
- Đã diễn tập khôi phục và DR.
- UAT có xác nhận của Vận hành, Tài chính, CSKH, Đội xe/Đối tác và IT.
- Hạn chế đã biết có người phụ trách và ngày hoàn thành.
