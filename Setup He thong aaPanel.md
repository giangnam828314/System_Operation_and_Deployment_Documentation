# Tài liệu chuẩn – Setup Hệ thống aaPanel

## 1. Mục đích

Tài liệu này tổng hợp toàn bộ quá trình thiết lập aaPanel trên VPS Ubuntu 24.04, các kiểm tra nền tảng, cấu hình SSH bảo mật, xử lý lỗi aaPanel Terminal và nguyên tắc triển khai stack dịch vụ sau khi Panel hoạt động.

> **Trạng thái tài liệu:** Ghi nhận theo quá trình triển khai thực tế ngày 16/08/2026.

---

## 2. Môi trường triển khai thực tế

| Thành phần | Giá trị |
|---|---|
| OS | Ubuntu 24.04.4 LTS (Noble Numbat) |
| Kernel | 6.8.0-137-generic x86_64 |
| CPU/RAM | RAM khoảng 7.8 GiB |
| Disk | khoảng 490 GB, LVM |
| Swap | 4 GB |
| aaPanel | Đã cài thành công |
| aaPanel port | 2891 |
| SSH port thực tế | 10012 |
| SSH PasswordAuthentication | no |
| SSH PermitRootLogin hiệu lực | no |
| SSH PublicKeyAuthentication | yes |
| SSH user quản trị | `namdag` |

---

## 3. Nguyên tắc triển khai

1. VPS mới phải được kiểm tra trạng thái trước khi cài dịch vụ.
2. Không cài thủ công Nginx/PHP/MariaDB/Docker bằng `apt` trước khi xác định cách aaPanel quản lý các dịch vụ này.
3. SSH phải giữ nguyên mô hình bảo mật: không bật đăng nhập root, không bật xác thực password nếu không cần thiết.
4. Chỉ mở những port cần thiết.
5. Mọi thay đổi cấu hình phải kiểm tra bằng `sshd -t` hoặc công cụ tương đương trước khi reload/restart dịch vụ.
6. Khi xử lý lỗi aaPanel Terminal, phân biệt rõ lỗi SSH server với lỗi Python/backend của aaPanel.

---

## 4. Kiểm tra VPS ban đầu

Sau khi vào shell root:

```bash
cat /etc/os-release
uname -a
df -h
free -h
docker --version
nginx -v
php -v
mariadb --version
```

Trạng thái thực tế ban đầu:

- Ubuntu 24.04.4 LTS.
- Docker chưa cài.
- Nginx chưa cài.
- PHP chưa cài.
- MariaDB client/server chưa cài.
- Disk và RAM còn dư lớn.
- Swap 4 GB đã tồn tại.

**Lưu ý:** Với VPS sạch dùng aaPanel, không nên cài các stack trên bằng `apt` trước nếu mục tiêu là để aaPanel quản lý chúng.

---

## 5. Cài đặt aaPanel

Script cài đặt sử dụng:

```bash
URL=https://www.aapanel.com/script/install_panel_en.sh && if [ -f /usr/bin/curl ];then curl -ksSO "$URL";else wget --no-check-certificate -O install_panel_en.sh "$URL";fi;bash install_panel_en.sh forum
```

Khi installer hỏi thư mục cài đặt `/www`, chọn `y`.

### Security Entrance

Security Entrance phải có dạng path, ví dụ:

```text
/a82c83v14
```

Không nhập chuỗi không có dấu `/` ở đầu.

Ví dụ truy cập Panel:

```text
https://SERVER_IP:2891/a82c83v14
```

Trong môi trường thực tế, aaPanel hoạt động ở port `2891`.

---

## 6. Kiểm tra aaPanel sau cài đặt

Kiểm tra trạng thái:

```bash
bt 14
```

Kiểm tra dịch vụ:

```bash
systemctl status nginx
systemctl status mariadb
systemctl status docker
```

Tại thời điểm đầu triển khai, các service ứng dụng có thể chưa được cài; mục tiêu của bước này là xác nhận Panel đang chạy trước khi triển khai stack.

---

## 7. Kiểm tra Firewall

aaPanel có trang:

```text
aaPanel → Security → Firewall
```

Không xóa rule chỉ dựa vào tên port. Trước khi xóa phải xác định dịch vụ thực tế đang listen bằng:

```bash
ss -lntup
```

Các port cần lưu ý:

