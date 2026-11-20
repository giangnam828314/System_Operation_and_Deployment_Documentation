# 00B — STATE MACHINE VÀ QUY TẮC CHUYỂN TRẠNG THÁI

**Phiên bản:** 0.1-dự thảo  
**Trạng thái:** Đề xuất chuẩn hóa — chưa được phê duyệt  
**Phạm vi:** Booking, DispatchRun, Offer, Assignment, Trip, Payment, Settlement và trạng thái khai thác Driver/Vehicle.

> Tài liệu này chuẩn hóa cách mô hình hóa vòng đời giữa các tài liệu 03, 04, 07, 08, 19, 21 và 25. Các tên trạng thái dưới đây là đề xuất để rà soát; không được coi là chính sách đã phê duyệt hoặc triển khai production trước khi chủ sở hữu nghiệp vụ xác nhận.

## 1. Mục tiêu và nguyên tắc

- Mỗi đối tượng nghiệp vụ có vòng đời và state machine riêng.
- Booking Service sở hữu Booking; Dispatch sở hữu DispatchRun, Offer, Assignment; Trip Service sở hữu Trip; Payment sở hữu PaymentTransaction; Settlement sở hữu SettlementBatch/Line.
- Chuyển trạng thái là command có kiểm tra, không phải cập nhật tùy ý trường status từ giao diện.
- Máy chủ là nguồn có thẩm quyền. Ứng dụng tài xế gửi yêu cầu/sự kiện, không tự xác nhận trạng thái cuối cùng.
- Tách trạng thái nghiệp vụ khỏi trạng thái kết nối. GPS cũ không đồng nghĩa tài xế offline.
- Trạng thái kết thúc không tự quay lại trạng thái trước; muốn tiếp tục phải tạo quy trình/đối tượng mới hoặc dùng command phục hồi được duyệt.
- Không xóa lịch sử. Chuyển trạng thái quan trọng phải lưu actor, thời gian máy chủ, nguồn, lý do và correlation_id.
- Lệnh ghi phải chống lặp và kiểm soát đồng thời bằng idempotency key, version/optimistic locking và ràng buộc dữ liệu.
- Chính sách chưa được duyệt giữ TBD; không tự đặt timeout, retry, thời gian chờ hay phí.

## 2. Danh mục State Machine chuẩn đề xuất

| Mã | Đối tượng | Module sở hữu | Trách nhiệm |
|---|---|---|---|
| SM-BKG | Booking | Booking | Yêu cầu dịch vụ của khách |
| SM-DSP | DispatchRun | Dispatch | Một lần tìm và phân phối ứng viên |
| SM-OFR | Offer | Dispatch | Lời mời cụ thể gửi đến tài xế |
| SM-ASG | Assignment | Dispatch | Kết quả phân công tài xế/xe |
| SM-TRP | Trip | Trip | Lần thực hiện dịch vụ vận chuyển |
| SM-PAY | PaymentTransaction | Payment | Một giao dịch thu/hoàn tiền |
| SM-SET | SettlementBatch | Settlement | Bảng kê đối soát theo đối tác/kỳ |
| SM-DRV | Driver availability | Fleet/Driver | Khả năng nhận chuyến của tài xế |
| SM-VHL | Vehicle availability | Fleet | Khả năng khai thác của xe |

Không tạo một trường trạng thái chung cho tất cả các vòng đời. Các đối tượng liên kết bằng ID và sự kiện nghiệp vụ.

## 3. State Machine chi tiết

### 3.1. SM-BKG — Booking

**Trạng thái đề xuất**

- DRAFT: đang nhập, chưa gửi yêu cầu chính thức.
- CONFIRMED: yêu cầu đã qua kiểm tra và được xác nhận tiếp nhận.
- READY_FOR_DISPATCH: đủ dữ liệu/điều kiện để chuyển điều phối.
- SEARCHING: đang có hoạt động tìm nguồn xe.
- ASSIGNED: đã có Assignment hiệu lực.
- IN_SERVICE: chuyến đã bắt đầu thực hiện.
- COMPLETED: dịch vụ hoàn thành.
- CANCELLED: yêu cầu bị hủy hợp lệ.
- EXPIRED: yêu cầu hết hiệu lực theo chính sách đã duyệt.
- UNFULFILLED: không thể bố trí dịch vụ sau quy trình xử lý được duyệt.
- EXCEPTION: cần can thiệp/rà soát; không phải trạng thái kết thúc mặc định.

