# Tài liệu Phát triển Dự án Video Dubbing AI - Back End

> **Tài liệu tổng hợp phát triển Backend VideoDubbing-AI**
>
> Repository nguồn: `daoaigiangnam/VideoDubing-AI`
>
> Baseline kỹ thuật đã sử dụng để kiểm tra: `fix-dubbing-audio-restore-8c94729`
>
> Repository tài liệu: `daoaigiangnam/System_Operation_and_Deployment_Documentation`
>
> Ngày tổng hợp: **16/08/2026**

---

## 1. Mục đích tài liệu

Tài liệu này tổng hợp quá trình xây dựng, sửa lỗi, kiểm thử và chuẩn hóa Backend của dự án **VideoDubbing-AI** từ các phiên trao đổi phát triển, log thực tế trên VPS/Docker và source code trên GitHub.

Tài liệu tập trung vào Backend, bao gồm:

- Kiến trúc API + Worker + dịch máy + TTS + FFmpeg.
- Luồng Download / Dubbing / Subtitle.
- Cơ chế Job chạy bất đồng bộ và monitor.
- Auto Detect ngôn ngữ nguồn bằng Whisper/faster-whisper.
- Translation qua LibreTranslate.
- TTS qua Edge TTS.
- Timing và chống tràn thời lượng khi dubbing.
- Audio separation, audio restore và audio mixing.
- Subtitle rendering và Unicode fonts.
- Các API public, API stage và API nội bộ.
- Database Job.
- Docker Compose và các service.
- Các lỗi thực tế đã gặp và cách xử lý.
- Các kết quả kiểm thử đã xác nhận.
- Các điểm cần đối chiếu trước khi triển khai lại từ Git.

> **Nguyên tắc:** những module đã kiểm thử thành công không tự ý thay đổi. Đặc biệt không thay đổi `dubbing.py`, `audio_fit.py`, `audio_mixer.py`, `timing.py`, font rendering hoặc final rendering nếu không có yêu cầu sửa lỗi cụ thể.

---

# 2. Tổng quan hệ thống

## 2.1. Kiến trúc logic

```text
Client / PHP / Shell monitor
          |
          | HTTP JSON
          v
+-------------------------+
| Node.js API :3000       |
| Express                 |
| Job / Download / Info   |
| Pipeline / Output       |
+------------+------------+
             |
             | HTTP internal
             v
+-------------------------+
| Python Worker :8000     |
| FastAPI                  |
| Download / Whisper       |
| Translation / TTS        |
| Timing / FFmpeg          |
+------+------------+------+
       |            |
       |            +------------------+
       |                               |
       v                               v
LibreTranslate :5000              Edge TTS
       |
       v
Translation                        Speech

             +-----------------------+
             |
             v
          FFmpeg
             |
             v
        Final MP4 / SRT
```

## 2.2. Docker services

Theo `docker-compose.yml` của baseline Git:

| Service | Vai trò | Port/container |
|---|---|---|
| `api` | Node.js REST API, Job API, Output API | `3000:3000` |
| `worker` | Python/FastAPI AI media worker | `8000` nội bộ |
| `libretranslate` | Translation provider | `5000` nội bộ |
| `cleanup` | Dọn storage/job cũ | không public |

API phụ thuộc Worker healthy; Worker phụ thuộc LibreTranslate healthy. citeturn1013file0

---

# 3. Các chế độ xử lý chính

## 3.1. FLAG = 0 — Download

```text
URL
 ↓
Download
 ↓
File gốc
```

Không Whisper, không Translate, không TTS.

## 3.2. FLAG = 1 — Dubbing

Quy tắc nghiệp vụ đã chốt trong quá trình phát triển:

```text
flag=1 + không target_language
    → giữ nguyên video / download-only

flag=1 + có target_language
    → Dubbing

source == target
    → giữ nguyên video, không Translate/TTS
```

Khi dubbing:

```text
Download
 ↓
Extract media
 ↓
Separate speech/background
 ↓
Whisper transcription
 ↓
Auto Detect source language nếu không chỉ định
 ↓
Translate source → target
 ↓
TTS target language
 ↓
Fit timing
 ↓
Mix dubbed speech
 ↓
Restore/background audio theo pipeline
 ↓
Render final MP4
```

## 3.3. FLAG = 2 — Subtitle

```text
Download
 ↓
Extract audio
 ↓
Separate speech
 ↓
Whisper
 ↓
Detect source
 ↓
Không target → subtitle source
Có target → translate → subtitle target
 ↓
Burn subtitle
 ↓
Giữ audio gốc
```

Subtitle **không TTS và không dubbing**.

---

# 4. Quy tắc ngôn ngữ đã chốt

## 4.1. Source language

Người dùng không bắt buộc nhập source language.