| Port | Mục đích | Nguyên tắc |
|---:|---|---|
| 22 | SSH mặc định | Không mở nếu SSH thực tế không dùng |
| 80 | HTTP | Mở khi triển khai web |
| 443 | HTTPS | Mở khi triển khai SSL |
| 2891 | aaPanel | Chỉ cho phép theo nhu cầu quản trị |
| 10012 | SSH thực tế | Port SSH của hệ thống này |
| 3306 | MariaDB | Không mở Internet |
| 6379 | Redis | Không mở Internet |
| 1433 | SQL Server | Không mở Internet; chỉ nội bộ/container |

Không mở FTP/passive FTP nếu không sử dụng.

---

## 8. Cấu hình SSH thực tế

Hệ thống này sử dụng:

```text
Port 10012
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Kiểm tra cấu hình hiệu lực bằng:

```bash
sshd -T | grep -E '^port |^permitrootlogin|^passwordauthentication|^pubkeyauthentication'
```

Kỳ vọng:

```text
port 10012
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
```

### Kiểm tra listener

```bash
ss -lntp | grep -E ':22|:10012'
```

Kết quả thực tế cho thấy `sshd` listen trên:

```text
0.0.0.0:10012
[::]:10012
```

### Khi thay đổi SSH config

Kiểm tra cú pháp:

```bash
sshd -t
```

Nếu trả `0`, có thể reload:

```bash
systemctl reload ssh
```

Không cần restart SSH chỉ vì sửa một directive; reload giảm rủi ro làm gián đoạn phiên quản trị hiện tại.

---

## 9. Lỗi aaPanel Terminal – phân tích đầy đủ

### 9.1 Triệu chứng ban đầu

Trong `aaPanel → Terminal`, Terminal liên tục:

```text
Trying to reconnect!
```

Traceback có lỗi:

```text
paramiko.ssh_exception.PasswordRequiredException:
Private key file is encrypted
```

Sau đó có thể xuất hiện:

```text
Account or Password incorrect: Authentication failed
```

hoặc:

```text
The specified host information is not found, please add it again!
```

### 9.2 Kiểm tra Python/Paramiko

aaPanel sử dụng Python:

```text
/www/server/panel/pyenv/bin/python3
```

Phiên bản thực tế:

```text
Python 3.12.3
```

Paramiko:

```text
3.4.0
```

Kiểm tra:

```bash
/www/server/panel/pyenv/bin/python3 --version
/www/server/panel/pyenv/bin/python3 -c "import paramiko; print(paramiko.__version__)"
```

Kiểm tra syntax:

```bash
/www/server/panel/pyenv/bin/python3 -m py_compile /www/server/panel/class_v2/ssh_terminal_v2.py
echo $?
```

Kết quả `0` xác nhận file Python compile được.

Cảnh báo TripleDES của Paramiko 3.4.0 là warning về thư viện mã hóa cũ, không phải nguyên nhân chính làm SSH Ed25519 thất bại.

---

## 10. Phân tích code aaPanel Terminal

File cần kiểm tra:

```text
/www/server/panel/class_v2/ssh_terminal_v2.py
```

Các class chính:

```python
class ssh_terminal:
class ssh_host_admin(ssh_terminal):
```

Các hàm quan trọng:

```text
connect()
is_local()
get_login_user()
get_ulist()
get_host_find()
create_host()
get_ssh_info()
save_ssh_info()
```

### 10.1 `connect()`

`connect()` gọi:

```python
self.is_local()
```

và nếu host là local:

```python
if self._host in ['127.0.0.1','localhost']:
    self._port = public.get_sshd_port()
