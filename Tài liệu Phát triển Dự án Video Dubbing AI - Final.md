# Tài liệu Phát triển Dự án Video Dubbing AI - Final

> **Tài liệu chuẩn Final về kiến trúc, phát triển, kiểm thử, triển khai và nguyên tắc bảo trì của dự án Video Dubbing AI.**
>
> **Repository nguồn:** `daoaigiangnam/Long_Tieng_Phu_De_Video`
>
> **Branch nguồn được kiểm tra:** `main`
>
> **Repository tài liệu:** `daoaigiangnam/System_Operation_and_Deployment_Documentation`
>
> **Ngày tổng hợp:** 16/08/2026
>
> **Trạng thái:** FINAL / Baseline phát triển

---

## 1. Mục đích và phạm vi

Tài liệu này là tài liệu phát triển chuẩn của dự án **Video Dubbing AI**, được tổng hợp từ source code và tài liệu đang tồn tại trên branch `main` của repository nguồn.

Mục tiêu của tài liệu là tạo một **baseline duy nhất** để đội phát triển, vận hành và bảo trì có thể dựa vào đó khi:

- Phát triển tính năng mới.
- Sửa lỗi.
- Kiểm thử.
- Build Docker image.
- Deploy lên server.
- Theo dõi Job và media pipeline.
- Bảo trì database và storage.
- Thay thế hoặc nâng cấp một AI/media provider mà không phá vỡ kiến trúc tổng thể.
- Đối chiếu code với thiết kế đã được chốt.

Phạm vi bao gồm:

1. Kiến trúc tổng thể.
2. Frontend/PHP integration.
3. Node.js REST API.
4. Python AI/media Worker.
5. Download và media inspection.
6. Speech separation.
7. Speech-to-text và language detection.
8. Translation.
9. Target-language TTS.
10. Sentence timing và audio fitting.
11. Audio mixing và final rendering.
12. Subtitle pipeline.
13. Job lifecycle và persistence.
14. Storage/cleanup.
15. Docker Compose.
16. CI/CD và GitHub Actions.
17. Kiểm thử và smoke/integration test.
18. Security boundaries.
19. Quy tắc phát triển và bảo trì.
20. Roadmap mở rộng sau baseline.

---

# 2. Nguyên tắc Final cần giữ nguyên

## 2.1. Kiến trúc cố định

Kiến trúc chuẩn là:

```text
Frontend / PHP / Client
        |
        | HTTP JSON
        v
Node.js API
        |
        +-------------------------+
        |                         |
        v                         v
   Job / DB / API          Python Worker
                                  |
              +-------------------+-------------------+
              |                   |                   |
              v                   v                   v
           yt-dlp              FFmpeg         AI / Media modules
                                                      |
                             +------------------------+-----------------------+
                             |                        |                       |
                             v                        v                       v
                         Whisper                 Translation               Edge TTS
                       / faster-whisper        / LibreTranslate          target voice
                             |                        |                       |
                             +------------------------+-----------------------+
                                                      |
                                                      v
                                             Timing / Audio / Render
                                                      |
                                                      v
                                                Final MP4 / SRT
```

**Không tự ý chuyển sang kiến trúc khác** nếu chưa có một architecture revision được phê duyệt.

Các thư mục logic cốt lõi được giữ nguyên:

```text
api/       Node.js REST API và orchestration
worker/    Python AI/media worker
storage/   runtime media storage, không commit vào Git
FrontEnd/  PHP/web integration và UI hiện hữu
docs/      tài liệu kỹ thuật
cleanup/   cleanup service
.github/   CI/CD automation
```

---

# 3. Trạng thái baseline nguồn

Repository `Long_Tieng_Phu_De_Video` hiện sử dụng branch mặc định `main`.

Source tree thể hiện đầy đủ các lớp:

- `api/`
- `worker/`
- `FrontEnd/`
- `database/`
- `cleanup/`
- `docs/`
- `docker-compose.yml`
- `.env.example`
- `.github/workflows/`

Ngoài phần code, repository hiện có các tài liệu về API, pipeline, architecture, installation, deployment update, cleanup và roadmap. Đây là cơ sở để tài liệu Final được xây dựng theo code thực tế thay vì chỉ dựa trên thiết kế lý thuyết.

---

# 4. Mô hình trách nhiệm của từng lớp

## 4.1. Frontend / PHP

Frontend có trách nhiệm:

- Nhận URL hoặc danh sách URL từ người dùng.
- Cho người dùng chọn chế độ Download / Dubbing / Subtitle.
- Chọn source language hoặc `auto`.
- Chọn target language.
- Gọi `/languages` và `/voices` để lấy catalog runtime.
- Gửi Job tới Node API.
- Theo dõi Job.
- Hiển thị tiến trình và kết quả.
- Cho phép tải output.

