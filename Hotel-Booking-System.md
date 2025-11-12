# 🏨 Hotel Booking System Design (30분 인터뷰용)

> **Booking.com/Airbnb 스타일 호텔 예약 시스템**

---

## 1️⃣ 문제 정의 (2분)

### 요구사항
- ✅ 호텔/객실 검색 (날짜, 위치, 인원)
- ✅ 실시간 객실 가용성 확인
- ✅ 예약 생성 (payment 제외)
- ✅ 예약 수정/취소
- ✅ 오버부킹 방지
- ✅ 동시성 제어 (같은 방 중복 예약 방지)

### 비기능 요구사항
- **성능**: 검색 < 500ms, 예약 < 2초
- **일관성**: 오버부킹 절대 불가 (Strong consistency)
- **가용성**: 99.9% uptime
- **확장성**: 100K+ 호텔, 1M+ 객실
- **동시 예약**: 초당 1,000+ 예약 처리

---

## 2️⃣ 규모 추정 (3분)

```
📊 Scale
├── 호텔: 100,000
├── 평균 객실/호텔: 50
├── 총 객실: 5M
├── DAU: 5M
├── 일일 검색: 50M (10 searches/user)
├── 일일 예약: 500K (10% conversion)
└── 피크 QPS: 
    ├── 검색: 10,000/sec
    └── 예약: 1,000/sec

💾 Storage
├── 호텔 정보: 100K × 10KB = 1GB
├── 객실 정보: 5M × 5KB = 25GB
├── 예약 데이터: 500K/day × 365 × 2KB = 365GB/year
├── 사용자 데이터: 10M × 2KB = 20GB
└── 총: ~500GB (첫 해)

📈 Bandwidth
├── 검색: 10K QPS × 50KB = 500MB/s = 4Gbps
├── 예약: 1K QPS × 5KB = 5MB/s = 40Mbps
└── 총: ~5Gbps (peak)

💰 비용
└── 월 ~$20,000 (compute + storage + DB)
```

---

## 3️⃣ 핵심 엔티티 (3분)

```typescript
// Hotel - 호텔
{
  hotelId: string,
  name: string,
  description: string,
  
  // 위치
  address: string,
  city: string,
  country: string,
  latitude: float,
  longitude: float,
  
  // 정보
  starRating: number,      // 1-5
  amenities: string[],     // ["wifi", "pool", "parking"]
  images: string[],
  
  // 정책
  checkInTime: time,       // "14:00"
  checkOutTime: time,      // "11:00"
  cancellationPolicy: string,
  
  createdAt: timestamp,
  updatedAt: timestamp
}

// RoomType - 객실 타입
{
  roomTypeId: string,
  hotelId: string,
  name: string,            // "Deluxe King Room"
  description: string,
  
  // 용량
  maxOccupancy: number,
  bedType: string,         // "King", "Queen", "Twin"
  
  // 가격
  basePrice: decimal,
  currency: string,
  
  // 편의시설
  amenities: string[],
  size: number,            // sqm
  images: string[],
  
  // 메타
  totalRooms: number,      // 이 타입의 총 객실 수
  createdAt: timestamp
}

// Room - 실제 객실 (물리적)
{
  roomId: string,
  roomTypeId: string,
  hotelId: string,
  roomNumber: string,      // "101", "102"
  floor: number,
  status: 'AVAILABLE' | 'MAINTENANCE' | 'OCCUPIED',
  createdAt: timestamp
}

// Booking - 예약
{
  bookingId: string,
  userId: string,
  hotelId: string,
  roomTypeId: string,
  roomId: string | null,   // 체크인 시 할당
  
  // 날짜
  checkInDate: date,       // "2025-11-15"
  checkOutDate: date,      // "2025-11-17"
  nights: number,          // 2
  
  // 게스트 정보
  guestName: string,
  guestEmail: string,
  guestPhone: string,
  numberOfGuests: number,
  
  // 가격
  totalPrice: decimal,
  priceBreakdown: {
    roomPrice: decimal,
    taxes: decimal,
    fees: decimal
  },
  
  // 상태
  status: 'PENDING' | 'CONFIRMED' | 'CANCELLED' | 'COMPLETED',
  
  // 메타
  createdAt: timestamp,
  updatedAt: timestamp,
  cancelledAt: timestamp | null
}

// RoomAvailability - 객실 가용성 (핵심!)
{
  availabilityId: string,
  hotelId: string,
  roomTypeId: string,
  date: date,              // "2025-11-15"
  
  // 재고
  totalRooms: number,      // 10
  availableRooms: number,  // 7
  bookedRooms: number,     // 3
  
  // 가격 (날짜별 변동 가능)
  price: decimal,
  
  updatedAt: timestamp
}

// Inventory - 재고 관리
{
  inventoryId: string,
  roomTypeId: string,
  date: date,
  allocated: number,       // 예약된 수
  blocked: number,         // 관리자가 막은 수
  available: number        // 가용 수 (computed)
}
```

**샤딩 전략**: 
- Hotels/Rooms: hotelId 기반
- Bookings: userId 또는 hotelId (선택 필요)
- Availability: hotelId + date 복합 키

