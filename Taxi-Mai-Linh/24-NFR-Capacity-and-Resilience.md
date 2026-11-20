# 24 — YÊU CẦU PHI CHỨC NĂNG, NĂNG LỰC TẢI VÀ KHẢ NĂNG CHỊU LỖI
**Trạng thái:** Không tự đặt giá trị mục tiêu. Đây là bảng yêu cầu cần đo lường và phê duyệt.

## 1. Danh mục mục tiêu mức dịch vụ cần duyệt
| Hạng mục | Chỉ số đo | Mục tiêu |
|---|---|---|
| Tính sẵn sàng | tỷ lệ sẵn sàng hàng tháng theo hành trình trọng yếu | TBD |
| Độ trễ booking | p95 thời gian phản hồi tạo booking | TBD |
| Độ trễ điều phối | p95 thời gian từ yêu cầu đến lời mời đầu tiên | TBD |
| Độ mới bản đồ | p95 tuổi dữ liệu vị trí hiển thị | TBD |
| Độ tin cậy API | tỷ lệ lỗi 5xx theo endpoint | TBD |
| Độ ổn định ứng dụng | tỷ lệ phiên không bị lỗi/crash | TBD |
| Khôi phục | RTO/RPO theo dịch vụ/nhóm dữ liệu | TBD |
| Hỗ trợ | thời gian phản hồi/khôi phục sự cố | TBD |

## 2. Đầu vào mô hình năng lực tải
Thu thập số trung bình và cao điểm:
- số booking/giờ và số yêu cầu tạo booking đồng thời cao nhất;
- số tài xế hoạt động và chu kỳ cập nhật GPS;
- số lời mời trên mỗi booking và phân bố số lần thử lại;
- số người dùng trung tâm điều hành đồng thời và số luồng bản đồ;
- số sự kiện chuyến đi mỗi giây;
- số callback thanh toán và kích thước lô đối soát;
- dung lượng lưu GPS, nhật ký, bằng chứng và kiểm toán.
Mô hình tải phải tính đến mùa cao điểm/sự kiện và mức tăng trưởng do doanh nghiệp phê duyệt.

## 3. Kịch bản kiểm thử tải
1. Tải ổn định ở khung giờ cao điểm đại diện.
2. Tải gấp 2 và 3 lần dự báo (hệ số cuối cùng do kiến trúc/kinh doanh duyệt).
3. Tăng đột biến booking trong khi nguồn xe hạn chế.
4. Bùng nổ GPS và thông điệp đến trễ/sai thứ tự.
5. Nhà cung cấp bản đồ, SMS/voice và thanh toán phản hồi chậm/hết thời gian.
6. Chuyển đổi dự phòng cơ sở dữ liệu và khôi phục hàng đợi bị tồn.
7. Trung tâm điều hành phát bản đồ/hàng đợi đến nhiều người dùng đồng thời.
8. Chạy lô đối soát với dung lượng lịch sử gần dữ liệu thật.

## 4. Mẫu thiết kế chịu lỗi
- Timeout, số lần thử lại có giới hạn, backoff lũy thừa và độ lệch ngẫu nhiên.
- Cầu dao và cô lập tài nguyên cho nhà cung cấp ngoài.
- Lệnh có tính lặp an toàn và bộ nhận sự kiện chống trùng.
- Mẫu Outbox/Inbox giao dịch.
- Hàng đợi thư lỗi (DLQ) có cảnh báo, phê duyệt phát lại và kiểm toán.
- Suy giảm có kiểm soát: điều phối thủ công và tổng đài dự phòng khi tự động hóa lỗi.
- Không được mất âm thầm sự kiện mốc chuyến, thanh toán hoặc bút toán.

## 5. Sao lưu và khôi phục thảm họa
Xác định RPO/RTO theo mức độ quan trọng. Bản sao lưu phải mã hóa, có bản bất biến/ngoại tuyến nếu khả thi, thông tin xác thực tách biệt, giám sát thành công và kiểm tra khôi phục định kỳ. Sổ tay DR phải bao gồm DNS/mạng, CSDL, bộ môi giới thông điệp, lưu trữ đối tượng, bí mật, tích hợp và truy cập ứng dụng. Một tác vụ sao lưu thành công không chứng minh dữ liệu có thể khôi phục.

## 6. Khả năng quan sát
Nhật ký có cấu trúc với correlation_id; chỉ số theo dịch vụ/endpoint/trạng thái; truy vết phân tán; luồng kiểm toán; kiểm tra tổng hợp. Không ghi dữ liệu cá nhân/bí mật vào log. Cảnh báo khi tạo booking lỗi, hàng đợi điều phối tồn, GPS cũ, lệch đối soát thanh toán, mất cân đối đối soát, tồn hàng đợi, CSDL quá tải, chứng thư hết hạn và sao lưu thất bại.

## 7. Nền tảng bảo mật
Mô hình đe dọa STRIDE; TLS khi truyền; mã hóa khi lưu; quản lý bí mật tập trung; MFA cho vai trò đặc quyền; quy trình phát triển an toàn; quét phụ thuộc/container; SAST/DAST; kiểm thử xâm nhập; SLA vá lỗi; quyền tối thiểu; lưu nhật ký kiểm toán; đánh giá tác động quyền riêng tư. Pháp chế/Tuân thủ phê duyệt nghĩa vụ áp dụng tại Việt Nam về dữ liệu cá nhân, vận tải, thanh toán và lưu trữ.