Frontend **không tự thực hiện AI processing**.

## 4.2. Node.js API

Node.js là lớp control plane.

Trách nhiệm:

- HTTP/JSON API.
- Validation request.
- Authentication/authorization ở lớp public integration khi được cấu hình.
- Job lifecycle.
- Persistence.
- Dispatch Job tới Worker.
- Proxy/health check Worker.
- Output download.
- Language/voice catalog exposure.
- Orchestration của pipeline.
- Không trực tiếp thực hiện CPU-heavy AI/media processing.

## 4.3. Python Worker

Python là data plane / processing plane.

Trách nhiệm:

- Download media.
- Inspect media.
- Extract video/audio.
- Speech/music separation.
- Whisper/faster-whisper transcription.
- Language detection.
- Translation client.
- Edge TTS.
- Timing.
- Audio fitting.
- Audio mix.
- Subtitle generation/rendering.
- FFmpeg final rendering.

## 4.4. LibreTranslate

LibreTranslate là service translation nội bộ.

Worker gọi qua HTTP trong Docker network.

Không để frontend gọi LibreTranslate trực tiếp.

## 4.5. Cleanup

Cleanup service chịu trách nhiệm xoá media/job artifacts cũ theo TTL cấu hình.

Cleanup phải được xem là **operational safety mechanism**, không phải một bước trong AI pipeline.

---

# 5. Docker runtime chuẩn

Docker Compose hiện định nghĩa 4 service chính:

| Service | Vai trò | Public/Internal |
|---|---|---|
| `api` | Node.js REST API | Public `3000` |
| `worker` | Python AI/media worker | Internal `8000` |
| `libretranslate` | Translation provider | Internal `5000` |
| `cleanup` | TTL cleanup | Internal |

Runtime dependencies:

```text
api
  -> worker healthy

worker
  -> libretranslate healthy
```

Shared storage:

```text
./storage -> /app/storage
```

API data:

```text
./api/data -> /app/data
```

LibreTranslate persistent model volume:

```text
libretranslate_models
```

`INTERNAL_TOKEN` phải tồn tại trước khi Compose khởi động API/Worker.

---

# 6. Media storage model

Storage runtime không được commit vào Git.

Một Job cần có namespace riêng, ví dụ:

```text
storage/
└── downloads/
    └── <job_id>/
        ├── input/
        ├── work/
        └── output/
            └── final.mp4
```

Nguyên tắc:

1. Job không được ghi đè artifact của Job khác.
2. Path output phải được kiểm tra nằm trong storage root.
3. Không nhận path filesystem tuỳ ý từ client rồi phục vụ trực tiếp.
4. File cuối cùng phải được liên kết với Job metadata.
5. Cleanup chỉ xoá artifact đã quá TTL và không còn trạng thái đang chạy.

---

# 7. Job lifecycle

Job là đơn vị xử lý trung tâm.

Mô hình tổng quát:

```text
waiting
   |
   v
running
   |
   +-----------> failed
   |
   v
completed
```

Các field nghiệp vụ quan trọng:

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
```

Nguyên tắc:

- Job được tạo trước khi xử lý.
- Worker nhận `job_id` và thực hiện xử lý bất đồng bộ.
- Client không chờ toàn bộ quá trình trong một HTTP request dài.
- Client polling `GET /api/v1/jobs/:id`.
- Khi completed, API cung cấp output URL.
- Khi failed, error phải được lưu vào Job.

---

# 8. Các workflow nghiệp vụ chính

## 8.1. Download Only

```text
URL(s)
  -> Node API
  -> Job
  -> Python downloader / yt-dlp
  -> completed file
```

Không chạy:

```text
Whisper
Translation
TTS
Timing
Audio mixing
```

Đây là luồng được expose độc lập để dùng như Download Manager.

## 8.2. Full Dubbing

```text
Source video
   |
   v
Download
   |
   v
Extract media
   |
   v
Separate speech/background
   |
   v
Whisper transcription + timestamps + source language
   |
   +------ source == target ------> ORIGINAL VIDEO
   |
   v
Translation source -> target
   |
   v
Target-language TTS
   |
   v
Sentence timing / audio fitting
   |
   v
Mix dubbed speech + background
   |
   v
FFmpeg final rendering
   |
   v
Final MP4
```

## 8.3. Subtitle workflow

Subtitle là một pipeline riêng về logic nghiệp vụ:

```text
Download
 -> Extract audio
 -> Speech separation nếu cần
 -> Whisper
 -> Source language detection
 -> Optional translation
 -> SRT generation
 -> Optional burn-in
 -> giữ audio gốc
