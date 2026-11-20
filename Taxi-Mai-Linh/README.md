# TÀI LIỆU PHÂN TÍCH, THIẾT KẾ VÀ TRIỂN KHAI HỆ THỐNG TAXI MAI LINH

> **Phiên bản:** 1.0-dự thảo · **Ngày cập nhật:** 14/11/2026  
> **Mô hình vận hành:** DIRECT + PARTNER trên một nền tảng điều phối thống nhất.  
> **Trạng thái:** Cơ sở lập kế hoạch phát triển — chờ phê duyệt nghiệp vụ, tài chính, pháp chế và kiến trúc.


## Tài liệu trục chính — đọc trước khi triển khai

- [00A — Thiết kế tổng thể và luồng vận hành hệ thống Taxi Mai Linh](./00A-Thiet-ke-tong-the-va-luong-van-hanh.md): luồng đầu cuối, Swimlane, chuỗi dữ liệu, trạng thái, ngoại lệ, ranh giới DIRECT/PARTNER và thứ tự phân rã Module.
- [00B — State Machine và quy tắc chuyển trạng thái](./00B-State-Machine-va-quy-tac-chuyen-trang-thai.md): vòng đời Booking, DispatchRun, Offer, Assignment, Trip, Payment, Settlement; guard, concurrency, ánh xạ trạng thái và test.

## Mục tiêu và đối tượng sử dụng

Bộ tài liệu dành cho Ban điều hành, Khối Vận hành, Tài chính, Pháp chế, BA/PO, UX/UI, Kiến trúc sư, Lập trình viên, QA và IT Operations để thống nhất yêu cầu, thiết kế, phát triển, kiểm thử, nghiệm thu và vận hành.

Bản 1.0 bổ sung đặc tả triển khai cho bộ tài liệu nền tảng 0.9. Đây chưa phải xác nhận sẵn sàng vận hành chính thức: chính sách nghiệp vụ, thông số tải, nhà cung cấp, yêu cầu pháp lý/bảo mật và dữ liệu nguồn vẫn phải được chủ sở hữu phê duyệt.

## Nguyên tắc nền tảng

1. Một nền tảng điều phối phục vụ xe/tài xế Mai Linh quản lý trực tiếp (**DIRECT**) và xe/tài xế đối tác (**PARTNER**).
2. `booking_channel` khác `fleet_type`. Kênh TỔNG ĐÀI/ỨNG DỤNG/WEB/API không quyết định nguồn xe DIRECT/PARTNER.
3. Có thể dùng chung bộ máy điều phối; hợp đồng, doanh thu, công nợ và đối soát phải phân biệt.
4. Cước và đối soát lưu bản chụp phiên bản biểu giá/hợp đồng/chính sách tại thời điểm nghiệp vụ.
5. Sự kiện vận hành và sổ cái phải truy vết; không xóa lịch sử để sửa dữ liệu.
6. Mọi lệnh ghi quan trọng phải có kiểm tra quyền, kiểm toán, chống xử lý lặp và kiểm soát đồng thời.
7. Chính sách chưa được phê duyệt phải ghi TBD; không dùng giá trị giả định làm cấu hình vận hành chính thức.

## Danh mục tài liệu nền tảng (00–17)