```

Trong hệ thống này, `public.get_sshd_port()` trả `10012`.

### 10.2 `is_local()`

Logic local của aaPanel tự động tìm key. Khi user là root, code ưu tiên:

```text
/root/.ssh/id_ed25519
/root/.ssh/id_ecdsa
/root/.ssh/id_rsa
/root/.ssh/id_rsa_bt
```

Trong máy này, `/root/.ssh/id_ed25519` tồn tại và là private key có passphrase.

Kiểm tra:

```bash
ssh-keygen -y -f /root/.ssh/id_ed25519 >/dev/null
```

Lệnh yêu cầu:

```text
Enter passphrase:
```

=> key được mã hóa bằng passphrase.

Trong khi Local Terminal tự lấy key nhưng không truyền `pkey_passwd`, gây:

```text
PasswordRequiredException: private key file is encrypted
```

---

## 11. Cơ chế `pkey_passwd` của aaPanel

aaPanel có hỗ trợ trường `pkey_passwd` trong backend và UI.

Các điểm quan trọng trong `ssh_terminal_v2.py`:

```python
self._key_passwd = ssh_info['pkey_passwd']
```

và backend test connection có logic:

```python
pkey_pwd = args.get('pkey_passwd', '').strip()
```

Khi private key có encryption nhưng không có passphrase đúng, backend có thể trả:

```text
SSH private key is encrypted, please provide pkey_passwd!
```

Điều này xác nhận lỗi ban đầu nằm ở việc Local server tự dùng private key encrypted nhưng không truyền passphrase phù hợp.

---

## 12. Tách lỗi SSH server khỏi lỗi aaPanel

Đây là bước quan trọng nhất trong quá trình xử lý.

SSH thực tế đã được chứng minh hoạt động bằng một key riêng:

```text
/home/namdag/.ssh/id_ed25519_aapanel
```

Key này không có passphrase và đã được thêm vào:

```text
/home/namdag/.ssh/authorized_keys
```

Test trực tiếp:

```bash
ssh -i /home/namdag/.ssh/id_ed25519_aapanel \
-p 10012 \
-o IdentitiesOnly=yes \
namdag@127.0.0.1
```

Kết quả thực tế:

```text
Welcome to Ubuntu 24.04.4 LTS
namdag@thientam:~$
```

=> SSH server, port, username và public-key authentication đều hoạt động.

Vì vậy lỗi còn lại là **logic Local Terminal của aaPanel**, không phải SSH daemon.

---

## 13. Tạo connection SSH mới trong aaPanel Terminal

Để tách biệt khỏi `Local server` mặc định, tạo connection:

```text
Server IP:    127.0.0.1
Port:         10012
SSH account:  namdag
Verification: Private key
```

Private key dùng:

```text
/home/namdag/.ssh/id_ed25519_aapanel
```

Key password: để trống.

**Không paste private key vào tài liệu Git.** Chỉ dùng trực tiếp trong Panel khi cấu hình.

Kết quả thực tế:

```text
Connection successful
```

=> aaPanel Python + Paramiko có thể kết nối SSH thành công với cấu hình này.

---

## 14. Vấn đề `ssh_info`

aaPanel lưu các SSH connection trong:

```text
/www/server/panel/config/ssh_info/
```

Các connection thực tế đã thấy:

```text
127.0.0.1/
127.0.0.1_10012/
127.0.0.1_10022/
```

Hàm:

```python
def get_ssh_info(self,host):
    info_file = self._save_path + host + '/info.json'
```

nên:

```text
get_ssh_info('127.0.0.1')
```

sẽ tìm:

```text
ssh_info/127.0.0.1/info.json
```

Trong khi `create_host()` lưu connection có port vào:

```python
host_path = self._save_path + args.host + "_" + args.port
```

nên connection 10012 nằm ở:

```text
ssh_info/127.0.0.1_10012/info.json
```

Test trực tiếp bằng Python cho kết quả:

```text
127.0.0.1 => NOT FOUND
127.0.0.1_10012 => FOUND
    host     = 127.0.0.1
    port     = 10012
    username = namdag
    password = NO
    pkey     = YES
    pkey_pwd = NO
127.0.0.1_10022 => FOUND
```

Điều này chứng minh connection 10012 đã được lưu/giải mã đúng.

---

## 15. Vấn đề `get_login_user()` và cấu hình SSH

Hàm `get_login_user()` của aaPanel đọc trực tiếp:

```text
/etc/ssh/sshd_config
```

và kiểm tra:

```python
PermitRootLogin yes|without-password
```

Trong hệ thống này, `/etc/ssh/sshd_config` có:

```text
PermitRootLogin yes
```

nhưng cấu hình **hiệu lực** từ:

```bash
sshd -T
```

lại là:

```text
permitrootlogin no
passwordauthentication no
pubkeyauthentication yes
port 10012
```

Đây là điểm bất nhất quan trọng: aaPanel nhìn vào file chính và có thể kết luận sai rằng root được phép SSH, trong khi `sshd` thực tế đang áp dụng `PermitRootLogin no` do cấu hình include/override.

### Người dùng `namdag` hợp lệ

```text
namdag:x:1000:1000:thientam:/home/namdag:/bin/bash
```

và:

```text
uid=1000(namdag) gid=1000(namdag)
```

`/home/namdag` tồn tại và shell là `/bin/bash`.

### Lỗi logic bổ sung trong `get_login_user()`

Cuối hàm hiện có:

```python
if len(can_login) >= 1:
    return root_group_user[0]["user"]