~~~mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> CONFIRMED: Submit / validation passed
  CONFIRMED --> READY_FOR_DISPATCH: Prerequisites passed
  READY_FOR_DISPATCH --> SEARCHING: Start dispatch
  SEARCHING --> ASSIGNED: Assignment activated
  SEARCHING --> EXCEPTION: Manual intervention
  EXCEPTION --> SEARCHING: Resume dispatch
  EXCEPTION --> UNFULFILLED: Approved decision
  ASSIGNED --> IN_SERVICE: Trip started
  IN_SERVICE --> COMPLETED: Trip completed
  DRAFT --> CANCELLED
  CONFIRMED --> CANCELLED
  READY_FOR_DISPATCH --> CANCELLED
  SEARCHING --> CANCELLED
  CONFIRMED --> EXPIRED
  READY_FOR_DISPATCH --> EXPIRED
~~~

**Quy tắc**
- SEARCHING phản ánh Booking đang được điều phối; chi tiết từng lượt tìm nằm ở DispatchRun.
- ASSIGNED chỉ được thiết lập sau khi Assignment được kích hoạt thành công với kiểm soát đồng thời.
- IN_SERVICE được đồng bộ từ sự kiện Trip hợp lệ; Booking không tự bắt đầu chuyến.
- Không có ứng viên không tự động chuyển Booking thành CANCELLED hoặc UNFULFILLED; giữ hàng đợi ngoại lệ và xử lý theo chính sách.
- Đổi điểm đón/điểm đến sau phân công phải kiểm tra trạng thái, báo giá, Offer và Assignment.
- Quyền hủy, thời điểm được hủy, phí hủy và điều kiện hết hạn: TBD.

### 3.2. SM-DSP — DispatchRun

Trạng thái đề xuất: CREATED → RUNNING → COMPLETED. Nhánh khác: NO_CANDIDATE, CANCELLED, FAILED, ESCALATED.

~~~mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> RUNNING: Start run
  RUNNING --> COMPLETED: Run closed
  RUNNING --> NO_CANDIDATE: No eligible candidate
  NO_CANDIDATE --> RUNNING: Retry with fresh snapshot
  NO_CANDIDATE --> ESCALATED: Retry policy exhausted
  CREATED --> CANCELLED
  RUNNING --> CANCELLED
  RUNNING --> FAILED: Technical failure
  FAILED --> RUNNING: Controlled retry
~~~

- Mỗi DispatchRun thuộc một Booking, có attempt_no, policy_version và thời gian bắt đầu/kết thúc.
- Snapshot ứng viên và dữ liệu đầu vào của mỗi lần chạy phải truy vết được.
- Retry không ghi đè lịch sử lần chạy cũ.
- NO_CANDIDATE là kết quả một lần tìm, không phải trạng thái kết thúc Booking.
- Cách đếm attempt, ngưỡng retry, timeout và điều kiện escalated: TBD.

### 3.3. SM-OFR — Offer

Trạng thái đề xuất: CREATED → SENT → ACCEPTED hoặc DECLINED / EXPIRED / CANCELLED / SUPERSEDED.

~~~mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> SENT: Delivery requested
  SENT --> ACCEPTED: Valid accept wins
  SENT --> DECLINED: Driver rejects
  SENT --> EXPIRED: Server expiry
  CREATED --> CANCELLED
  SENT --> CANCELLED
  SENT --> SUPERSEDED: Another offer wins
~~~

**Guard khi chấp nhận**
1. Offer thuộc Booking/DispatchRun còn hợp lệ.
2. Offer đang SENT và chưa hết hạn theo thời gian máy chủ.
3. Booking vẫn cho phép điều phối.
4. Tài xế và xe vẫn đạt điều kiện khai thác.
5. Tài xế/xe không có phân công hoặc chuyến xung đột.
6. Chưa có Assignment khác thắng cho Booking.
7. Idempotency key hợp lệ; yêu cầu lặp trả kết quả đã ghi nhận.

Lần chấp nhận hợp lệ đầu tiên thắng. Offer cạnh tranh được đóng phù hợp. Trạng thái hiển thị trên thiết bị không thay thế trạng thái Offer do máy chủ quản lý. Timeout và cách gửi tuần tự/theo nhóm: TBD.

