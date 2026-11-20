# GitHub Actions – Self-hosted Runner và Runner Group

> **Mục đích:** Tổng hợp kiến thức, kiến trúc, mục đích sử dụng, cách triển khai và nguyên tắc vận hành **GitHub Actions Self-hosted Runner** và **Runner Group** cho hệ thống Server/DevOps.
>
> **Bối cảnh thực tế:** Tài liệu được tổng hợp từ quá trình trao đổi về việc triển khai Runner trên VPS Ubuntu 24.04 + aaPanel, mục tiêu CI/CD, tránh xung đột port, phân tách Production/Staging/AI, bảo mật, rollback và khả năng mở rộng/HA.
>
> **Phạm vi:** Tài liệu tập trung vào GitHub Actions Runner, không thay thế tài liệu riêng về aaPanel, Docker, Nginx/Caddy, database HA hoặc Load Balancer.

---

## 1. Executive Summary

**Self-hosted Runner** biến một máy chủ do mình quản lý thành **worker thực thi GitHub Actions**. GitHub giữ vai trò điều phối workflow; Runner chạy các lệnh CI/CD trên chính hạ tầng của mình.

Mô hình cơ bản:

```text
Developer
   │
   │ git push / Pull Request / manual / schedule
   ▼
GitHub Repository
   │
   ▼
GitHub Actions Workflow
   │
   ▼
Self-hosted Runner
   │
   ├── Checkout
   ├── Test
   ├── Build
   ├── Docker
   ├── Deploy
   ├── Health Check
   └── Rollback
   │
   ▼
Production / Staging / AI Infrastructure
```

GitHub xác nhận self-hosted runner có thể là máy Linux, Windows hoặc macOS đáp ứng yêu cầu hệ điều hành, kiến trúc, tài nguyên và kết nối tới GitHub Actions. Ubuntu 20.04+ được hỗ trợ; Linux runner có thể chạy Docker container actions/service containers khi Docker được cài đặt. citeturn632625search0

**Kết luận cốt lõi:**

- Runner là **worker CI/CD**, không phải web server.
- Runner **không phải Load Balancer**.
- Runner **không tự tạo Application HA**.
- Runner **không phải cơ chế failover application**.
- Runner có thể tham gia vào hệ thống HA/automation nếu bên dưới đã có nhiều Runner và/hoặc nhiều server application.

---

## 2. Self-hosted Runner được triển khai nhằm mục đích gì?

### 2.1. Tự động hóa CI/CD

Thay vì phải SSH vào server rồi thao tác thủ công:

```text
SSH
→ git pull
→ npm install
→ build
→ restart
```

Có thể chuyển thành:

```text
Git push
→ GitHub Actions
→ Self-hosted Runner
→ Test
→ Build
→ Deploy
→ Verify
```

Đây là mục đích quan trọng nhất.

### 2.2. Chạy trực tiếp trên hạ tầng do mình kiểm soát

Runner có thể sử dụng các phần mềm, filesystem, network và tài nguyên đã chuẩn bị trên server:

```text
Ubuntu 24.04
├── Node.js
├── PHP
├── Python
├── Docker
├── FFmpeg
├── yt-dlp
├── Composer
├── CUDA / GPU
└── Internal services
```

Điều này đặc biệt hữu ích với các workload AI/Video có dependency nặng hoặc phần cứng đặc thù.

### 2.3. Thao tác với hệ thống nội bộ

Runner nằm trong infrastructure có thể gọi trực tiếp:

```text
127.0.0.1 services
Docker network
Redis
Database nội bộ
Private API
NAS / mounted storage
```

GitHub-hosted runner nằm ngoài infrastructure nên không có cùng mức truy cập nội bộ.

### 2.4. Chuẩn hóa vận hành

Runner có thể thực hiện script chuẩn hóa:

```text
Backup
Health Check
Migration
Cleanup
Deploy
Restart
Rollback
```

Từ đó biến quy trình vận hành thành một pipeline có log, trạng thái và lịch sử chạy trên GitHub Actions.

---

## 3. Self-hosted Runner không phải là gì?