---

## 4️⃣ API 설계 (2분)

```http
# 호텔 검색
GET /api/v1/hotels/search?city=Seoul&checkIn=2025-11-15&checkOut=2025-11-17&guests=2
→ 200 OK
{
  "hotels": [
    {
      "hotelId": "hotel_123",
      "name": "Grand Hotel Seoul",
      "starRating": 5,
      "location": {...},
      "availableRoomTypes": [
        {
          "roomTypeId": "rt_456",
          "name": "Deluxe King",
          "availableRooms": 5,
          "price": 150.00,
          "maxOccupancy": 2
        }
      ],
      "lowestPrice": 150.00
    }
  ],
  "total": 156
}

# 객실 타입 상세 + 가용성 확인
GET /api/v1/hotels/{hotelId}/room-types/{roomTypeId}/availability?checkIn=2025-11-15&checkOut=2025-11-17
→ 200 OK
{
  "roomTypeId": "rt_456",
  "name": "Deluxe King",
  "description": "...",
  "availability": [
    { "date": "2025-11-15", "available": 5, "price": 150.00 },
    { "date": "2025-11-16", "available": 5, "price": 180.00 }
  ],
  "totalPrice": 330.00
}

# 예약 생성 (중요!)
POST /api/v1/bookings
{
  "hotelId": "hotel_123",
  "roomTypeId": "rt_456",
  "checkInDate": "2025-11-15",
  "checkOutDate": "2025-11-17",
  "guestInfo": {
    "name": "John Doe",
    "email": "john@example.com",
    "phone": "+1-555-0100"
  },
  "numberOfGuests": 2
}
→ 201 Created
{
  "bookingId": "booking_789",
  "status": "CONFIRMED",
  "totalPrice": 330.00,
  "confirmationCode": "ABC123XYZ"
}

# 예약 조회
GET /api/v1/bookings/{bookingId}
→ 200 OK

# 예약 취소
POST /api/v1/bookings/{bookingId}/cancel
{
  "reason": "Change of plans"
}
→ 200 OK
{
  "bookingId": "booking_789",
  "status": "CANCELLED",
  "refundAmount": 330.00
}

# 내 예약 목록
GET /api/v1/users/me/bookings?status=CONFIRMED
→ 200 OK
{
  "bookings": [...]
}
```

---

## 5️⃣ 시스템 아키텍처 (10분)

### High-Level Design

```
┌─────────────┐
│   Clients   │ Web/Mobile App
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────┐
│     CDN     │ Static assets
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ API Gateway │ Auth, Rate Limit
└──────┬──────┘
       │
       ▼
┌────────────────────────────────────┐
│      Application Services          │
│  ┌──────────┐  ┌──────────────┐  │
│  │ Search   │  │  Booking     │  │
│  │ Service  │  │  Service     │  │
│  └────┬─────┘  └──────┬───────┘  │
│       │               │           │
│  ┌────┴─────┐  ┌──────┴───────┐  │
│  │Inventory │  │  Payment     │  │
│  │ Service  │  │  Service     │  │
│  └──────────┘  └──────────────┘  │
└────────────────────────────────────┘
       │               │
       ▼               ▼
┌─────────────┐ ┌─────────────┐
│ PostgreSQL  │ │ Redis       │
│  (ACID)     │ │ (Cache)     │
└─────────────┘ └─────────────┘
       │
       ▼
┌─────────────┐
│Elasticsearch│ Hotel/Room search
└─────────────┘
```

### 상세 아키텍처

```
┌───────────────────────────────────────────────────┐
│                 Client Layer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Web    │  │  Mobile  │  │  Hotel   │       │
│  │   App    │  │   App    │  │  Portal  │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└────────────────────────┬──────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────┐
│              Load Balancer (ALB)                   │
└────────────────────────┬──────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────┐
│                API Gateway                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │   Auth   │  │   Rate   │  │  Router  │       │
│  │  (JWT)   │  │ Limiter  │  │          │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└────────────────────────┬──────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Search     │ │   Booking    │ │  Inventory   │
│   Service    │ │   Service    │ │   Service    │
│              │ │              │ │              │
│ - Filter     │ │ - Reserve    │ │ - Check      │
│ - Sort       │ │ - Confirm    │ │ - Lock       │
│ - Cache      │ │ - Cancel     │ │ - Release    │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       │                │                │
       ▼                ▼                ▼
┌────────────────────────────────────────────────┐
│              Cache Layer (Redis)               │
│  ┌──────────────┐  ┌──────────────┐          │
│  │ Search Cache │  │ Inventory    │          │
│  │ (5 min)      │  │ Lock         │          │
│  └──────────────┘  └──────────────┘          │
└────────────────────────┬───────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │Elasticsearch │ │   Message    │
│   Primary    │ │   Cluster    │ │    Queue     │
│              │ │              │ │  (RabbitMQ)  │
│ ACID + Locks │ │ Search Index │ │              │
└──────┬───────┘ └──────────────┘ └──────┬───────┘
       │                                  │
       ▼                                  ▼
┌──────────────┐                  ┌──────────────┐
│ Read Replica │                  │ Notification │
│   (Slave)    │                  │   Service    │
└──────────────┘                  └──────────────┘
```

