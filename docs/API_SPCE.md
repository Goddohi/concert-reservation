# 콘서트 예약 서비스 API 명세서

## API Specs

### 1. 유저 대기열 토큰
#### 1-1. 대기열 추가
- 서비스를 이용할 토큰을 발급
- 토큰은 유저의 UUID 와 해당 유저의 대기열을 관리할 수 있는 정보 ( 대기 순서 or 잔여 시간 등 ) 를 포함
- 이후 모든 API 는 위 토큰을 이용해 대기열 검증을 통과해야 이용

**Request**
```
POST /api/queue/token
{ "userId": "uuid" }
- 추후 OAuth/Jwt인증 추가시 Authorization: Bearer <token>로 대체 
```
**Response**
```
{ 
    "userId": "uuid",
    "token": "string",
    "queuePosition": 182,
    "estimatedWaitTime": 120,   // seconds
    "status": "WAITING"         // WAITING | READY | EXPIRED
}
```


#### 1-2. 대기열 상태 조회
- 발급받은 Queue-Token 을 이용하여 현재 대기열 상태를 확인
- 현재 순번, 예상 대기시간, 상태(WAITING, READY, ACTIVE, EXPIRED)를 반환
- 클라이언트는 이 API를 폴링하여 자신의 대기 상태를 확인

**Request**
```
GET /api/queue/status
// Header
Queue-Token: <issued_token>

```

```

// Response
{
  "queuePosition": 5,
  "estimatedWaitTime": 60,  // seconds
  "status": "WAITING"       // WAITING | READY | ACTIVE | EXPIRED
}
```

### 2. 예약 가능 날짜 / 좌석 API
#### 2-1. 콘서트 조회 (페이징)
- 여러 콘서트 목록을 페이징 형태로 조회합니다.
**Request**
```
GET /api/concerts?page=0&size=10
Accept: application/json
```

**Response**
```
{
 "page": 0,
 "size": 10,
 "totalElements": 123,
 "totalPages": 13,
 "content": [
   {
     "concertId": 10421,
     "title": "Gomdol Live 2025",
     "venue": "서울공연장1홀",
     "runtimeMin": 180,
     "cast": "곰돌",
     "startDt": "2025-09-10",
     "endDt": "2025-09-12"
   },
   {
     "concertId": 10423,
     "title": "Rock Night",
     "venue": "제주도공연장",
     "runtimeMin": 120,
     "cast": "관식이, 돌밤이",
     "startDt": "2025-10-01",
     "endDt": "2025-10-02"
   }
 ]
}
```

#### 2-2. 콘서트를 선택 시 공연 스케줄 조회
- 특정 콘서트의 공연 일정을 조회합니다.
- **동시성 고려**: 목록에서는 정확한 잔여 좌석 수를 제공하지 않습니다. 대신 오픈 여부만 제공합니다. 상세 좌석은 (2-3)에서 조회.

**Request**
```
GET /api/concerts/{concertId}/schedules
Accept: application/json
```

**Response**
```
{
"concertId": "1231",
"schedules": [
  {
    "scheduleId": "12378",
    "startAt": "2025-09-29T19:00:00+09:00",
    "openAt" : "2025-09-01T19:00:00+09:00",
    "status": "ON_SALE"           
  },
  {
    "scheduleId": "12379",
    "startAt": "2025-09-30T19:00:00+09:00",
    "openAt" : "2025-09-01T19:00:00+09:00",
    "status": "ON_SALE"
  }
]
}
```

### 3. 예약 가능한 스케줄의 남은 좌석 정보 조회
- 특정 스케줄에서 예약 가능한 좌석 현황을 조회합니다.
- **대기열 필수**: 이 API는 Queue-Token 헤더 검증

**Request**
```
GET /api/concerts/{concertId}/schedules/{scheduleId}/seats
Queue-Token: <issued_token>
Accept: application/json
```

**Response**
```
{
 "concertId": "12342",
 "scheduleId": "14232",
 "totalSeats": 50,
 "availableCount": 34,
 "seats": [
  { "seatNo": 1 , "grade" : 'SKYLOUNGE'},
  { "seatNo": 2 , "grade" : 'A2'},
  { "seatNo": 5 , "grade" : 'C3'}
 ]
}
```

### 4. 좌석 예약 요청 API
```
POST /api/concerts/{concertId}/schedules/{scheduleId}/holds
Queue-Token: <issued_token>
Content-Type: application/json

{
  "seats": [ 12, 13, 14 ]
}
```

```
201 Created
{
  "reservation_id": "123124",
  "concertId": "10421",
  "scheduleId": "12378",
  "seats": [
    { "seatNo": 12 , "grade" : 'B2', "price" : 130000},
    { "seatNo": 13 , "grade" : 'B2', "price" : 130000},
    { "seatNo": 14 , "grade" : 'B2', "price" : 130000}
  ],
  
  "heldUntil": "2025-09-07T19:05:00+09:00"
}
```
```

409 Conflict
{
  "code": "SEAT_CONFLICT",
  "message": "이미 다른 고객님이 선점한 좌석입니다."
}
```

- 날짜와 좌석 정보를 입력받아 좌석을 예약 처리하는 API 를 작성합니다.
- 좌석 예약과 동시에 해당 좌석은 그 유저에게 약 5분간 임시 배정됩니다. ( 시간은 정책에 따라 자율적으로 정의합니다. )
- 만약 배정 시간 내에 결제가 완료되지 않는다면 좌석에 대한 임시 배정은 해제되어야 하며 다른 사용자는 예약할 수 없어야 한다.

### 5. 잔액 충전 / 조회 API
- 사용자 식별자 및 충전할 금액을 받아 잔액을 충전
- 사용자 식별자를 통해 해당 사용자의 잔액을 조회

#### 5-1. 잔액 충전
**Request**
```
POST /api/balance/charge
Content-Type: application/json

{ "userId": "uuid", "amount": 100000 }
```

**Response**
```
{ "balance": 150000 }
```

---

#### 5-2. 잔액 조회

**Request**
```
GET /api/balance?userId=uuid
```

**Response**
```
{ "balance": 150000 }
```
- uuid가 없거나 지갑이 없을경우 0원으로 반환

**Response**
```
{ "balance": 0 }
```

### 6. 결제 API
- 결제 처리하고 결제 내역을 생성하는 API 를 작성합니다.
- 결제가 완료되면 해당 좌석의 소유권을 유저에게 배정하고 대기열 토큰을 만료시킵니다.

```
POST /api/payments
Queue-Token: <issued_token>
Content-Type: application/json
{
  "userId": "uuid",
  "reservation_id": "123124",
  "amount": 150000,
  "methods": [
    { "type": "POINT", "amount": 50000 },
    { "type": "CARD", "amount": 100000, "provider": "HYUNDAI_CARD" }
  ]
}
```

```
{
  "paymentId": "123124",
  "reservation_id": "123124",
  "userId": "uuid",
  "amount": 150000,
  "status": "COMPLETED",
  "createdAt": "2025-09-07T19:01:12+09:00",
  "methods": [
    { "type": "POINT", "amount": 50000 },
    { "type": "CARD", "amount": 100000, "provider": "HYUNDAI_CARD" }
  ]
}
```
**Response – 실패 (잔액 부족)**
```
402 Payment Required
{
  "code": "INSUFFICIENT_FUNDS",
  "message": "결제 실패: 잔액부족"
}
```