Cần phân biệt rõ để tránh thiết kế sai:

| Thành phần | Vai trò |
|---|---|
| Self-hosted Runner | Máy thực thi job GitHub Actions |
| Runner Group | Phân vùng quyền và tập hợp Runner |
| Runner Label | Mô tả đặc tính để định tuyến job |
| Nginx/Caddy | Web server / Reverse Proxy |
| Load Balancer | Phân phối traffic và hỗ trợ application failover |
| Docker | Runtime/container platform |
| systemd/PM2 | Quản lý process/service |
| Database HA | Đảm bảo khả năng chịu lỗi tầng database |

**Runner không thay thế các thành phần trên.**

---

## 4. Runner Group là gì?

Runner Group là cơ chế của GitHub để **tổ chức Runner và tạo security boundary** quanh chúng. Runner Group cho phép kiểm soát repository nào được phép sử dụng tập Runner đó, và có thể định tuyến job vào group cụ thể. Một Runner chỉ thuộc một group tại một thời điểm. citeturn632625search1turn632625search4

Ví dụ:

```text
Organization
│
├── PRODUCTION
│   ├── prod-01
│   └── prod-02
│
├── STAGING
│   └── staging-01
│
├── BUILD
│   └── build-01
│
└── AI
    ├── gpu-01
    └── gpu-02
```

### 4.1. Runner Group dùng để làm gì?

- Giới hạn repository nào được phép chạy trên Runner.
- Tách Production / Staging / Build / AI.
- Tạo boundary về quyền sử dụng hạ tầng.
- Định tuyến workflow tới đúng nhóm Runner.
- Tổ chức nhiều Runner khi hạ tầng lớn lên.
- Kết hợp với labels để chọn đúng loại máy.

GitHub cho phép Runner Group có policy truy cập theo một danh sách repository hoặc toàn bộ repository trong organization; theo tài liệu hiện hành, mặc định runner group chỉ cho private repositories truy cập và có thể override theo chính sách. citeturn632625search4

---

## 5. Runner Label là gì?

Label mô tả đặc tính của Runner và được dùng để định tuyến job.

Runner tự động có các label như:

```text
self-hosted
linux / windows / macOS
x64 / ARM / ARM64
```

Có thể tạo label tùy chỉnh:

```text
production
staging
ubuntu24
node
php
 docker
gpu
cuda
```

Labels được kết hợp theo kiểu **AND**: Runner phải có đầy đủ các label yêu cầu mới đủ điều kiện nhận job. citeturn632625search2turn632625search5

Ví dụ:

```yaml
runs-on: [self-hosted, linux, x64, production]
```

Chỉ Runner có tất cả bốn label này mới có thể nhận job.

---

## 6. Runner Group và Label khác nhau thế nào?

| Tiêu chí | Runner Group | Runner Label |
|---|---|---|
| Mục tiêu | Security boundary + tổ chức | Mô tả đặc tính Runner |
| Phạm vi | Group của Runner | Thuộc tính Runner |
| Repository access | Có | Không phải cơ chế chính |
| Chọn loại máy | Có thể gián tiếp | Rất phù hợp |
| Ví dụ | `PRODUCTION` | `gpu`, `ubuntu24`, `x64` |
| Workflow | `group: PRODUCTION` | `labels: production` |

Khi kết hợp, job phải thỏa **cả group và label**. citeturn632625search5

Ví dụ:

```yaml
runs-on:
  group: PRODUCTION
  labels: ubuntu-24.04
```

---

## 7. Runner có chiếm Port của Server không?

### 7.1. Câu trả lời: Không theo cách hiểu thông thường

Self-hosted Runner chủ yếu giao tiếp outbound với GitHub Actions. Nó **không phải một HTTP server** cần mở `:3000`, `:8080`, `:80` hoặc `:443` để nhận request từ GitHub.

GitHub yêu cầu Runner có thể giao tiếp với GitHub Actions và dùng kết nối HTTPS outbound; vì vậy việc cài Runner không đồng nghĩa với việc phải chiếm thêm port web đang được aaPanel/Nginx/Caddy sử dụng. citeturn632625search0