Nếu không truyền source hoặc truyền các giá trị:

```text
auto
automatic
detect
detected
"
null
```

thì Backend chuẩn hóa thành `None/null`, để Whisper tự phát hiện ngôn ngữ.

Không truyền literal `"auto"` vào Whisper vì Whisper/faster-whisper yêu cầu mã hợp lệ hoặc `None` cho auto-detect.

## 4.2. Target language

Target language là ngôn ngữ mà người dùng muốn nhận được:

```text
source: ko
 target: vi
```

→ Korean → Vietnamese dubbing/subtitle.

## 4.3. Same-language passthrough

Nếu:

```text
source_language == target_language
```

thì không cần dịch và không cần TTS.

Video được giữ nguyên theo chính sách passthrough.

---

# 5. Translation và mã ngôn ngữ

LibreTranslate trong Docker sử dụng image:

```text
libretranslate/libretranslate:v1.9.6
```

và được cấu hình tải các model:

```text
en, vi, zh
```

citeturn1013file0

## 5.1. Lỗi tiếng Trung đã gặp

LibreTranslate trả lỗi:

```json
{"error":"zh is not supported"}
```

trong khi `/languages` trả:

```json
[
  {"code":"en","targets":["en","vi","zh-Hans"]},
  {"code":"zh-Hans","targets":["en","vi","zh-Hans"]},
  {"code":"vi","targets":["en","vi","zh-Hans"]}
]
```

Vì vậy `zh`, `zh-CN`, `zh-Hans` phải được chuẩn hóa thành `zh-Hans` khi gọi LibreTranslate. Source code `translate_service.py` đã có alias mapping này. citeturn1014file0

## 5.2. Translation HTTP API nội bộ

Worker gọi:

```http
POST http://libretranslate:5000/translate
Content-Type: application/json
```

Payload:

```json
{
  "q": "text cần dịch",
  "source": "zh-Hans",
  "target": "vi",
  "format": "text"
}
```

Response kỳ vọng:

```json
{
  "translatedText": "..."
}
```

Worker có kiểm tra JSON lỗi, HTTP error, connection error và empty translation. citeturn1015file0

---

# 6. API Contract — Node.js Backend

Base URL mặc định:

```text
http://127.0.0.1:3000
```

Khi public qua reverse proxy/domain, dùng domain thực tế.

---

## 6.1. GET `/health`

Health check của API.

### Response

```json
{
  "success": true,
  "service": "videodubing-api",
  "status": "ok"
}
```

Đây là endpoint dùng để monitor API. citeturn1001file0

---

## 6.2. GET `/api/v1`

API root/version/feature discovery.

Response hiện công bố:

```json
{
  "success": true,
  "name": "VideoDubing-AI API",
  "version": "v1",
  "features": {
    "multilingual_dubbing": true,
    "same_language_passthrough": true,
    "stage_apis": [
      "extract",
      "separate",
      "transcribe",
      "translate",
      "tts",
      "render"
    ],
    "language_api": true,
    "voice_api": true
  }
}
```

citeturn1001file0

---

## 6.3. GET `/api/v1/worker/health`

API proxy tới Worker health.

Response dạng:

```json
{
  "success": true,
  "data": {
    "success": true,
    "service": "videodubing-worker",
    "status": "ok",
    "version": "0.2.0"
  }
}
```

---

# 7. API quan trọng nhất — `/api/v1/download`

```http
POST /api/v1/download
Content-Type: application/json
```

Đây là API chính dành cho UI/PHP.

Endpoint nhận cả:

```json
{"url":"..."}
```

hoặc:

```json
{"urls":["...","..."]}
```

Route chuẩn hóa thành danh sách URL và tạo một Job cho mỗi URL. citeturn1000file0

## 7.1. FLAG 0 — Download only

```json
{
  "flag": 0,
  "urls": [
    "https://www.youtube.com/watch?v=VIDEO_ID"
  ]
}
```

Nhiều URL:

```json
{
  "flag": 0,
  "urls": [
    "https://www.youtube.com/watch?v=VIDEO_1",
    "https://www.youtube.com/watch?v=VIDEO_2"
  ]
}
```

## 7.2. FLAG 1 — Dubbing

Dubbing có target:

```json
{
  "flag": 1,
  "urls": [
    "https://www.youtube.com/watch?v=VIDEO_ID"
  ],
  "target_language": "vi"
}
```

Source có thể bỏ trống để auto-detect.

Chỉ định source:

```json
{
  "flag": 1,
  "urls": ["https://www.youtube.com/watch?v=VIDEO_ID"],
  "source_language": "ko",
  "target_language": "vi"
}
```

Target không nhập theo yêu cầu UI mới:

