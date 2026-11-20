# Cách bảo mật cho Linux

## 1. Mục đích

Tài liệu này tổng hợp toàn bộ quá trình hardening một Ubuntu Server mới trong quá trình triển khai VPS **Thiên Tâm**, tập trung vào SSH, tài khoản quản trị, UFW, AppArmor, cập nhật bảo mật và kiểm tra bề mặt dịch vụ.

> Phạm vi: Ubuntu Server mới cài, quản trị qua SSH/PuTTY, SSH chỉ cho phép từ mạng `192.168.100.0/24`, SSH dùng key, không đăng nhập root/password trực tiếp.

---

## 2. Baseline ban đầu

SSH ban đầu đang listen trên port `10012`:

```bash
sudo ss -lntp | grep ssh
```

Kết quả:

```text
0.0.0.0:10012
[::]:10012
```

Cấu hình SSH ban đầu cho phép:

```text
port 10012
permitrootlogin yes
pubkeyauthentication yes
passwordauthentication yes
maxauthtries 6
```

Kiểm tra cấu hình hiệu lực:

```bash
sudo sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|pubkeyauthentication|allowusers|allowgroups|maxauthtries'
```

### Tài khoản ban đầu

Có tài khoản quản trị:

```text
namdag
UID=1000
/home/namdag
/bin/bash
```

`namdag` thuộc nhóm `sudo`, nên có thể dùng `sudo` để quản trị hệ thống.

`/root/.ssh/authorized_keys` ban đầu rỗng.

`/home/namdag/.ssh/authorized_keys` ban đầu cũng rỗng.

---

## 3. Xây dựng tài khoản quản trị SSH bằng key

### 3.1. Dùng tài khoản `namdag`

Kiểm tra:

```bash
id namdag
groups namdag
```

Kết quả xác nhận `namdag` có nhóm `sudo`.

### 3.2. SSH key

Máy quản trị Windows đã có bộ key:

```text
D:\Working\SSH_KEY\
├── id_namdag
├── id_namdag.pub
└── id_namdag-putty.ppk
```

Có thể dùng một public key trên nhiều Linux server về mặt kỹ thuật, nhưng về bảo mật production nên ưu tiên **mỗi server một key riêng** để khi một private key bị lộ có thể thu hồi độc lập.

Không đưa private key (`id_namdag`, `.ppk`) lên server.

Chỉ đưa public key (`id_namdag.pub`) vào:

```text
/home/namdag/.ssh/authorized_keys
```

### 3.3. Quyền SSH key

```bash
sudo chown -R namdag:namdag /home/namdag/.ssh
sudo chmod 700 /home/namdag/.ssh
sudo chmod 600 /home/namdag/.ssh/authorized_keys
```

### 3.4. Kiểm tra đăng nhập bằng PuTTY

Đăng nhập bằng:

```text
Host: IP_VPS
Port: 10012
User: namdag
Private key: id_namdag-putty.ppk
```

Xác minh sudo:

```bash
sudo whoami
```

Kết quả phải là:

```text
root
```

**Chỉ sau khi `namdag` đăng nhập bằng key thành công mới được khóa root/password SSH.**

---

## 4. Hardening SSH

### 4.1. File cấu hình riêng

Tạo:

```bash
sudo nano /etc/ssh/sshd_config.d/99-security.conf
```

