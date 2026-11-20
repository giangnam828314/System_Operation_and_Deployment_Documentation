# Bitrix24 Evidence Images

Thư mục này dành cho ảnh minh họa/screenshot dùng trong tài liệu Bitrix24.

## Quy ước

- `01-installation/` — OS, resource, SSH, Web Server, PHP, PHP-FPM, Database
- `02-restore/` — disk mount, copy, checksum, restore DB/source
- `03-security/` — permission, SELinux, SSH hardening
- `04-acceptance/` — Domain, SSL, Login, CRM, Deals, Users, Tasks, Upload, logs
- `06-troubleshooting/` — before/after và lỗi thực tế

## Quy tắc bảo mật

Không upload:

- Password
- API token
- SSH private key
- SSL private key
- Database dump
- Backup dữ liệu thật
- Screenshot chứa dữ liệu khách hàng nhạy cảm

Khi chưa có screenshot thực tế, Markdown có thể dùng placeholder như `![Screenshot](./images/...)` nhưng không được tạo ảnh “giả bằng chứng”. Chỉ thay placeholder bằng screenshot thực tế sau khi thao tác đã được kiểm chứng.