```json
{
  "flag": 1,
  "urls": ["https://www.youtube.com/watch?v=VIDEO_ID"]
}
```

→ download/giữ nguyên, không TTS, không translation.

## 7.3. FLAG 2 — Subtitle

Subtitle source:

```json
{
  "flag": 2,
  "urls": ["https://www.youtube.com/watch?v=VIDEO_ID"]
}
```

Subtitle translated:

```json
{
  "flag": 2,
  "urls": ["https://www.youtube.com/watch?v=VIDEO_ID"],
  "target_language": "vi"
}
```

Audio gốc được giữ lại.

## 7.4. Các trường tùy chọn

```json
{
  "flag": 1,
  "urls": ["..."],
  "extractor": null,
  "type": "video",
  "format": "bv*+ba/b",
  "quality": null,
  "source_language": null,
  "target_language": "vi",
  "tts_voice": null,
  "translation_endpoint": null,
  "whisper_model": "small",
  "keep_background": true,
  "sync": true
}
```

`tts_voice` bỏ trống thì Worker chọn voice mặc định theo target language.

---

# 8. API `/api/v1/jobs`

## 8.1. POST `/api/v1/jobs`

Tạo một Job trực tiếp.

Request tối thiểu:

```json
{
  "url": "https://www.youtube.com/watch?v=VIDEO_ID"
}
```

Có thể truyền:

```text
extractor
type
format
quality
source_language
target_language
tts_voice
translation_endpoint
whisper_model
keep_background
sync
```

Job được dispatch xuống Worker ngay. Route trả HTTP `202`. citeturn1002file0

## 8.2. GET `/api/v1/jobs/:id`

Theo dõi Job.

Ví dụ:

```bash
curl -s http://127.0.0.1:3000/api/v1/jobs/JOB_ID
```

Response chứa Job và `output_url` khi status `completed`.

Các field chính:

```text
id
url
extractor
type
title
thumbnail
duration
format
quality
filename
filepath
filesize
progress
speed
eta
status
retry_count
worker
metadata
error
created_at
updated_at
finished_at
output_url
```

---

# 9. API `/api/v1/output/:id`

```http
GET /api/v1/output/:id
```

Chỉ trả file khi Job đã `completed`.

Nếu chưa hoàn tất:

```text
409 Job is <status>
```

Nếu không tìm thấy Job:

```text
404 Job not found
```

Output của dubbing/subtitle ưu tiên:

```text
storage/downloads/<job_id>/output/final.mp4
```

Nếu là download-only, endpoint có thể trả `job.filepath`. Path được kiểm tra để không thoát khỏi storage root. citeturn1006file0

---

# 10. API `/api/v1/info`

**Theo source code hiện tại:**

```http
POST /api/v1/info
Content-Type: application/json
```

Request:

```json
{
  "url": "https://www.youtube.com/watch?v=VIDEO_ID"
}
```

Response gồm:

```text
id
extractor
title
description
thumbnail
duration
uploader
channel
webpage_url
formats[]
```

API sử dụng `yt-dlp` để lấy metadata. citeturn1003file0

> Lưu ý: tài liệu API cũ trong `API_GUIDE.md` có ghi `GET /api/v1/info?url=...`; source route hiện tại là `POST` + JSON body. Khi tích hợp mới phải ưu tiên source code hiện tại.

---

# 11. API `/api/v1/languages`

```http
GET /api/v1/languages
```

API proxy tới Worker `/languages` và lấy catalog ngôn ngữ runtime từ Edge TTS. citeturn1004file0

Không nên hard-code danh sách ngôn ngữ trong frontend/PHP.

---

# 12. API `/api/v1/voices`

```http
GET /api/v1/voices
```

Hoặc:

```http
GET /api/v1/voices?language=vi
```

API proxy tới Worker `/voices`.

Response mỗi voice có:

```text
id
name
locale
language
gender
styles
```

citeturn1005file0

---

# 13. API `/api/v1/monitoring`

```http
GET /api/v1/monitoring
```

Endpoint yêu cầu Internal Token.

Dùng cho monitoring nội bộ, không phải API frontend public. citeturn1007file0

---

# 14. Stage API `/api/v1/pipeline/:stage`

```http
POST /api/v1/pipeline/extract
POST /api/v1/pipeline/separate
POST /api/v1/pipeline/transcribe
POST /api/v1/pipeline/translate
POST /api/v1/pipeline/tts
POST /api/v1/pipeline/render
```

Danh sách stage được source code giới hạn tại:

```text
extract
separate
transcribe
translate
tts
render
```

Không có stage public tên `dubbing`; full dubbing là pipeline nghiệp vụ được thực hiện bởi task download/full processor, không nên tự tạo task `dubbing` mới nếu chưa thiết kế lại worker contract. citeturn1010file0