---

### 주요 컴포넌트 상세

#### 1. Search Service

```go
type SearchService struct {
    es    *elasticsearch.Client
    cache *Redis
    db    *PostgreSQL
}

func (ss *SearchService) SearchHotels(req *SearchRequest) ([]*Hotel, error) {
    // 1. Build cache key
    cacheKey := fmt.Sprintf("search:%s:%s:%s:%d", 
        req.City, req.CheckIn, req.CheckOut, req.Guests)
    
    // 2. Check cache
    if cached := ss.cache.Get(cacheKey); cached != nil {
        return cached, nil
    }
    
    // 3. Search Elasticsearch
    query := ss.buildSearchQuery(req)
    hotels := ss.es.Search(query)
    
    // 4. Get availability for each hotel (parallel)
    var wg sync.WaitGroup
    for _, hotel := range hotels {
        wg.Add(1)
        go func(h *Hotel) {
            defer wg.Done()
            h.AvailableRoomTypes = ss.getAvailableRoomTypes(
                h.HotelId, req.CheckIn, req.CheckOut, req.Guests)
        }(hotel)
    }
    wg.Wait()
    
    // 5. Filter out hotels with no availability
    hotels = ss.filterAvailable(hotels)
    
    // 6. Sort by price/rating
    sort.Slice(hotels, func(i, j int) bool {
        return hotels[i].LowestPrice < hotels[j].LowestPrice
    })
    
    // 7. Cache results (5 min)
    ss.cache.Set(cacheKey, hotels, 5*time.Minute)
    
    return hotels, nil
}

func (ss *SearchService) buildSearchQuery(req *SearchRequest) map[string]interface{} {
    return map[string]interface{}{
        "query": map[string]interface{}{
            "bool": map[string]interface{}{
                "must": []map[string]interface{}{
                    {
                        "match": map[string]interface{}{
                            "city": req.City,
                        },
                    },
                },
                "filter": []map[string]interface{}{
                    {
                        "geo_distance": map[string]interface{}{
                            "distance": "50km",
                            "location": map[string]float64{
                                "lat": req.Latitude,
                                "lon": req.Longitude,
                            },
                        },
                    },
                    {
                        "range": map[string]interface{}{
                            "starRating": map[string]int{
                                "gte": req.MinStarRating,
                            },
                        },
                    },
                },
            },
        },
        "sort": []map[string]interface{}{
            {
                "_geo_distance": map[string]interface{}{
                    "location": map[string]float64{
                        "lat": req.Latitude,
                        "lon": req.Longitude,
                    },
                    "order": "asc",
                },
            },
        },
    }
}
```

#### 2. Inventory Service (핵심!)

**가용성 체크 전략**:

```go
type InventoryService struct {
    db    *PostgreSQL
    cache *Redis
}

// 날짜 범위의 가용성 확인
func (is *InventoryService) CheckAvailability(
    roomTypeId string,
    checkIn time.Time,
    checkOut time.Time,
) (bool, error) {
    // 1. Generate date range
    dates := generateDateRange(checkIn, checkOut)
    
    // 2. Check each date (체크아웃 날짜는 제외)
    for _, date := range dates[:len(dates)-1] {
        available, err := is.getAvailableRooms(roomTypeId, date)
        if err != nil {
            return false, err
        }
        
        // 하나라도 없으면 예약 불가
        if available <= 0 {
            return false, nil
        }
    }
    
    return true, nil
}

// 특정 날짜의 가용 객실 수 조회
func (is *InventoryService) getAvailableRooms(
    roomTypeId string,
    date time.Time,
) (int, error) {
    // 1. Try cache first
    cacheKey := fmt.Sprintf("avail:%s:%s", roomTypeId, date.Format("2006-01-02"))
    if cached := is.cache.Get(cacheKey); cached != nil {
        return cached.(int), nil
    }
    
    // 2. Query database
    var availability struct {
        TotalRooms     int
        BookedRooms    int
        BlockedRooms   int
    }
    
    err := is.db.QueryRow(`
        SELECT 
            rt.total_rooms,
            COALESCE(COUNT(b.booking_id), 0) as booked_rooms,
            COALESCE(i.blocked, 0) as blocked_rooms
        FROM room_types rt
        LEFT JOIN bookings b ON b.room_type_id = rt.room_type_id
            AND b.status IN ('CONFIRMED', 'PENDING')
            AND $2 >= b.check_in_date
            AND $2 < b.check_out_date
        LEFT JOIN inventory i ON i.room_type_id = rt.room_type_id
            AND i.date = $2
        WHERE rt.room_type_id = $1
        GROUP BY rt.room_type_id, i.blocked
    `, roomTypeId, date).Scan(&availability.TotalRooms, 
                             &availability.BookedRooms, 
                             &availability.BlockedRooms)
    
    if err != nil {
        return 0, err
    }
    
    // 3. Calculate available
    available := availability.TotalRooms - 
                 availability.BookedRooms - 
                 availability.BlockedRooms
    
    // 4. Cache it (5 min)
    is.cache.Set(cacheKey, available, 5*time.Minute)
    
    return available, nil
}

// 예약 시 재고 차감 (중요!)
func (is *InventoryService) Reserve(
    roomTypeId string,
    checkIn time.Time,
    checkOut time.Time,
) error {
    // 모든 날짜의 캐시 무효화
    dates := generateDateRange(checkIn, checkOut)
    for _, date := range dates[:len(dates)-1] {
        cacheKey := fmt.Sprintf("avail:%s:%s", roomTypeId, date.Format("2006-01-02"))
        is.cache.Delete(cacheKey)
    }
    
    // DB는 booking 생성 시 자동 차감됨
    return nil
}
```