Cấu hình cuối cùng:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
AllowUsers namdag
```

### 4.2. Lưu ý về `50-cloud-init.conf`

Ubuntu cloud-init có file:

```text
/etc/ssh/sshd_config.d/50-cloud-init.conf
```

Trong quá trình kiểm tra phát hiện:

```text
PasswordAuthentication yes
```

Giá trị này khiến cấu hình hiệu lực vẫn là `yes`. File được chỉnh thành:

```text
PasswordAuthentication no
```

Sau đó kiểm tra toàn bộ cấu hình:

```bash
sudo sshd -T | grep -E 'port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|maxauthtries|allowusers'
```

Kết quả cuối cùng:

```text
port 10012
maxauthtries 3
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
kbdinteractiveauthentication no
allowusers namdag
```

Kiểm tra forwarding:

```bash
sudo sshd -T | grep -E 'x11forwarding|allowagentforwarding|allowtcpforwarding'
```

Kết quả cuối:

```text
x11forwarding no
allowtcpforwarding no
allowagentforwarding no
```

### 4.3. Kiểm tra cú pháp trước khi reload

```bash
sudo sshd -t
```

Không có output nghĩa là cấu hình hợp lệ.

Sau đó reload thay vì restart:

```bash
sudo systemctl reload ssh
```

Kiểm tra:

```bash
sudo systemctl status ssh --no-pager
```

### 4.4. Quy tắc vận hành an toàn

Luôn giữ phiên SSH/PuTTY hiện tại mở khi thay đổi SSH.

Mở một phiên mới để test `namdag` trước khi đóng phiên cũ.

Không chạy `systemctl reload ssh` hoặc thay đổi `PermitRootLogin`/`PasswordAuthentication` trước khi xác nhận tài khoản quản trị mới đăng nhập được bằng key.

---

## 5. UFW Firewall

### 5.1. Trạng thái ban đầu

```bash
sudo ufw status numbered
```

Ban đầu:

```text
Status: inactive
```

### 5.2. Chính sách mặc định

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 5.3. Chỉ cho phép SSH từ mạng quản trị

SSH `10012/tcp` chỉ cho phép từ:

```text
192.168.100.0/24
```

Rule:

```bash
sudo ufw allow from 192.168.100.0/24 to any port 10012 proto tcp
```

### 5.4. Web public

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### 5.5. Bật UFW

Sau khi kiểm tra rules:

```bash
sudo ufw enable
```

Kết quả cuối:

```text
[ 1] 10012/tcp  ALLOW IN  192.168.100.0/24
[ 2] 80/tcp     ALLOW IN  Anywhere
[ 3] 443/tcp    ALLOW IN  Anywhere
[ 4] 80/tcp (v6)  ALLOW IN  Anywhere (v6)
[ 5] 443/tcp (v6) ALLOW IN  Anywhere (v6)
```

Kiểm tra:

```bash
sudo ufw status verbose
```

Kết quả:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
```

### 5.6. Nguyên tắc

Không mở SSH `10012` cho `Anywhere` nếu SSH chỉ cần dùng từ VPN/LAN quản trị.

Không mở các port dịch vụ như database/Redis/Docker API ra Internet nếu không thực sự cần.

---

## 6. Kiểm tra listening ports

Sau khi hardening OS mới:

```bash
sudo ss -lntup
```

Baseline quan sát được:

```text
127.0.0.53:53     systemd-resolve
127.0.0.54:53     systemd-resolve
0.0.0.0:10012     sshd
[::]:10012        sshd
```

Không phát hiện tại thời điểm kiểm tra các port public kiểu:

```text
3306   MySQL
5432   PostgreSQL
6379   Redis
2375   Docker API
9000   MinIO
8080   API
```

Sau khi cài Docker/Caddy phải kiểm tra lại:

```bash
sudo ss -lntup
docker ps
```

Không nên public database/Redis/management API nếu container chỉ cần giao tiếp nội bộ.

---

## 7. AppArmor

Kiểm tra:

```bash
sudo aa-status
```

Kết quả trên OS mới:

```text
AppArmor module loaded
118 profiles loaded
23 profiles enforce
4 profiles complain
91 profiles unconfined
1 process with defined profile in enforce mode
```

Các profile Transmission ở `complain` không phải vấn đề nếu server không chạy Transmission.

Không nên tự động chuyển hàng loạt profile `unconfined` sang `enforce` khi chưa có yêu cầu, vì có thể gây lỗi ứng dụng.

Điểm quan trọng là AppArmor module đã loaded và các profile phù hợp đang được enforce.

---

## 8. Automatic Security Updates

Kiểm tra:

