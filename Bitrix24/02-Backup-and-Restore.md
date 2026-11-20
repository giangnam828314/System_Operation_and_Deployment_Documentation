# 02 — Bitrix24 Backup & Restore — Step by Step

## 1. Scope

Áp dụng cho migration/restore Bitrix24 sang server mới.

**Nguyên tắc:** với backup Web lớn (~131 GB theo phương án thực tế), ưu tiên copy trực tiếp từ server nguồn sang HDD rời kết nối trực tiếp với server thay vì tải qua máy tính cá nhân.

---

## 2. Backup to external HDD

### Step 1 — Identify disk

```bash
lsblk -f
blkid
```

Xác định đúng device, ví dụ `/dev/sdb1`.

📸 Evidence: screenshot `lsblk -f`.

### Step 2 — Check filesystem

```bash
lsblk -f /dev/sdb1
```

Nếu là exFAT mà server chưa hỗ trợ, cài gói exFAT phù hợp với OS.

### Step 3 — Create mount point

```bash
mkdir -p /mnt/bitrix-backup
mount /dev/sdb1 /mnt/bitrix-backup
```

Check:

```bash
df -h /mnt/bitrix-backup
mount | grep bitrix-backup
```

### Step 4 — Write test

```bash
touch /mnt/bitrix-backup/.write-test
ls -l /mnt/bitrix-backup/.write-test
rm -f /mnt/bitrix-backup/.write-test
```

**Expected:** create/delete thành công.

### Step 5 — Copy Web backup

Ví dụ với file backup đã tạo:

```bash
rsync -avh --progress /path/to/backup_web.tar.gz /mnt/bitrix-backup/
```

Hoặc nếu file rất lớn và muốn copy đơn giản:

```bash
cp -v /path/to/backup_web.tar.gz /mnt/bitrix-backup/
```

Theo dõi dung lượng:

```bash
du -sh /mnt/bitrix-backup/backup_web.tar.gz
df -h /mnt/bitrix-backup
```

### Step 6 — Copy SQL backup

```bash
rsync -avh --progress /path/to/file_backup.sql /mnt/bitrix-backup/
```

### Step 7 — Verify integrity

Tại nguồn:

```bash
sha256sum /path/to/backup_web.tar.gz
sha256sum /path/to/file_backup.sql
```

Tại HDD:

```bash
sha256sum /mnt/bitrix-backup/backup_web.tar.gz
sha256sum /mnt/bitrix-backup/file_backup.sql
```

**Expected:** checksum giống nhau.

---

## 3. Restore Web source

### Step 1 — Create destination

```bash
mkdir -p /home/bitrix/www
```

### Step 2 — Extract backup

```bash
cd /home/bitrix/www
sudo tar -xzvf /mnt/bitrix-backup/backup_web.tar.gz
```

> Với archive chứa đường dẫn đặc biệt, kiểm tra nội dung trước khi extract:
>
> ```bash
> tar -tzf /mnt/bitrix-backup/backup_web.tar.gz | head -50
> ```

### Step 3 — Verify source structure

```bash
ls -la /home/bitrix/www
ls -la /home/bitrix/www/bitrix
```

Kiểm tra các file cấu hình chính, ví dụ `.settings.php` và/hoặc `bitrix/php_interface/dbconn.php` tùy version/mô hình triển khai.

---

## 4. Restore Database

Bitrix documentation xác nhận quy trình restore bao gồm bước restore database; trong trường hợp migration có thể tạo Database nếu chưa tồn tại. citeturn871693search4

### Step 1 — Confirm DB service

```bash
systemctl is-active mariadb
# hoặc
systemctl is-active mysqld
```

### Step 2 — Create DB if required

```sql
CREATE DATABASE `<DB_NAME>` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Step 3 — Test DB account

```bash
mysql -u <DB_USER> -p <DB_NAME>
```

### Step 4 — Import SQL

Theo mô hình SQL dump:

```bash
mysql -u <DB_USER> -p <DB_NAME> < /mnt/bitrix-backup/file_backup.sql
```

Tài liệu Bitrix cũng minh họa restore bằng cú pháp `mysql <database> < <sql_file>`. citeturn871693search2

### Step 5 — Monitor import

Theo dõi process/log nếu database lớn. Không kết luận thành công chỉ vì terminal không báo lỗi.

Kiểm tra row/table:

```bash
mysql -u <DB_USER> -p -e "USE <DB_NAME>; SHOW TABLES;"
```

### Step 6 — Verify application data

Đăng nhập Bitrix24 và kiểm tra:

- CRM
- Companies/Contacts nếu có
- Deals
- Users
- Tasks
- Kanban
- Workflows
- File attachments

---

## 5. Verify configuration after restore

### Step 1 — Check DB config

Xác nhận thông tin connection trong cấu hình ứng dụng khớp DB mới.

### Step 2 — Clear/refresh cache if required

Thực hiện theo cơ chế cache của bản Bitrix đang triển khai; không xóa tùy tiện các thư mục production ngoài phạm vi đã xác định.

### Step 3 — Run Bitrix system check

Sau migration, Bitrix khuyến nghị kiểm tra các thông số hệ thống và administrative settings, đặc biệt khi domain thay đổi. citeturn871693search9

---

## 6. Security cleanup

Sau khi restore bằng workflow có chứa local backup/helper scripts, cần xóa các bản sao local không còn cần thiết và script phục vụ restore theo hướng dẫn của Bitrix. citeturn871693search4turn871693search2

**Không xóa backup duy nhất** trước khi có xác nhận nghiệm thu và rollback artifact đầy đủ.

---

## 7. Restore acceptance checkpoint

- [ ] Web archive extracted successfully
- [ ] Database imported successfully
- [ ] DB connection works
- [ ] CRM data visible
- [ ] Users visible
- [ ] Deals visible
- [ ] Tasks visible
- [ ] File upload/download tested
- [ ] No DB import errors
- [ ] No critical application errors
- [ ] Backup/restore evidence captured

## 8. Evidence package

```text
02-restore/
├── 01-lsblk.png
├── 02-mount.png
├── 03-write-test.png
├── 04-copy-progress.png
├── 05-checksum-source.png
├── 06-checksum-destination.png
├── 07-source-structure.png
├── 08-db-import.png
├── 09-db-tables.png
└── 10-functional-test.png
```