```

Subtitle không tự động chuyển thành dubbing nếu người dùng không yêu cầu TTS.

---

# 9. Download subsystem

Download module là subsystem dùng chung cho Download-only và Full Dubbing.

Nguyên tắc:

- Centralize download behavior.
- Không duplicate logic yt-dlp giữa nhiều workflow.
- Có inspect/info trước khi cần.
- Giữ fallback strategy của project.

Download strategy hiện tại ưu tiên yt-dlp thông thường; Android client strategy là fallback khi strategy thông thường thất bại.

Không được để mỗi route tự có một implementation downloader riêng.

---

# 10. Language model

## 10.1. Source language

`source_language`:

- Có thể là mã ISO/ứng dụng tương ứng.
- Hoặc `auto`.

Khi `auto`, Worker phải truyền giá trị phù hợp với Whisper/faster-whisper để model thực sự tự phát hiện.

Không được giả định source là English.

## 10.2. Target language

`target_language` là language của output.

Ví dụ:

```text
source = ko
target = vi
```

→ Korean speech được translate và TTS sang Vietnamese.

## 10.3. Runtime language catalog

Frontend không nên hard-code toàn bộ provider catalog.

Sử dụng:

```http
GET /api/v1/languages
```

Catalog runtime là nguồn tham chiếu authoritative cho language capability hiện tại của TTS provider cấu hình.

---

# 11. Voice selection

Voice catalog được lấy runtime:

```http
GET /api/v1/voices
GET /api/v1/voices?language=vi
```

Khi target language thay đổi:

```text
target_language changed
        |
        v
GET /voices?language=<target>
        |
        v
refresh voice dropdown
```

Quy tắc bắt buộc:

1. Voice phải cùng target language.
2. Không tự động fallback sang English khi target không hỗ trợ.
3. Nếu `voice_id` bỏ trống, backend có thể chọn verified default voice của target language.
4. Provider catalog có thể thay đổi theo thời gian; không coi bảng hard-code là authoritative.

Các default voice đã được tài liệu dự án xác nhận gồm:

```text
English:    en-US-AriaNeural
Vietnamese: vi-VN-HoaiMyNeural
```

Các voice khác phải được xác minh trước khi coi là project default.

---

# 12. Translation subsystem

Translation flow:

```text
Whisper transcript
      |
      v
Normalize language code
      |
      v
LibreTranslate /translate
      |
      v
Translated transcript
```

Payload chuẩn:

```json
{
  "q": "text",
  "source": "en",
  "target": "vi",
  "format": "text"
}
```

Response yêu cầu:

```json
{
  "translatedText": "..."
}
```

Phải kiểm tra:

- HTTP error.
- Connection error.
- Provider error object.
- Empty translation.
- Unsupported language.

## 12.1. Language alias normalization

Các mã ngôn ngữ không hoàn toàn tương thích giữa Whisper và LibreTranslate phải được normalize trước khi gọi provider.

Ví dụ tiếng Trung:

```text
zh
zh-CN
zh-Hans
```

có thể cần map về mã provider hợp lệ như:

```text
zh-Hans
```

Không được giả định mã của một provider có thể gửi nguyên trạng cho provider khác.

---

# 13. Speech-to-text

Whisper/faster-whisper là nguồn tham chiếu timestamp cho dubbing.

Worker cần giữ ít nhất:

```text
segment.start
segment.end
segment.text
source_language
```

Khi cần timing chính xác hơn, word timestamps được sử dụng theo khả năng của cấu hình model.

Nguyên tắc:

- Whisper chịu trách nhiệm nhận dạng speech.
- Whisper language detection là cơ sở để quyết định same-language bypass khi source=`auto`.
- Không dùng transcript không có timestamp làm nguồn duy nhất để đồng bộ dubbing.

---

# 14. Same-language bypass — quy tắc rất quan trọng

Nếu actual source language bằng target language:

```text
source == target
```

thì:

```text
return original video
```

Không chạy:

```text
Translation
TTS
Audio remix
Final re-render
```

Lợi ích:

- Không làm giảm chất lượng audio/video do re-encode không cần thiết.
- Tiết kiệm CPU.
- Giảm độ trễ.
- Giảm chi phí AI/provider.
- Giữ nguyên nội dung gốc.

Đây là business rule bắt buộc của V1.

---

# 15. TTS subsystem

Target-language TTS phải được gọi bằng voice thuộc target language.

```text
translated segment
       |
       v
edge TTS
       |
       v
audio segment
```

TTS không được thực hiện trước translation trong Full Dubbing.

Thứ tự chính xác:

```text
Transcript
 -> Translate
 -> TTS