#### 3. Booking Service (동시성 제어!)

**문제**: 여러 사용자가 동시에 마지막 남은 방을 예약하려고 할 때

**해결책**: Pessimistic Locking + Transaction ✅

```go
type BookingService struct {
    db        *PostgreSQL
    inventory *InventoryService
    payment   *PaymentService
}

func (bs *BookingService) CreateBooking(req *BookingRequest) (*Booking, error) {
    // Start transaction
    tx, err := bs.db.Begin()
    if err != nil {
        return nil, err
    }
    defer tx.Rollback()
    
    // 1. Lock room type for update (pessimistic lock)
    // 다른 트랜잭션은 이 락이 풀릴 때까지 대기
    var roomType RoomType
    err = tx.QueryRow(`
        SELECT room_type_id, total_rooms, base_price
        FROM room_types
        WHERE room_type_id = $1
        FOR UPDATE
    `, req.RoomTypeId).Scan(&roomType.RoomTypeId, 
                            &roomType.TotalRooms, 
                            &roomType.BasePrice)
    if err != nil {
        return nil, err
    }
    
    // 2. Check availability (within transaction)
    available := bs.checkAvailabilityInTx(tx, req.RoomTypeId, req.CheckIn, req.CheckOut)
    if !available {
        return nil, ErrNoAvailability
    }
    
    // 3. Calculate price
    totalPrice := bs.calculatePrice(req.RoomTypeId, req.CheckIn, req.CheckOut)
    
    // 4. Create booking
    booking := &Booking{
        BookingId:     generateId(),
        UserId:        req.UserId,
        HotelId:       req.HotelId,
        RoomTypeId:    req.RoomTypeId,
        CheckInDate:   req.CheckIn,
        CheckOutDate:  req.CheckOut,
        Nights:        calculateNights(req.CheckIn, req.CheckOut),
        GuestName:     req.GuestInfo.Name,
        GuestEmail:    req.GuestInfo.Email,
        NumberOfGuests: req.NumberOfGuests,
        TotalPrice:    totalPrice,
        Status:        "CONFIRMED",
        CreatedAt:     time.Now(),
    }
    
    // 5. Insert booking
    _, err = tx.Exec(`
        INSERT INTO bookings (booking_id, user_id, hotel_id, room_type_id,
                             check_in_date, check_out_date, nights,
                             guest_name, guest_email, number_of_guests,
                             total_price, status, created_at)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13)
    `, booking.BookingId, booking.UserId, booking.HotelId, booking.RoomTypeId,
       booking.CheckInDate, booking.CheckOutDate, booking.Nights,
       booking.GuestName, booking.GuestEmail, booking.NumberOfGuests,
       booking.TotalPrice, booking.Status, booking.CreatedAt)
    
    if err != nil {
        return nil, err
    }
    
    // 6. Commit transaction
    if err := tx.Commit(); err != nil {
        return nil, err
    }
    
    // 7. Invalidate cache (async)
    go bs.inventory.Reserve(req.RoomTypeId, req.CheckIn, req.CheckOut)
    
    // 8. Send confirmation email (async)
    go bs.sendConfirmationEmail(booking)
    
    return booking, nil
}

// 트랜잭션 내에서 가용성 체크 (락 걸린 상태)
func (bs *BookingService) checkAvailabilityInTx(
    tx *sql.Tx,
    roomTypeId string,
    checkIn time.Time,
    checkOut time.Time,
) bool {
    dates := generateDateRange(checkIn, checkOut)
    
    for _, date := range dates[:len(dates)-1] {
        var available int
        err := tx.QueryRow(`
            SELECT 
                rt.total_rooms - 
                COALESCE(COUNT(b.booking_id), 0) - 
                COALESCE(i.blocked, 0)
            FROM room_types rt
            LEFT JOIN bookings b ON b.room_type_id = rt.room_type_id
                AND b.status IN ('CONFIRMED', 'PENDING')
                AND $2 >= b.check_in_date
                AND $2 < b.check_out_date
            LEFT JOIN inventory i ON i.room_type_id = rt.room_type_id
                AND i.date = $2
            WHERE rt.room_type_id = $1
            GROUP BY rt.room_type_id, rt.total_rooms, i.blocked
        `, roomTypeId, date).Scan(&available)
        
        if err != nil || available <= 0 {
            return false
        }
    }
    
    return true
}

// 예약 취소
func (bs *BookingService) CancelBooking(bookingId string, userId string) error {
    tx, err := bs.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // 1. Get booking (with lock)
    var booking Booking
    err = tx.QueryRow(`
        SELECT booking_id, user_id, status, check_in_date, room_type_id
        FROM bookings
        WHERE booking_id = $1
        FOR UPDATE
    `, bookingId).Scan(&booking.BookingId, &booking.UserId, 
                       &booking.Status, &booking.CheckInDate, &booking.RoomTypeId)
    
    if err != nil {
        return err
    }
    
    // 2. Check ownership
    if booking.UserId != userId {
        return ErrUnauthorized
    }
    
    // 3. Check if cancellable
    if booking.Status != "CONFIRMED" && booking.Status != "PENDING" {
        return ErrCannotCancel
    }
    
    // 4. Check cancellation policy (e.g., 24 hours before check-in)
    if time.Until(booking.CheckInDate) < 24*time.Hour {
        return ErrTooLateToCancel
    }
    
    // 5. Update status
    _, err = tx.Exec(`
        UPDATE bookings
        SET status = 'CANCELLED', cancelled_at = $2
        WHERE booking_id = $1
    `, bookingId, time.Now())
    
    if err != nil {
        return err
    }
    
    // 6. Commit
    if err := tx.Commit(); err != nil {
        return err
    }
    
    // 7. Invalidate cache (재고 다시 늘어남)
    go bs.inventory.Release(booking.RoomTypeId, booking.CheckInDate, booking.CheckOutDate)
    
    // 8. Process refund (async)
    go bs.payment.ProcessRefund(bookingId)
    
    return nil
}
```

