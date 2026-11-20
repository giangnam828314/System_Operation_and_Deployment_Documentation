# Ý Tưởng về các hình thức Khách hàng phải trả khi sử dụng Dịch vụ

## 1. Mục đích tài liệu

Tài liệu này tổng hợp và chuẩn hóa ý tưởng về mô hình thương mại hóa cho **Video Dubbing AI**, tập trung vào cách khách hàng trả phí khi sử dụng các dịch vụ AI xử lý video.

Mục tiêu chính:

- Thiết kế mô hình thu phí dễ hiểu với khách hàng.
- Cân bằng doanh thu với chi phí server, API, GPU/CPU, storage và bandwidth.
- Tính mức sử dụng theo **thời lượng video**, không tính theo thời gian server xử lý.
- Cho phép nhiều Job chạy **song song**.
- Không để Job song song làm vượt quota của khách hàng.
- Chỉ ghi nhận usage chính thức khi Job xử lý thành công.
- Khi Job thất bại hoặc bị hủy theo chính sách, phần quota đã giữ phải được trả lại.
- Tạo nền tảng có thể mở rộng cho Dubbing, Subtitle, Text-to-Speech và các tính năng AI khác.

---

# 2. Định hướng mô hình thương mại

Video Dubbing AI nên được xây dựng theo mô hình **Subscription theo thời gian + hạn mức sử dụng** thay vì chỉ bán thời gian sử dụng không giới hạn.

Mô hình đề xuất:

> Khách hàng mua một gói có thời hạn, trong thời hạn đó được sử dụng dịch vụ trong phạm vi quota được cấp.

Ví dụ:

- 99.000đ / 30 ngày.
- 199.000đ / 30 ngày.
- 399.000đ / 30 ngày.
- Có thể có gói năm hoặc Business.

Điểm quan trọng: **không nên hiểu gói 30 ngày là xử lý không giới hạn**.

Gói phải đồng thời có:

1. Thời hạn sử dụng.
2. Quota xử lý.
3. Giới hạn đồng thời nếu cần.
4. Chính sách Fair Use.
5. Có thể có giới hạn storage hoặc thời gian lưu file.

---

# 3. Vì sao không nên chỉ bán "không giới hạn"

Nếu bán ví dụ:

> 99.000đ / 30 ngày, Dubbing không giới hạn.

thì một khách hàng có thể gửi lượng video rất lớn, khiến chi phí thực tế tăng theo:

- CPU/GPU.
- Whisper hoặc speech-to-text.
- TTS.
- API AI bên ngoài nếu sử dụng.
- FFmpeg.
- Storage.
- Bandwidth.
- Thời gian Worker.
- Queue và tài nguyên xử lý song song.

Trong khi doanh thu trên mỗi khách hàng gần như cố định.

Vì vậy mô hình an toàn hơn là:

**Gói thời gian + quota theo thời lượng video.**

---

# 4. Đơn vị tính phí: thời lượng Video

Đây là nguyên tắc cốt lõi của mô hình.

## 4.1. Không tính theo thời gian Server xử lý

Nếu video dài 10 phút và server xử lý mất:

- 2 phút;
- 10 phút;
- 30 phút;
- 1 giờ;

thì khách hàng vẫn chỉ sử dụng quota tương ứng với **10 phút video**.

Không tính thời gian CPU/GPU thực tế vào quota của khách hàng.

## 4.2. Tính theo duration của video

Ví dụ:

- Video 10:00 → 10 phút quota.
- Video 13:43 → 14 phút quota.
- Video 35:10 → 36 phút quota.
- Video 60:00 → 60 phút quota.

Có thể chuẩn hóa bằng:

```text
quota_minutes = ceil(duration_seconds / 60)
```

Ví dụ:

```text
823 giây
823 / 60 = 13,716...
ceil() = 14 phút
```

## 4.3. Lợi ích

Khách hàng dễ hiểu:

> Video dài bao nhiêu phút thì sử dụng bấy nhiêu phút quota.

Doanh nghiệp cũng dễ dự toán:

> Tổng quota bán ra = tổng số phút video mà khách hàng được phép xử lý.

