# 23 — MA TRẬN VAI TRÒ VÀ PHÂN QUYỀN
**Nguyên tắc:** quyền tối thiểu cần thiết, tách biệt nhiệm vụ, giới hạn phạm vi theo chi nhánh/khu vực/đối tác và kiểm toán các thao tác đặc quyền.

| Năng lực | Tổng đài | Điều phối | Giám sát | Quản lý đội xe | Tài chính | Người dùng đối tác | Quản trị bảo mật |
|---|---:|---:|---:|---:|---:|---:|---:|
| Tạo booking | Có | Giới hạn | Có | Không | Không | Phạm vi riêng TBD | Không |
| Xem dữ liệu cá nhân khách | Tối thiểu | Tối thiểu | Theo phạm vi | Không | Đã che | Chỉ booking của mình TBD | Mặc định không |
| Xem hàng đợi điều phối | Không | Có | Có | Không | Không | Phạm vi riêng TBD | Không |
| Phân xe thủ công | Không | Có | Duyệt ngoại lệ | Không | Không | Không | Không |
| Hủy booking | Đề nghị | Theo chính sách | Duyệt ngoại lệ | Không | Không | Yêu cầu của mình TBD | Không |
| Quản lý tài xế/xe | Không | Chỉ xem | Chỉ xem | Có | Xem giới hạn | Đội xe của mình TBD | Không |
| Quản lý biểu giá | Không | Không | Chỉ xem | Không | Rà soát | Không | Không |
| Duyệt biểu giá | Không | Không | Người duyệt nghiệp vụ TBD | Không | Người duyệt tài chính TBD | Không | Không |
| Xem thanh toán | Không | Giới hạn | Chỉ xem | Không | Có | Bảng kê của mình TBD | Không |
| Hoàn tiền | Không | Không | Đề nghị/duyệt TBD | Không | Kiểm soát hai cấp TBD | Không | Không |
| Lập đối soát | Không | Không | Chỉ xem | Không | Có | Xem bảng kê của mình TBD | Không |
| Duyệt đối soát | Không | Không | Không | Không | Người duyệt riêng TBD | Không | Không |
| Quản lý người dùng/vai trò | Không | Không | Không | Không | Không | Không | Có |
| Xem nhật ký kiểm toán | Không | Theo phạm vi | Theo phạm vi | Theo phạm vi | Phạm vi tài chính | Phạm vi của mình TBD | Có |

## Quy tắc phân quyền
- Bắt buộc kiểm tra quyền ở máy chủ; ẩn nút trên giao diện không phải là biện pháp bảo mật.
- Kiểm tra phạm vi chi nhánh/đối tác/khu vực ở cả truy vấn và lệnh cập nhật.
- Ghi nhật ký các lần đọc dữ liệu nhạy cảm theo chính sách.
- Quyền truy cập khẩn cấp phải có thời hạn, lý do, cảnh báo và hậu kiểm.
- Cấp tài khoản, đổi vai trò và vô hiệu hóa tài khoản đều phải được kiểm toán.
- Cơ chế người lập/người duyệt của Tài chính: người tạo không được tự duyệt biểu giá, hoàn tiền hoặc đối soát, trừ ngoại lệ được phê duyệt chính thức.
- Tài khoản dịch vụ chỉ có phạm vi quyền hẹp, có chủ sở hữu và lịch xoay vòng thông tin xác thực.

## Các quyết định cần phê duyệt
Chủ sở hữu nghiệp vụ phải duyệt vai trò chính xác, phạm vi dữ liệu, hạn mức hoàn tiền, luồng duyệt biểu giá, luồng duyệt đối soát, quyền khẩn cấp và năng lực cổng đối tác.