### 3.4. SM-ASG — Assignment

Trạng thái đề xuất: PENDING → ACTIVE → COMPLETED. Nhánh khác: RELEASED, CANCELLED, REASSIGNED, FAILED.

~~~mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> ACTIVE: Atomic activation succeeds
  PENDING --> FAILED: Validation fails
  PENDING --> CANCELLED
  ACTIVE --> COMPLETED: Linked trip completed
  ACTIVE --> RELEASED: Approved release
  ACTIVE --> REASSIGNED: Replacement activated
  ACTIVE --> CANCELLED: Approved cancellation
~~~

- Assignment là bản ghi phân công, không phải trạng thái Booking hoặc Trip.
- Một Booking tối đa có một Assignment đang hiệu lực tại một thời điểm.
- Tài xế/xe không được có Assignment/Trip xung đột theo chính sách.
- Kích hoạt phải là thao tác nguyên tử: kiểm tra Booking, Offer, Driver, Vehicle và ghi Assignment.
- Phân xe lại phải kết thúc Assignment cũ có thời điểm/lý do; Assignment mới có ID riêng.
- Định nghĩa chính xác “đang hiệu lực”, khóa tài nguyên và điều kiện giải phóng: cần chốt trong thiết kế dữ liệu/kiến trúc.

### 3.5. SM-TRP — Trip

Trạng thái đề xuất: CREATED → DRIVER_EN_ROUTE → ARRIVED → IN_PROGRESS → COMPLETED. Nhánh ngoại lệ: CANCELLED, NO_SHOW, INCIDENT, DISPUTED, NEEDS_REVIEW.

~~~mermaid
stateDiagram-v2
  [*] --> CREATED
  CREATED --> DRIVER_EN_ROUTE
  DRIVER_EN_ROUTE --> ARRIVED
  ARRIVED --> IN_PROGRESS: Start trip accepted
  ARRIVED --> NO_SHOW: Approved conditions
  DRIVER_EN_ROUTE --> CANCELLED
  ARRIVED --> CANCELLED
  IN_PROGRESS --> COMPLETED: Completion validated
  DRIVER_EN_ROUTE --> INCIDENT
  ARRIVED --> INCIDENT
  IN_PROGRESS --> INCIDENT
  INCIDENT --> IN_PROGRESS: Resume approved
  INCIDENT --> NEEDS_REVIEW
  COMPLETED --> DISPUTED: Dispute opened
~~~

- Trip liên kết một Booking và một Assignment; không tự chọn tài xế/xe ngoài Assignment hiệu lực.
- TripEvent lưu actor, server timestamp, source, device timestamp nếu có, vị trí/độ chính xác nếu có, sequence và correlation_id.
- Sự kiện offline có client UUID/sequence; máy chủ kiểm tra trùng, thứ tự và trạng thái trước khi xác nhận.
- Không cho phép hoàn tất Trip nhiều lần gây lặp tính cước/hạch toán.
- DISPUTED là xử lý tranh chấp sau hoàn thành, không phải chuyến đang chạy.
- Đề xuất Incident Case là đối tượng riêng; Trip state phản ánh chuyến bị gián đoạn/rà soát.
- Điều kiện no-show, thời gian chờ, bằng chứng, quyền hủy và điều kiện tiếp tục sau incident: TBD.

### 3.6. SM-PAY — PaymentTransaction

Trạng thái đề xuất: PENDING → INITIATED → SUCCEEDED hoặc FAILED / EXPIRED / CANCELLED / RECONCILIATION_REQUIRED.

~~~mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> INITIATED
  INITIATED --> SUCCEEDED: Verified provider success
  INITIATED --> FAILED: Verified failure
  INITIATED --> EXPIRED
  INITIATED --> RECONCILIATION_REQUIRED: Unknown outcome
  FAILED --> INITIATED: Retry allowed
  SUCCEEDED --> RECONCILIATION_REQUIRED: Reconciliation mismatch
~~~

- SUCCEEDED chỉ ghi khi xác thực kết quả; không dựa vào xác nhận chủ quan của tài xế.
- Callback phải xác thực chữ ký, chống replay và idempotency.
- Timeout mạng không đồng nghĩa thất bại; nếu chưa rõ kết quả, chuyển đối soát thay vì thu tiền lặp.
- Hoàn tiền là transaction mới có tham chiếu giao dịch gốc; không sửa giao dịch thu tiền đã thành công.
- Trip hoàn thành và Payment thành công là hai vòng đời độc lập.
- Cash handling, retry, partial refund và thời điểm ghi Ledger: TBD, cần Tài chính duyệt.

