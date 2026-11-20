# 19 — QUY TẮC ĐIỀU PHỐI CHI TIẾT VÀ XỬ LÝ TRẠNG THÁI
**Trạng thái:** Đặc tả dự thảo; trọng số xếp hạng và ngưỡng vận hành phải được Khối Vận hành phê duyệt.

## 1. Dữ liệu đầu vào điều phối
Thông tin booking bắt buộc: booking_id, thời điểm yêu cầu, kênh đặt, tọa độ/địa chỉ đón, điểm đến nếu có, hạng dịch vụ, số hành khách, yêu cầu hỗ trợ/thiết bị, giờ đón mong muốn, ràng buộc khách hàng và mã báo giá.

Thông tin nguồn xe: mã tài xế/xe, DIRECT/PARTNER, trạng thái sẵn sàng, thời điểm GPS gần nhất, độ chính xác vị trí, điều kiện xe/dịch vụ, tình trạng giấy tờ/giấy phép, ca và trạng thái tài xế, booking/chuyến đang hoạt động, trạng thái khai thác của đối tác.

## 2. Điều kiện loại trừ bắt buộc
Ứng viên bị loại nếu không đạt bất kỳ điều kiện bắt buộc nào:
- Tài khoản hoặc trạng thái khai thác của tài xế không hợp lệ.
- Xe không hoạt động, không sẵn sàng, sai hạng dịch vụ hoặc giấy tờ tuân thủ đã hết hạn/bị khóa.
- Tài xế không sẵn sàng hoặc đang có phân xe/chuyến xung đột.
- Vị trí GPS cũ hơn ngưỡng được phê duyệt.
- Không đáp ứng số hành khách, khả năng hỗ trợ hoặc thiết bị bắt buộc.
- Đối tác/hợp đồng PARTNER bị đình chỉ hoặc không hiệu lực theo dịch vụ/thời gian/khu vực.
- Ứng viên nằm ngoài phạm vi tìm kiếm được cấu hình.
Ngưỡng và ma trận điều kiện pháp lý: Khối Vận hành, Đội xe và Pháp chế xác nhận.

## 3. Chấm điểm ứng viên (có thể cấu hình và giải thích được)
Sau khi qua điều kiện loại trừ, hệ thống tính điểm theo các yếu tố đã duyệt:
- thời gian/quãng đường dự kiến đến điểm đón;
- độ mới và độ tin cậy của GPS;
- mức phù hợp dịch vụ/loại xe;
- thời gian sẵn sàng và cân bằng tải tài xế;
- cân bằng khu vực và ràng buộc vận hành;
- điều kiện đội xe và hợp đồng.
Trọng số đang TBD. Bộ máy phải trả về thành phần điểm và lý do loại để kiểm toán; không được ngầm ưu tiên DIRECT hoặc PARTNER khi chưa có chính sách phê duyệt.

## 4. Quy trình gửi lời mời nhận chuyến
1. Tạo lần điều phối với bản chụp danh sách ứng viên bất biến và phiên bản chính sách.
2. Gửi lời mời cho một hoặc nhiều ứng viên theo phương thức tuần tự/theo nhóm đã duyệt.
3. Mỗi lời mời có thời điểm hết hạn và offer_id duy nhất.
4. Lệnh chấp nhận phải kiểm tra nguyên tử: booking còn được điều phối, lời mời còn hạn, tài xế/xe còn đủ điều kiện và không có phân công xung đột.
5. Lần chấp nhận hợp lệ đầu tiên thắng; các lời mời cạnh tranh được đóng ở trạng thái bị thay thế.
6. Từ chối, hết giờ, không liên lạc được hoặc mất điều kiện phải ghi lý do và kích hoạt chính sách thử lại.
7. Lần thử lại dùng bản chụp nguồn xe mới và ghi số lần thử.
8. Nếu hết số lần tự động, chuyển sang hàng đợi can thiệp thủ công; không tự hủy booking.