Mô hình:

```text
aaPanel / Nginx / Caddy
├── :80
├── :443
└── các port application

Self-hosted Runner
└── outbound HTTPS → GitHub
```

### 7.2. Cái gì mới gây port conflict?

**Application được Runner khởi chạy**, chứ không phải bản thân Runner.

Ví dụ:

```text
App A → :3000
App B → :3000
```

Nếu cả hai cùng bind `:3000` trên cùng IP thì xảy ra `EADDRINUSE`.

Vì vậy cần thiết kế **Application Port Isolation** độc lập với Runner Group.

---

## 8. Thiết kế Port Isolation cho aaPanel

Với VPS aaPanel, nên quy hoạch:

```text
Internet
   │
   ▼
Nginx/Caddy :80/:443
   │
   ├── review360.id.vn
   ├── domain-B.vn
   └── api.review360.id.vn
             │
             ▼
        127.0.0.1:3100
```

Các application nội bộ có thể dùng các port khác nhau:

```text
Media API       → 127.0.0.1:3100
Service B       → 127.0.0.1:3200
Service C       → 127.0.0.1:3300
```

Runner chỉ thực hiện:

```text
Deploy
→ Build
→ Restart
→ Health Check
```

Runner **không cần giữ một port riêng cho mỗi project**.

---

## 9. Mục tiêu triển khai thực tế trên VPS ThienTam

Bối cảnh đề xuất:

```text
Ubuntu 24.04
+ aaPanel
+ Nginx/Caddy
+ Docker
+ Node.js
+ PHP
+ Python
```

Kiến trúc ban đầu:

```text
GitHub
│
└── Runner Group: PRODUCTION
      │
      └── thientam-prod
            ├── self-hosted
            ├── linux
            ├── x64
            └── production
```

**Giai đoạn đầu chỉ cần 1 Runner Production trên VPS.**

Không cần tạo nhiều Runner chỉ vì có nhiều repository. Một Runner có thể xử lý nhiều job theo quyền/labels phù hợp; khi workload hoặc yêu cầu isolation tăng thì mới thêm Runner.

---

## 10. Cài đặt Self-hosted Runner trên Ubuntu 24.04

### 10.1. Tạo user riêng

Không nên chạy Runner bằng `root` trên production.

```bash
sudo adduser --disabled-password --gecos "" github-runner
```

Nếu Runner cần điều khiển Docker:

```bash
sudo usermod -aG docker github-runner
```

Kiểm tra:

```bash
id github-runner
```

### 10.2. Tạo thư mục

```bash
sudo mkdir -p /opt/actions-runner
sudo chown -R github-runner:github-runner /opt/actions-runner
```

Chuyển user:

```bash
sudo su - github-runner
cd /opt/actions-runner
```

### 10.3. Download Runner

Trên GitHub:

```text
Repository/Organization
→ Settings
→ Actions
→ Runners
→ New self-hosted runner
→ Linux
→ x64
```

GitHub sẽ cung cấp đúng URL/package/command cho phiên bản Runner hiện tại. Không nên tự hard-code một version cũ trong tài liệu vận hành.

### 10.4. Register Runner

Ví dụ cấu trúc:

```bash
./config.sh \
  --url https://github.com/OWNER/REPOSITORY \
  --token YOUR_TEMPORARY_REGISTRATION_TOKEN
```

Token đăng ký Runner là giá trị tạm thời; không ghi vào Git, không đưa vào tài liệu công khai.

### 10.5. Test foreground

```bash
./run.sh
```

Khi thấy Runner connected/listening for jobs thì dừng foreground test và cài dạng service.

### 10.6. Cài systemd service

```bash
sudo ./svc.sh install github-runner
sudo ./svc.sh start
sudo ./svc.sh status
```

Mục tiêu:

```text
Server reboot
→ systemd
→ Runner tự khởi động
```

---

## 11. Kiểm tra sau khi cài

### Trên server

```bash
sudo ./svc.sh status
```

```bash
ps aux | grep Runner
```