```

---

# 16. Timing engine

Timing là một trong các module quan trọng nhất của hệ thống.

Whisper timestamps là scheduling reference.

Mục tiêu:

```text
preserve source anchors
preserve pauses where possible
prevent overlap
adapt speech rate
avoid destructive clipping
keep final duration under control
```

## 16.1. Không overlap

Hai segment speech không được chồng nhau.

Nếu segment mới cần nhiều thời gian hơn slot ban đầu:

```text
Do NOT:
- hard clip câu
- silently truncate words
- để hai voice segment overlap

Do:
- adapt speaking rate
- fit audio where possible
- move later segment forward when required
```

## 16.2. Sentence timing

Mỗi segment cần được xử lý dựa trên:

```text
source start
source end
translated text length
TTS duration
previous scheduled end
next segment anchor
```

Timing code phải duy trì invariant:

```text
segment[i].scheduled_end <= segment[i+1].scheduled_start
```

hoặc điều chỉnh schedule để bảo đảm không overlap.

---

# 17. Audio subsystem

Audio pipeline có các lớp chính:

```text
source audio
    |
    +--> speech/vocals
    |
    +--> background/music

translated speech
    |
    v
fit/timing
    |
    v
mixed audio
```

Các module Worker hiện có gồm nhóm:

- `audio_duration.py`
- `audio_fit.py`
- `audio_mixer.py`
- `audio_separation.py`
- `audio_service.py`
- `audio_timeline.py`

Các module này không được thay đổi đồng loạt chỉ để “refactor cho đẹp” nếu chưa có test bảo vệ behavior.

---

# 18. Speech / background separation

Demucs được sử dụng cho speech/music separation theo pipeline hiện tại.

Mục tiêu:

```text
Source audio
   |
   v
Demucs
   +---- speech/vocal stem
   +---- background/music stem
```

Background được giữ lại khi workflow yêu cầu `keep_background=true`.

Không được nhầm separation với volume mute đơn giản; mục tiêu là tái tạo nền âm thanh để ghép với dubbed speech.

---

# 19. Final rendering

FFmpeg là media renderer cuối.

```text
video-only
 + background
 + dubbed speech
 ----------------
       |
       v
   final.mp4
```

Final render phải:

- Giữ duration hợp lý theo source.
- Không làm mất audio ngoài ý muốn.
- Không truncate dubbed content ngoài quy tắc timing.
- Ghi output đúng storage namespace của Job.

---

# 20. Subtitle subsystem

Subtitle data được tạo từ transcript/timestamps.

Định dạng chuẩn V1 là SRT.

```text
1
00:00:01,000 --> 00:00:03,000
Hello everyone.
```

Nếu target language được yêu cầu:

```text
Whisper
 -> Translate
 -> target subtitle
```

Nếu không yêu cầu target:

```text
Whisper
 -> source subtitle
