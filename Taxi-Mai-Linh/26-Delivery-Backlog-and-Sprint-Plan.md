# 26 — DANH SÁCH CÔNG VIỆC PHÁT TRIỂN VÀ KẾ HOẠCH SPRINT
**Cơ sở lập kế hoạch:** phân rã công việc đề xuất, chưa phải lịch cam kết. Chưa có dữ liệu năng lực đội dự án và thời gian cung ứng của nhà cung cấp.

## Phân rã theo nhóm chức năng lớn
| Nhóm | Phạm vi | Phụ thuộc | Bằng chứng hoàn thành |
|---|---|---|---|
| EP-01 Nền tảng | kho mã, CI/CD, môi trường, bí mật, giám sát | kiến trúc | dịch vụ triển khai được, bằng chứng pipeline |
| EP-02 Định danh và phân quyền | đăng nhập, MFA, vai trò, phạm vi, kiểm toán | EP-01 | kiểm thử phân quyền |
| EP-03 Danh mục nguồn xe | tài xế, xe, đối tác, điều kiện | EP-02 | quy trình dữ liệu gốc được duyệt |
| EP-04 Đặt xe | kênh, điểm dừng, kiểm tra, vòng đời | EP-02 | kiểm thử booking đầu-cuối |
| EP-05 Định giá | phiên bản biểu giá, báo giá, mô phỏng | quyết định chính sách | ví dụ được Tài chính duyệt |
| EP-06 Điều phối | ứng viên, lời mời, nhận, thử lại, dự phòng thủ công | EP-03/04 | kiểm thử tranh chấp/đồng thời |
| EP-07 Ứng dụng tài xế | trực tuyến, lời mời, mốc chuyến, ngoại tuyến | EP-03/06 | kiểm thử thiết bị/UAT |
| EP-08 Chuyến đi | trạng thái, bằng chứng, ngoại lệ | EP-06/07 | kiểm thử vòng đời chuyến |
| EP-09 Thanh toán | thu tiền, callback, hoàn tiền, đối soát | EP-05/08 | môi trường thử nhà cung cấp và đối soát |
| EP-10 Sổ cái/đối soát | bút toán, bản chụp hợp đồng PARTNER, bảng kê | EP-09 | Tài chính ký duyệt |
| EP-11 Trung tâm điều hành | hàng đợi, bản đồ, bộ lọc, điều phối thủ công | EP-04/06/08 | UAT điều phối viên |
| EP-12 Báo cáo | dữ liệu đọc vận hành/tài chính | sự kiện nghiệp vụ | đối chiếu chỉ số |
| EP-13 Gia cố và phát hành | tải, bảo mật, DR, chuyển đổi, hỗ trợ | tất cả | hồ sơ quyết định phát hành |

## Quy tắc lập kế hoạch Sprint
- Thời lượng Sprint và năng lực thực tế TBD sau khi đánh giá đội dự án.
- Ưu tiên lát cắt dọc có thể kiểm thử đầu-cuối.
- Không đưa công việc vào Sprint nếu thiếu tiêu chí nghiệm thu, phụ thuộc API/dữ liệu và trạng thái UX.
- Mỗi hạng mục phải tính đến bảo mật, giám sát và kiểm toán.
- Định nghĩa hoàn thành: rà soát mã, kiểm thử tự động, kiểm thử hợp đồng, rà soát migration, telemetry, tài liệu, bằng chứng QA và không còn lỗi nghiêm trọng.

## Ví dụ câu chuyện người dùng
- Là nhân viên tổng đài, tôi có thể tạo booking và nhận mã tra cứu ổn định.
- Là điều phối viên, tôi xem được ứng viên đủ điều kiện và lý do ứng viên bị loại.
- Là tài xế, tôi nhận lời mời còn hạn và chỉ thấy phân xe đã được máy chủ xác nhận.
- Là nhân viên Tài chính, tôi đối soát tổng thanh toán và sổ cái theo chuyến/kỳ.
- Là nhân viên Tài chính, tôi lập bảng kê đối tác theo đúng phiên bản hợp đồng áp dụng cho từng chuyến.
- Là kiểm toán viên, tôi truy được ai đã thay đổi booking, biểu giá, thanh toán hoặc đối soát và vì sao.

## Đầu ra cần có khi lập kế hoạch
Danh sách nhân sự/năng lực, thời gian cung ứng của nhà cung cấp, danh mục dữ liệu chuyển đổi, phạm vi MVP, năng lực Sprint, bản đồ phụ thuộc, lịch triển khai, sổ đăng ký rủi ro và phê duyệt ngân sách.