Request có thể dùng:

```json
{
  "payload": {
    "...": "..."
  },
  "job_id": "optional"
}
```

Route loại bỏ các field điều khiển Job khỏi payload trước khi dispatch. Response HTTP `202` trả Job + `stage` + `status_url`. citeturn1008file0

---

# 15. API Internal — Worker callback

Các endpoint dưới `/internal` yêu cầu Internal Token.

## POST `/internal/jobs/:id/progress`

Worker gửi:

```json
{
  "progress": 70,
  "speed": null,
  "eta": null,
  "status": "translating"
}
```

`progress` phải là number từ 0 đến 100.

## POST `/internal/jobs/:id/complete`

Worker gửi kết quả hoàn tất. API lưu:

```text
title
duration
filename
filepath
filesize
metadata.media
metadata.final_path
metadata.subtitle_path
metadata.dubbed_speech_path
```

Sau đó progress = 100 và status = `completed`.

## POST `/internal/jobs/:id/fail`

Worker gửi:

```json
{
  "error": "chi tiết lỗi"
}
```

API chuyển Job sang `failed`. citeturn1009file0

---

# 16. Worker API — FastAPI :8000

Worker không public ra Internet; Docker Compose chỉ `expose: 8000`.

## GET `/health`

```json
{
  "success": true,
  "service": "videodubing-worker",
  "status": "ok",
  "version": "0.2.0"
}
```

## GET `/languages`

Lấy toàn bộ language code từ Edge TTS voice catalog.

## GET `/voices`

Có thể lọc:

```http
GET /voices?language=ko
```

## POST `/jobs`

Worker nhận:

```json
{
  "job_id": "UUID",
  "task": "download",
  "payload": {
    "url": "..."
  }
}
```

Worker kiểm tra task trong `SUPPORTED_TASKS`, sau đó chạy bất đồng bộ bằng `ThreadPoolExecutor(max_workers=2)`. citeturn1011file0

### Task hiện có

```text
download
download_only
extract
separate
transcribe
translate
tts
render
subtitle
```

> **Điểm kỹ thuật cần chú ý:** source branch `fix-dubbing-audio-restore-8c94729` hiện có `generate_dubbed_speech()` trong `dubbing.py`, nhưng snapshot Git của `processor.py` được kiểm tra ngày 16/08/2026 không chứa đầy đủ nhánh full dubbing trong `process_download()`. Trong khi VPS thực tế đã chạy thành công Dubbing. Vì vậy source Git và runtime VPS cần được đối chiếu trước lần build/deploy kế tiếp; không được ghi đè runtime đang chạy tốt bằng snapshot chưa đồng bộ.

---

# 17. Translation service

`translate_service.py` là lớp chuẩn hóa ngôn ngữ + tạo translated transcript.

Các alias quan trọng:

```text
zh       → zh-Hans
zh-cn    → zh-Hans
zh-hans  → zh-Hans
zh-simplified → zh-Hans
```

Các code regional khác được rút về base language, ví dụ:

```text
en-US → en
vi-VN → vi
```

Sau translation, mỗi segment được thêm:

```json
{
  "target_text": "..."
}
```

citeturn1014file0

---

# 18. Dubbing engine và Timing

Dubbing engine hiện có chính sách:

1. Không cắt mất câu nói.
2. Ưu tiên giữ timing nguồn khi có thể.
3. Khi video ngắn và bản dịch dài, sử dụng global `atempo` có kiểm soát.
4. Có giới hạn an toàn tốc độ.

Các hằng số của `dubbing.py`:

```text
DUB_NORMAL_MAX_SPEED = 1.15
DUB_GLOBAL_MAX_SPEED = 1.60
DUB_MIN_SPEED = 1.00
DUB_GAP = 0.03
DUB_EPSILON = 0.05
```

citeturn1016file0

## Hai pass quan trọng

### Pass 1

TTS toàn bộ câu trước để lấy **duration thực tế**.

### Pass 2

Scheduler xây timeline từ duration thật.

Nếu tổng speech vượt video nhưng vẫn nằm trong giới hạn 1.60x, hệ thống dùng global compression.

Nếu cần nhanh hơn 1.60x thì Job fail với thông báo an toàn thay vì tạo speech quá méo.

Sau khi mix, hệ thống kiểm tra lại duration cuối cùng; đây là kiểm tra authoritative trước khi render. citeturn1016file0

---

# 19. Các lỗi timing đã gặp trong quá trình phát triển

Đã từng gặp:

```text
Dubbed speech exceeds video duration (16.17s > 10.08s)
```

sau đó:

```text
Dubbed speech exceeds video duration (14.24s > 10.08s)
```

và một phiên bản policy khác báo:

```text
Translated speech is too long to fit the source video naturally even at 1.4...
```