**동시성 제어 전략 비교**:

| 전략 | 장점 | 단점 | 선택 |
|------|------|------|------|
| **Pessimistic Lock** | 오버부킹 완전 방지 | Throughput 감소 | ✅ |
| Optimistic Lock | 높은 throughput | 충돌 시 재시도 필요 | ❌ |
| Distributed Lock (Redis) | 확장성 | 복잡도, Redis 의존 | ❌ |

**선택 이유**: 
- 호텔 예약은 **절대** 오버부킹 불가
- Throughput 감소는 수용 가능 (초당 1,000 예약은 충분히 처리)
- 구현 단순, PostgreSQL 기본 기능 활용

---

## 6️⃣ 핵심 플로우 (5분)

### 검색 플로우

```
1. User: "Seoul, 11/15-11/17, 2 guests"
2. Client → API Gateway → Search Service
3. Search Service → Redis (cache check)
4. Cache miss → Elasticsearch (hotels in Seoul)
5. For each hotel (parallel):
   → Inventory Service → Check availability
   → Get price for dates
6. Filter hotels with availability
7. Sort by price
8. Cache results (5 min)
9. Return to client

Timeline: ~500ms
```

### 예약 플로우 (중요!)

```
1. User clicks "Book Now"
2. Client → API Gateway → Booking Service

3. Start DB Transaction
4. Lock room_type row (FOR UPDATE)
   ├── Other concurrent requests WAIT here
   └── Ensures no overbooking

5. Check availability (within transaction)
   ├── Count current bookings
   ├── Check blocked rooms
   └── Calculate available = total - booked - blocked

6. If available > 0:
   ├── Create booking record
   ├── Status: CONFIRMED
   └── Commit transaction
   Else:
   └── Rollback, return "No availability"

7. Lock released → Other transactions proceed

8. Invalidate cache (async)
9. Send confirmation email (async)
10. Return booking details to client

Timeline: ~1.5 seconds
Critical section (lock held): ~500ms
```

**동시 예약 시나리오**:

```
시간축:
T0: User A와 User B가 동시에 마지막 1개 방 예약 시도

T1: User A의 트랜잭션 시작
    └── SELECT ... FOR UPDATE (Lock 획득)

T2: User B의 트랜잭션 시작
    └── SELECT ... FOR UPDATE (Lock 대기)

T3: User A: 가용성 확인 (1개 available)
    User A: Booking 생성
    User A: Commit

T4: User A의 Lock 해제

T5: User B의 Lock 획득
    User B: 가용성 확인 (0개 available)
    User B: Rollback
    User B: Return "No availability"

결과: User A 성공, User B 실패 (정상!)
```

### 취소 플로우

```
1. User clicks "Cancel Booking"
2. Client → API Gateway → Booking Service

3. Start DB Transaction
4. Lock booking row (FOR UPDATE)

5. Check ownership (userId matches)
6. Check status (CONFIRMED or PENDING)
7. Check cancellation policy (>24 hours before)

8. Update status to CANCELLED
9. Commit transaction

10. Invalidate availability cache (재고 복구)
11. Process refund (async)
12. Send cancellation email (async)

Timeline: ~1 second
```

---

## 7️⃣ 주요 설계 결정 (3분)

### 1. 재고 관리 방식: Pre-calculated vs On-demand

**Option A: Pre-calculated Table (선택) ✅**