```bash
systemctl is-enabled unattended-upgrades 2>/dev/null
```

Kết quả:

```text
enabled
```

Kiểm tra/cài:

```bash
sudo apt install unattended-upgrades -y
```

Trong phiên kiểm tra, package đã ở phiên bản mới nhất và service đã enabled.

---

## 9. Kiểm tra user và UID 0

Liệt kê user thường có shell:

```bash
getent passwd | awk -F: '$3 >= 1000 && $7 ~ /(bash|sh)$/ {print $1, $3, $6, $7}'
```

Kết quả:

```text
namdag 1000 /home/namdag /bin/bash
```

Kiểm tra UID 0:

```bash
awk -F: '$3 == 0 {printf "%s UID=%s\\n", $1, $3}' /etc/passwd
```

Kết quả:

```text
root UID=0
```

Đây là trạng thái bình thường; không có tài khoản UID 0 bất thường.

---

## 10. SSH forwarding

Trạng thái ban đầu:

```text
x11forwarding no
allowtcpforwarding yes
allowagentforwarding yes
```

Vì VPS chỉ sử dụng để quản trị bằng SSH/PuTTY và không cần SSH tunnel/agent forwarding, đã harden thành:

```text
x11forwarding no
allowtcpforwarding no
allowagentforwarding no
```

Nếu sau này cần `ssh -L`, `ssh -R`, VS Code Remote SSH tunneling hoặc agent forwarding thì phải xem xét bật lại tính năng tương ứng.

---

## 11. Vì sao không chỉ đổi port SSH?

Đổi port `22` thành `10012` có tác dụng giảm scan/bot tự động nhưng **không phải biện pháp bảo mật chính**.

Các lớp quan trọng hơn:

1. SSH key.
2. Tắt `PermitRootLogin`.
3. Tắt `PasswordAuthentication`.
4. Chỉ cho phép user cần thiết.
5. UFW default deny.
6. Giới hạn source IP/subnet.
7. Tắt forwarding không cần thiết.
8. Cập nhật bảo mật.
9. Kiểm tra listening ports và service.

---

## 12. Có nên cài Fail2ban?

Với cấu hình hiện tại:

```text
Internet
  -> UFW
  -> SSH chỉ từ 192.168.100.0/24
  -> SSH key only
```

Fail2ban không còn là lớp bắt buộc cho SSH, đặc biệt nếu `192.168.100.0/24` là mạng VPN/private đáng tin cậy.

Nếu SSH sau này phải public Internet, có thể bổ sung Fail2ban:

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

---

## 13. IPv6

SSH đang listen cả IPv4 và IPv6:

```text
0.0.0.0:10012
[::]:10012
```

UFW hiện chỉ cho phép SSH `10012` từ IPv4 subnet `192.168.100.0/24`. Do default incoming deny, SSH IPv6 không được mở public bởi rule hiện tại.

Không nên tắt IPv6 chỉ vì nhìn thấy `[::]:10012`; chỉ tắt nếu hệ thống thực sự không dùng IPv6 và đã kiểm tra DNS/service/network dependency.

---

## 14. Các kiểm tra audit hữu ích

### SSH log

```bash
sudo journalctl -u ssh --since today
```

Failed login:

```bash
sudo journalctl -u ssh | grep -i "failed"
```

### Reboot / session

```bash
last -x | head -20
```

### SSH keys

```bash
sudo find /home /root -name authorized_keys -type f -exec ls -l {} \;
```

### Service đang chạy

```bash
systemctl --type=service --state=running
```

### Process sử dụng CPU

```bash
ps aux --sort=-%cpu | head -20
```

### Port

```bash
sudo ss -lntup
```

---

## 15. Cấu hình cuối cùng của VPS Thiên Tâm

### SSH

```text
Port                    10012
PermitRootLogin         no
PasswordAuthentication  no
KbdInteractiveAuthentication no
PubkeyAuthentication    yes
PermitEmptyPasswords    no
MaxAuthTries            3
LoginGraceTime          30
X11Forwarding           no
AllowTcpForwarding      no
AllowAgentForwarding    no
AllowUsers              namdag
```