```bash
curl -I https://github.com
```

### Trên GitHub

```text
Settings
→ Actions
→ Runners
```

Runner mong muốn:

```text
thientam-prod
● Idle
```

GitHub sẽ chỉ giao job tới Runner đang online và idle có labels/groups phù hợp. Nếu chưa có Runner phù hợp, job nằm ở trạng thái queued; tài liệu GitHub nêu rằng nếu job bị queued quá 24 giờ thì sẽ fail. citeturn632625search0

---

## 12. Workflow test tối thiểu

Tạo:

```text
.github/workflows/runner-test.yml
```

```yaml
name: Test Self Hosted Runner

on:
  push:
    branches:
      - main

jobs:
  test-runner:
    runs-on: [self-hosted, linux, x64, production]

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: System information
        run: |
          hostname
          whoami
          uname -a

      - name: Node
        run: |
          node --version || true
          npm --version || true

      - name: Docker
        run: |
          docker --version || true

      - name: Runner test
        run: echo "Self-hosted Runner is working"
```

Cách dùng labels theo tổ hợp là cơ chế định tuyến chuẩn của GitHub Actions. citeturn632625search5

---

## 13. Concurrency để tránh deployment đè nhau

Port conflict không phải vấn đề duy nhất. Hai deployment chạy cùng lúc vào cùng một release directory cũng có thể phá production.

Nên khóa deployment:

```yaml
concurrency:
  group: production-deploy
  cancel-in-progress: false
```

Mô hình:

```text
Deploy A → RUNNING
Deploy B → WAITING
Deploy C → WAITING

A DONE
↓
B START
↓
B DONE
↓
C START
```

Điều này ngăn nhiều pipeline cùng sửa một production target tại cùng thời điểm.

---

## 14. CI/CD chuẩn nên đi theo pipeline nào?

```text
Developer
   │
   ▼
git push
   │
   ▼
GitHub Actions
   │
   ├── Checkout
   ├── Lint
   ├── Unit Test
   ├── Build
   ├── Security/quality checks
   ├── Backup (trước deploy nếu cần)
   ├── Deploy
   ├── Health Check
   └── Rollback nếu thất bại
```

Không nên coi CI/CD chỉ là:

```bash
git pull
npm install
pm2 restart app
```

Một deployment production nên có **Verify** sau Deploy.

---

## 15. Health Check và Rollback

Một flow an toàn:

```text
Build
 ↓
Deploy
 ↓
Health Check
 ├── OK → Release
 └── FAIL → Rollback
```

Ví dụ health endpoint:

```bash
curl -fsS http://127.0.0.1:3100/health
```

Nếu application trả về lỗi:

```text
Deploy FAIL
→ log failure
→ rollback version cũ
→ health check lần nữa
```

---

## 16. Blue-Green Deployment để tránh downtime

Nếu muốn giảm downtime và giảm rủi ro port/process conflict:

```text
                 Nginx/Caddy
                      │
               ┌──────┴──────┐
               │             │
               ▼             ▼
          BLUE :3100     GREEN :3200
          version A      version B
```

Traffic đang ở BLUE.

Runner:

```text
Deploy version B vào GREEN
→ Health Check GREEN
→ Nếu OK: chuyển traffic sang GREEN
→ Tắt BLUE sau khi ổn định
```

Nếu GREEN lỗi:

```text
GREEN ❌
→ giữ BLUE
→ không ảnh hưởng production
```

Blue-Green là **deployment strategy**, không phải tính năng riêng của Runner.

---

## 17. Runner Failover có nghĩa gì?

Cần phân biệt hai loại failover.

### 17.1. Runner failover

Có nhiều Runner cùng đủ điều kiện:

```text
PRODUCTION GROUP
├── runner-01 ✅
├── runner-02 ✅
└── runner-03 ✅
```

Nếu `runner-01` offline, các job mới có thể được nhận bởi Runner khác phù hợp. GitHub định tuyến job theo runner group + labels và trạng thái online/idle. citeturn632625search0turn632625search5

### 17.2. Application failover

Đây là chuyện khác:

