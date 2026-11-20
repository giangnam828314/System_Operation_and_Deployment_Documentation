# 01 — Bitrix24 Step-by-Step Installation

> **Purpose:** triển khai môi trường Bitrix24 self-hosted/box trên Linux server mới, chuẩn bị cho restore dữ liệu.
>
> **Target:** IT/System Administrator.
>
> **Rule:** Không thực hiện trên Production nếu chưa có backup, rollback plan và cửa sổ bảo trì được phê duyệt.

## 0. Information sheet

| Item | Value |
|---|---|
| OS | `<AlmaLinux version>` |
| Server IP | `<IP>` |
| SSH Port | `<PORT>` |
| Admin User | `<admin>` |
| Bitrix path | `/home/bitrix/www/` |
| PHP target | `<8.2/8.4/... — must be validated>` |
| DB engine/version | `<MariaDB/MySQL>` |
| DB name | `<DB_NAME>` |
| DB user | `<DB_USER>` |
| Domain | `<FQDN>` |
| Backup mount | `<MOUNT_POINT>` |

---

## 1. Pre-flight check

### 1.1 Check OS

```bash
cat /etc/os-release
uname -r
```

**Expected:** đúng OS/version đã được phê duyệt.

📸 **Evidence:** lưu screenshot terminal.

### 1.2 Check CPU / RAM / Disk

```bash
lscpu | head
free -h
df -h
lsblk
```

**Expected:** tài nguyên đủ cho workload; không có filesystem quan trọng gần đầy.

📸 **Evidence:** `free -h`, `df -h`, `lsblk`.

### 1.3 Update package metadata

```bash
dnf update -y
```

**Expected:** hoàn tất không có lỗi dependency.

> Sau bước này nên reboot nếu kernel/core packages được cập nhật và policy vận hành yêu cầu.

---

## 2. Create secure admin access

### 2.1 Create admin account

```bash
useradd -m -s /bin/bash <ADMIN_USER>
passwd <ADMIN_USER>
```

> Password thật **không được ghi vào GitHub**.

### 2.2 Grant sudo

```bash
usermod -aG wheel <ADMIN_USER>
```

Test:

```bash
su - <ADMIN_USER>
sudo -v
sudo id
```

**Expected:** `sudo id` cho thấy command chạy bằng quyền root.

### 2.3 SSH hardening

Edit SSH configuration theo policy hiện hành:

```bash
sudo vi /etc/ssh/sshd_config
```

Các nguyên tắc bắt buộc:

```text
PermitRootLogin no
```

Giữ SSH port custom đã được phê duyệt:

```text
Port <APPROVED_PORT>
```

Validate trước khi restart:

```bash
sudo sshd -t
```

Sau khi config hợp lệ:

```bash
sudo systemctl restart sshd
sudo systemctl status sshd --no-pager
```

📸 **Evidence:** `sshd_config` relevant lines + `sshd -t` + active service.

> Không đóng phiên SSH hiện tại cho tới khi đăng nhập thành công bằng account quản trị mới.

---

## 3. Install Web Server / PHP / Database

Bitrix hiện yêu cầu PHP **từ 8.2.0 trở lên từ 01/02/2026** và khuyến nghị PHP **8.4+**. Phiên bản cuối cùng phải được chốt sau khi kiểm thử tương thích ứng dụng/customization. citeturn871693search0

Tài liệu Bitrix hiện tại cũng công bố bộ package tham chiếu gồm Apache 2.4.x, PHP 8.4.x, MariaDB 10.11.x, Node/NPM và Redis cho box product; version package thực tế phải khớp repository/build được phê duyệt. citeturn871693search1

### 3.1 Install Apache

```bash
dnf install httpd -y
systemctl enable --now httpd
systemctl status httpd --no-pager
```

**Expected:** `active (running)`.

📸 **Evidence:** service status.

### 3.2 Install PHP

Cài PHP **đúng version đã chốt**. Ví dụ package release cho PHP 8.4 theo tài liệu Bitrix:

```bash
dnf install php84-release -y
```

Sau đó cài các extension cần thiết theo repository/build:

```bash
dnf install -y \
  php php-cli php-common php-devel php-gd php-ldap php-mbstring \
  php-mysqlnd php-opcache php-pdo php-pear php-pear-DB \
  php-pecl-apcu php-pecl-memcached php-process php-xml php-zipstream
```