### 3.7. SM-SET — SettlementBatch

Trạng thái đề xuất: DRAFT → CALCULATED → REVIEWED → APPROVED → LOCKED → PAID. Nhánh khác: REJECTED, ON_HOLD, CANCELLED.

~~~mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> CALCULATED
  CALCULATED --> REVIEWED
  REVIEWED --> APPROVED
  REVIEWED --> REJECTED
  REJECTED --> DRAFT: Rework with audit
  APPROVED --> LOCKED
  LOCKED --> PAID: Payout confirmed
  CALCULATED --> ON_HOLD: Variance/dispute
  REVIEWED --> ON_HOLD: Variance/dispute
  ON_HOLD --> REVIEWED: Issue resolved
~~~

- Batch xác định theo partner, contract/policy version, kỳ, tiền tệ và chiều phân tách được duyệt.
- Mỗi dòng truy về Trip, LedgerEntry và snapshot hợp đồng/chính sách.
- Không đưa trùng căn cứ tài chính vào một bảng kê, trừ điều chỉnh được định nghĩa rõ.
- LOCKED không được sửa/xóa; sai lệch sau khóa dùng adjustment batch/dòng điều chỉnh liên kết.
- PAID cần bằng chứng chi trả và tham chiếu.
- Maker-checker và phân quyền lập/kiểm tra/duyệt/khóa/chi trả: đối chiếu tài liệu 23, cần phê duyệt.

### 3.8. SM-DRV và SM-VHL — trạng thái khai thác Driver/Vehicle

**Driver availability đề xuất:** OFFLINE, AVAILABLE, RESERVED, ON_TRIP, PAUSED, SUSPENDED.

**Vehicle availability đề xuất:** AVAILABLE, RESERVED, ON_TRIP, PAUSED, MAINTENANCE, SUSPENDED.

- GPS freshness, network connectivity và availability state là ba thuộc tính khác nhau.
- Không tự chuyển Driver/Vehicle sang OFFLINE chỉ vì GPS cũ.
- AVAILABLE chỉ được đặt khi tài khoản, giấy tờ, ca, xe, hợp đồng và xung đột chuyến đều đạt.
- Tài liệu 04 có danh sách Vehicle riêng; tài liệu 07 mô tả hành trình Driver App. Danh sách trên cần Fleet/Vận hành xác nhận.

## 4. Bảng ánh xạ trạng thái giữa tài liệu hiện có

| Đối tượng | Tên trong tài liệu nguồn | Đề xuất chuẩn trong 00B | Nhận xét |
|---|---|---|---|
| Booking | 03: CONFIRMED / READY_FOR_DISPATCH | Giữ nguyên | Phù hợp |
| Booking | 03: ASSIGNED | ASSIGNED | Chỉ sau Assignment hiệu lực |
| Booking | 03: EN_ROUTE_TO_PICKUP, ARRIVED, IN_TRIP | Chuyển sang Trip: DRIVER_EN_ROUTE, ARRIVED, IN_PROGRESS | Không đặt hành trình tài xế trong Booking |
| Booking | 04: DRAFT, CONFIRMED, READY_FOR_DISPATCH, SEARCHING, ASSIGNED, IN_SERVICE, COMPLETED | Giữ trạng thái phù hợp; IN_SERVICE đồng bộ từ Trip IN_PROGRESS | Cần duyệt tên cuối |
| Booking | 19: REQUESTED, QUOTED, DISPATCHING, DRIVER_EN_ROUTE, ARRIVED, IN_PROGRESS | Phân tách Booking, Quote, DispatchRun và Trip | Không giữ thành một chuỗi Booking |
| Offer | 19: CREATED, SENT, ACCEPTED, DECLINED, EXPIRED, CANCELLED, SUPERSEDED | Giữ nguyên | Bổ sung guard/server-time expiry |
| Assignment | 04: CREATED, OFFERED, ACCEPTED, ACTIVE, REJECTED, EXPIRED, WITHDRAWN, REASSIGNED | Assignment PENDING → ACTIVE; Offer quản lý ACCEPTED/REJECTED/EXPIRED | Tránh trộn hai aggregate |
| Trip | 04: ASSIGNED, EN_ROUTE_TO_PICKUP, ARRIVED, IN_TRIP, COMPLETED | CREATED, DRIVER_EN_ROUTE, ARRIVED, IN_PROGRESS, COMPLETED | ASSIGNED thuộc Booking/Assignment |
| Trip | 07: EN_ROUTE, IN_TRIP, NO_SHOW_FLOW, INCIDENT | DRIVER_EN_ROUTE, IN_PROGRESS, NO_SHOW, INCIDENT | NO_SHOW_FLOW có thể là quy trình con |
| Payment | 04/08: SUCCESS, EXPIRED | Đề xuất SUCCEEDED, EXPIRED | Chuẩn tên; cập nhật API sau phê duyệt |
| Settlement | 04/08: DRAFT, CALCULATED, REVIEWED, APPROVED, LOCKED, PAID | Giữ nguyên | Phù hợp |