Sau khi hoàn thiện cơ chế scheduler + global compression, test thực tế đã đạt:

```text
TOTAL     : 1
COMPLETED : 1
FAILED    : 0
```

Dubbing tiếng Trung → tiếng Việt đã chạy thành công và có audio.

---

# 20. Audio architecture

## 20.1. Media extraction

Video được tách thành:

```text
video_only
original audio
speech track
```

Speech được chuẩn hóa để Whisper xử lý.

## 20.2. Audio separation

Pipeline sử dụng `audio_separation.py`/`audio_service.py` để chuẩn bị:

```text
speech/vocals
background/no-vocals
```

## 20.3. TTS

TTS được sinh theo từng segment.

Mỗi segment có:

```text
raw TTS
fitted WAV
start
end
duration
fit information
tts attempt
```

## 20.4. Audio mixer

`audio_mixer.py` dùng FFmpeg `adelay` để đưa từng TTS segment về timeline và `amix` để tạo speech track tổng.

## 20.5. Audio restore

Một lỗi quan trọng từng xảy ra ở Subtitle: video có subtitle nhưng không có tiếng.

Nguyên nhân là subtitle renderer chỉ nhận video + subtitle, không nhận lại audio.

Sau khi sửa, subtitle renderer nhận:

```text
video_path
subtitle_path
output_path
audio_path
```

và giữ audio gốc.

---

# 21. Subtitle rendering và Unicode fonts

Subtitle renderer dùng FFmpeg subtitle filter và encode video bằng H.264.

Đã kiểm thử thực tế:

- 🇨🇳 Chinese: hiển thị đúng sau khi cài font Unicode/CJK.
- 🇰🇷 Korean: hiển thị đúng.
- 🇬🇧 English: hiển thị đúng.
- 🇻🇳 Vietnamese: hiển thị đúng.

Lỗi trước đó là tiếng Trung xuất hiện ô vuông `□□□`.

Đã đóng gói cài font/fallback trong Docker để xử lý Unicode đa ngôn ngữ, đặc biệt CJK.

Không nên sửa subtitle renderer khi font pipeline đang hoạt động tốt.

---

# 22. Các test thực tế đã thực hiện

## 22.1. Chinese → Vietnamese dubbing

Video test XHS:

```text
http://xhslink.com/o/60UT7QLpsPd
```

Kết quả cuối:

```text
COMPLETED: 1
FAILED: 0
```

Đã xác nhận dubbing có audio.

## 22.2. Chinese subtitle

Video:

```text
http://xhslink.com/o/7ILytKPJc8x
```

Subtitle Chinese hiển thị sau khi xử lý font.

## 22.3. Korean subtitle

Video YouTube:

```text
https://www.youtube.com/watch?v=OZfAam19aOU
```

Korean subtitle đã test thành công, không còn ô vuông.

## 22.4. English → Vietnamese dubbing

Video test:

```text
https://youtu.be/ntlFB_blHBo?si=vzY3t2wnOf7ma0zc
```

Job đã đạt:

```text
COMPLETED: 1
FAILED: 0
```

## 22.5. Same-language test

English source = English target đã được test và xác nhận OK theo nguyên tắc passthrough.

---

# 23. Monitor script

Script sử dụng:

```bash
./monitor-job.sh
```

Màn hình monitor hiển thị:

```text
JOBS
RUNNING
COMPLETED
FAILED
POLL
JOB ID
STATUS
PROGRESS
RESULT
ERROR
OUTPUT
```

Ví dụ:

```text
./monitor-job.sh "URL" vi 1
```

Trong quá trình debug đã bỏ `clear`/sửa `clear_screen()` để giữ lại:

```text
REQUEST PAYLOAD
API RESPONSE
```

Điều này giúp xác nhận API thực sự nhận:

```json
{
  "flag": 1,
  "urls": ["..."],
  "target_language": "vi"
}
```

---

# 24. Job model và SQLite

Database sử dụng SQLite.

Bảng `jobs` có các nhóm field:

### Identity

```text
id
url
extractor
type
```

### Media metadata

```text
title
thumbnail
duration
format
quality
filename
filepath
filesize
```

### Progress

```text
progress
speed
eta
status
```

### Execution

```text
retry_count
worker
metadata
error
```

### Timestamp

```text
created_at
updated_at
finished_at
```

Index:

```text
idx_jobs_status
idx_jobs_created
idx_jobs_extractor
```

citeturn1018file0

---

# 25. Job lifecycle

```text
waiting
  ↓
accepted
  ↓
queued
  ↓
processing
  ↓
downloading
  ↓
extracting
  ↓
separating
  ↓
transcribing
  ↓
translating
  ↓
tts
  ↓
rendering
  ↓
completed
```