```text
Load Balancer
    │
 ┌──┴──┐
 ▼     ▼
App A App B
```

Nếu Server A chết, Load Balancer chuyển traffic sang Server B.

**Runner failover không tự tạo application failover.**

Nếu chỉ có:

```text
Server A
└── runner-01
```

và Server A chết thì application cũng chết, dù GitHub có thể còn ghi nhận Runner offline.

---

## 18. Kiến trúc HA đúng

### 18.1. HA tầng CI/CD

```text
GitHub
   │
   ▼
PRODUCTION GROUP
   ├── Runner-A
   └── Runner-B
```

### 18.2. HA tầng Application

```text
Load Balancer
   │
   ├── Server-A
   └── Server-B
```

### 18.3. HA tầng Database

```text
DB-A
 ⇅
DB-B
```

### 18.4. Storage

Dữ liệu quan trọng nên nằm trong hệ thống storage có chiến lược backup/replication phù hợp.

**Muốn có HA thực sự phải thiết kế HA ở từng tầng.**

---

## 19. Mô hình Runner Group được đề xuất cho hệ thống

### Giai đoạn 1 – VPS hiện tại

```text
PRODUCTION
└── thientam-prod
```

Labels:

```text
self-hosted
linux
x64
production
ubuntu24
```

### Giai đoạn 2 – Có Staging

```text
PRODUCTION
└── prod-01

STAGING
└── staging-01
```

Flow:

```text
Feature
→ PR
→ Test
→ Staging
→ Approval
→ Production
```

### Giai đoạn 3 – Production HA

```text
PRODUCTION
├── prod-runner-01
└── prod-runner-02

Production Infrastructure
├── server-01
└── server-02
```

### Giai đoạn 4 – AI/GPU

```text
AI
├── gpu-01
└── gpu-02
```

Workflow:

```yaml
runs-on: [self-hosted, linux, x64, gpu, cuda]
```

---

## 20. Runner Group không giải quyết Port Conflict

Sai lầm phổ biến:

```text
Runner Group = chống đụng port
```

Không đúng.

Đúng là:

```text
Runner Group
→ Access / organization / repository boundary

Runner Label
→ Capability routing

Application architecture
→ Port isolation

Concurrency
→ Prevent deployment race

Blue-Green
→ Isolate release versions

Nginx/Caddy
→ Reverse proxy / public ports
```

---

## 21. Security – nguyên tắc bắt buộc

### 21.1. Chỉ dùng Self-hosted Runner cho repository tin cậy/private

GitHub cảnh báo self-hosted runners với public repository có rủi ro, vì fork có thể tạo pull request dẫn tới workflow chạy mã nguy hiểm trên máy Runner. GitHub khuyến nghị chỉ dùng self-hosted runners với private repositories. citeturn632625search3

### 21.2. Không chạy Runner bằng root

Dùng user riêng:

```text
github-runner
```

và cấp đúng quyền cần thiết.

### 21.3. Secrets không đưa vào Git

Không commit:

```text
password
API key
private key
registration token
production credential
```

Dùng GitHub Secrets/Variables hoặc cơ chế credential an toàn phù hợp.

### 21.4. Không cho workflow quyền rộng hơn cần thiết

Đặc biệt tránh workflow tùy ý thao tác:

```bash
rm -rf /www/*
```

hoặc restart toàn server nếu job chỉ cần deploy một service.

### 21.5. Tách Runner theo trust boundary

Ví dụ:

```text
PRODUCTION ≠ STAGING ≠ BUILD ≠ AI
```

nếu các workload có mức tin cậy/quyền khác nhau.

### 21.6. Không dùng production Runner cho code chưa tin cậy

Pull Request từ code chưa review không nên được trao quyền trực tiếp tới production runner.

GitHub mô tả rõ rằng nếu runner bị compromise, attacker có thể tác động tới secrets, GITHUB_TOKEN và hệ thống mà Runner có quyền truy cập. citeturn632625search8

---

## 22. Khi nào cần nhiều Runner?

Không cần nhiều Runner chỉ vì có nhiều repository.