```

đây là lỗi logic; khi `can_login` có user nhưng `root_group_user` rỗng, phải trả:

```python
return can_login[0]["user"]
```

Đây là lỗi cần ghi nhận trong tài liệu; **không tự động sửa code aaPanel trong môi trường production nếu chưa có kế hoạch backup/version pinning**.

---

## 16. Những điều KHÔNG nên làm

Không nên chữa lỗi aaPanel Terminal bằng cách:

```text
- bật PermitRootLogin yes
- bật PasswordAuthentication yes
- đổi SSH port về 22
- mở thêm firewall port vô điều kiện
- xóa private key hiện tại
- sửa source aaPanel trực tiếp mà không backup
```

Đặc biệt không gửi hoặc lưu private key vào Git.

---

## 17. Khuyến nghị xử lý Local Terminal

### Phương án ưu tiên

Giữ SSH server ở trạng thái bảo mật:

```text
root SSH        disabled
Password SSH    disabled
Public Key      enabled
SSH port        10012
```

Dùng connection riêng trong aaPanel Terminal:

```text
127.0.0.1:10012
namdag
Private key
```

Connection này đã được xác nhận bằng `Connection successful`.

### Nếu cần Local server mặc định

Cần sửa hoặc nâng cấp logic aaPanel `is_local()`/`get_login_user()` để:

1. Đánh giá đúng effective `PermitRootLogin` thay vì chỉ đọc `/etc/ssh/sshd_config`.
2. Khi root không được SSH, chọn user hợp lệ như `namdag`.
3. Cho phép Local Terminal sử dụng key riêng như:
   ```text
   /home/namdag/.ssh/id_ed25519_aapanel
   ```
4. Không làm thay đổi chính sách SSH bảo mật.

Nếu phải sửa source aaPanel, phải backup file trước:

```bash
cp /www/server/panel/class_v2/ssh_terminal_v2.py \
   /www/server/panel/class_v2/ssh_terminal_v2.py.bak-$(date +%Y%m%d-%H%M%S)