| # | Tài liệu | Phạm vi |
|---|---|---|
| 00 | [Tổng quan dành cho lãnh đạo](./00-Executive-Overview.md) | Mục tiêu, phạm vi, nguyên tắc, cổng phê duyệt |
| 01 | [Yêu cầu nghiệp vụ](./01-BRD-Business-Requirements.md) | Mục tiêu, bên liên quan, năng lực, KPI |
| 02 | [Mô hình vận hành](./02-Operating-Model.md) | DIRECT/PARTNER, vai trò, điều kiện khai thác |
| 03 | [Quy trình nghiệp vụ đầu cuối](./03-End-to-End-Workflows.md) | Đặt xe, điều phối, chuyến đi, thanh toán, đối soát, sự cố |
| 04 | [Đặc tả yêu cầu phần mềm](./04-SRS-Functional-Requirements.md) | Mã yêu cầu, quy tắc nghiệp vụ, trạng thái |
| 05 | [Đặc tả giao diện và trải nghiệm](./05-UIUX-Specification.md) | Danh mục màn hình, trường dữ liệu, tương tác, trạng thái |
| 06 | [Điều phối và trung tâm điều hành](./06-Dispatch-Control-Tower.md) | Tìm xe, bản đồ, hàng đợi, ngoại lệ |
| 07 | [Ứng dụng tài xế](./07-Driver-App.md) | Hành trình di động, trạng thái, ngoại tuyến/an toàn |
| 08 | [Cước, thanh toán và đối soát](./08-Fare-Payment-Settlement.md) | Biểu giá, thanh toán, sổ cái, đối soát đối tác |
| 09 | [Mô hình dữ liệu và ERD](./09-Data-Model-ERD.md) | Thực thể, quan hệ, ràng buộc |
| 10 | [API và tích hợp](./10-API-Integration.md) | API, sự kiện, hợp đồng, tích hợp |
| 11 | [Kiến trúc, NFR và bảo mật](./11-Architecture-NFR-Security.md) | Kiến trúc logic, chịu lỗi, bảo mật, DR |
| 12 | [Kiểm thử và nghiệm thu](./12-QA-UAT-Acceptance.md) | Chiến lược kiểm thử, ca kiểm thử, cổng nghiệm thu |
| 13 | [Kế hoạch phát triển và triển khai](./13-Delivery-Deployment-Plan.md) | Nhóm công việc, môi trường, chuyển đổi dữ liệu |
| 14 | [Vận hành và sổ tay xử lý](./14-Operations-Runbooks.md) | Giám sát, sự cố, sao lưu/khôi phục |
| 15 | [Sổ đăng ký quyết định](./15-Decision-Register.md) | Nội dung TBD, chủ sở hữu, mức ưu tiên, phê duyệt |
| 16 | [Ma trận truy vết](./16-Traceability-Matrix.md) | Mục tiêu → yêu cầu → luồng → giao diện/API → kiểm thử |
| 17 | [Bảng thuật ngữ](./17-Glossary.md) | Thuật ngữ thống nhất |

## Đặc tả bổ sung cho giai đoạn sẵn sàng phát triển (18–30)

| # | Tài liệu | Nội dung |
|---|---|---|
| 18 | [Thiết kế tổng thể triển khai](./18-Implementation-Blueprint.md) | Miền nghiệp vụ, nguyên tắc bất biến, kiến trúc, lát cắt phát hành |
| 19 | [Quy tắc điều phối chi tiết](./19-Detailed-Dispatch-Rules.md) | Điều kiện loại trừ, chấm điểm, lời mời, đồng thời, ngoại lệ |
| 20 | [Hợp đồng API và ví dụ](./20-API-Contracts-Examples.md) | Danh mục endpoint, JSON, cấu trúc lỗi/sự kiện |
| 21 | [Từ điển dữ liệu và ràng buộc](./21-Data-Dictionary-and-Constraints.md) | Trường thực thể, chỉ mục, tiền tệ, vòng đời, toàn vẹn |
| 22 | [Đặc tả hành vi màn hình](./22-UI-Screen-Behavior-Spec.md) | Danh mục màn hình, thao tác, kiểm tra, trạng thái |
| 23 | [Ma trận vai trò và phân quyền](./23-Role-Permission-Matrix.md) | Phạm vi vai trò, thao tác nhạy cảm, người lập/người duyệt |
| 24 | [NFR, năng lực tải và chịu lỗi](./24-NFR-Capacity-and-Resilience.md) | Mục tiêu dịch vụ, dữ liệu tải, kiểm thử tải, DR, bảo mật |
| 25 | [Danh mục kiểm thử và nghiệm thu](./25-QA-Test-Catalog.md) | Đầu-cuối, bảo mật, đối soát tài chính, nghiệm thu |
| 26 | [Danh sách công việc và kế hoạch Sprint](./26-Delivery-Backlog-and-Sprint-Plan.md) | Nhóm chức năng, phụ thuộc, câu chuyện người dùng, tiêu chí hoàn thành |
| 27 | [Danh sách sẵn sàng vận hành chính thức](./27-Production-Readiness-Checklist.md) | Kiểm tra phát hành và biên bản quyết định |
| 28 | [Tài liệu họp chốt quyết định](./28-Decision-Workshop-Pack.md) | Quyết định nghiệp vụ, người phụ trách, bằng chứng, phê duyệt |
| 29 | [Ma trận tích hợp và nhà cung cấp](./29-Integration-and-Provider-Matrix.md) | Yêu cầu nhà cung cấp, xử lý lỗi, tiếp nhận tích hợp |
| 30 | [Kế hoạch chuyển đổi dữ liệu và quay lui](./30-Data-Migration-and-Cutover-Plan.md) | Ánh xạ, diễn tập, chuyển hệ thống, quay lui |