Nên thêm Runner khi có một trong các lý do:

1. **Concurrency:** nhiều job cần chạy đồng thời.
2. **Isolation:** Production không muốn chạy cùng workload Build/AI.
3. **Special hardware:** GPU/CPU/RAM/FFmpeg/CUDA khác nhau.
4. **Security boundary:** workload có mức tin cậy khác nhau.
5. **HA:** cần Runner dự phòng.
6. **Performance:** job build/test nặng.

Ví dụ:

```text
1 Runner
→ đủ cho workload nhỏ

2–3 Runner
→ concurrency / isolation

Dedicated GPU Runner
→ AI workload

2 Production Runners
→ CI/CD redundancy
```

---

## 23. Các use case quan trọng

### Use Case A – Deploy Node.js

```text
git push
→ test
→ npm ci
→ npm test
→ build
→ restart service
→ health check
```

### Use Case B – Deploy Docker

```text
git push
→ test
→ docker build
→ docker compose up -d
→ health check
```

### Use Case C – Backup trước Deploy

```text
backup DB
→ backup config
→ deploy
→ verify
```

### Use Case D – Rollback

```text
release N
→ health check FAIL
→ rollback release N-1
```

### Use Case E – AI workload

```text
push
→ GPU Runner
→ Python tests
→ model test
→ FFmpeg test
→ build artifact
```

### Use Case F – Scheduled Operations

Runner có thể thực thi workflow theo schedule/manual dispatch để thực hiện:

```text
Backup
Cleanup
Health Check
Maintenance
```

---

## 24. Những gì Runner KHÔNG tự giải quyết

| Vấn đề | Công cụ/kiến trúc cần có |
|---|---|
| Port conflict | Application port design |
| Zero downtime | Blue-Green / Rolling / Canary |
| Application failover | Load Balancer + multiple app servers |
| Database failover | Database HA/replication |
| Shared storage HA | Replication/Object Storage/backup |
| Secret management | GitHub Secrets/OIDC/Vault tùy kiến trúc |
| Process crash | systemd/PM2/Docker restart policies |
| Container orchestration | Docker Compose/Kubernetes/etc. |

---

## 25. Khuyến nghị kiến trúc cho hệ thống ThienTam

### Hiện tại

```text
                         GitHub
                           │
                           ▼
                  PRODUCTION GROUP
                           │
                           ▼
                    thientam-prod
                           │
                           ▼
               Ubuntu 24.04 + aaPanel
                           │
           ┌───────────────┼────────────────┐
           ▼               ▼                ▼
        Nginx/Caddy      Docker         Application
          :80/:443       services          ports
```

### Nguyên tắc

- Chỉ 1 Production Runner ở giai đoạn đầu.
- Runner chạy bằng user riêng.
- Cài systemd service.
- Không mở port riêng cho Runner.
- Ứng dụng dùng port riêng theo service.
- Public access đi qua Nginx/Caddy.
- Dùng `concurrency` để tránh deploy chồng nhau.
- Có Health Check sau Deploy.
- Có Rollback cho release quan trọng.
- Không dùng production runner cho code không tin cậy.

---

## 26. Kiến trúc mở rộng cho hệ thống AI/Media API

```text
                         GitHub
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     PRODUCTION          STAGING             AI
          │                 │                 │
      prod-01          staging-01          gpu-01
          │                                     │
          ▼                                     ▼
    VPS / Docker                         GPU / CUDA / AI
          │                                     │
          └───────────────┬─────────────────────┘
                          ▼
                 Shared CI/CD Policy
```

Workflow định tuyến theo từng group/label.

---

## 27. Quy trình triển khai chuẩn

### Bước 1 – Xác định workload

```text
Production / Staging / Build / AI
```

### Bước 2 – Xác định trust boundary

```text
Ai được phép chạy code ở đâu?
```

### Bước 3 – Thiết kế Runner Group

```text
PRODUCTION
STAGING
BUILD
AI
```

### Bước 4 – Thiết kế labels

```text
production
ubuntu24
x64
gpu
cuda
docker
```