```

Subtitle rendering phải chú ý Unicode fonts, đặc biệt với:

- Vietnamese.
- Chinese.
- Arabic.
- Indic scripts.
- Các ngôn ngữ có shaping/RTL.

Không được coi `font` chỉ là vấn đề giao diện; đây là media rendering dependency.

---

# 21. API contract chuẩn

Base API:

```text
/api/v1
```

## Core

```text
GET  /health
GET  /api/v1
POST /api/v1/info
POST /api/v1/jobs
GET  /api/v1/jobs/:id
GET  /api/v1/output/:id
POST /api/v1/download
GET  /api/v1/languages
GET  /api/v1/voices
GET  /api/v1/worker/health
```

## Stage APIs

```text
POST /api/v1/pipeline/extract
POST /api/v1/pipeline/separate
POST /api/v1/pipeline/transcribe
POST /api/v1/pipeline/translate
POST /api/v1/pipeline/tts
POST /api/v1/pipeline/render
```

Stage APIs phải trả về `job_id` và được xử lý bất đồng bộ.

Frontend production bình thường nên dùng `/jobs` hoặc `/download`; stage APIs phục vụ:

- Diagnostic.
- Automated workflow.
- Contract test.
- Advanced integration.
- Isolated module testing.

---

# 22. Download API contract

## Flag 0 — Download only

```json
{
  "flag": 0,
  "urls": [
    "https://example.com/video"
  ]
}
```

## Flag 1 — Full Dubbing

```json
{
  "flag": 1,
  "urls": [
    "https://example.com/video"
  ],
  "source_language": "auto",
  "target_language": "vi",
  "voice_id": "vi-VN-HoaiMyNeural"
}
```

Optional:

```text
whisper_model
keep_background
sync
format
quality
```

Nếu source/target giống nhau, worker áp dụng passthrough.

---

# 23. `/api/v1/jobs` contract

Minimal request:

```json
{
  "url": "https://example.com/video"
}
```

Extended request:

```json
{
  "url": "https://example.com/video",
  "source_language": "auto",
  "target_language": "vi",
  "voice_id": "vi-VN-HoaiMyNeural",
  "whisper_model": "small",
  "keep_background": true,
  "sync": true
}
```

Response tạo Job là asynchronous, nên client phải nhận `job_id` rồi polling.

---

# 24. API security boundaries

## 24.1. Internal token

API và Worker dùng `INTERNAL_TOKEN` để xác thực internal communication.

Token không được hard-code vào source.

## 24.2. Environment variables

Secret phải nằm ở environment/.env/secret manager tùy môi trường.

Không commit:

```text
.env
provider credentials
API keys
internal secrets
private certificates
```

## 24.3. Output path security

Output endpoint chỉ được phục vụ file nằm trong storage root.

Không được cho phép:

```text
../../etc/passwd
absolute arbitrary path
symlink escape
```

## 24.4. Download URL validation

URL người dùng gửi vào phải được validate/normalize trước khi đưa tới downloader.

Cần giới hạn:

- URL schemes.
- Request size.
- Job size/duration nếu business rule yêu cầu.
- Batch size.

---

# 25. Database

Database của API lưu Job metadata và trạng thái xử lý.

SQLite phù hợp cho baseline triển khai nhỏ/đơn node hiện tại.

Khi scale lớn hơn, database persistence có thể được tách thành service riêng **nhưng đây là architecture revision**, không tự ý làm trong một feature nhỏ.

Nguyên tắc migration:

1. Migration phải versioned.
2. Không sửa trực tiếp production DB mà bỏ qua migration history.
3. Sau migration phải chạy smoke test Job lifecycle.
4. Backup trước migration destructive.

---

# 26. Cleanup / TTL

Cleanup service có các biến cấu hình dạng:

```text
CLEANUP_ENABLED
CLEANUP_TTL_HOURS
CLEANUP_INTERVAL_MINUTES
```

Cleanup phải:

- Chỉ xoá artifact quá TTL.
- Không xoá Job đang chạy.
- Không phá output của Job còn được phép tải.
- Không xoá database record chưa đến policy retention.

Cleanup là một phần của production hygiene và phải được test riêng.

---

# 27. CI chuẩn

GitHub Actions hiện có CI cho `main` và pull request vào `main`.

## Worker tests

CI thực hiện:

```text
Setup Python 3.11
Install FFmpeg
Install worker dependencies
compileall
pytest -m "not integration"
```

## API tests

```text
Setup Node.js 20
npm install
npm test
```

## Compose validation

```text
docker compose --env-file .env.example config -q
```

## Docker build

Chỉ build image sau khi các bước test/validation chính thành công.

---

# 28. AI/media integration tests

Ngoài unit/contract test, project có workflow AI integration chạy deterministic media tests.

Các bài test bao gồm nhóm:

```text
test_media_fixture.py
test_audio_pipeline.py
test_media_pipeline.py
```

Workflow này kiểm tra media toolchain và audio/media processing không phụ thuộc vào một video ngẫu nhiên từ internet.

Nguyên tắc quan trọng:

- Unit test kiểm tra logic.
- Contract test kiểm tra interface.
- Deterministic media test kiểm tra FFmpeg/audio/video behavior.
- AI smoke test kiểm tra model/provider availability ở mức cần thiết.
- Không phụ thuộc toàn bộ CI vào một URL YouTube ngoài internet.

---

# 29. CD chuẩn

Sau khi CI trên `main` thành công, CD publish Docker images lên GitHub Container Registry.

Images:

```text
ghcr.io/<owner>/videodubing-api
ghcr.io/<owner>/videodubing-worker
```

Tagging hiện tại:

```text
:<commit-sha>
:latest
```

Trong production, ưu tiên deploy bằng immutable SHA tag hơn `latest`.

Luồng:

```text
push main
   |
   v
CI
   |
   +-- worker tests
   +-- API tests
   +-- compose validation
   +-- docker build
   |
   v
CD
   |
   v
GHCR
```

---

# 30. Quy trình phát triển feature mới — chuẩn Final

Mỗi feature phải đi qua các bước:

### Bước 1 — Xác định layer

Trả lời rõ feature thuộc:

```text
Frontend
API
Worker
Translation
TTS
Timing
Media
Database
Deployment
```

### Bước 2 — Không phá architecture

Không di chuyển folder hoặc thay đổi contract liên service chỉ vì feature nhỏ.

### Bước 3 — Viết/điều chỉnh contract trước

Xác định:

```text
Request
Response
Job fields
Error contract
Progress contract
```

### Bước 4 — Implement đúng owner

- API orchestration → Node.
- AI/media → Python.
- UI behavior → PHP/frontend.
- Deployment → Docker/GitHub Actions.

### Bước 5 — Test

Tối thiểu:

```text
Unit test
Contract test
Integration test liên quan
```

### Bước 6 — Regression

Phải kiểm tra các workflow đã có:

```text
Download
Dubbing
Same-language bypass
Subtitle
Output download
Job polling
```

### Bước 7 — CI

Không merge nếu CI baseline fail.

### Bước 8 — Deploy

Chỉ deploy image/commit đã qua CI thành công.

---

# 31. Quy trình sửa lỗi

Một bug phải được xử lý theo thứ tự:

```text
Reproduce
  -> Capture logs
  -> Identify owning layer
  -> Add regression test
  -> Fix
  -> Run targeted tests
  -> Run full regression
  -> Deploy