```sql
CREATE TABLE room_availability (
    availability_id UUID PRIMARY KEY,
    hotel_id UUID,
    room_type_id UUID,
    date DATE,
    total_rooms INT,
    available_rooms INT,
    booked_rooms INT,
    price DECIMAL
);

-- 매일 밤 배치 작업으로 업데이트
-- 또는 예약/취소 시마다 업데이트
```

**장점**:
- 빠른 조회 (1 query)
- 간단한 쿼리

**단점**:
- 동기화 복잡
- 저장 공간 많이 사용
- 데이터 정합성 위험

**Option B: On-demand Calculation (채택) ✅**

```sql
SELECT 
    rt.total_rooms - COUNT(b.booking_id) as available
FROM room_types rt
LEFT JOIN bookings b ON b.room_type_id = rt.room_type_id
    AND b.status = 'CONFIRMED'
    AND $date >= b.check_in_date
    AND $date < b.check_out_date
WHERE rt.room_type_id = $room_type_id
GROUP BY rt.room_type_id, rt.total_rooms;
```

**장점**:
- 항상 정확 (Single source of truth)
- 저장 공간 절약
- 구현 단순

**단점**:
- 조회 시 계산 필요

**최적화**: Redis 캐싱으로 성능 보완

### 2. 동시성 제어: Pessimistic vs Optimistic

**Pessimistic Locking (선택) ✅**

```sql
SELECT * FROM room_types
WHERE room_type_id = $1
FOR UPDATE;
-- 다른 트랜잭션은 대기
```

**장점**:
- 오버부킹 완전 방지
- 충돌 없음
- 구현 단순

**단점**:
- Lock 대기 시간
- Throughput 감소

**Optimistic Locking**

```sql
-- Version field 사용
UPDATE bookings
SET version = version + 1
WHERE booking_id = $1 AND version = $2;
-- 실패 시 재시도
```

**장점**:
- 높은 throughput
- Lock 대기 없음

**단점**:
- 충돌 시 재시도 필요
- 오버부킹 가능성
- 구현 복잡

**결정**: Pessimistic ✅ (호텔 예약은 정확성이 최우선)

### 3. 검색: SQL vs Elasticsearch

**선택**: Elasticsearch ✅

**이유**:
- 지리적 검색 (geo_distance)
- Full-text search (호텔 이름, 설명)
- Faceted search (가격 범위, 별점 필터)
- 빠른 성능
- 확장성

**SQL로는**:
```sql
-- 느림 (100K 호텔에서)
SELECT * FROM hotels
WHERE city = 'Seoul'
  AND ST_Distance(location, $user_location) < 50000
  AND star_rating >= 4
ORDER BY price;
```

**Elasticsearch로**:
```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"city": "Seoul"}},
        {"geo_distance": {
          "distance": "50km",
          "location": {"lat": 37.5, "lon": 127}
        }}
      ]
    }
  }
}
```

### 4. Room 할당: 예약 시 vs 체크인 시

**선택**: 체크인 시 할당 ✅

```
예약 시:
└── roomId = NULL (아직 특정 방 할당 안 됨)
    └── roomTypeId만 지정

체크인 시:
└── 실제 Room 할당 (101호, 102호 등)
```

**이유**:
- 유연성 (방 정비, 업그레이드 가능)
- 관리 용이
- Industry standard

---

## 8️⃣ 확장성 & 최적화 (2분)

### Database Sharding

**전략**: Geographic sharding

```
Shard 1: Hotels in Asia-Pacific
Shard 2: Hotels in Europe
Shard 3: Hotels in Americas

각 Shard:
├── PostgreSQL Primary
├── 2 Read Replicas
└── Redis cache
```

**장점**:
- 대부분의 검색은 single shard
- Region별 독립 확장
- 법적 요구사항 충족 (data residency)

### Caching 전략

```
캐싱 레이어:

L1: CDN (CloudFlare)
├── 호텔 이미지
├── Static assets
└── TTL: 7 days

L2: Redis (Application)
├── 검색 결과 (5분 TTL)
├── 호텔 정보 (1시간 TTL)
├── 가용성 (5분 TTL)
└── 가격 (10분 TTL)

L3: Database Query Cache
└── PostgreSQL shared_buffers

캐시 무효화:
├── 예약 생성 → 가용성 캐시 삭제
├── 예약 취소 → 가용성 캐시 삭제
└── 가격 변경 → 가격 캐시 삭제
```

### Read Scaling

```go
// 읽기는 Replica로
func (s *SearchService) GetHotel(hotelId string) (*Hotel, error) {
    // Use read replica
    return s.readDB.QueryHotel(hotelId)
}

// 쓰기는 Primary로
func (s *BookingService) CreateBooking(req *BookingRequest) (*Booking, error) {
    // Use primary
    return s.primaryDB.InsertBooking(req)
}
```

### 인덱스 최적화