### Bước 5 – Cài Runner

```text
user riêng
→ /opt/actions-runner
→ register
→ service
```

### Bước 6 – Test Runner

```text
workflow test
```

### Bước 7 – CI

```text
lint → test → build
```

### Bước 8 – CD

```text
backup → deploy → verify
```

### Bước 9 – Reliability

```text
health check
→ rollback
→ concurrency
```

### Bước 10 – HA khi cần

```text
multiple runners
+
multiple application servers
```

---

## 28. Checklist vận hành

### GitHub

- [ ] Repository là private nếu Runner chứa production quyền truy cập.
- [ ] Runner Group có policy repository phù hợp.
- [ ] Labels được đặt đúng.
- [ ] Workflow dùng group/labels chính xác.
- [ ] Secrets không hard-code.
- [ ] Production deployment có approval/branch policy phù hợp.

### Server

- [ ] Ubuntu hỗ trợ.
- [ ] User `github-runner` riêng.
- [ ] Docker được cài nếu workflow cần Docker.
- [ ] `/opt/actions-runner` đúng owner.
- [ ] Runner chạy bằng systemd.
- [ ] Outbound HTTPS hoạt động.
- [ ] Không mở port không cần thiết.

### Application

- [ ] Port mapping rõ ràng.
- [ ] Public traffic đi qua reverse proxy.
- [ ] Health endpoint tồn tại.
- [ ] Deployment có concurrency.
- [ ] Có backup nếu migration ảnh hưởng dữ liệu.
- [ ] Có rollback strategy.

### HA

- [ ] Xác định Runner HA có thực sự cần không.
- [ ] Application HA được thiết kế riêng.
- [ ] Database HA được thiết kế riêng.
- [ ] Storage/backup có kế hoạch riêng.

---

## 29. Kết luận

Self-hosted Runner là một **worker CI/CD nằm trong hạ tầng của mình**. Nó giúp GitHub Actions thực thi trực tiếp các công việc Build/Test/Deploy/Operate trên VPS hoặc server nội bộ.

Runner Group là lớp **tổ chức và kiểm soát truy cập**, còn Label là lớp **định tuyến theo khả năng**.

Ba nguyên tắc cần nhớ:

```text
1. Runner = thực thi job
2. Runner Group = security/access boundary
3. Application architecture = quyết định port và HA
```

Không nên dùng Runner Group như một cơ chế chống port conflict. Không nên coi nhiều Runner là đồng nghĩa với application failover. Muốn HA đầy đủ phải thiết kế riêng cho CI/CD, application, database và storage.

### Kiến trúc khuyến nghị cho giai đoạn hiện tại

```text
GitHub
  │
  ▼
Runner Group: PRODUCTION
  │
  ▼
thientam-prod
  │
  ├── CI
  ├── CD
  ├── Backup
  ├── Health Check
  └── Rollback
  │
  ▼
Ubuntu 24.04 + aaPanel
  │
  ├── Nginx/Caddy :80/:443
  ├── Docker
  ├── Node/PHP/Python
  └── Application ports
```

Khi hệ thống tăng tải hoặc tăng mức độ quan trọng, mở rộng theo thứ tự:

```text
1 Runner
→ Runner Group + Staging
→ Multiple Production Runners
→ Blue-Green/Canary
→ Multiple Application Servers
→ Load Balancer
→ Database HA
→ Full HA architecture
```

---

## 30. Tài liệu tham khảo chính thức

- GitHub Docs – Self-hosted runners: https://docs.github.com/en/actions/reference/runners/self-hosted-runners
- GitHub Docs – Adding self-hosted runners: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners
- GitHub Docs – Runner groups: https://docs.github.com/en/actions/concepts/runners/runner-groups
- GitHub Docs – Managing access with runner groups: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/manage-access
- GitHub Docs – Using labels with self-hosted runners: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/apply-labels
- GitHub Docs – Using self-hosted runners in a workflow: https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/use-in-a-workflow
- GitHub Docs – Compromised runners: https://docs.github.com/en/actions/concepts/security/compromised-runners