---

# 5. Vì hệ thống chỉ nhận URL

Phiên bản hiện tại của Video Dubbing AI có thể thiết kế theo flow:

```text
User nhập URL
    ↓
Media API /api/info
    ↓
yt-dlp lấy metadata
    ↓
Lấy duration
    ↓
Hiển thị thời lượng + quota cần dùng
    ↓
User chọn thao tác
```

Không cần upload file ở bước đầu.

---

# 6. Lấy thời lượng Video bằng API

## 6.1. Video từ nền tảng hỗ trợ bởi yt-dlp

Media API hiện tại sử dụng `yt-dlp` có thể lấy metadata của URL.

Ví dụ dữ liệu chuẩn hóa:

```json
{
  "success": true,
  "data": {
    "title": "Video Test",
    "duration": 823,
    "duration_minutes": 14,
    "thumbnail": "...",
    "extractor": "youtube"
  }
}
```

Trong đó:

- `duration`: số giây thực tế.
- `duration_minutes`: số phút quota sau khi làm tròn lên.
- `extractor`: nền tảng/extractor.

## 6.2. FFprobe

Nếu sau này hệ thống cần xử lý file nội bộ hoặc file đã được tải xuống, có thể sử dụng FFprobe để lấy duration.

Tuy nhiên với flow chỉ nhận URL, `yt-dlp` là nguồn lấy metadata thuận tiện nhất.

---

# 7. Flow UX với khách hàng

Khi khách hàng nhập URL, hệ thống nên kiểm tra trước để hiển thị rõ chi phí dự kiến.

Ví dụ:

```text
Video Test
Thời lượng: 13:43
Quota cần sử dụng: 14 phút
Quota còn khả dụng: 286 phút

[ Bắt đầu Dubbing ]
```

Ở bước `/api/info` **chưa trừ quota**.

Lý do: khách hàng có thể chỉ kiểm tra Video mà chưa thực sự thực hiện thao tác.

---

# 8. Thời điểm bắt đầu tính/quản lý quota

Khi khách hàng nhấn:

> Bắt đầu Dubbing

hoặc:

> Tạo phụ đề

hoặc:

> Chuyển văn bản thành giọng nói

thì hệ thống mới bắt đầu quy trình quota.

Tuy nhiên, không nên hiểu là "trừ quota vĩnh viễn ngay lập tức".

Cách đúng là:

> **Reserve quota khi Job được chấp nhận.**

Sau đó:

- SUCCESS → chuyển Reserve thành Used.
- FAILED/CANCELLED theo chính sách → Release Reserve.

---

# 9. Vì sao cần Reserve quota

Hệ thống Video Dubbing AI cho phép **nhiều Job chạy song song**.

Ví dụ khách hàng có:

```text
Quota Total = 300 phút
Quota Used = 100 phút
Quota Reserved = 50 phút
```

Quota còn khả dụng:

```text
Available = Total - Used - Reserved
          = 300 - 100 - 50
          = 150 phút
```

Nếu khách hàng tạo một Job 40 phút:

```text
150 >= 40 → được phép
```

Sau khi reserve:

```text
Total = 300
Used = 100
Reserved = 90
Available = 110
```

Job được chạy ngay, kể cả song song với Job khác.

---

# 10. Tách 3 trạng thái quota

Mỗi Subscription nên theo dõi tối thiểu:

```text
quota_total
quota_used
quota_reserved
```

Công thức:

```text
quota_available = quota_total - quota_used - quota_reserved
```

## Ý nghĩa

### quota_total
Tổng quota được cấp trong gói.

### quota_used
Quota đã tiêu thụ chính thức bởi các Job SUCCESS.

### quota_reserved
Quota đang được giữ cho các Job đang chờ/xử lý.

### quota_available
Quota còn có thể cấp cho Job mới.

**Mọi lần kiểm tra quota để tạo Job mới phải sử dụng `quota_available`, không chỉ sử dụng `quota_total - quota_used`.**

---

# 11. Flow đầy đủ của một Job