```sql
-- Bookings table
CREATE INDEX idx_bookings_room_type_dates 
ON bookings(room_type_id, check_in_date, check_out_date)
WHERE status IN ('CONFIRMED', 'PENDING');

CREATE INDEX idx_bookings_user 
ON bookings(user_id, created_at DESC);

-- Hotels table
CREATE INDEX idx_hotels_city 
ON hotels(city);

CREATE INDEX idx_hotels_location 
ON hotels USING GIST(location);

-- Room types table
CREATE INDEX idx_room_types_hotel 
ON room_types(hotel_id);
```

---

## 9️⃣ 특수 시나리오 (추가 질문 대비)

### Q1: 피크 시즌 (블랙 프라이데이) 대응?

```
문제:
├── 트래픽 10배 증가
├── 동시 예약 급증
└── 가용성 0인 경우 많음

해결책:

1. Read Scaling
   ├── Read replica 추가 (10 → 20)
   ├── Redis cache 확장
   └── CDN 활용

2. Write Scaling
   ├── Connection pool 증가
   ├── 트랜잭션 타임아웃 단축
   └── Lock timeout 설정

3. Queue System
   ├── 예약 요청을 큐에 넣기
   ├── Worker pool이 순차 처리
   └── 사용자에게 "처리 중" 표시

4. Graceful Degradation
   ├── 캐시 TTL 연장 (5분 → 30분)
   ├── 실시간 가용성 대신 "거의 확실" 표시
   └── Non-critical 기능 비활성화

코드:
func (bs *BookingService) CreateBooking(req *BookingRequest) (*Booking, error) {
    // Peak season: use queue
    if isPeakSeason() {
        jobId := bs.queue.Push(req)
        return &Booking{
            BookingId: jobId,
            Status: "PROCESSING",
        }, nil
    }
    
    // Normal: direct processing
    return bs.createBookingDirect(req)
}
```

### Q2: 오버부킹을 허용해야 한다면? (항공사처럼)

```
전략:
├── Overbooking percentage: 5-10%
├── No-show 예측 모델
└── 보상 정책

구현:
type RoomType struct {
    TotalRooms       int
    OverbookingLimit int  // totalRooms * 1.1
}

func checkAvailability(roomTypeId string, date time.Time) bool {
    booked := getBookedCount(roomTypeId, date)
    limit := getOverbookingLimit(roomTypeId)
    
    return booked < limit  // 10% 오버부킹 허용
}

// 실제 체크인 시
func checkIn(bookingId string) error {
    // 만약 실제 방이 없다면
    if !hasPhysicalRoom() {
        // 1. 업그레이드 제안
        // 2. 다른 호텔 예약 + 교통비
        // 3. 보상금 지급
        return offerCompensation()
    }
    
    return assignRoom(bookingId)
}
```

### Q3: 그룹 예약 (10개 방 동시 예약)?

```
문제:
├── 10개 방을 한 트랜잭션에서 처리
├── 일부만 가능한 경우 어떻게?
└── Lock 시간 길어짐

해결책:

Option A: All-or-Nothing
func CreateGroupBooking(req *GroupBookingRequest) error {
    tx := db.Begin()
    defer tx.Rollback()
    
    // Lock all room types
    for _, roomType := range req.RoomTypes {
        tx.Exec("SELECT * FROM room_types WHERE room_type_id = $1 FOR UPDATE", roomType)
    }
    
    // Check all availability
    for _, roomType := range req.RoomTypes {
        if !checkAvail(roomType) {
            return ErrInsufficientRooms
        }
    }
    
    // Create all bookings
    for _, roomType := range req.RoomTypes {
        tx.InsertBooking(roomType)
    }
    
    tx.Commit()
    return nil
}

Option B: Partial Booking (선택)
func CreateGroupBooking(req *GroupBookingRequest) (*GroupBookingResult, error) {
    result := &GroupBookingResult{
        Requested: req.RoomCount,
        Booked: 0,
        Failed: 0,
    }
    
    // Try to book as many as possible
    for i := 0; i < req.RoomCount; i++ {
        err := createSingleBooking(req)
        if err == nil {
            result.Booked++
        } else {
            result.Failed++
        }
    }
    
    return result, nil
}
```

### Q4: 가격 변동 (Dynamic Pricing)?

```
전략:
├── 수요 기반 가격 조정
├── 날짜별 다른 가격
├── 실시간 경쟁사 모니터링
└── 머신러닝 가격 예측

구현:
type DynamicPricing struct {
    basePrice    float64
    demandFactor float64  // 0.5 - 2.0
    seasonFactor float64  // 0.8 - 1.5
    dayOfWeek    float64  // 0.9 - 1.2 (주말 비쌈)
}

func calculatePrice(roomTypeId string, date time.Time) float64 {
    base := getRoomTypeBasePrice(roomTypeId)
    
    // 가용성 기반
    available := getAvailableRooms(roomTypeId, date)
    total := getTotalRooms(roomTypeId)
    occupancyRate := 1 - float64(available)/float64(total)
    demandFactor := 0.5 + (occupancyRate * 1.5)  // 0.5 ~ 2.0
    
    // 날짜 기반
    seasonFactor := getSeasonFactor(date)
    dayOfWeek := getDayOfWeekFactor(date)
    
    // 최종 가격
    price := base * demandFactor * seasonFactor * dayOfWeek
    
    // 반올림
    return math.Round(price*100) / 100
}

// 가격 업데이트 (배치 작업)
func updatePrices() {
    // 매 시간마다 실행
    for _, roomType := range getAllRoomTypes() {
        for _, date := range getNext90Days() {
            price := calculatePrice(roomType.Id, date)
            updateRoomPrice(roomType.Id, date, price)
        }
    }
}
```