Bảng này là ánh xạ thiết kế, không phải lệnh tự động đổi dữ liệu. Nếu có dữ liệu vận hành, phải lập migration mapping và đối soát trước khi chuyển đổi.

## 5. Transition Contract — hợp đồng chuyển trạng thái

Mọi API/command thay đổi trạng thái phải xác định tối thiểu:

| Thuộc tính | Ý nghĩa |
|---|---|
| command | Tên lệnh nghiệp vụ, ví dụ AcceptOffer, StartTrip |
| aggregate_id | ID đối tượng cần chuyển trạng thái |
| expected_state | Trạng thái client kỳ vọng |
| expected_version | Phiên bản đối tượng client đã đọc |
| idempotency_key | Khóa chống xử lý lặp |
| actor | Người/hệ thống thực hiện |
| source | Driver App, Call Center, Control Tower, API, System |
| reason_code | Mã lý do khi cần |
| correlation_id | Liên kết xuyên Module |
| occurred_at | Thời gian sự kiện từ client nếu có |
| server_recorded_at | Thời gian máy chủ ghi nhận |

Máy chủ kiểm tra quyền, trạng thái hiện tại, version, guard nghiệp vụ và điều kiện đồng thời trước khi ghi.

**Kết quả command**
- Thành công: trả trạng thái mới, version mới và ID sự kiện.
- Cùng idempotency key đã xử lý: trả lại kết quả đã lưu.
- Xung đột version/trạng thái: trả lỗi conflict, không ghi một phần.
- Không đủ quyền: từ chối và audit theo chính sách.
- Không đạt guard: trả mã lỗi nghiệp vụ ổn định.
- Lỗi kỹ thuật chưa rõ kết quả: client truy vấn trạng thái/đối soát, không gửi lệnh mới bằng khóa khác một cách mù quáng.

## 6. Ma trận chuyển trạng thái và guard tối thiểu

| Aggregate | Command/sự kiện | Guard tối thiểu | Tác động cần kiểm soát |
|---|---|---|---|
| Booking | SubmitBooking | Dữ liệu bắt buộc, vùng/dịch vụ hợp lệ, quyền tạo | Ghi Booking và audit |
| Booking | StartDispatch | Booking đủ điều kiện, chưa hủy/hết hạn | Tạo DispatchRun |
| Booking | ActivateAssignment | Assignment được kích hoạt hợp lệ | Cập nhật Booking ASSIGNED |
| Booking | CancelBooking | Vai trò, trạng thái, chính sách hủy | Dừng Offer/Dispatch; phí nếu được duyệt |
| DispatchRun | StartRun | Booking cho phép điều phối | Snapshot ứng viên/chính sách |
| DispatchRun | RetryRun | Lý do retry, giới hạn và policy hợp lệ | Tạo attempt mới, không ghi đè |
| Offer | SendOffer | Ứng viên đã qua eligibility | Gửi thông báo, đặt expires_at |
| Offer | AcceptOffer | Chưa hết hạn; Driver/Vehicle đủ điều kiện; không xung đột | Atomic assignment activation |
| Offer | DeclineOffer | Offer SENT; actor đúng tài xế | Đóng Offer, lưu reason |
| Assignment | ActivateAssignment | Booking/Offer/resource còn hợp lệ | Ghi Assignment nguyên tử |
| Assignment | Release/Reassign | Quyền, lý do, trạng thái hợp lệ | Kết thúc Assignment cũ, tạo liên kết thay thế |
| Trip | MarkArrived | Trip DRIVER_EN_ROUTE; actor hợp lệ | Ghi TripEvent và vị trí nếu có |
| Trip | StartTrip | Trip ARRIVED; điều kiện bắt đầu đạt | Ghi thời gian bắt đầu |
| Trip | CompleteTrip | Trip IN_PROGRESS; dữ liệu hoàn tất hợp lệ | Chốt Trip một lần; phát sự kiện tính cước |
| Trip | ReportNoShow | Điều kiện chờ/bằng chứng được duyệt | Mở luồng no-show |
| Payment | InitiatePayment | Trip/fare hợp lệ; amount/currency hợp lệ | Tạo PaymentTransaction |
| Payment | ProviderCallback | Chữ ký hợp lệ, chống replay, đúng reference | Cập nhật một lần; phát sự kiện tài chính |
| Payment | Refund | Quyền duyệt, giao dịch gốc, hạn mức | Tạo transaction hoàn tiền riêng |
| Settlement | CalculateBatch | Nguồn và contract snapshot hợp lệ | Lưu kết quả tính/phiên bản |
| Settlement | Approve/Lock/Pay | Maker-checker, quyền, kiểm tra sai lệch | Audit, khóa batch, lưu bằng chứng |

