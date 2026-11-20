# Bitrix24 — Installation, Restore & Cut-over Documentation

Bộ tài liệu **Step-by-Step** triển khai, khôi phục, bảo mật, nghiệm thu và chuyển đổi Bitrix24.

## Documents

0. [00 — Step-by-Step AlmaLinux 9 Installation](./00-Step-by-Step-AlmaLinux-9-Installation.md)
1. [01 — Step-by-Step Installation](./01-Step-by-Step-Installation.md)
2. [02 — Backup & Restore](./02-Backup-and-Restore.md)
3. [03 — Permission & Security](./03-Permission-and-Security.md)
4. [04 — Domain, SSL & Functional Test](./04-Domain-SSL-and-Functional-Test.md)
5. [05 — Cut-over & Rollback](./05-Cut-over-and-Rollback.md)
6. [06 — Troubleshooting](./06-Troubleshooting.md)
7. [07 — Final Acceptance Checklist](./07-Final-Acceptance-Checklist.md)
8. [Images / Evidence](./images/README.md)

## Execution flow

```text
Pre-flight
   ↓
AlmaLinux 9 Installation
   ↓
OS Readiness Check
   ↓
Security / SSH Hardening
   ↓
Bitrix Environment Installation
   ↓
Storage & Backup
   ↓
Database Restore
   ↓
Bitrix Source Restore
   ↓
Permission Validation
   ↓
Domain / SSL
   ↓
Functional Test
   ↓
Cut-over
   ↓
Post-cut-over Monitoring
   ↓
Acceptance / Sign-off
```

## Image policy

- Bước cài AlmaLinux 9 có **ảnh minh họa thao tác đầy đủ** lấy từ Installation Guide chính thức của AlmaLinux và được nhúng trực tiếp trong Markdown.
- Các bước kiểm tra sau cài, cấu hình Bitrix, SSH, Database, Permission, Domain/SSL và Acceptance phải dùng **screenshot thực tế của môi trường triển khai**.
- Không dùng screenshot Internet làm Evidence của một lần triển khai thực tế.
- Không tạo screenshot giả để đánh dấu Pass.

## Important

- Các giá trị như IP, hostname, DB name, DB user, domain, mount point và PHP target phải được chốt theo môi trường thực tế.
- Không commit secret/password/private key/database dump/backup dữ liệu thật lên GitHub.
- Với backup Web lớn khoảng 131 GB, quy trình ưu tiên copy trực tiếp từ Server sang HDD rời kết nối với Server thay vì tải qua workstation.
- SSH quản trị dùng account riêng + `sudo`; không cho phép SSH root trực tiếp.
- PHP target phải được chốt sau khi kiểm thử compatibility; không dùng PHP 8.1 làm target khuyến nghị cho tài liệu mới.

## Official references

### AlmaLinux

- AlmaLinux Installation Guide: https://wiki.almalinux.org/documentation/installation-guide
- AlmaLinux After-Installation Guide: https://wiki.almalinux.org/documentation/after-installation-guide.html

### Bitrix24

- Bitrix server requirements: https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=32&LESSON_ID=2593
- Bitrix package installation: https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=32&LESSON_ID=15296
- Bitrix backup/restore: https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=48&LESSON_ID=6979