## 5. Đồng thời và chống xử lý lặp
- Dùng ràng buộc cơ sở dữ liệu/giao dịch để bảo đảm mỗi booking chỉ có một phân xe đang hiệu lực.
- Áp dụng ràng buộc tương tự cho tài xế và xe nếu chính sách yêu cầu.
- Dùng cơ chế so sánh-và-cập-nhật theo phiên bản/trạng thái booking.
- API chấp nhận yêu cầu phải có khóa idempotency. Gửi lại cùng khóa trả về kết quả ban đầu.
- Nếu nhiều yêu cầu chấp nhận chạy đồng thời cho cùng booking, chỉ một yêu cầu thành công; các yêu cầu khác nhận lỗi xung đột.
- Bộ tiêu thụ hàng đợi phải chịu được việc một thông điệp được giao nhiều lần.

## 6. Điều phối thủ công
Điều phối viên chỉ được chọn ứng viên đã vượt qua các điều kiện bắt buộc. Nếu chọn khác thứ tự điểm mềm, phải nhập mã lý do và ghi chú. Không được bỏ qua điều kiện loại trừ bắt buộc, trừ khi có chính sách khẩn cấp riêng được phê duyệt và kiểm toán.

## 7. Mô hình trạng thái
Booking: DRAFT → REQUESTED → QUOTED (không bắt buộc) → DISPATCHING → ASSIGNED → DRIVER_EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED.  
Trạng thái kết thúc thay thế: CANCELLED, EXPIRED, UNFULFILLED. Chỉ được bỏ qua trạng thái khi quy trình đã duyệt cho phép; máy chủ kiểm tra mọi chuyển trạng thái.

Lời mời: CREATED → SENT → ACCEPTED | DECLINED | EXPIRED | CANCELLED | SUPERSEDED.  
Chuyến đi: CREATED → DRIVER_EN_ROUTE → ARRIVED → STARTED → COMPLETED; ngoại lệ: CANCELLED, DISPUTED, NEEDS_REVIEW.  
Thanh toán: INITIATED → PENDING → SUCCEEDED | FAILED | CANCELLED | REFUNDED | PARTIALLY_REFUNDED | RECONCILIATION_REQUIRED.

## 8. Ma trận xử lý ngoại lệ
| Tình huống | Hệ thống xử lý | Người xử lý |
|---|---|---|
| Không có nguồn xe đủ điều kiện | giữ booking trong hàng đợi ngoại lệ, ghi phạm vi đã tìm | điều phối viên liên hệ khách/áp dụng phương án dự phòng đã duyệt |
| Tài xế không phản hồi | hết hạn lời mời, ghi trạng thái gửi | chỉ can thiệp nếu lỗi lặp lại |
| Tài xế hủy sau khi nhận | giải phóng phân xe an toàn, tạo lần điều phối mới | điều phối viên theo dõi ảnh hưởng khách hàng |
| GPS cũ | loại hoặc giảm điểm theo ngưỡng được duyệt | liên hệ tài xế/kiểm tra vị trí thủ công |
| Khách đổi điểm đón | tăng phiên bản booking, vô hiệu báo giá/lời mời cũ nếu cần | xác nhận lại điều kiện |
| Ứng dụng mất kết nối | xếp hàng dữ liệu an toàn; chặn lệnh trùng không an toàn | tổng đài dự phòng |
| Tranh chấp cước chuyến | tạm giữ phần đối soát liên quan theo chính sách | Tài chính/Vận hành rà soát |

## 9. Chỉ số vận hành
Tỷ lệ chấp nhận lời mời, thời gian phân xe, sai số ETA đón, tỷ lệ không có xe, tỷ lệ tài xế hủy, tỷ lệ can thiệp thủ công, số lần phân trùng (mục tiêu bằng 0), phân bố độ mới GPS. Mục tiêu số cần phê duyệt.
