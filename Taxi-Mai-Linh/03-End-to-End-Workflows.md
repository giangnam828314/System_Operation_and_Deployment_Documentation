# 03 — CÁC QUY TRÌNH NGHIỆP VỤ ĐẦU CUỐI

## 1. Đặt xe → Điều phối → Chuyến đi → Thanh toán

```mermaid
flowchart TD
 A["Khách hàng / Tổng đài / Ứng dụng / Web / API"] --> B["Tạo booking"]
 B --> C{"Dữ liệu hợp lệ?"}
 C -- Không --> C1["Báo lỗi trường dữ liệu; giữ nội dung đã nhập"]
 C1 --> B
 C -- Có --> D["CONFIRMED / READY_FOR_DISPATCH"]
 D --> E["Tìm ứng viên điều phối"]
 E --> F{"Có ứng viên đủ điều kiện?"}
 F -- Không --> G["Hàng đợi ngoại lệ / Giám sát"]
 G --> E
 F -- Có --> H["Gửi lời mời tài xế"]
 H --> I{"Chấp nhận trước khi hết hạn?"}
 I -- Không --> E
 I -- Có --> J["ASSIGNED"]
 J --> K["EN_ROUTE_TO_PICKUP"]
 K --> L["ARRIVED"]
 L --> M["IN_TRIP"]
 M --> N["COMPLETED"]
 N --> O["Chốt cước"]
 O --> P["Thanh toán / Sổ cái"]
 P --> Q["Báo cáo / CSKH / Đối soát"]
```

## 2. Booking tức thời — Tổng đài

1. Agent nhận cuộc gọi, tìm khách theo số liên hệ, tạo hồ sơ tối thiểu nếu chưa có.
2. Nhập pickup; chọn geocoding suggestion; xác nhận pin và hướng dẫn đón.
3. Nhập destination và ordered stop list nếu được phép.
4. Chọn service/vehicle type, thời gian, hành khách, hành lý, ghi chú.
5. Validation và duplicate warning; xác nhận booking.
6. Booking vào dispatch queue.
7. No candidate/offer timeout: booking vẫn tồn tại và vào exception queue.

## 3. Booking đặt trước

```mermaid
stateDiagram-v2
 [*] --> DRAFT
 DRAFT --> SCHEDULED: Validate scheduled time
 SCHEDULED --> READY_FOR_DISPATCH: Lead-time reached
 READY_FOR_DISPATCH --> SEARCHING
 SEARCHING --> ASSIGNED: Driver accepts
 SEARCHING --> EXCEPTION: No eligible candidate
 EXCEPTION --> SEARCHING: Retry / manual action
 SCHEDULED --> CANCELLED: Cancellation policy
 ASSIGNED --> CANCELLED: If policy permits
```

Lead-time, cutoff và xử lý booking không có xe là TBD. Không tự hủy booking khi thiếu nguồn lực.

## 4. Vòng đời lời mời điều phối

```mermaid
sequenceDiagram
 participant E as Dispatch Engine
 participant V as Eligibility Service
 participant A as Assignment Service
 participant D as Driver App
 E->>V: Find candidates
 V-->>E: Eligible ranked candidates
 E->>A: Create time-bound offer
 A-->>D: Push offer
 alt Driver accepts
  D->>A: ACCEPT
  A->>A: Atomic assignment lock
  A-->>E: ASSIGNED
 else Reject or timeout
  D->>A: REJECT / no response
  A->>A: Close offer and record reason
  A-->>E: Try next / exception
 end
```

Ranking weights, DIRECT/PARTNER priority, radius, timeout and retry count are TBD.

## 5. Máy trạng thái chuyến đi

```mermaid
stateDiagram-v2
 [*] --> ASSIGNED
 ASSIGNED --> EN_ROUTE_TO_PICKUP
 EN_ROUTE_TO_PICKUP --> ARRIVED
 ARRIVED --> IN_TRIP
 IN_TRIP --> COMPLETED
 ASSIGNED --> CANCELLED
 EN_ROUTE_TO_PICKUP --> CANCELLED
 ARRIVED --> NO_SHOW
 ARRIVED --> CANCELLED
 IN_TRIP --> INCIDENT
 INCIDENT --> IN_TRIP: Resume if approved
 INCIDENT --> COMPLETED: End under incident flow
```

Server validates every transition. Events store event ID, actor, server time, device time, source and reason.

## 6. Phân xe lại

- Check booking/trip state and dispatcher permission.
- Require reason (vehicle fault, driver reject, GPS issue, customer request, dispatcher action, other).
- Close prior assignment; never delete it.
- Re-evaluate eligibility and create next offer.
- Recalculate ETA and notify customer/operations according to policy.
- Preserve before/after and actor in audit.

## 7. Quy trình xử lý ngoại lệ

| Case | Evidence | Result |
|---|---|---|
| No-show | Arrival time, wait duration, reason/evidence | NO_SHOW + policy |
| Customer cancel | Actor, time, reason, state | CANCELLED |
| Driver cancel | Driver, reason, time | Re-dispatch/escalate |
| Vehicle failure | Vehicle, GPS, incident report | Reassign |
| SOS | Timestamp, location, trip, acknowledgement | High-severity incident |
| Payment exception | Provider ref, callback, status | Finance queue |

## 8. Đối soát đối tác

```mermaid
flowchart LR
 A["Period close"] --> B["Partner + contract scope"]
 B --> C["Collect eligible completed trips"]
 C --> D["Apply policy snapshot"]
 D --> E["Gross / deductions / share / net"]
 E --> F["Finance review"]
 F --> G{"Variance?"}
 G -- Yes --> H["Evidence and adjustment"]
 H --> F
 G -- No --> I["Approve"]
 I --> J["Lock batch"]
 J --> K["Record payout"]
 K --> L["PAID"]
```

After LOCKED, use a linked adjustment batch instead of changing historical ledger lines.

## 9. Các sự kiện kiểm toán tối thiểu

BOOKING_CREATED/CONFIRMED/UPDATED/CANCELLED; DISPATCH_SEARCH_STARTED; DRIVER_OFFERED/ACCEPTED/REJECTED/EXPIRED; VEHICLE_ASSIGNED/REASSIGNED; DRIVER_EN_ROUTE/ARRIVED; TRIP_STARTED/COMPLETED/CANCELLED/NO_SHOW; FARE_CALCULATED; PAYMENT_INITIATED/SUCCEEDED/FAILED/REFUNDED; INCIDENT_CREATED/ACKNOWLEDGED/RESOLVED; SETTLEMENT_CALCULATED/APPROVED/LOCKED/PAID.