Đây là guard tối thiểu ở mức thiết kế; từng endpoint cần đặc tả chi tiết trong tài liệu API và từng tính năng.

## 7. Lỗi, xung đột và nhất quán

### 7.1. Lỗi chuyển trạng thái
- Không cho phép chuyển trạng thái ngoài state machine đã duyệt.
- Không tự sửa trạng thái để “cho chạy tiếp”.
- Trả lỗi có mã ổn định, thông điệp an toàn và correlation_id.
- Ghi nhận thao tác bị từ chối nếu nhạy cảm hoặc cần kiểm toán.

### 7.2. Đồng thời và tranh chấp tài nguyên
- Accept Offer phải có atomic compare-and-set/transaction.
- Ràng buộc dữ liệu bảo đảm tối đa một Assignment hiệu lực cho mỗi Booking.
- Nhiều tài xế cùng chấp nhận: chỉ một request thắng; request còn lại nhận conflict.
- Khóa Driver/Vehicle phải có thứ tự nhất quán để hạn chế deadlock.
- Không dựa riêng vào khóa giao diện hoặc cache để bảo đảm tính duy nhất.

### 7.3. Sự kiện và tích hợp
- Sự kiện phát ra sau khi transaction nghiệp vụ commit; cân nhắc transactional outbox.
- Consumer phải idempotent vì message có thể giao nhiều lần.
- Lưu event_id, aggregate_id, aggregate_version, event_type, occurred_at, recorded_at, correlation_id.
- Xử lý sự kiện sai thứ tự bằng version/sequence và quy tắc nghiệp vụ.
- Khi chưa rõ kết quả thanh toán, dùng reconciliation; không tự kết luận FAILED.

## 8. Quyết định cần chủ sở hữu xác nhận

| ID | Câu hỏi cần quyết định | Owner đề xuất | Trạng thái |
|---|---|---|---|
| DEC-SM-01 | Booking dùng CONFIRMED hay REQUESTED sau khi gửi? | Vận hành/Booking PO | TBD |
| DEC-SM-02 | QUOTED là state Booking hay Quote độc lập? | Pricing/Booking PO | TBD |
| DEC-SM-03 | SEARCHING là state Booking hay suy ra từ DispatchRun RUNNING? | Kiến trúc/Dispatch PO | TBD |
| DEC-SM-04 | UNFULFILLED là state kết thúc hay mã kết quả đóng yêu cầu? | Vận hành/CSKH | TBD |
| DEC-SM-05 | Assignment có PENDING hay tạo trực tiếp ACTIVE khi nhận Offer? | Dispatch/Architecture | TBD |
| DEC-SM-06 | Tên Trip chuẩn là IN_PROGRESS hay IN_TRIP? | Vận hành/Driver App | TBD |
| DEC-SM-07 | Incident là Trip state, Incident Case riêng hay kết hợp? | Vận hành/An toàn | TBD |
| DEC-SM-08 | Payment dùng SUCCESS hay SUCCEEDED; refund là transaction riêng hay state? | Tài chính/Payment | TBD |
| DEC-SM-09 | Điều kiện, thời gian chờ và bằng chứng NO_SHOW? | Vận hành/CSKH/Pháp chế | TBD |
| DEC-SM-10 | Quy tắc hủy/đổi sau từng mốc Booking, Offer, Assignment, Trip? | Vận hành/CSKH/Tài chính | TBD |
| DEC-SM-11 | Driver/Vehicle xử lý RESERVED, mất GPS, mất mạng thế nào? | Fleet/IT | TBD |
| DEC-SM-12 | Timeout, retry, số attempt và điều kiện ESCALATED? | Vận hành | TBD |
| DEC-SM-13 | Maker-checker cho APPROVE/LOCK/PAID? | Tài chính/Phân quyền | TBD |