Có thể chuyển sang:

```text
failed
cancelled
```

Progress được Worker callback về API qua `/internal/jobs/:id/progress`.

---

# 26. Docker Compose và deployment

Compose baseline:

```text
api
worker
cleanup
libretranslate
```

API:

```text
3000:3000
```

Worker:

```text
8000 nội bộ
```

LibreTranslate:

```text
5000 nội bộ
```

Shared storage:

```text
./storage:/app/storage
```

API database:

```text
./api/data:/app/data
```

Worker dùng:

```text
API_URL=http://api:3000
TRANSLATION_ENDPOINT=http://libretranslate:5000
```

citeturn1013file0

---

# 27. Cleanup service

Cleanup dùng các biến:

```text
CLEANUP_ENABLED
CLEANUP_TTL_HOURS
CLEANUP_INTERVAL_MINUTES
JOBS_DIR
DATABASE_PATH
```

Mục tiêu là dọn storage/job cũ theo TTL để không làm đầy disk.

Baseline Compose mặc định:

```text
CLEANUP_ENABLED=false
CLEANUP_TTL_HOURS=2
CLEANUP_INTERVAL_MINUTES=5
```

---

# 28. Security / Internal Token

API ↔ Worker callback sử dụng `INTERNAL_TOKEN`.

Các endpoint nội bộ:

```text
/api/v1/monitoring
/internal/jobs/:id/progress
/internal/jobs/:id/complete
/internal/jobs/:id/fail
```

không nên expose public trực tiếp.

Docker Compose yêu cầu `INTERNAL_TOKEN` trong `.env`.

---

# 29. PHP Integration

PHP không cần chạy AI trực tiếp.

PHP gọi Node API.

Ví dụ Dubbing:

```php
$payload = [
    'flag' => 1,
    'urls' => ['https://www.youtube.com/watch?v=VIDEO_ID'],
    'target_language' => 'vi',
];

$ch = curl_init('http://127.0.0.1:3000/api/v1/download');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => ['Content-Type: application/json'],
    CURLOPT_POSTFIELDS => json_encode($payload),
    CURLOPT_RETURNTRANSFER => true,
]);
$response = curl_exec($ch);
curl_close($ch);
```

Sau khi nhận Job ID:

```text
GET /api/v1/jobs/:id
```

Khi completed:

```text
GET /api/v1/output/:id
```

Đây là contract phù hợp để frontend/PHP chỉ đóng vai trò client/orchestrator.

---

# 30. Download engine và yt-dlp

Downloader sử dụng `yt-dlp` để:

- lấy metadata;
- download video/audio;
- hỗ trợ nhiều nền tảng;
- kiểm tra artifact trước khi đưa sang pipeline tiếp theo.

Trong quá trình phát triển đã thống nhất:

```text
yt-dlp bình thường
       ↓
   thành công
       ↓
 tiếp tục pipeline

Nếu fail/validation không đạt
       ↓
 fallback phù hợp
```

Không ép mọi YouTube request sử dụng Android client ngay từ đầu.

---

# 31. Các vấn đề kỹ thuật đã xử lý

## 31.1. Translation Chinese `zh` không được hỗ trợ

Lỗi:

```text
Translation HTTP 400
response={"error":"zh is not supported"}
```

Giải pháp:

```text
zh → zh-Hans
```

## 31.2. LibreTranslate connection refused

Đã gặp:

```text
ConnectionRefusedError: [Errno 111] Connection refused
```

Nguyên nhân thực tế là LibreTranslate đang boot/update language models.

Sau khi container healthy, `/languages` hoạt động bình thường.

## 31.3. Dubbing tràn thời lượng

Đã thay đổi từ fit từng câu đơn thuần sang scheduler + global compression có giới hạn.

## 31.4. Subtitle mất audio

Đã sửa renderer để truyền audio gốc vào subtitle render.

## 31.5. Chinese subtitle ô vuông

Đã cài bộ font Unicode/CJK và Fontconfig trong Docker.

## 31.6. Monitor che mất request/response

Đã chỉnh `clear_screen()` trong môi trường debug để giữ payload/response trên terminal.

---

# 32. Quy trình debug chuẩn

## Bước 1 — Health

```bash
curl -s http://127.0.0.1:3000/health
```

## Bước 2 — Worker

```bash
curl -s http://127.0.0.1:3000/api/v1/worker/health
```

## Bước 3 — Languages

```bash
curl -s http://127.0.0.1:3000/api/v1/languages
```

## Bước 4 — Test translation

Trong Worker:

```bash
docker compose exec -T worker python3 -c '...'
```

Đặc biệt kiểm tra `zh-Hans → vi` nếu gặp lỗi Chinese.

## Bước 5 — Tạo Job