```text
User nhập URL
      ↓
POST /api/info
      ↓
Media API / yt-dlp
      ↓
Lấy duration
      ↓
Quy đổi duration → quota_minutes
      ↓
Hiển thị cho User
      ↓
User bấm thao tác
      ↓
POST /api/dubbing/create
      ↓
Kiểm tra Subscription
      ↓
Kiểm tra quota_available
      ↓
Atomic Reserve quota
      ↓
Create Job
      ↓
Queue
      ↓
Worker xử lý song song
      ↓
      ┌───────────────────┐
      │                   │
   SUCCESS             FAILED
      │                   │
      ↓                   ↓
Reserve → Used       Release Reserve
      │                   │
      ↓                   ↓
Job Completed        Job Failed
```

---

# 12. Khi Job thành công

Ví dụ Video dài 13:43.

Quota cần dùng:

```text
14 phút
```

Trước khi chạy:

```text
quota_reserved += 14
```

Sau khi xử lý thành công:

```text
quota_reserved -= 14
quota_used += 14
```

Ví dụ:

```text
Before:
Used = 300
Reserved = 114

After SUCCESS:
Used = 314
Reserved = 100
```

Khách hàng chính thức mất 14 phút quota.

---

# 13. Khi Job thất bại

Các trường hợp có thể thất bại:

- URL không còn khả dụng.
- yt-dlp không tải được.
- Nguồn Video lỗi.
- FFmpeg lỗi.
- Whisper lỗi.
- TTS lỗi.
- Dubbing lỗi.
- Worker lỗi.
- Server lỗi khiến Job không hoàn thành.

Nếu Job thất bại theo chính sách hoàn quota:

```text
quota_reserved -= quota_minutes
quota_used không thay đổi
```

Ví dụ:

```text
Before:
Used = 300
Reserved = 114

Job 14 phút FAILED

After:
Used = 300
Reserved = 100
```

Khách hàng không mất quota cho Job thất bại.

---

# 14. Khi Job bị Cancel

Nếu hệ thống cho phép người dùng hủy Job:

```text
RESERVED → RELEASE
```

Không chuyển sang Used nếu Job chưa hoàn tất theo điều kiện tính phí của sản phẩm.

Chính sách cuối cùng phải ghi rõ trong Terms/Usage Policy.

---

# 15. Chạy nhiều Job song song

Ví dụ:

```text
Quota Total = 300
Used = 0
Reserved = 0
Available = 300
```

User gửi:

```text
Job A = 100 phút
Job B = 80 phút
Job C = 70 phút
```

Xử lý:

```text
Job A:
Reserve 100
Available = 200

Job B:
Reserve 80
Available = 120

Job C:
Reserve 70
Available = 50
```

Cả A, B và C có thể chạy song song.

Trạng thái:

```text
Total = 300
Used = 0
Reserved = 250
Available = 50
```

Nếu tiếp tục tạo Job D = 60 phút:

```text
Available = 50
60 > 50
```

→ Không cho tạo Job D.

Như vậy hệ thống không bao giờ cho các Job song song vượt quota.

---

# 16. Bắt buộc Atomic khi Reserve

Đây là yêu cầu kỹ thuật quan trọng.

Không được xử lý theo kiểu không đồng bộ:

```text
SELECT quota
↓
PHP tính toán
↓
UPDATE quota
```

vì nhiều request đồng thời có thể cùng đọc một giá trị quota và cùng cho phép Job.

Phải dùng transaction/row lock hoặc cơ chế atomic tương đương:

```text
BEGIN TRANSACTION

LOCK quota row

Tính:
quota_total - quota_used - quota_reserved

Nếu đủ:
    quota_reserved += duration
    tạo Job

COMMIT
```

Nếu không đủ:

```text
ROLLBACK
→ Reject Job
```

Điều này đặc biệt quan trọng với hệ thống chạy Worker song song.

---

# 17. Ví dụ kiểm tra quota hoàn chỉnh

Khách hàng có:

```text
Total = 1.000 phút
Used = 300 phút
Reserved = 100 phút
```

Do đó:

```text
Available = 1.000 - 300 - 100
          = 600 phút
```

Khách hàng gửi các Job:

