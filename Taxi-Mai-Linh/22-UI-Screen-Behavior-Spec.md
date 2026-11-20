# 22 — ĐẶC TẢ HÀNH VI CÁC MÀN HÌNH
Các hình SVG hiện tại là bản phác thảo ý tưởng. Tài liệu này xác định hành vi cần có trước khi thiết kế giao diện chi tiết.

## 1. Tiêu chuẩn giao diện chung
- Trung tâm điều hành trên máy tính tối ưu từ độ phân giải 1366×768; quy tắc co giãn cần xác nhận.
- Ứng dụng tài xế thiết kế theo phiên bản Android/iOS được hỗ trợ và phê duyệt.
- Mỗi màn hình phải có trạng thái đang tải, trống, lỗi, mất kết nối, không đủ quyền và dữ liệu cũ.
- Thao tác phá hủy dữ liệu hoặc tài chính phải xác nhận và nhập lý do.
- Hiển thị giờ địa phương nhất quán; lưu UTC.
- Điều hướng và che dữ liệu theo vai trò/quyền.
- Ký hiệu bản đồ phải thể hiện trạng thái, lần cập nhật gần nhất và nhãn không phụ thuộc màu sắc.

## 2. Danh mục màn hình
| Mã | Màn hình | Vai trò chính | Thao tác chính |
|---|---|---|---|
| UI-001 | Đăng nhập/MFA | mọi người dùng | đăng nhập, khôi phục, xác thực nhiều lớp |
| UI-002 | Trung tâm điều hành | điều phối/giám sát | hàng đợi, bản đồ, phân xe, chuyển cấp |
| UI-003 | Tiếp nhận booking | tổng đài | tạo/sửa/xác nhận booking |
| UI-004 | Chi tiết booking | vận hành | xem dòng thời gian, hủy, điều phối lại |
| UI-005 | Danh mục tài xế/xe | quản lý đội xe | tạo, xác minh, tạm khóa |
| UI-006 | Danh mục đối tác | quản lý đối tác/tài chính | quản lý hồ sơ/trạng thái đối tác |
| UI-007 | Quản lý biểu giá | quản trị giá | soạn, mô phỏng, duyệt, kích hoạt |
| UI-008 | Chi tiết chuyến | vận hành | mốc chuyến, bằng chứng, tranh chấp |
| UI-009 | Thanh toán/đối soát | tài chính | rà soát, đối soát, yêu cầu hoàn tiền |
| UI-010 | Đối soát đối tác | tài chính | lập, rà soát, duyệt, xuất |
| UI-011 | Báo cáo | quản lý | lọc, xuất, xem chi tiết |
| UI-012 | Người dùng/vai trò/kiểm toán | quản trị bảo mật | cấp/thu quyền, xem kiểm toán |
| UI-013 | Trang chủ tài xế | tài xế | trực tuyến/ngoại tuyến, xem trạng thái |
| UI-014 | Chi tiết lời mời | tài xế | chấp nhận/từ chối, đồng hồ đếm ngược |
| UI-015 | Chuyến đang thực hiện | tài xế | dẫn đường, đến điểm đón, bắt đầu, hoàn tất |

## 3. Trung tâm điều hành (UI-002)
Bố cục: thanh đầu trang (môi trường/giờ/người dùng), dải KPI, cột trái là hàng đợi ngoại lệ/điều phối, giữa là bản đồ, bên phải là chi tiết booking/ứng viên, dưới cùng là dòng thời gian sự kiện.

Bộ lọc: khu vực, tuổi booking, hạng dịch vụ, kênh, trạng thái, loại đội xe, đối tác (theo quyền).
Cột hàng đợi: mã tra cứu, giờ yêu cầu, khu vực đón, thời gian chờ, dịch vụ, trạng thái, nguồn lực được gán, ngoại lệ.
Thao tác: mở chi tiết, bắt đầu điều phối, chọn ứng viên thủ công, phân lại, chuyển cấp, hủy (cần quyền và lý do).

Chốt an toàn: không hiển thị dữ liệu cá nhân khách hàng khi không có quyền; chọn thủ công không được bỏ qua điều kiện bắt buộc; GPS cũ phải được đánh dấu rõ.
Nghiệm thu: hàng đợi/bản đồ phản ánh dữ liệu máy chủ; thao tác đồng thời dùng kiểm soát phiên bản; mọi ghi đè đều được kiểm toán.

## 4. Màn hình tiếp nhận booking (UI-003)
Các phần: tìm/tạo khách hàng; điểm đón/điểm trả; hạng dịch vụ; số hành khách/yêu cầu; giờ đón; ghi chú; tóm tắt báo giá; metadata kênh/nguồn.
Kiểm tra: bắt buộc điểm đón; thể hiện độ tin cậy địa chỉ/tọa độ; số khách trong giới hạn dịch vụ đã duyệt; cảnh báo booking trùng; lưu thông tin đồng ý liên hệ khi cần.
Thao tác: lưu nháp, yêu cầu báo giá, xác nhận booking, hủy nháp.
Nếu sửa địa chỉ sau khi báo giá: vô hiệu báo giá cũ hoặc tính lại. Không âm thầm dùng giá đã lỗi thời.

## 5. Màn hình ứng dụng tài xế (UI-013–015)
Trang chủ: bật/tắt trực tuyến, kết nối, quyền GPS, xe và ca làm việc.
Lời mời: tóm tắt điểm đón, dịch vụ, cước ước tính nếu chính sách cho phép, đồng hồ hết hạn, nút nhận/từ chối và lý do từ chối.
Chuyến đang chạy: lộ trình, gọi/che số liên hệ, đến điểm đón, bắt đầu, hoàn tất; thao tác khẩn cấp/hỗ trợ.
Ngoại tuyến: phân biệt rõ thao tác đang xếp hàng và thao tác đã được máy chủ xác nhận. Không hiển thị nhận/bắt đầu/hoàn tất là thành công trước khi máy chủ phản hồi; lệnh gửi lại dùng khóa chống trùng.
An toàn: giảm thao tác khi xe đang chạy; không bắt nhập văn bản khi lái xe; cần Pháp chế rà soát.

## 6. Màn hình tài chính
Biểu giá: bản nháp có phiên bản, thời hạn hiệu lực, bảng thành phần, mô phỏng, phê duyệt hai cấp, kích hoạt và quay lại bằng phiên bản mới (không sửa phiên bản lịch sử).
Thanh toán: trạng thái nhà cung cấp, trạng thái nội bộ, số tiền, tiền tệ, mã tham chiếu, trạng thái đối soát; hoàn tiền là lệnh riêng có phân quyền.
Đối soát: kỳ, bản chụp hợp đồng, tổng gộp, khấu trừ, điều chỉnh, số ròng phải trả, tranh chấp, luồng phê duyệt, mã kiểm tra tệp xuất.

## 7. Hồ sơ bàn giao thiết kế
Các trang Figma theo vai trò; thư viện thành phần; bộ quy chuẩn giao diện; điểm ngắt màn hình; nguyên mẫu tương tác; kiểm tra từng trường; khả năng tiếp cận; bản địa hóa; ma trận quyền; liên kết API; thiết kế trạng thái lỗi. Hình SVG hiện tại không phải giao diện sản phẩm hoàn chỉnh.