## Hình ảnh và sơ đồ

- [Sơ đồ bối cảnh hệ thống](./assets/01-system-context.svg)
- [Bản phác thảo trung tâm điều hành](./assets/02-control-tower-wireframe.svg)
- [Bản phác thảo màn hình tiếp nhận tổng đài](./assets/03-call-center-booking-wireframe.svg)
- [Bản phác thảo ứng dụng tài xế](./assets/04-driver-app-wireframe.svg)
- [Sơ đồ kiến trúc hệ thống logic](./assets/05-logical-architecture.svg)
- [Sơ đồ tuần tự đặt xe – điều phối – chuyến đi](./assets/06-booking-dispatch-trip-sequence.svg)

Các hình là bản phác thảo SVG có thể chỉnh sửa, không phải giao diện Figma đã nghiệm thu hoặc bản đồ/GPS trực tiếp. Số liệu trên hình chỉ là dữ liệu minh họa.

## Cách sử dụng theo vai trò

- Ban điều hành/Vận hành/Tài chính/Pháp chế: xác nhận tài liệu 00–03, 08, 15 và 28.
- BA/PO: quản lý mã yêu cầu tại 04, 16; bổ sung tiêu chí nghiệm thu và truy vết quyết định.
- UX/UI: dùng 05–07, 22 để dựng nguyên mẫu Figma đầy đủ trạng thái.
- Kiến trúc sư/Lập trình viên: dùng 09–11, 18–21, 23–24, 29 để chốt thiết kế vật lý, OpenAPI và lược đồ dữ liệu.
- QA: dùng 12, 16, 25 để xây bộ kiểm thử và bằng chứng.
- IT Operations: dùng 13–14, 24, 27, 30 để chuẩn bị môi trường, DR, chuyển đổi và vận hành.

## Các cổng phê duyệt

1. **Cơ sở nghiệp vụ:** nghiệp vụ DIRECT/PARTNER, đặt xe, điều phối, cước, hủy, thanh toán và đối soát được ký duyệt.
2. **Cơ sở giải pháp:** kiến trúc, ERD vật lý, OpenAPI, mô hình bảo mật, NFR và nhà cung cấp được duyệt.
3. **Sẵn sàng phát triển:** mỗi hạng mục có tiêu chí nghiệm thu, trạng thái UX, phụ thuộc API/dữ liệu và độ bao phủ kiểm thử.
4. **Sẵn sàng UAT:** môi trường, dữ liệu thử, ca kiểm thử và người nghiệm thu được xác nhận.
5. **Sẵn sàng vận hành chính thức:** danh sách kiểm tra mục 27 đạt; khôi phục/DR đã thử; tài chính đã đối soát; hỗ trợ/on-call sẵn sàng.

## Trạng thái tài liệu

Dự thảo → Đang rà soát → Đã phê duyệt → Đường cơ sở. Khi đã xác lập đường cơ sở, mọi thay đổi phải qua yêu cầu thay đổi và đánh giá ảnh hưởng.

> **Không tự giả định:** biểu giá, phụ phí, phí hủy/không có mặt, tỷ lệ chia PARTNER, ưu tiên DIRECT/PARTNER, bán kính tìm xe, thời hạn lời mời, SLA/KPI, tải thiết kế, RPO/RTO, quy định lưu dữ liệu và nhà cung cấp tích hợp. Các nội dung này phải được chủ sở hữu phê duyệt trước khi vận hành chính thức.