```

Không sửa “cho chạy được” bằng workaround che lỗi ở layer khác.

Ví dụ không được:

```text
Worker trả voice sai language
-> Frontend tự đổi language thành English
```

Mà phải sửa tại validation/voice selection layer.

---

# 32. Quy tắc bảo vệ các module quan trọng

Các module media core có rủi ro regression cao:

```text
dubbing.py
audio_fit.py
audio_mixer.py
timing.py
final_renderer.py
subtitle_renderer.py
transcriber.py
```

Không được refactor lớn nếu không có:

1. Test trước thay đổi.
2. Test sau thay đổi.
3. Media fixture để so sánh behavior.
4. Xác nhận không phá timing/output duration.

---

# 33. Quy tắc versioning provider

AI providers và media tools có thể thay đổi catalog hoặc behavior.

Các nhóm cần quản lý version:

- Python dependencies.
- Node dependencies.
- FFmpeg.
- LibreTranslate image.
- Whisper/faster-whisper model.
- Edge TTS voice catalog.
- yt-dlp.

Khi provider thay đổi:

```text
provider update
 -> smoke test
 -> voice catalog test
 -> translation test
 -> media test
 -> release
```

Không coi provider update là một thay đổi vô hại.

---

# 34. Quan sát hệ thống / monitoring

Các health endpoint chuẩn:

```text
GET /health
GET /api/v1/worker/health
```

Cần monitor:

```text
API availability
Worker availability
LibreTranslate availability
Job failure rate
Job duration
Queue/active job count
Storage usage
Cleanup health
```

Log tối thiểu phải trả lời được:

- Job nào lỗi?
- Stage nào lỗi?
- Provider nào lỗi?
- Input/output path nào?
- Error gốc là gì?
- Retry bao nhiêu lần?

---

# 35. Performance model

Các bước tốn CPU/RAM nhất thường là:

```text
Whisper
Demucs
TTS
FFmpeg encoding
```

Scale đúng cách là scale Worker, không phải cứ tăng Node API.

Khi workload tăng:

```text
API remains lightweight
       |
       v
more worker capacity
       |
       +---- GPU worker nếu cần
       +---- CPU worker nếu phù hợp
```

Nhưng mở rộng multi-worker/queue architecture là phase nâng cấp riêng và phải bảo vệ Job locking/idempotency.

---

# 36. Idempotency và retry

Job processing phải được thiết kế để retry không tạo ra artifact mồ côi hoặc Job duplicate không kiểm soát.

Retry cần cân nhắc:

- Download failure.
- Temporary provider failure.
- Translation timeout.
- TTS transient failure.
- FFmpeg failure.

Không retry vô hạn.

Mỗi retry phải tăng `retry_count` và ghi error context.

---

# 37. Error handling chuẩn

Error phải được phân loại ít nhất:

```text
VALIDATION_ERROR
DOWNLOAD_ERROR
MEDIA_ERROR
TRANSCRIPTION_ERROR
TRANSLATION_ERROR
TTS_ERROR
TIMING_ERROR
RENDER_ERROR
INTERNAL_ERROR
```

Nguyên tắc:

- User-facing message đủ hiểu.
- Internal log có technical detail.
- Không leak secret/provider credentials.
- Không báo “success” khi artifact chưa hoàn chỉnh.

---

# 38. Security baseline

Production deployment cần tối thiểu:

- HTTPS tại reverse proxy.
- Firewall chỉ mở public ports cần thiết.
- API/Worker internal network được giới hạn.
- `INTERNAL_TOKEN` đủ mạnh.
- Secret không commit Git.
- Storage directory không public trực tiếp nếu không cần.
- Output download qua API validation.
- Rate limit/public abuse control theo mức sử dụng.
- Request size limit.
- Batch size limit.
- Log rotation.
- OS/Docker patching.

Nếu hệ thống cung cấp dịch vụ công cộng, cần bổ sung quota/billing/rate limiting ở tầng business.

---

# 39. Sẵn sàng tích hợp Frontend/PHP

Frontend PHP không cần biết nội bộ Worker hoạt động thế nào.

Contract tối thiểu:

```text
POST /api/v1/download
POST /api/v1/jobs
GET  /api/v1/jobs/:id
GET  /api/v1/output/:id
GET  /api/v1/languages
GET  /api/v1/voices?language=...
```

Frontend chỉ nên biết:

```text
create job
poll job
show progress
show result
```

Không đặt business logic AI vào JavaScript/PHP nếu logic đó đã thuộc Worker/API.

---

# 40. Quy tắc mở rộng multilingual

Hệ thống được thiết kế để multilingual.

Muốn thêm language mới phải kiểm tra đủ:

```text
Language catalog
Translation support
TTS voice support
Subtitle rendering/font
Whisper recognition
Timing behavior
```

Không coi “provider có voice” đồng nghĩa “pipeline hoàn toàn hỗ trợ language”.

Một language chỉ được coi là verified khi tối thiểu đã qua:

1. Language discovery.
2. Voice discovery.
3. Translation test.
4. TTS test.
5. Timing test.
6. Final render test.
7. Unicode subtitle test nếu subtitle được hỗ trợ.

---

# 41. Quy tắc mở rộng provider

Architecture cho phép thay thế provider miễn là giữ internal contracts.

Ví dụ:

```text
LibreTranslate
    |
    +--> replacement translation provider