Check:

```bash
php -v
php -m
```

**Expected:** PHP target đúng version; extension cần thiết có mặt.

📸 **Evidence:** `php -v`, `php -m`.

### 3.3 Install PHP-FPM

```bash
dnf install php-fpm -y
systemctl enable --now php-fpm
systemctl status php-fpm --no-pager
```

### 3.4 Install MariaDB

```bash
dnf install mariadb-server mariadb -y
systemctl enable --now mariadb
systemctl status mariadb --no-pager
```

**Expected:** DB service active.

### 3.5 Optional / architecture-dependent services

Tùy architecture Bitrix box và các module sử dụng, cài Node/NPM, Redis và các dependency liên quan theo build được phê duyệt. Không cài thêm package chỉ vì “có trong mẫu” nếu hệ thống thực tế không cần.

---

## 4. Configure Database

### 4.1 Create database

```sql
CREATE DATABASE `<DB_NAME>` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 4.2 Create application DB user

```sql
CREATE USER '<DB_USER>'@'localhost' IDENTIFIED BY '<DB_PASSWORD>';
GRANT ALL PRIVILEGES ON `<DB_NAME>`.* TO '<DB_USER>'@'localhost';
FLUSH PRIVILEGES;
```

> Không commit `<DB_PASSWORD>` thật vào repository.

### 4.3 Test DB connectivity

```bash
mysql -u <DB_USER> -p <DB_NAME>
```

**Expected:** đăng nhập thành công.

---

## 5. Prepare Bitrix directories

```bash
mkdir -p /home/bitrix/www
mkdir -p /home/bitrix/backup
chown -R bitrix:bitrix /home/bitrix
```

> Nếu user/group `bitrix` chưa tồn tại, phải tạo theo architecture và deployment method đã được phê duyệt; không tự ý đổi ownership toàn hệ thống.

---

## 6. Configure Web Server / PHP-FPM

1. Cấu hình VirtualHost/Server block cho Domain.
2. Cấu hình DocumentRoot trỏ tới Bitrix path.
3. Kết nối PHP requests tới PHP-FPM.
4. Validate config.

Apache example validation:

```bash
apachectl configtest
systemctl reload httpd
```

PHP-FPM:

```bash
systemctl status php-fpm --no-pager
journalctl -u php-fpm --since "-30 min" --no-pager
```

📸 **Evidence:** configtest + service status + relevant log.

---

## 7. Firewall / Network

Chỉ mở port cần thiết theo network policy:

```bash
firewall-cmd --list-all
```

HTTP/HTTPS:

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

SSH dùng custom port đã phê duyệt, không mở rộng quyền truy cập ngoài phạm vi cần thiết.

---

## 8. System validation before restore

Run:

```bash
systemctl --failed
systemctl is-active httpd
systemctl is-active php-fpm
systemctl is-active mariadb
php -v
mysql --version
df -h
```

**Expected:**

- Không có failed service không giải thích được.
- Apache/NGINX, PHP-FPM, Database đang active.
- Disk còn đủ dung lượng cho source + database + temporary space.
- PHP version đúng target.

---

## 9. Go / No-Go checkpoint

### GO only when

- [ ] OS/version đúng
- [ ] Admin account + sudo hoạt động
- [ ] Root SSH login disabled
- [ ] Web Server active
- [ ] PHP target validated
- [ ] PHP-FPM active
- [ ] Database active
- [ ] Database connectivity verified
- [ ] Disk capacity verified
- [ ] Backup source verified
- [ ] Rollback plan approved

### Evidence package

`01-installation/`

```text
01-os.png
02-resource.png
03-ssh-hardening.png
04-httpd-status.png
05-php-version.png
06-php-extensions.png
07-php-fpm-status.png
08-db-status.png
09-db-connectivity.png
10-system-validation.png
```

---

## 10. Official reference

- Bitrix server requirements: https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=32&LESSON_ID=2593
- Bitrix package installation: https://dev.1c-bitrix.ru/learning/course/?COURSE_ID=32&LESSON_ID=15296

Tài liệu này intentionally giữ các placeholder ở những chỗ phụ thuộc môi trường. Không commit credentials, private keys, database dump hoặc backup thật vào repository.