```text
A = 100 phút
B = 200 phút
C = 150 phút
D = 200 phút
```

Sau A:

```text
Available = 500
```

Sau B:

```text
Available = 300
```

Sau C:

```text
Available = 150
```

D cần 200 phút nhưng chỉ còn 150 phút.

→ D không được Reserve.

A/B/C vẫn có thể tiếp tục chạy song song.

---

# 18. Khi Job A/B/C hoàn thành khác nhau

Không cần chờ các Job hoàn thành theo thứ tự tạo.

Ví dụ:

```text
A SUCCESS
B FAILED
C SUCCESS
```

Hệ thống xử lý độc lập:

A:

```text
Reserved A → Used A
```

B:

```text
Reserved B → Release
```

C:

```text
Reserved C → Used C
```

Điều này phù hợp với kiến trúc Worker chạy song song hiện tại.

---

# 19. Kiến trúc quota đề xuất

## Subscription

```text
subscriptions
-------------------------
id
user_id
plan_id
started_at
expired_at
quota_total
quota_used
quota_reserved
status
created_at
updated_at
```

## Jobs

```text
jobs
-------------------------
id
user_id
url
extractor
type
duration_seconds
quota_minutes
status
progress
filepath
error
worker
created_at
started_at
finished_at
heartbeat_at
updated_at
```

## Usage history

```text
usage_transactions
-------------------------
id
user_id
job_id
minutes
type
status
description
created_at
```

Bảng `usage_transactions` là lịch sử tiêu thụ quota, giúp:

- User xem lịch sử.
- Admin đối soát.
- Debug.
- Xử lý khiếu nại.
- Audit hệ thống.

---

# 20. Subscription và quota theo thời gian

Mô hình thương mại có thể bắt đầu từ các gói như:

| Gói | Giá tham khảo | Thời hạn | Quota |
|---|---:|---:|---:|
| Free | 0đ | 7 ngày | Hạn mức nhỏ |
| Basic | 99.000đ | 30 ngày | Ví dụ 300 phút |
| Pro | 199.000đ | 30 ngày | Ví dụ 1.000 phút |
| Premium | 399.000đ | 30 ngày | Ví dụ 3.000 phút |
| Business | 999.000đ | 30 ngày | Ví dụ 50 giờ |

Các con số trên là **mức đề xuất ban đầu**, cần kiểm chứng bằng cost model thực tế trước khi phát hành thương mại.

---

# 21. Một quota tổng hay nhiều quota

Có hai hướng.

## Hướng A - Một quota tổng

Ví dụ:

```text
Pro = 1.000 phút / tháng
```

Khách hàng tự chia cho:

- Dubbing.
- Subtitle.
- TTS.

Ưu điểm:

- Rất dễ hiểu.
- Dễ lập trình.
- Dễ quảng bá.
- Quota linh hoạt.

Đây là **hướng được ưu tiên cho phiên bản đầu tiên**.

## Hướng B - Quota riêng theo tính năng

Ví dụ:

```text
Dubbing = 500 phút
Subtitle = 1.000 phút
TTS = 500 phút
```

Ưu điểm:

- Dễ kiểm soát cost nếu từng dịch vụ có chi phí rất khác nhau.

Nhược điểm:

- Khó hiểu hơn.
- UX phức tạp hơn.

Có thể triển khai sau khi có dữ liệu sử dụng thực tế.

---

# 22. Cân bằng chi phí Server và số lượng User

Giá bán không nên xác định trước rồi mới nghĩ đến cost.

Trình tự đúng:

```text
Tính cost / phút
      ↓
Tính cost server
      ↓
Tính cost AI/API
      ↓
Tính storage + bandwidth
      ↓
Tính tỷ lệ user thực sự sử dụng
      ↓
Xây dựng quota mỗi gói
      ↓
Định giá
      ↓
Kiểm tra biên lợi nhuận
```

Ví dụ nội bộ có thể theo dõi:

```text
Server cost
Storage cost
Bandwidth cost
AI/API cost
Worker cost
Payment gateway fee
Support cost
```

Cần phân biệt:

**Customer Usage**

```text
Video duration = 14 phút
```

