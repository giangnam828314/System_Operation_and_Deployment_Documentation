# 18 — BẢN THIẾT KẾ TỔNG THỂ TRIỂN KHAI
**Phiên bản:** 1.0-dự thảo | **Trạng thái:** Cơ sở lập kế hoạch phát triển, chờ phê duyệt nghiệp vụ và kiến trúc

## 1. Mục đích và ranh giới triển khai
Tài liệu chuyển các yêu cầu nghiệp vụ và thiết kế phiên bản 0.9 thành các nhóm công việc có thể giao cho đội triển khai. Tài liệu không cho phép đưa cấu hình chưa được phê duyệt vào môi trường vận hành chính thức.

### Phạm vi
- Tiếp nhận đặt xe thống nhất từ tổng đài, ứng dụng khách hàng, web và các kênh đối tác/API được duyệt.
- Điều phối thống nhất nguồn xe DIRECT và PARTNER.
- Quy trình ứng dụng tài xế, vòng đời chuyến đi, tính cước và ghi nhận thanh toán.
- Hạch toán hợp đồng/tài chính riêng cho DIRECT và PARTNER.
- Trung tâm điều hành, quản trị khách hàng/tài xế/xe/đối tác, báo cáo, nhật ký kiểm toán và tích hợp.

### Ngoài phạm vi cho đến khi được phê duyệt
- Định giá động tự động, phụ thu theo nhu cầu, khách hàng thân thiết, ví lưu trữ tiền, nhà cung cấp thanh toán không tiền mặt, nhà cung cấp hóa đơn điện tử, quy tắc tiếp nhận đội xe bên thứ ba, chuyến xuyên biên giới.
- Mọi diễn giải pháp lý/hợp đồng chưa được Pháp chế, Tài chính và Vận hành Mai Linh phê duyệt.

## 2. Các miền nghiệp vụ và quyền sở hữu
| Miền nghiệp vụ | Chịu trách nhiệm quản lý | Không chịu trách nhiệm |
|---|---|---|
| Định danh và phân quyền | người dùng, vai trò, phiên đăng nhập, chính sách truy cập | quan hệ lao động tài xế/hợp đồng đối tác |
| Khách hàng | hồ sơ, chấp thuận, địa điểm lưu | tính giá chuyến |
| Danh mục nguồn xe | loại đội xe, tài xế, xe, điều kiện khai thác | xếp hạng điều phối |
| Đặt xe | yêu cầu, kênh, điểm đón/trả, vòng đời | đối soát |
| Điều phối | đánh giá xe phù hợp, lời mời, phân xe | sổ cái cước |
| Chuyến đi | phân xe được chấp nhận, mốc trạng thái, bằng chứng chuyến | tính tiền phải trả đối tác |
| Định giá | phiên bản biểu giá, báo giá, tính cước | thu tiền |
| Thanh toán | yêu cầu/lần thanh toán, trạng thái, đối soát | soạn biểu giá |
| Sổ cái tài chính | bút toán bất biến, điều chỉnh | điều phối vận hành |
| Đối soát đối tác | bảng kê, khoản khấu trừ, khoản phải trả | bảng lương trực tiếp |
| Thông báo | mẫu tin, lần gửi | nguồn dữ liệu trạng thái chuyến |
| Báo cáo | dữ liệu đọc và số liệu tổng hợp | ghi giao dịch |

Mỗi miền tự quản lý dữ liệu của mình. Thay đổi giữa các miền phải đi qua API/sự kiện có phiên bản, không ghi trực tiếp vào cơ sở dữ liệu của miền khác.

## 3. Mã định danh chuẩn
Khuyến nghị dùng mã UUID/ULID không mang ý nghĩa nghiệp vụ. Mã tra cứu booking hiển thị cho người dùng phải tách biệt và không để lộ ID tuần tự của cơ sở dữ liệu.
- booking_id, dispatch_id, offer_id, assignment_id, trip_id
- customer_id, driver_id, vehicle_id, fleet_owner_id, partner_id
- tariff_version_id, quote_id, payment_id, ledger_entry_id, settlement_id
- correlation_id, idempotency_key, event_id

Định dạng và kiểu dữ liệu vật lý là quyết định kiến trúc; UUID chỉ là khuyến nghị, chưa phải ràng buộc đã chốt.

