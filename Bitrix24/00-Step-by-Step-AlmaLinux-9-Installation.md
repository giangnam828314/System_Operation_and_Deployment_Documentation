# 00 — Step-by-Step cài đặt AlmaLinux 9 cho Bitrix24

> **Mục tiêu:** Cài một AlmaLinux 9 server sạch, tối thiểu và sẵn sàng cho các bước cài Web Server, PHP/PHP-FPM, Database và Bitrix24.
>
> **Quan trọng:** Đây là baseline OS. Các giá trị IP, hostname, disk, gateway, DNS và tài khoản phải thay bằng thông tin của môi trường thực tế.
>
> **Ảnh minh họa:** Các ảnh giao diện trong tài liệu được lấy từ Installation Guide chính thức của AlmaLinux. Một số ảnh hiển thị giao diện AlmaLinux 9.1; UI có thể thay đổi nhẹ giữa các minor release 9.x.

---

## 1. Chuẩn bị trước khi cài

### 1.1. Thông số cần chốt

| Hạng mục | Giá trị |
|---|---|
| AlmaLinux | 9.x |
| Architecture | x86_64 |
| ISO | `AlmaLinux-9-latest-x86_64-dvd.iso` hoặc ISO đã được phê duyệt |
| Installation profile | **Minimal Install** / Server |
| Hostname | `bitrix24-prod` |
| IP | `<SERVER_IP>` |
| Prefix/Netmask | `<PREFIX>` |
| Gateway | `<GATEWAY>` |
| DNS | `<DNS1>`, `<DNS2>` |
| Timezone | `Asia/Ho_Chi_Minh` |
| Admin user | `<ADMIN_USER>` |
| SSH port | `<SSH_PORT>` |
| System disk | `<SYSTEM_DISK>` |

AlmaLinux hỗ trợ BIOS/UEFI và x86_64; Installation Guide hiện hành cung cấp các loại ISO `boot`, `minimal` và `dvd`. Với server Bitrix24, chọn image/profile theo chuẩn triển khai đã được phê duyệt. citeturn1view0

---

## 2. Download ISO AlmaLinux 9

Khuyến nghị lấy ISO từ mirror/repository chính thức và chọn đúng architecture.

```bash
curl -O https://repo.almalinux.org/almalinux/9/isos/x86_64/AlmaLinux-9-latest-x86_64-dvd.iso
```

> Không commit file ISO vào Git repository.

### Evidence cần lưu

- ☐ Tên file ISO
- ☐ Version
- ☐ Architecture
- ☐ SHA256 checksum

Nguồn chính thức: urlAlmaLinux Installation Guidehttps://wiki.almalinux.org/documentation/installation-guide

---

## 3. Verify ISO trước khi boot

### 3.1. Download public key và CHECKSUM

```bash
curl -O https://repo.almalinux.org/almalinux/RPM-GPG-KEY-AlmaLinux-9
curl -O https://repo.almalinux.org/almalinux/9/isos/x86_64/CHECKSUM
gpg --import RPM-GPG-KEY-AlmaLinux-9
gpg --verify CHECKSUM
```

Kết quả cần tìm:

```text
gpg: Good signature from "AlmaLinux OS 9 <packager@almalinux.org>"
```

### 3.2. Tính SHA256

```bash
sha256sum AlmaLinux-9-latest-x86_64-dvd.iso
cat CHECKSUM | grep -E 'SHA256.*AlmaLinux-9-latest-x86_64-dvd.iso'
```

Hai giá trị SHA256 phải khớp. Quy trình xác minh này được AlmaLinux hướng dẫn chính thức. citeturn1view0

---

## 4. Tạo USB boot

Nếu cài server vật lý, có thể dùng Rufus/Fedora Media Writer trên Windows hoặc `dd` trên Linux.

Ví dụ Linux:

```bash
lsblk
sudo dd if=./AlmaLinux-9-latest-x86_64-dvd.iso of=/dev/sdX status=progress conv=fsync bs=4M
```

> **CẢNH BÁO:** `/dev/sdX` phải là USB. Chọn sai thiết bị sẽ làm mất dữ liệu trên disk đó.

AlmaLinux cũng hướng dẫn Rufus cho Windows và `dd` cho Linux. citeturn1view0