với:

**Internal Infrastructure Cost**

```text
CPU time = X phút
GPU time = Y phút
Storage = Z MB
Bandwidth = N MB
```

Hai loại số liệu này phục vụ hai mục đích khác nhau.

---

# 23. Fair Use Policy

Ngay cả khi có quota theo phút, vẫn nên có giới hạn kỹ thuật để bảo vệ hệ thống.

Ví dụ:

- Số Job chạy đồng thời.
- Thời lượng tối đa của một Video.
- Kích thước file kết quả.
- Số lần retry.
- Storage lưu trữ.
- Thời gian giữ file kết quả.
- Rate limit API.
- Các giới hạn chống abuse.

Fair Use không thay thế quota; Fair Use là lớp bảo vệ bổ sung.

---

# 24. Thanh toán Online

Sau khi mô hình Subscription + Quota ổn định, có thể tích hợp Payment Gateway.

Các gateway có thể cân nhắc:

- PayOS.
- VNPAY.
- MoMo.
- ZaloPay.
- Stripe nếu mở rộng quốc tế.
- PayPal nếu cần.

Phiên bản đầu tiên có thể ưu tiên một gateway dễ tích hợp, sau đó bổ sung các gateway khác.

---

# 25. Luồng thanh toán

```text
User
 ↓
Chọn gói
 ↓
Create Payment Order
 ↓
Payment Gateway
 ↓
User thanh toán
 ↓
Webhook / Callback
 ↓
Verify transaction
 ↓
Xác nhận tiền thành công
 ↓
Activate Subscription
 ↓
Cấp quota
```

Không được cấp gói chỉ dựa vào việc User quay lại `return_url`.

Phải xác thực giao dịch bằng webhook/callback và chữ ký giao dịch theo quy định của gateway.

---

# 26. Tách Payment khỏi Usage

Nên xem đây là hai module độc lập:

```text
PAYMENT
 ├── Order
 ├── Transaction
 ├── Gateway
 ├── Webhook
 └── Payment status

SUBSCRIPTION / QUOTA
 ├── Plan
 ├── Subscription
 ├── Total
 ├── Used
 ├── Reserved
 └── Usage history
```

Thanh toán thành công → cấp Subscription/Quota.

Job thành công → cập nhật Usage.

Không nên trộn Payment Transaction với Job Usage.

---

# 27. Flow kiến trúc tổng thể

```text
                     USER
                       │
                       ▼
                 Đăng nhập/Đăng ký
                       │
                       ▼
                  Chọn gói dịch vụ
                       │
                       ▼
                   PAYMENT
                       │
                Thanh toán Online
                       │
                       ▼
                 Subscription
                       │
                       ▼
                 QUOTA TOTAL
                       │
                       ▼
                 Nhập URL Video
                       │
                       ▼
                    /api/info
                       │
                       ▼
                     yt-dlp
                       │
                       ▼
                    Duration
                       │
                       ▼
               Hiển thị quota cần
                       │
                       ▼
                User chọn thao tác
                       │
                       ▼
              ATOMIC QUOTA CHECK
                       │
                       ▼
                RESERVE QUOTA
                       │
                       ▼
                   CREATE JOB
                       │
                       ▼
                     QUEUE
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
       Worker A     Worker B     Worker C
          │            │            │
          └────────────┼────────────┘
                       ▼
                  Process Video
                       │
                 ┌─────┴─────┐
                 ▼           ▼
              SUCCESS      FAILED
                 │           │
                 ▼           ▼
           Reserve → Used   Release
                 │           │
                 └─────┬─────┘
                       ▼
                 Usage History
```

---

# 28. Quy tắc nghiệp vụ cần chốt

## Rule 1
Quota của khách hàng được tính theo **thời lượng Video**, không theo thời gian Server xử lý.

## Rule 2
`duration_seconds` phải lấy từ backend đáng tin cậy, không tin giá trị do Frontend tự gửi.

## Rule 3
`quota_minutes = ceil(duration_seconds / 60)` là quy tắc cơ bản.

## Rule 4
Chỉ khi User thực sự bắt đầu Job mới Reserve quota.