Edge TTS
    |
    +--> replacement TTS provider

Whisper
    |
    +--> replacement ASR provider
```

Adapter/provider logic nên được cô lập trong service module.

Core business flow vẫn giữ:

```text
Transcribe
 -> Translate
 -> TTS
 -> Timing
 -> Mix
 -> Render
```

---

# 42. Database migration policy

Mỗi thay đổi schema phải có migration file.

Ví dụ baseline hiện dùng migration đặt tên theo ngày cho FrontEnd/database.

Quy tắc:

```text
new feature
 -> migration
 -> backup
 -> migrate
 -> test
```

Không commit code yêu cầu cột database mới nhưng quên migration.

---

# 43. Release checklist

Trước release:

```text
[ ] Branch đúng main/release
[ ] API tests pass
[ ] Worker tests pass
[ ] Compose validation pass
[ ] Docker build pass
[ ] Deterministic media tests pass
[ ] Download flow pass
[ ] Dubbing flow pass
[ ] Same-language passthrough pass
[ ] Subtitle flow pass
[ ] Output download pass
[ ] Language/voice catalog pass
[ ] No secrets committed
[ ] Migration verified
[ ] Storage cleanup verified
[ ] Health endpoints pass
[ ] Backup/rollback plan available
```

---

# 44. Deployment checklist

## Pre-deploy

```text
[ ] Backup DB
[ ] Backup required storage
[ ] Verify .env / secrets
[ ] Verify Docker availability
[ ] Pull/build correct images
[ ] Verify image SHA
```

## Deploy

```text
docker compose pull

docker compose up -d
```

## Verify

```text
GET /health
GET /api/v1/worker/health
```

Sau đó chạy smoke test:

```text
Download
 -> Job completed
 -> Output downloadable
```

và một Full Dubbing test nhỏ.

---

# 45. Rollback

Rollback phải dùng immutable image/commit.

Ưu tiên:

```text
GHCR image:<commit-sha>
```

Không phụ thuộc duy nhất vào:

```text
:latest
```

Rollback workflow:

```text
Identify bad release
 -> stop promotion
 -> restore previous image SHA
 -> restore DB if migration destructive
 -> smoke test
 -> reopen traffic