---

## 5. Boot vào AlmaLinux Installer

Cắm USB/attach ISO vào server/VM và boot từ ISO.

![AlmaLinux Installer Welcome Menu](https://wiki.almalinux.org/images/installation-guide-welcome-menu.webp)

### Chọn

**Test this media & install AlmaLinux 9.x**

Khuyến nghị chọn mục test media để kiểm tra ISO trước khi vào installer. Nếu đã verify ISO bằng checksum và muốn bỏ qua media test, có thể chọn **Install AlmaLinux 9.x**. citeturn1view0

### Evidence

- ☐ Chụp màn hình Welcome Menu

---

## 6. Chọn ngôn ngữ

![AlmaLinux Installation Language](https://wiki.almalinux.org/images/installation-guide-installation-language.webp)

Khuyến nghị cho server:

- Language: `English`
- Keyboard: `English (US)` hoặc layout theo chuẩn vận hành

Chọn **Continue**.

### Expected Result

Xuất hiện màn hình **Installation Summary**.

---

## 7. Installation Summary — kiểm tra toàn bộ trước khi cài

![AlmaLinux Installation Summary](https://wiki.almalinux.org/images/installation-guide-installation-summary.webp)

Các mục có dấu cảnh báo/đỏ phải được cấu hình trước khi nút **Begin Installation** được bật. AlmaLinux chia cấu hình thành Localization, Software, System và User Settings. citeturn1view0

---

# 8. Localization

## 8.1. Keyboard

![Keyboard](https://wiki.almalinux.org/images/installation-guide-keyboard.webp)

Chọn keyboard layout đúng với người quản trị. Thông thường dùng `English (US)`.

**Done** → quay lại Installation Summary.

## 8.2. Language Support

Giữ English hoặc bổ sung ngôn ngữ nếu có yêu cầu.

## 8.3. Time & Date

![Time and Date](https://wiki.almalinux.org/images/installation-guide-time-date.webp)

Thiết lập:

- Region: `Asia`
- City: `Ho_Chi_Minh`
- Network Time: **ON** nếu hạ tầng NTP cho phép

**Expected Result:** Server sử dụng đúng timezone và đồng bộ thời gian.

---

# 9. Software

## 9.1. Installation Source

Nếu dùng DVD ISO, source thường được nhận diện là Local Media. Nếu dùng Boot ISO, installer có thể sử dụng mirror/repository qua mạng. citeturn1view0

Kiểm tra source không có cảnh báo.

## 9.2. Software Selection

![Software Selection](https://wiki.almalinux.org/images/installation-guide-software-selection.webp)

Đối với Bitrix24 server, ưu tiên:

**Base Environment → Minimal Install**

Không cài Desktop/GNOME nếu đây là production server và không có yêu cầu đặc biệt.

> Mục tiêu là giữ OS tối giản; Web Server/PHP/Database/Bitrix sẽ được cài ở các bước tiếp theo.

**Done**.

---

# 10. System — Installation Destination

![Installation Destination](https://wiki.almalinux.org/images/installation-guide-installation-destination.webp)

Chọn đúng system disk.

### Khuyến nghị

- Nếu server mới và chỉ có một system disk: có thể dùng **Automatic**.
- Nếu có yêu cầu partition riêng cho `/home`, `/var`, `/var/log`, backup hoặc data: chọn **Custom** và dùng layout đã được phê duyệt.
- Không chọn nhầm disk chứa dữ liệu backup.

### Với Bitrix24

Nếu dữ liệu Web/Upload lớn, cần thiết kế storage trước khi production. Không mặc định coi system disk là nơi chứa backup 131 GB.

**Done**.

---

# 11. KDUMP

![KDUMP](https://wiki.almalinux.org/images/installation-guide-kdump.webp)

Giữ cấu hình theo chuẩn server của tổ chức.

- Nếu yêu cầu crash dump: Enable kdump.
- Nếu không dùng: phải có quyết định vận hành rõ ràng trước khi disable.

AlmaLinux installer mặc định cung cấp cấu hình kdump và automatic memory reservation. citeturn1view0

**Done**.

---

# 12. Network & Host Name

![Network and Host Name](https://wiki.almalinux.org/images/installation-guide-network.webp)

Bật NIC cần sử dụng và cấu hình hostname.

Ví dụ:

```text
Host Name: bitrix24-prod
```

Nếu dùng static IP, nhập:

```text
IP Address: <SERVER_IP>
Prefix:     <PREFIX>
Gateway:    <GATEWAY>
DNS:        <DNS1>, <DNS2>
```

### Kiểm tra

- ☐ NIC Connected
- ☐ IP đúng
- ☐ Gateway đúng
- ☐ DNS đúng
- ☐ Hostname đúng

**Done**.

---

# 13. Security Profile

AlmaLinux installer có Security Profile/SCAP. Các profile có thể bao gồm CIS, ANSSI, PCI-DSS, STIG và các baseline khác tùy release. citeturn1view0

### Khuyến nghị

Không tự chọn profile compliance chỉ vì thấy tên "CIS" hoặc "STIG". Chỉ chọn khi tổ chức đã xác định baseline và đã kiểm thử compatibility với Bitrix24.

Nếu chưa có baseline được phê duyệt:

**No profile selected** → ghi nhận quyết định trong change/implementation record.

---

# 14. Root Password

![Root configuration](https://wiki.almalinux.org/images/installation-guide-root.webp)

Đây là tài khoản đặc quyền cao nhất.

### Nguyên tắc của tài liệu Bitrix24

Sau khi OS cài xong:

- Dùng tài khoản admin riêng.
- Quản trị bằng `sudo`.
- **Không cho phép SSH trực tiếp bằng root.**
- Không ghi password vào Git.

Nếu password root được đặt trong quá trình cài, lưu trong password manager của tổ chức, không lưu trong Markdown.

---

# 15. Create Administrative User

![User Creation](https://wiki.almalinux.org/images/installation-guide-user.webp)

Tạo tài khoản quản trị riêng, ví dụ:

```text
Full Name: Bitrix Administrator
User Name: <ADMIN_USER>
```

Bật:

**Make this user administrator**

Sau khi cài xong kiểm tra:

```bash
id <ADMIN_USER>
groups <ADMIN_USER>
sudo whoami
```

Expected:

```text
root
```

### Evidence bắt buộc

- ☐ Screenshot User Creation
- ☐ `id <ADMIN_USER>`
- ☐ `sudo whoami = root`

---

# 16. Final Installation Summary

Trước khi bấm **Begin Installation**, kiểm tra:

```text
☐ Keyboard
☐ Language
☐ Time & Date
☐ Installation Source
☐ Software Selection = Minimal Install
☐ Installation Destination
☐ KDUMP
☐ Network & Host Name
☐ Security Profile / decision recorded
☐ Root Password / policy
☐ Administrative User
```

![Begin Installation](https://wiki.almalinux.org/images/installation-guide-begin-installation.webp)

Chỉ khi tất cả mục đã đúng mới chọn **Begin Installation**.

---

# 17. Installation Progress

Installer sẽ bắt đầu tạo filesystem, cài package và cấu hình hệ thống.

Không reset/power-off server trong quá trình này.

![Installation completed / reboot](https://wiki.almalinux.org/images/installation-guide-reboot.webp)

Khi quá trình hoàn tất, chọn **Reboot System**.

---

# 18. Sau reboot — đăng nhập và kiểm tra OS

Đăng nhập bằng tài khoản admin đã tạo.

### 18.1. OS

```bash
cat /etc/os-release
hostnamectl
uname -r
```

Expected:

- AlmaLinux 9.x
- Hostname đúng
- Kernel đang chạy bình thường

### 18.2. CPU / RAM / Disk

```bash
lscpu
free -h
df -h
lsblk
```

### 18.3. Network

```bash
ip addr
ip route
resolvectl status
ping -c 4 <GATEWAY>
ping -c 4 8.8.8.8
```

Nếu DNS được cấu hình:

```bash
getent hosts repo.almalinux.org
```

### 18.4. Time

```bash
timedatectl
```

Expected:

```text
Time zone: Asia/Ho_Chi_Minh
System clock synchronized: yes
```

---

# 19. Update OS sau khi cài

Trước khi cài application stack:

```bash
sudo dnf update -y
```

Kiểm tra:

```bash
sudo dnf history info last
```

Nếu kernel/system package yêu cầu reboot:

```bash
sudo reboot
```

Sau reboot kiểm tra lại:

```bash
uname -r
cat /etc/os-release
```

---

# 20. Kiểm tra SELinux

```bash
getenforce
sestatus
```

### Nguyên tắc

Không tắt SELinux một cách tùy tiện để "fix" lỗi Bitrix.

Nếu application gặp SELinux denial, xác định nguyên nhân và tạo policy/config phù hợp thay vì disable toàn bộ security control.

---

# 21. Kiểm tra Firewall

AlmaLinux sử dụng `firewalld` cho firewall management. citeturn0search4

```bash
sudo systemctl status firewalld --no-pager
sudo firewall-cmd --state
sudo firewall-cmd --list-all
```

Không mở port tùy tiện.

Các port Production sẽ được mở ở bước triển khai tương ứng, sau khi xác định architecture:

```text
SSH      = <SSH_PORT>
HTTP     = 80   (nếu cần)
HTTPS    = 443
Database = KHÔNG public nếu không có yêu cầu đặc biệt
```

---

# 22. SSH Hardening — bước bắt buộc trước khi cài Bitrix

Tài liệu vận hành Bitrix24 yêu cầu sử dụng admin riêng + `sudo`, giữ SSH port bảo mật và **không SSH trực tiếp bằng root**.

### 22.1. Kiểm tra user

```bash
id <ADMIN_USER>
groups <ADMIN_USER>
```

### 22.2. Kiểm tra sudo

```bash
sudo -l -U <ADMIN_USER>
```

### 22.3. Kiểm tra SSH effective configuration

```bash
sudo sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries'
```

Mục tiêu production:

```text
permitrootlogin no
```

> Chỉ disable password authentication sau khi đã kiểm tra SSH key login thành công và có phiên console/out-of-band để tránh lockout.

---

# 23. Final OS Readiness Checklist

| # | Check | Expected Result | Status |
|---|---|---|---|
| 1 | OS version | AlmaLinux 9.x | ☐ |
| 2 | Hostname | Đúng hostname | ☐ |
| 3 | IP | Đúng IP | ☐ |
| 4 | Gateway | Reachable | ☐ |
| 5 | DNS | Resolve được | ☐ |
| 6 | Timezone | Asia/Ho_Chi_Minh | ☐ |
| 7 | NTP | Synchronized | ☐ |
| 8 | Disk | Đủ dung lượng | ☐ |
| 9 | RAM | Đạt yêu cầu | ☐ |
| 10 | Admin user | Có sudo | ☐ |
| 11 | SELinux | Policy được xác nhận | ☐ |
| 12 | Firewalld | Policy được xác nhận | ☐ |
| 13 | SSH | Admin + sudo | ☐ |
| 14 | Root SSH | Disabled | ☐ |
| 15 | OS Update | Completed | ☐ |

---

# 24. Evidence phải lưu

Lưu screenshot vào:

```text
Bitrix24/images/01-almalinux-installation/
```

Tên file chuẩn:

```text
01-boot-menu.png
02-language.png
03-installation-summary.png
04-keyboard.png
05-time-date.png
06-software-selection.png
07-installation-destination.png
08-network-hostname.png
09-kdump.png
10-user-creation.png
11-final-summary.png
12-installation-progress.png
13-reboot.png
14-os-version.png
15-network-test.png
16-disk-memory.png
17-selinux.png
18-firewalld.png
19-ssh-hardening.png
```

> Các file từ `14` trở đi phải ưu tiên là **screenshot thực tế của server triển khai**, không dùng ảnh minh họa từ Internet làm evidence.

---

## 25. Chuyển sang bước tiếp theo

Sau khi toàn bộ **OS Readiness Checklist = PASS**, chuyển sang:

**01 — Step-by-Step Installation**

→ cài Web Server  
→ PHP/PHP-FPM  
→ Database  
→ Bitrix Environment  
→ các thành phần phụ trợ  
→ kiểm tra service.

---

## Reference

- AlmaLinux Installation Guide: https://wiki.almalinux.org/documentation/installation-guide
- AlmaLinux After-Installation Guide: https://wiki.almalinux.org/documentation/after-installation-guide.html