Dùng `monitor-job.sh` hoặc POST `/api/v1/download`.

## Bước 6 — Theo dõi Job

```bash
curl -s http://127.0.0.1:3000/api/v1/jobs/JOB_ID | python3 -m json.tool
```

## Bước 7 — Worker log

```bash
docker compose logs --tail=300 worker
```

## Bước 8 — API log

```bash
docker compose logs --tail=300 api
```

## Bước 9 — Output

```bash
curl -L -o output.mp4 \
  http://127.0.0.1:3000/api/v1/output/JOB_ID
```

---

# 33. Kiểm thử API bằng curl

## Download

```bash
curl -X POST http://127.0.0.1:3000/api/v1/download \
  -H 'Content-Type: application/json' \
  -d '{
    "flag": 0,
    "urls": ["https://youtu.be/VIDEO_ID"]
  }'
```

## Dubbing Korean → Vietnamese

```bash
curl -X POST http://127.0.0.1:3000/api/v1/download \
  -H 'Content-Type: application/json' \
  -d '{
    "flag": 1,
    "urls": ["https://www.youtube.com/watch?v=OZfAam19aOU"],
    "target_language": "vi"
  }'
```

Source để auto-detect nên bỏ qua `source_language`.

## Subtitle Korean source

```bash
curl -X POST http://127.0.0.1:3000/api/v1/download \
  -H 'Content-Type: application/json' \
  -d '{
    "flag": 2,
    "urls": ["https://www.youtube.com/watch?v=OZfAam19aOU"]
  }'
```

## Subtitle Korean → Vietnamese

```bash
curl -X POST http://127.0.0.1:3000/api/v1/download \
  -H 'Content-Type: application/json' \
  -d '{
    "flag": 2,
    "urls": ["https://www.youtube.com/watch?v=OZfAam19aOU"],
    "target_language": "vi"
  }'
```

---

# 34. Test matrix chuẩn

| Test | Source | Target | Flag | Kết quả mong muốn |
|---|---|---|---:|---|
| Download | bất kỳ | — | 0 | Download nguyên bản |
| Dubbing | Auto detect | vi | 1 | Dubbing sang Việt |
| Dubbing | ko | vi | 1 | Korean → Vietnamese |
| Dubbing | zh-Hans | vi | 1 | Chinese → Vietnamese |
| Same language | en | en | 1 | Giữ nguyên |
| Same language | ko | ko | 1 | Giữ nguyên |
| Subtitle source | Auto detect | — | 2 | Subtitle source |
| Subtitle Chinese | zh-Hans | — | 2 | Chinese subtitle |
| Subtitle Korean | ko | — | 2 | Korean subtitle |
| Subtitle translated | en | vi | 2 | Vietnamese subtitle + audio English |

---

# 35. Nguyên tắc không phá vỡ baseline

Các module sau đã được kiểm thử và cần xem là **frozen baseline** nếu không có yêu cầu sửa lỗi:

```text
dubbing.py
audio_fit.py
audio_mixer.py
audio_duration.py
timing.py
subtitle_renderer.py
final_renderer.py
translation_edge.py
translate_service.py
font installation / Fontconfig
```

Không thay đổi chỉ vì muốn “refactor cho đẹp”. Mọi thay đổi phải có test hồi quy.

---

# 36. Git baseline

Branch kỹ thuật được sử dụng trong quá trình test:

```text
fix-dubbing-audio-restore-8c94729
```

Trong phiên làm việc, VPS đã push và xác nhận:

```text
c6ab66f fix: improve translation fallback and job monitor
```

và:

```text
HEAD -> fix-dubbing-audio-restore-8c94729
origin/fix-dubbing-audio-restore-8c94729
```

Đây là branch cần ưu tiên khi đối chiếu với runtime đang chạy, thay vì tự động lấy `main`.

---

# 37. Cảnh báo đồng bộ Git ↔ Runtime

Trong quá trình tổng hợp ngày 16/08/2026, source Git của branch `fix-dubbing-audio-restore-8c94729` có một điểm cần lưu ý:

- `api/src/routes/download.js` cho phép `target_language` optional.
- `download.service.js` phân loại flag và hiện tạo task `download` khi `flag=1` + có target.
- `worker/main.py` có các task `download`, `download_only`, `subtitle`, `extract`, `separate`, `transcribe`, `translate`, `tts`, `render` nhưng không có task tên `dubbing`.
- `worker/dubbing.py` tồn tại và đã chứa thuật toán dubbing/timing mới.
- `worker/processor.py` trên snapshot Git được đọc trong quá trình lập tài liệu chưa thể hiện đầy đủ nhánh full dubbing trong `process_download()` như phiên bản runtime VPS đã test thành công.