## 4. Các nguyên tắc bất biến bắt buộc
1. Kênh đặt xe không quyết định loại đội xe.
2. Mỗi lần phân xe phải tham chiếu một tài xế và một xe đủ điều kiện tại thời điểm phân.
3. Mỗi booking chỉ có tối đa một phân xe đang hiệu lực; mỗi tài xế/xe chỉ có tối đa một chuyến/phân xe đang hoạt động, trừ khi mô hình nhiều điểm dừng được duyệt có quy định khác.
4. Báo giá lưu phiên bản biểu giá và dữ liệu đầu vào đã sử dụng.
5. Dữ liệu tài chính của chuyến đã hoàn tất không được sửa trực tiếp; sửa sai bằng bút toán bù trừ.
6. Đối soát PARTNER dựa trên bản chụp hợp đồng đã duyệt và các bút toán sổ cái, không dựa trên cấu hình hợp đồng hiện tại có thể thay đổi.
7. Mọi chuyển trạng thái phải lưu người thực hiện, thời điểm, lý do, mã tương quan và nguồn phát sinh.
8. Lệnh gửi lại phải có tính lặp an toàn.
9. Phản hồi từ nhà cung cấp thanh toán phải được xác thực, chống trùng và đối soát.
10. Không đưa hệ thống vào vận hành chính thức khi các quyết định chính sách trọng yếu còn bỏ ngỏ.

## 5. Mô hình kết nối dịch vụ tham chiếu
Ứng dụng người dùng → Cổng API/BFF → dịch vụ nghiệp vụ → cơ sở dữ liệu giao dịch; sự kiện bất đồng bộ → bộ môi giới thông điệp → dịch vụ thông báo/tích hợp/báo cáo. Có thể dùng chỉ mục địa lý riêng để tìm xe, nhưng Danh mục nguồn xe vẫn là nơi có thẩm quyền xác nhận điều kiện tài xế/xe.
- Gọi đồng bộ: lệnh cần kiểm tra và trả kết quả ngay.
- Sự kiện bất đồng bộ: thông báo, phân tích dữ liệu, xuất dữ liệu đối tác, tích hợp không cần phản hồi tức thì.
- Mẫu Outbox: bảo đảm sự kiện được phát sau khi giao dịch đã ghi thành công.
- Mẫu Inbox/chống trùng: ngăn xử lý tác động phụ nhiều lần.
- Saga/bộ điều phối quy trình: quản lý luồng đặt xe–điều phối–chuyến đi–thanh toán qua nhiều miền.

## 6. Môi trường và cổng phát hành
| Môi trường | Mục đích | Dữ liệu |
|---|---|---|
| Máy phát triển cá nhân | vòng lặp phát triển | dữ liệu giả lập |
| DEV | tích hợp dùng chung | dữ liệu giả lập |
| SIT | kiểm thử liên dịch vụ | dữ liệu giả lập/đã ẩn danh |
| UAT | nghiệp vụ nghiệm thu | dữ liệu thử đã được duyệt/ẩn danh |
| Staging | diễn tập gần giống thật | dữ liệu đã ẩn danh |
| Production | vận hành thật | chỉ dữ liệu sản xuất |

Thông tin bí mật phải tách theo từng môi trường. Không đưa dữ liệu cá nhân thật hoặc khóa thanh toán sản xuất vào môi trường thấp hơn.

## 7. Tiêu chí sẵn sàng phát triển
- Quy tắc nghiệp vụ đã duyệt hoặc được loại rõ khỏi bản phát hành đầu tiên.
- Lược đồ API và hợp đồng sự kiện có quản lý phiên bản.
- ERD và kịch bản chuyển đổi cơ sở dữ liệu đã được rà soát.
- Giao diện có trạng thái đang tải, trống, lỗi, không đủ quyền, mất kết nối và xung đột.
- Tiêu chí nghiệm thu có thể kiểm thử.
- Giám sát, nhật ký kiểm toán và phương án quay lui đã được thiết kế.
- Mỗi mục TBD có người phụ trách và ngày dự kiến quyết định.

## 8. Chia lát phát hành đề xuất
R0 Nền tảng: định danh, kiểm toán, dữ liệu danh mục, danh mục nguồn xe.  
R1 Đặt xe: tiếp nhận, khung báo giá, vòng đời booking.  
R2 Điều phối: tìm xe, gửi lời mời, nhận/từ chối, điều phối thủ công.  
R3 Chuyến đi: ứng dụng tài xế, mốc chuyến, bằng chứng GPS.  
R4 Cước/thanh toán: bộ máy biểu giá, ghi nhận thanh toán, đối soát.  
R5 Tài chính: hạch toán DIRECT, bản chụp hợp đồng PARTNER và đối soát.  
R6 Vận hành: trung tâm điều hành, báo cáo, cảnh báo, sổ tay xử lý, gia cố hệ thống.  
Thứ tự trên là đề xuất; phụ thuộc và ưu tiên kinh doanh cần được xác nhận.

## 9. Điều kiện không phát hành
Không phát hành nếu còn lỗi bảo mật nghiêm trọng chưa xử lý; chưa thử khôi phục dữ liệu; đối soát thanh toán/sổ cái thất bại; vẫn có thể phân trùng xe; không thể khôi phục điều phối thủ công; kiểm thử nghiệm thu trọng yếu thất bại; hoặc chưa có người chịu trách nhiệm hỗ trợ/on-call.