Các quyết định phải ghi vào tài liệu 15 — Sổ đăng ký quyết định và cập nhật ngược vào SRS/API/Data Dictionary/Test Catalog.

## 9. Ma trận kiểm thử bổ sung

| Mã test | Đối tượng | Kịch bản | Kết quả mong đợi |
|---|---|---|---|
| SM-001 | Booking | Submit thiếu trường bắt buộc | Từ chối, không chuyển state |
| SM-002 | Booking | Không có ứng viên | Booking còn tồn tại; DispatchRun ghi NO_CANDIDATE/ngoại lệ |
| SM-003 | Booking | Hủy ở trạng thái không cho phép | Từ chối theo guard |
| SM-004 | Offer | Chấp nhận Offer hết hạn | Từ chối server-side |
| SM-005 | Offer | Hai tài xế chấp nhận đồng thời | Chỉ một request thắng |
| SM-006 | Assignment | Tạo Assignment thứ hai đang hiệu lực | DB/service chặn |
| SM-007 | Assignment | Reassign | Assignment cũ có dấu vết; Assignment mới có ID riêng |
| SM-008 | Trip | Bắt đầu khi chưa ARRIVED | Từ chối |
| SM-009 | Trip | CompleteTrip nhiều lần | Một lần hoàn tất và một lần phát sinh nghiệp vụ tài chính |
| SM-010 | Trip | Sự kiện offline gửi lại | Deduplicate; hiển thị kết quả đồng bộ |
| SM-011 | Trip | Sự kiện đến sai thứ tự | Không làm hỏng state; trả conflict/review |
| SM-012 | Payment | Callback lặp | Một lần đổi trạng thái |
| SM-013 | Payment | Timeout, provider chưa rõ kết quả | Reconciliation required, không thu tiền lặp |
| SM-014 | Payment | Refund | Transaction riêng, liên kết transaction gốc |
| SM-015 | Settlement | Approve/Lock không đủ quyền | Từ chối và audit |
| SM-016 | Settlement | Sửa batch LOCKED | Từ chối; yêu cầu adjustment |
| SM-017 | Driver/Vehicle | GPS cũ nhưng availability còn AVAILABLE | Không tự đổi OFFLINE |
| SM-018 | Cross-module | Event phát lặp | Consumer idempotent, không nhân đôi tác động |

Các test bổ sung cho E2E-001…E2E-020 tại tài liệu 25; không thay thế kiểm thử chi tiết của từng Module.

## 10. Điều kiện thông qua 00B

1. Vận hành/CSKH xác nhận Booking, hủy, no-show và ngoại lệ.
2. Dispatch/Fleet xác nhận Offer, Assignment, trạng thái tài xế/xe và điều kiện phân công.
3. Đội tài xế xác nhận mốc Trip, offline và incident.
4. Tài chính xác nhận Payment, hoàn tiền, Ledger và Settlement.
5. Kiến trúc xác nhận aggregate ownership, transaction boundary, concurrency và event contract.
6. BA/QA cập nhật SRS, API, Data Dictionary, Traceability và Test Catalog.
7. Các quyết định TBD có owner, hạn chốt và bằng chứng phê duyệt.

**Kết luận:** 00B đề xuất tách rõ vòng đời Booking – DispatchRun – Offer – Assignment – Trip – Payment – Settlement. Đây là điều kiện nền để thiết kế API và dữ liệu nhất quán. Sau khi chốt các quyết định trạng thái, bước tiếp theo là lập Module Catalog và phân rã Booking thành các Feature Spec độc lập.