Điều này **không được tự động sửa trong tài liệu** bằng cách đoán. Đây là một điểm cần đồng bộ source trước deployment tiếp theo.

### Quy tắc xử lý

```text
Runtime đang chạy tốt
        ↓
Không overwrite bằng Git snapshot chưa đối chiếu
        ↓
Đối chiếu file + commit
        ↓
Test
        ↓
Push
        ↓
Deploy
```

---

# 38. Trạng thái kiểm thử tổng hợp

## Đã xác nhận

- API health: OK.
- Job creation: OK.
- Job monitor: OK.
- Chinese translation sau chuẩn hóa `zh-Hans`: OK.
- Chinese → Vietnamese dubbing: OK.
- Dubbing có audio: OK.
- Timing compression: OK sau khi hoàn thiện policy.
- English → Vietnamese dubbing: OK.
- Chinese subtitle: OK.
- Korean subtitle: OK.
- Unicode/CJK font rendering: OK.
- Same-language English passthrough: OK.
- Subtitle giữ audio gốc sau sửa renderer: OK.

## Không được coi là đã hoàn tất nếu chưa kiểm tra

- Tất cả ngôn ngữ TTS runtime.
- Tất cả cặp translation source/target.
- Deployment clean từ Git branch vào Docker mà không dựa vào container hiện tại.
- Full regression test sau mỗi thay đổi lớn.

---

# 39. Quy trình phát triển chuẩn từ đây

```text
1. Xác định yêu cầu mới
        ↓
2. Xác định module chịu trách nhiệm
        ↓
3. Đọc code hiện tại trên branch baseline
        ↓
4. Không sửa module frozen nếu không liên quan
        ↓
5. Sửa tối thiểu
        ↓
6. Unit/stage test
        ↓
7. Integration test
        ↓
8. Test bằng monitor-job.sh
        ↓
9. Kiểm tra API response
        ↓
10. Kiểm tra worker logs
        ↓
11. Kiểm tra output thực tế
        ↓
12. Commit
        ↓
13. Push branch
        ↓
14. Pull/rebuild Docker
        ↓
15. Regression test
        ↓
16. Đánh dấu module frozen nếu đạt
```

---

# 40. Contract dành cho Frontend/PHP

Frontend chỉ cần hiểu 3 thao tác:

```text
📥 Download  → flag=0
🎙 Dubbing   → flag=1
📝 Subtitle  → flag=2
```

## Dubbing

UI:

```text
URL
Target language
(optional voice)
```

Source language không cần bắt buộc nhập vì hệ thống có Auto Detect.

Nếu Target bỏ trống:

```text
flag=1
→ giữ nguyên video/download
```

Nếu Target có:

```text
flag=1
→ Dubbing
```

## Subtitle

UI:

```text
URL
Target language optional
```

Không target:

```text
source subtitle
```

Có target:

```text
translated subtitle
```

Audio gốc giữ nguyên.

---

# 41. Kết luận kiến trúc

Backend VideoDubbing-AI đã hình thành thành một hệ thống media pipeline bất đồng bộ với các lớp:

```text
REST API
   ↓
Job Service
   ↓
Worker dispatch
   ↓
Download / Media
   ↓
AI transcription
   ↓
Translation
   ↓
TTS
   ↓
Timing
   ↓
Audio mix
   ↓
Subtitle / Final render
   ↓
Output API
```

Các nguyên tắc quan trọng nhất:

1. **API không thực hiện AI nặng trực tiếp.** API tạo và theo dõi Job.
2. **Worker xử lý media/AI bất đồng bộ.**
3. **Whisper tự detect source khi source không được cung cấp.**
4. **LibreTranslate phải dùng canonical language code; Chinese cần `zh-Hans`.**
5. **Dubbing không được cắt mất lời nói.**
6. **Timing ưu tiên tự nhiên; global compression có giới hạn an toàn.**
7. **Subtitle giữ audio gốc.**
8. **Font Unicode/CJK phải được đóng gói cùng môi trường Docker.**
9. **Output chỉ được cung cấp khi Job completed và file tồn tại.**
10. **Không thay đổi baseline đã test thành công nếu không có lý do kỹ thuật rõ ràng.**
11. **Git branch baseline và runtime VPS phải được đồng bộ trước deployment.**

---

# 42. Tài liệu nguồn và repository

- Source code: `daoaigiangnam/VideoDubing-AI`
- Branch baseline: `fix-dubbing-audio-restore-8c94729`
- Tài liệu triển khai/vận hành: `daoaigiangnam/System_Operation_and_Deployment_Documentation`

Tài liệu này được xây dựng để trở thành **tài liệu Backend chuẩn của dự án VideoDubbing-AI**, dùng làm căn cứ khi phát triển tiếp, debug, test, triển khai Docker và tích hợp PHP/Frontend.
