# 30 — KẾ HOẠCH CHUYỂN ĐỔI DỮ LIỆU, CHUYỂN HỆ THỐNG VÀ QUAY LUI

## 1. Nguyên tắc chuyển đổi
- Kiểm kê mọi hệ thống nguồn và chủ sở hữu dữ liệu trước khi ánh xạ.
- Chỉ chuyển dữ liệu đã được duyệt và cần cho ngày phát hành.
- Giữ mã định danh nguồn trong bảng đối chiếu.
- Tài chính phải xác nhận độc lập dữ liệu tài chính.
- Che/ẩn danh dữ liệu cá nhân ở môi trường không sản xuất.
- Không thực hiện thay đổi phá hủy dữ liệu nguồn trong buổi diễn tập.

## 2. Nhóm dữ liệu
Khách hàng, tài xế, xe, tổ chức/hợp đồng đối tác, biểu giá, booking đang mở, lịch sử chuyến (nếu cần), mã tham chiếu thanh toán, số dư/khoản đối soát chưa đóng, người dùng/vai trò, dữ liệu danh mục và lịch sử kiểm toán. Mỗi nhóm phải có quyết định đưa vào/loại ra rõ ràng.

## 3. Mẫu ánh xạ dữ liệu
| Thực thể/trường nguồn | Thực thể/trường đích | Quy tắc chuyển đổi | Quy tắc null/mặc định | Chủ sở hữu | Kiểm tra |
|---|---|---|---|---|---|
| TBD | TBD | TBD | không tự đặt mặc định | Chủ dữ liệu | số lượng + mẫu + tổng |

## 4. Trình tự diễn tập
1. Khóa bảng ánh xạ và kịch bản trích xuất.
2. Chụp dữ liệu nguồn và ghi mốc trích xuất.
3. Nạp dữ liệu danh mục/dữ liệu gốc.
4. Nạp hợp đồng/biểu giá cùng ngày hiệu lực và lịch sử phiên bản.
5. Nạp khách hàng/nguồn xe và bảng đối chiếu ID.
6. Nạp dữ liệu vận hành/tài chính đang mở nếu được duyệt.
7. Kiểm tra tham chiếu, trùng lặp, định dạng, múi giờ và ánh xạ trạng thái.
8. Đối soát số bản ghi và tổng tài chính.
9. Chạy kiểm tra nghiệp vụ và lấy xác nhận.
10. Ghi thời lượng, lỗi và kịch bản đã sửa; lặp lại đến khi ổn định.

## 5. Phương án chuyển hệ thống
- Xác định thời gian bảo trì/đóng băng và thông báo các bên.
- Chụp dữ liệu nguồn cuối cùng và sao lưu.
- Áp dụng phần dữ liệu phát sinh sau lần diễn tập thành công gần nhất.
- Chỉ chuyển lưu lượng khi các cổng xác nhận đạt.
- Giám sát booking, điều phối, thanh toán và tích hợp.
- Giữ hệ thống nguồn ở chế độ chỉ đọc trong cửa sổ quyết định quay lui nếu khả thi.

## 6. Điều kiện kích hoạt quay lui
Ví dụ cần ngưỡng được duyệt trước: tạo booking thất bại, lỗi tranh chấp phân xe, mất cân đối tài chính, lỗi bảo mật nghiêm trọng, hỏng dữ liệu, nhà cung cấp trọng yếu không hoạt động hoặc khôi phục/DR thất bại. Ngưỡng và người có quyền quyết định đang TBD.

## 7. An toàn khi quay lui
Không đảo ngược mù quáng các tác động thanh toán/thông báo bên ngoài. Dùng giao dịch bù trừ và đối soát. Giữ nguyên bằng chứng kiểm toán. Kế hoạch quay lui phải nêu rõ quay lại DNS/ứng dụng, tương thích CSDL, phát lại/giữ sự kiện, xử lý callback nhà cung cấp và thông báo khách/tài xế.

## 8. Ký xác nhận
Chủ sở hữu dữ liệu, Tài chính, Vận hành, IT và Bảo mật ký xác nhận đối soát và danh sách chuyển hệ thống trước khi quyết định PHÁT HÀNH.