```

---

# 46. Các điểm kỹ thuật đã được xác nhận trong baseline

Các quy tắc sau được thể hiện trực tiếp trong source/docs của branch `main`:

1. Node.js API + Python Worker là kiến trúc cố định.
2. Download được dùng chung cho Download-only và Full Dubbing.
3. Source language có thể `auto`.
4. Target language là input bắt buộc về logic cho Full Dubbing.
5. Voice phải cùng target language.
6. Không silent English fallback.
7. Source == target → original passthrough.
8. Stage APIs là asynchronous và dùng `job_id`.
9. Runtime `/languages` và `/voices` được dùng để discover capability.
10. TTS timing dùng timestamp làm scheduling reference.
11. Generated speech không được silently hard-trim.
12. Speech segment không được overlap.
13. Docker Compose có API, Worker, LibreTranslate và cleanup.
14. CI kiểm tra worker, API, Compose và Docker build.
15. CD publish API/Worker images lên GHCR sau CI thành công.

---

# 47. Những gì không được tự ý thay đổi

Không tự ý thay đổi:

```text
Directory architecture
Job contract
/api/v1 contract
source==target behavior
voice language validation
stage API semantics
storage security model
Internal API token mechanism
CI/CD promotion rule
```

Một thay đổi phá contract cần được coi là **architecture/API revision**, không phải bugfix nhỏ.

---

# 48. Quy ước commit

Khuyến nghị:

```text
feat: ...
fix: ...
refactor: ...
test: ...
docs: ...
ci: ...
cd: ...
perf: ...
security: ...
```

Commit phải mô tả đúng technical scope.

Ví dụ:

```text
fix: prevent translated speech overlap
feat: add target-language voice discovery
fix: normalize zh-Hans for translation
ci: add deterministic audio pipeline test
```

---

# 49. Quy ước branch

Baseline:

```text
main
```

Khuyến nghị phát triển:

```text
feature/<name>
fix/<name>
refactor/<name>
release/<version>
```

Không push thử nghiệm trực tiếp lên production branch nếu chưa qua CI.

---

# 50. Phân biệt Code chuẩn và tài liệu lịch sử

Trong quá trình phát triển đã tồn tại nhiều tài liệu, log và ghi chú của các giai đoạn trước.

Quy tắc Final:

```text
Current main source code
        >
Current architecture/API docs
        >
Historical notes/logs
```

Nếu tài liệu lịch sử mâu thuẫn với source `main`, phải ưu tiên source hiện tại và cập nhật tài liệu.

---

# 51. Roadmap sau Final — không phá V1

Các hướng mở rộng có thể triển khai sau baseline:

## Phase A — Production hardening

- Rate limiting.
- Auth/session hoàn chỉnh cho public API.
- Centralized logging.
- Metrics.
- Storage quota.
- Job concurrency control.
- Better retry policy.

## Phase B — Scale

- PostgreSQL/MySQL nếu cần.
- Queue broker.
- Multiple Worker replicas.
- GPU worker pool.
- Object storage.
- CDN/output acceleration.

## Phase C — Commercial service

- User quota.
- Credit/billing.
- Usage accounting.
- Reverse transaction model nếu business chọn cách tính theo consumption.
- Subscription/credit packages.
- Per-minute/per-job charging.

## Phase D — AI quality

- Better voice cloning/provider options.
- Speaker diarization.
- Multi-speaker voice assignment.
- Prosody/style transfer.
- Advanced lip-sync.
- Voice consistency across long-form video.

Mỗi phase phải đi qua architecture review trước khi thay đổi baseline.

---

# 52. Kết luận Final

**Video Dubbing AI** hiện có một nền tảng kỹ thuật rõ ràng:

```text
Frontend/PHP
     |
     v
Node.js API
     |
     v
Python Worker
     |
     +--> Download
     +--> FFmpeg
     +--> Demucs
     +--> Whisper/faster-whisper
     +--> Translation
     +--> Edge TTS
     +--> Timing
     +--> Audio mixing
     +--> Subtitle
     +--> Final render
     |
     v
Storage / Job metadata
```

Đây là baseline cần được sử dụng làm **chuẩn phát triển chính thức**.

Nguyên tắc quan trọng nhất của toàn bộ hệ thống là:

> **Giữ architecture ổn định, giữ API contract ổn định, bảo vệ timing/audio pipeline bằng regression tests, và chỉ mở rộng provider/hạ tầng qua các adapter hoặc architecture revision rõ ràng.**

Khi phát triển tiếp, mọi thay đổi nên trả lời được bốn câu hỏi:

```text
1. Thay đổi ở layer nào?
2. Contract nào bị ảnh hưởng?
3. Test nào bảo vệ behavior?
4. Rollback bằng cách nào?
```

Nếu bốn câu hỏi này đều có câu trả lời rõ ràng, thay đổi mới đủ điều kiện để đưa vào baseline phát triển tiếp theo.

---

# 53. Source of Truth

## Repository nguồn

`daoaigiangnam/Long_Tieng_Phu_De_Video`

Branch:

```text
main
```

## Repository tài liệu

`daoaigiangnam/System_Operation_and_Deployment_Documentation`

## Tài liệu kỹ thuật nguồn quan trọng

```text
README.md
docs/ARCHITECTURE.md
docs/API_PIPELINE.md
docs/API_GUIDE.md
docs/DEPLOY_UPDATE.md
docs/CLEANUP_TTL.md
docs/INSTALL_UBUNTU.md
docs/INSTALL_UBUNTU_FULL.md
docs/ROADMAP.md
docker-compose.yml
.github/workflows/ci.yml
.github/workflows/ai-integration.yml
.github/workflows/ai-model-smoke.yml
.github/workflows/compose-smoke.yml
.github/workflows/cd.yml
```

---

**FINAL — 16/08/2026**