```

Sau khi sửa phải kiểm tra:

```bash
/www/server/panel/pyenv/bin/python3 -m py_compile \
/www/server/panel/class_v2/ssh_terminal_v2.py
```

và phải trả `0`.

---

## 18. Quy trình Setup aaPanel chuẩn đề xuất

### Giai đoạn 1 – Chuẩn bị

- [ ] Kiểm tra OS, kernel, RAM, disk, swap.
- [ ] Xác nhận VPS sạch.
- [ ] Xác định mô hình quản lý package: aaPanel hay apt.

### Giai đoạn 2 – Cài aaPanel

- [ ] Chạy script cài đặt chính thức.
- [ ] Thiết lập Security Entrance dạng `/path`.
- [ ] Ghi nhận port Panel.
- [ ] Đăng nhập và kiểm tra Local Node Online.

### Giai đoạn 3 – Bảo mật nền tảng

- [ ] Xác định SSH port thực tế.
- [ ] Tắt root SSH nếu mô hình cho phép.
- [ ] Tắt PasswordAuthentication.
- [ ] Bật PublicKeyAuthentication.
- [ ] Kiểm tra bằng `sshd -T`.
- [ ] `sshd -t` trước khi reload.
- [ ] Kiểm tra firewall và chỉ mở port cần thiết.

### Giai đoạn 4 – Kiểm tra aaPanel Terminal

- [ ] Test SSH bên ngoài aaPanel.
- [ ] Test Python/Paramiko của aaPanel.
- [ ] Phân biệt lỗi SSH với lỗi Local Terminal.
- [ ] Kiểm tra `ssh_info`.
- [ ] Test connection trong aaPanel.
- [ ] Không hạ bảo mật SSH chỉ để sửa Terminal.

### Giai đoạn 5 – Cài stack ứng dụng

Sau khi Panel và SSH ổn định mới triển khai:

- [ ] Nginx.
- [ ] PHP 8.3.
- [ ] MariaDB.
- [ ] Redis.
- [ ] Docker.
- [ ] SQL Server bằng Docker.
- [ ] Website/domain.
- [ ] SSL.

### Giai đoạn 6 – Kiểm thử

- [ ] Kiểm tra service status.
- [ ] Kiểm tra port.
- [ ] Kiểm tra database connection.
- [ ] Kiểm tra Docker container.
- [ ] Kiểm tra domain/SSL.
- [ ] Kiểm tra backup.
- [ ] Ghi lại cấu hình thực tế vào Git.

---

## 19. Các lệnh kiểm tra chuẩn

### Hệ thống

```bash
cat /etc/os-release
uname -a
df -h
free -h
```

### SSH

```bash
ss -lntp | grep -E ':22|:10012'
sshd -t
sshd -T | grep -E '^port |^permitrootlogin|^passwordauthentication|^pubkeyauthentication'
```

### Python aaPanel

```bash
/www/server/panel/pyenv/bin/python3 --version
/www/server/panel/pyenv/bin/python3 -c "import paramiko; print(paramiko.__version__)"
```

### aaPanel

```bash
bt 14
```

### Docker

```bash
docker --version
docker ps
```

---

## 20. Bài học từ quá trình thực tế

1. Không nên kết luận lỗi Terminal là lỗi SSH server khi SSH thủ công vẫn hoạt động.
2. `sshd_config` và `sshd -T` có thể cho kết quả khác nhau do cấu hình include/override; quản trị viên phải phân biệt **configured** và **effective configuration**.
3. Khi debug aaPanel, cần kiểm tra cả Python runtime, Paramiko, source module, cấu trúc `ssh_info` và luồng `is_local()`.
4. Connection test thành công trong aaPanel là bằng chứng mạnh rằng Python/Paramiko/SSH key có thể hoạt động với đúng thông tin kết nối.
5. Không nên sửa source aaPanel ngay khi chưa xác định luồng dữ liệu.
6. Không bao giờ lưu private key, password hoặc secret vào Git.
7. Với Local Terminal của aaPanel, cần ưu tiên user SSH hợp lệ và key không passphrase chuyên dụng nếu module không truyền được passphrase.

---

## 21. Trạng thái hiện tại

### Đã xác nhận

- Ubuntu 24.04.4 LTS hoạt động.
- aaPanel đã cài và Panel hoạt động ở port 2891.
- SSH listen trên port 10012.
- `PermitRootLogin` hiệu lực là `no`.
- `PasswordAuthentication` là `no`.
- `PublicKeyAuthentication` là `yes`.
- `namdag` đăng nhập SSH bằng public key thành công.
- Private key `/home/namdag/.ssh/id_ed25519_aapanel` dùng được.
- aaPanel Test Connection với `127.0.0.1:10012 / namdag / Private key` thành công.
- Python 3.12.3 và Paramiko 3.4.0 hoạt động.
- `ssh_terminal_v2.py` compile thành công.

### Còn cần hoàn thiện

- Hoàn thiện Local server mặc định của aaPanel nếu muốn dùng nút Local server thay vì connection riêng.
- Tiếp tục cài stack Nginx/PHP/MariaDB/Redis/Docker.
- Triển khai SQL Server bằng Docker.
- Hoàn thiện SSL/domain/firewall theo mô hình production.

---

## 22. Bảo mật tài liệu Git

Không lưu các nội dung sau vào repository:

```text
- Private key
- SSH passphrase
- Root password
- Database password
- API key
- Token
- Cookie/session
- Backup secret
```

Chỉ ghi:

- đường dẫn file;
- tên biến;
- cấu trúc cấu hình;
- lệnh đã dùng;
- giá trị mẫu/placeholder;
- trạng thái đã kiểm tra.

---

## 23. Tài liệu liên quan trong repository

- `Cách bảo mật cho Linux.md`
- `Backup Hệ thống Thiện tâm.md`
- `Quy-trinh-Migration-He-thong-tu-VPS-cu-sang-VPS-moi-Su-dung-Docker.md`
- `Quá Trình Cài đặt và Kiểm tra - Thiện tâm (GPT).md`