## Rule 5
Mọi Job mới phải kiểm tra:

```text
quota_total - quota_used - quota_reserved
```

## Rule 6
Reserve phải atomic/transactional.

## Rule 7
Job SUCCESS → chuyển Reserved thành Used.

## Rule 8
Job FAILED/CANCELLED theo chính sách → Release Reserved.

## Rule 9
Các Job có thể chạy song song, miễn tổng Reserved không vượt Available quota.

## Rule 10
Usage history phải được lưu để audit và đối soát.

## Rule 11
Payment status và Job usage là hai hệ thống logic độc lập.

## Rule 12
Không bán Unlimited nếu chưa có cơ chế kiểm soát Fair Use và cost rõ ràng.

---

# 29. Những vấn đề cần xử lý ở giai đoạn triển khai

Trước khi code production, cần chốt thêm:

1. Gói tháng có được cộng dồn quota hay không.
2. Khi gia hạn sớm thì quota xử lý thế nào.
3. Khi Subscription hết hạn nhưng Job đang chạy thì xử lý thế nào.
4. Retry Job có reserve lại quota hay giữ reservation cũ.
5. Job bị crash rồi được retry có tránh double charge hay không.
6. Job SUCCESS nhưng file output bị mất sau đó có hoàn quota hay không.
7. Chính sách Cancel.
8. Chính sách Refund Payment.
9. Thời gian giữ file kết quả.
10. Tối đa bao nhiêu Job song song cho mỗi gói.
11. Tối đa bao nhiêu phút cho một Video.
12. Rate limit cho API.
13. Cơ chế chống duplicate webhook Payment.
14. Idempotency cho Usage transaction.

---

# 30. Kết luận và mô hình được đề xuất chốt

Mô hình phù hợp nhất cho Video Dubbing AI ở giai đoạn hiện tại là:

> **Subscription theo thời gian + Quota theo thời lượng Video + Reservation khi bắt đầu Job + Commit khi Job SUCCESS.**

Cụ thể:

```text
Khách hàng mua GÓI
        ↓
Có thời hạn sử dụng
        ↓
Được cấp QUOTA theo phút
        ↓
Nhập URL Video
        ↓
API lấy DURATION
        ↓
Biết chính xác số phút sẽ sử dụng
        ↓
User bấm thao tác
        ↓
CHECK AVAILABLE QUOTA
        ↓
RESERVE QUOTA
        ↓
Job chạy song song
        ↓
SUCCESS → Reserved chuyển Used
FAILED → Reserved được Release
```

Mô hình này giải quyết đồng thời ba bài toán:

### Với khách hàng

- Dễ hiểu giá trị được mua.
- Tính theo thời lượng Video trực quan.
- Không bị tính theo việc Server xử lý nhanh hay chậm.
- Job lỗi có thể được hoàn quota theo chính sách.

### Với hệ thống

- Cho phép chạy nhiều Job song song.
- Không vượt quota.
- Chống race condition bằng atomic reserve.
- Có lịch sử Usage rõ ràng.
- Dễ mở rộng thành nhiều tính năng AI.

### Với doanh nghiệp

- Có doanh thu định kỳ.
- Có giới hạn cost theo quota.
- Dễ dự toán hạ tầng.
- Dễ xây dựng nhiều gói giá.
- Có thể mở rộng sang Business/API.

---

# 31. Trạng thái tài liệu

**Định hướng nghiệp vụ được đề xuất:** Ready for implementation.

**Chưa chốt:** giá bán thực tế, quota thực tế từng gói, giới hạn Fair Use, Payment Gateway cụ thể, chính sách Refund/Cancel và các tham số vận hành production.

**Nguyên tắc kiến trúc đã định hướng:**

```text
PAYMENT
   ↓
SUBSCRIPTION
   ↓
QUOTA
   ↓
RESERVE
   ↓
PARALLEL JOB PROCESSING
   ↓
SUCCESS → USED
FAILED  → RELEASE
```

Tài liệu này là cơ sở nghiệp vụ để triển khai tiếp module **Subscription / Payment / Quota / Usage** cho dự án Video Dubbing AI.