---

## 🔟 모니터링 (1분)

### 핵심 메트릭

```
Golden Signals:

Latency:
├── Search p95 < 500ms
├── Booking p95 < 2s
└── Availability check < 100ms

Traffic:
├── 10K searches/sec
├── 1K bookings/sec
└── QPS by endpoint

Errors:
├── Failed bookings < 1%
├── Overbooking = 0 (!!!)
├── 4xx rate < 2%
└── 5xx rate < 0.1%

Saturation:
├── DB connections < 80%
├── Lock wait time < 200ms
├── Cache hit rate > 85%
└── Disk I/O < 80%

비즈니스 메트릭:
├── Booking conversion rate
├── Average booking value
├── Cancellation rate
├── Occupancy rate by hotel
└── Revenue per available room
```

### 알람

```yaml
Critical (PagerDuty):
  - Overbooking detected (!!!)
  - Database down
  - Booking error rate > 5%
  - Payment processing failed

Warning (Slack):
  - Search latency p95 > 1s
  - Booking latency p95 > 3s
  - Cache hit rate < 70%
  - Lock wait time > 500ms
  - High cancellation rate (>20%)
```

---

## 📊 트레이드오프 정리

| 측면 | 선택 | 트레이드오프 |
|------|------|-------------|
| **동시성** | Pessimistic Lock | Throughput ↓, 정확성 ↑↑ |
| **재고 계산** | On-demand | 계산 비용 ↑, 정확성 ↑ |
| **검색** | Elasticsearch | 복잡도 ↑, 성능 ↑↑ |
| **Room 할당** | 체크인 시 | 유연성 ↑, 복잡도 ↑ |
| **일관성** | Strong (ACID) | 가용성 ↓, 정확성 ↑↑ |
| **캐싱** | Aggressive | Stale 가능, 성능 ↑ |

---

## ✅ 요약 (1분)

### 핵심 설계

1. **Pessimistic Locking**: 오버부킹 완전 방지
2. **On-demand 재고 계산**: 항상 정확한 가용성
3. **Elasticsearch**: 강력한 지리적 검색
4. **Redis Cache**: 읽기 성능 최적화
5. **Transaction Isolation**: ACID 보장

### 달성 목표

- ✅ 100K+ 호텔, 5M+ 객실
- ✅ < 500ms 검색
- ✅ < 2초 예약
- ✅ 0% 오버부킹
- ✅ 1,000+ bookings/sec

### 기술 스택

**Backend**: Go (동시성)  
**DB**: PostgreSQL (ACID + Locks)  
**Search**: Elasticsearch (지리 검색)  
**Cache**: Redis (성능)  
**Queue**: RabbitMQ (피크 대응)

### 가장 중요한 포인트

**동시성 제어** = 시스템의 핵심!
```sql
BEGIN TRANSACTION;
SELECT * FROM room_types 
WHERE room_type_id = $1
FOR UPDATE;  -- 이 락이 오버부킹을 방지!
-- Check availability
-- Create booking
COMMIT;
```

---

## 📝 화이트보드 그림 가이드

```
면접 시 그릴 다이어그램:

1단계: High-level (2분)
[Client] → [API] → [Search Service] → [Elasticsearch]
                    [Booking Service] → [PostgreSQL]
                                           (ACID + Locks)

2단계: 예약 플로우 (3분)
Request → Start TX → Lock Room Type (FOR UPDATE)
                     ↓
                  Check Availability
                     ↓
               available > 0?
              /              \
           Yes                No
            ↓                  ↓
      Create Booking      Rollback
      Commit TX           Return Error
            ↓
      Return Success

3단계: 동시 예약 시나리오 (2분)
Time →
User A: [──TX Start──Lock──Check─Create─Commit──]
User B:          [──TX Start──Wait────Lock──Check(0)─Rollback─]
                                ↑
                        User A가 락 해제될 때까지 대기

4단계: 재고 계산 (1분)
Available = Total Rooms - Booked Rooms - Blocked Rooms

Query:
SELECT total_rooms - COUNT(bookings) 
FROM room_types 
LEFT JOIN bookings ON ...
WHERE date BETWEEN check_in AND check_out
```

---

**면접 시간 배분**:
- 문제 이해: 2분
- Scale 추정: 3분
- High-level 설계: 3분
- Deep dive (동시성 제어!): 12분
- 특수 케이스: 5분
- Q&A: 5분

**핵심 강조 포인트**:
1. **동시성 제어**: Pessimistic Lock의 필요성
2. **오버부킹 방지**: 절대 발생하면 안 됨
3. **ACID**: 호텔 예약은 Strong Consistency 필요
4. **성능 vs 정확성**: 정확성 우선!

**끝!** 🎉