### Firewall

```text
Default incoming        deny
Default outgoing        allow
SSH 10012               192.168.100.0/24 only
HTTP 80                 public
HTTPS 443               public
```

### Host security

```text
AppArmor                loaded/enforced
unattended-upgrades     enabled
UID 0                   only root
Normal login shell      only namdag
```

---

## 16. Baseline sau hardening

```text
                 INTERNET
                    │
          ┌─────────┴─────────┐
          │                   │
        80/443              SSH 10012
          │                   │
          │         192.168.100.0/24 only
          │                   │
          ▼                   ▼
       Web/Caddy           namdag
                              │
                           SSH Key
                              │
                            sudo
                              │
                              ▼
                             root
```

Không có SSH password.

Không có root SSH.

Không có SSH public Internet nếu source không nằm trong `192.168.100.0/24`.

---

## 17. Checklist triển khai OS mới

- [x] Kiểm tra SSH hiện tại.
- [x] Xác nhận port SSH.
- [x] Xác nhận user quản trị.
- [x] Xác nhận user có sudo.
- [x] Cài public key cho user quản trị.
- [x] Test đăng nhập bằng PuTTY/SSH key.
- [x] `PermitRootLogin no`.
- [x] `PasswordAuthentication no`.
- [x] `KbdInteractiveAuthentication no`.
- [x] `PubkeyAuthentication yes`.
- [x] `MaxAuthTries 3`.
- [x] `AllowUsers namdag`.
- [x] `X11Forwarding no`.
- [x] `AllowTcpForwarding no`.
- [x] `AllowAgentForwarding no`.
- [x] `sshd -t` trước reload.
- [x] Reload SSH và test phiên mới.
- [x] UFW default deny incoming.
- [x] UFW allow SSH theo subnet.
- [x] UFW allow 80/443.
- [x] Kiểm tra `ss -lntup`.
- [x] Kiểm tra AppArmor.
- [x] Kiểm tra unattended-upgrades.
- [x] Kiểm tra user shell.
- [x] Kiểm tra UID 0.

---

## 18. Giai đoạn bảo mật tiếp theo sau khi cài Docker/Caddy

Sau khi bắt đầu triển khai ứng dụng, trọng tâm bảo mật chuyển sang:

- Docker daemon và Docker socket.
- Không public Docker API `2375/2376`.
- Container network.
- Chỉ publish port thực sự cần.
- Không public DB/Redis/MinIO nếu không cần.
- Caddy reverse proxy.
- TLS/HTTPS.
- Secrets và file `.env`.
- Quyền filesystem/volume.
- Backup và restore test.
- Monitoring và alerting.
- Log rotation.
- Update image/package định kỳ.

**Nguyên tắc:** không mở port chỉ vì container có service đang listen; trước tiên xác định service đó cần public Internet hay chỉ cần giao tiếp nội bộ Docker/private network.

---

## 19. Ghi chú vận hành an toàn

1. Luôn giữ một phiên SSH đang hoạt động khi chỉnh `sshd_config`.
2. Luôn chạy `sudo sshd -t` trước reload/restart.
3. Sau khi reload, mở một phiên SSH mới để xác nhận.
4. Không gửi private key vào chat, Git hoặc server.
5. Public key có thể được chia sẻ; private key phải được bảo vệ.
6. Không dùng chung private key cho nhiều server production nếu có thể tạo key riêng.
7. Mỗi lần cài thêm dịch vụ phải kiểm tra lại `sudo ss -lntup` và firewall.
8. Với hệ thống quan trọng, nên quản trị SSH thông qua VPN/private network thay vì public Internet.

---

## 20. Trạng thái hoàn tất của phiên Thiên Tâm

**SSH + Firewall + OS hardening cơ bản đã được hoàn tất và có thể dùng làm baseline trước khi triển khai Docker/Caddy.**
