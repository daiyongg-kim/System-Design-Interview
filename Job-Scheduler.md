# 🚀 Job Scheduler System Design (30분 인터뷰용)

> **글로벌 클라우드 기반 Job 스케줄링 서비스**

---

## 1️⃣ 문제 정의 (2분)

### 요구사항
- ✅ 일회성 & 반복 실행 Job 스케줄링
- ✅ Cron 표현식 지원
- ✅ Job 실행 이력 추적
- ✅ 실패 시 자동 재시도
- ✅ Webhook 알림

### 비기능 요구사항
- **성능**: 초당 10,000+ job 실행
- **정확도**: 스케줄 시간 오차 < 1초
- **가용성**: 99.9% uptime
- **확장성**: Horizontal scaling

---

## 2️⃣ 규모 추정 (3분)

```
📊 Scale
├── 사용자: 1M
├── Active Jobs: 10M
├── 일일 실행: 100M
├── 피크: 10,000 jobs/sec
└── 평균 실행 시간: 2분

💾 Storage
├── Job 메타데이터: 200GB
├── 실행 이력 (90일): 45TB
└── 로그 (S3): 450TB

💰 비용
└── 월 ~$70,000 (compute + storage + network)
```

---

## 3️⃣ 핵심 엔티티 (3분)

```typescript
// Job - 스케줄링할 작업
{
  jobId: string,
  userId: string,
  type: 'ONE_TIME' | 'RECURRING',
  scheduleExpression: string,  // "0 0 * * *"
  status: 'ACTIVE' | 'PAUSED',
  payload: JSON,
  retryPolicy: {
    maxRetries: 3,
    retryInterval: 300
  },
  nextExecutionAt: timestamp
}

// JobExecution - 실행 기록
{
  executionId: string,
  jobId: string,
  status: 'RUNNING' | 'SUCCEEDED' | 'FAILED',
  startedAt: timestamp,
  duration: number,
  attemptNumber: number
}
```

**샤딩 전략**: userId 기반 (한 사용자의 모든 job이 같은 샤드에)

---

## 4️⃣ API 설계 (2분)

```http
# Job 생성
POST /api/v1/jobs
{
  "name": "daily-report",
  "type": "RECURRING",
  "scheduleExpression": "0 0 * * *",
  "payload": {...}
}
→ 201 Created

# Job 즉시 실행
POST /api/v1/jobs/{jobId}/execute
→ 202 Accepted { executionId: "..." }

# 실행 상태 조회
GET /api/v1/executions/{executionId}
→ 200 OK { status: "RUNNING", ... }
```

**비동기 API**: 즉시 202 반환, 클라이언트가 폴링 또는 Webhook 사용

---

## 5️⃣ 시스템 아키텍처 (10분)

### High-Level Design

```
┌─────────────┐
│   Clients   │ Web/Mobile/CLI/SDK
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────┐
│ API Gateway │ Auth, Rate Limit, Routing
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────┐
│   Application Layer             │
│  ┌────────┐  ┌──────────────┐  │
│  │  Job   │  │  Scheduler   │  │
│  │Manager │  │   Engine     │  │
│  └───┬────┘  └──────┬───────┘  │
└──────┼───────────────┼──────────┘
       │               │
       ▼               ▼
┌─────────────┐  ┌─────────────┐
│ PostgreSQL  │  │  RabbitMQ   │ Priority Queue
│ (Metadata)  │  │ (Job Queue) │
└─────────────┘  └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │Worker Pool  │ K8s Pods
                 │Auto-scaling │
                 └──────┬──────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  S3 + TSDB  │ Logs + Metrics
                 └─────────────┘
```

### 주요 컴포넌트

#### 1. Scheduler Engine
```go
// Time Wheel Algorithm (O(1))
type TimeWheel struct {
    slots      [3600]*JobList  // 1시간을 1초 단위로
    currentPos int
}

// 매 초마다 tick
func (tw *TimeWheel) tick() {
    jobs := tw.slots[tw.currentPos]
    for _, job := range jobs {
        dispatcher.Send(job)  // RabbitMQ로 전송
    }
    tw.currentPos = (tw.currentPos + 1) % 3600
}
```

**장점**: O(1) insertion/deletion, 캐시 지역성 우수

#### 2. Job Queue (RabbitMQ)
```
Priority Queue:
├── High (priority 10)
├── Normal (priority 5)
└── Low (priority 1)

Features:
├── 메시지 영속성
├── Dead Letter Queue
└── Priority 지원
```

#### 3. Worker Pool
```yaml
# Kubernetes HPA
minReplicas: 100
maxReplicas: 2000
metrics:
  - type: Resource
    name: cpu
    targetAverage: 70%
  - type: Custom
    name: queue_depth
    targetValue: 10  # worker당 10개
```

**Pull Model**: Worker가 큐에서 job을 가져감
- 자연스러운 backpressure
- Worker 독립적 확장

---

## 6️⃣ 핵심 플로우 (5분)

### Job 생성 → 실행

```
1. Client → API Gateway (JWT 인증)
2. API Gateway → Job Manager (요청 검증)
3. Job Manager → PostgreSQL (저장)
4. Job Manager → Scheduler Engine (다음 실행 시간 계산)
5. 스케줄 시간 도달
6. Scheduler → RabbitMQ (job 푸시)
7. Worker ← RabbitMQ (job 풀)
8. Worker → 실행 (격리된 컨테이너)
9. Worker → PostgreSQL (결과 저장)
10. Worker → Webhook (알림)
```

**Timeline**: Job 생성 ~80ms, 실행 지연 < 5초

### 재시도 로직

```go
func shouldRetry(exec *Execution) bool {
    if exec.Attempt >= job.MaxRetries {
        return false  // DLQ로 이동
    }
    
    // Exponential backoff
    delay := retryInterval * pow(2, exec.Attempt)
    
    scheduleRetry(exec, delay)
    return true
}
```

---

## 7️⃣ 주요 설계 결정 (3분)

### 1. Time Wheel vs Heap
**선택**: Time Wheel ✅
- O(1) vs O(log n)
- 대량 job에 유리
- Linux kernel, Kafka 등에서 사용

### 2. Pull vs Push
**선택**: Pull Model ✅
- Worker가 자기 속도로 처리
- 자연스러운 backpressure
- 확장 용이

### 3. At-Least-Once vs Exactly-Once
**선택**: At-Least-Once ✅
- Exactly-once는 분산 환경에서 어려움
- 사용자가 idempotent job 작성
- Idempotency key로 중복 방지

```go
// Idempotency key: jobId + scheduledAt
key := fmt.Sprintf("%s:%d", jobId, scheduledAt.Unix())
if !cache.SetNX(key, "executing") {
    return  // 이미 실행 중이거나 완료
}
```

### 4. Database Sharding
**선택**: userId 기반 샤딩
```go
shardId = hash(userId) % numShards
```
- 사용자의 모든 데이터가 한 샤드에
- 분산 트랜잭션 불필요
- 확장 간단

---

## 8️⃣ 고가용성 & 확장성 (2분)

### Multi-Region Active-Active

```
Global DNS (CloudFlare)
├── US-EAST-1 (33% traffic)
├── EU-WEST-1 (33% traffic)
└── AP-SE-1 (33% traffic)

각 Region:
├── 3 AZs
├── Multi-AZ RDS
└── Auto-scaling workers
```

### 장애 복구

```
노드 장애:
└── K8s가 자동 재시작 (30초)

DB 장애:
└── Replica로 자동 failover (1분)

Region 장애:
└── 다른 region으로 트래픽 라우팅 (5분)

RTO: 5분, RPO: 1분
```

---

## 9️⃣ 병목 & 최적화 (추가 질문 대비)

### 잠재적 병목점

1. **Scheduler Engine 병목**
   - 해결: Time Wheel + Consistent Hashing으로 분산
   - 각 스케줄러가 특정 job subset만 담당

2. **Database 쓰기 병목**
   - 해결: Write-through cache + Batch updates
   - 실행 이력은 TimescaleDB 사용

3. **Queue 백로그**
   - 해결: Worker auto-scaling + Priority queue
   - Queue depth 기반 HPA

4. **Worker 리소스 부족**
   - 해결: Spot instances + Reserved capacity
   - 70% cost reduction

### 캐싱 전략

```
Redis 캐싱:
├── Job metadata (TTL: 1시간)
├── User quota (TTL: 5분)
├── Rate limit counters (TTL: 1분)
└── Idempotency keys (TTL: 24시간)

Cache-aside pattern
```

---

## 🔟 모니터링 (1분)

### 핵심 메트릭

```
Golden Signals:
├── Latency: p95 스케줄링 지연 < 1초
├── Traffic: 10K jobs/sec 처리
├── Errors: 실패율 < 1%
└── Saturation: Worker 사용률 < 80%

비즈니스 메트릭:
├── Job 생성률
├── 실행 성공률
└── SLA 준수율 (99.9%)
```

### 알람

```
Critical (PagerDuty):
├── API 에러율 > 5%
├── DB/Queue 다운
└── Worker pool < 10%

Warning (Slack):
├── 레이턴시 p95 > 500ms
└── 큐 depth > 10K
```

---

## 📊 트레이드오프 정리

| 측면 | 선택 | 트레이드오프 |
|------|------|-------------|
| **실행 보장** | At-least-once | Exactly-once는 복잡도 ↑ |
| **스케줄링** | Time Wheel | 메모리 사용 ↑, 정확도 ↑ |
| **실행 모델** | Pull | 약간의 지연 ↑, 확장성 ↑ |
| **샤딩** | userId | Cross-shard query 불가 |
| **일관성** | Eventually | 가용성 ↑, 강한 일관성 ↓ |

---

## 🎯 추가 질문 대비

### Q1: 100만 개 job이 동시에 실행되어야 한다면?
```
해결책:
1. Worker pool을 2000+로 확장
2. 지역별로 분산 (3 regions × 700 = 2100)
3. Batch processing (100 jobs → 1 worker)
4. 우선순위 기반 실행
```

### Q2: Job이 무한 루프에 빠지면?
```
보호 장치:
1. Timeout 설정 (job별)
2. CPU/Memory limit (컨테이너)
3. Watchdog이 stuck job 감지
4. 강제 종료 + DLQ 이동
```

### Q3: Scheduler 노드가 죽으면?
```
복구:
1. Health check가 장애 감지 (10초)
2. Consistent hashing으로 job 재분배
3. 다른 스케줄러가 인계 (30초)
4. 새 노드 자동 프로비저닝 (5분)

→ 최대 1분 지연, job 손실 없음
```

### Q4: 비용을 50% 줄이려면?
```
최적화:
1. Spot instances (70% 할인)
2. Reserved instances (40% 할인)
3. S3 Intelligent Tiering
4. Off-peak 시간 worker 축소
5. 로그 압축 + 90일 후 Glacier
```

---

## ✅ 요약 (1분)

### 핵심 설계
1. **Time Wheel** 스케줄러로 O(1) 성능
2. **Pull model** worker로 자연스러운 확장
3. **Priority queue** + Auto-scaling
4. **Multi-region** active-active
5. **userId 샤딩**으로 locality 확보

### 달성 목표
- ✅ 10K+ jobs/sec
- ✅ < 1초 스케줄링 정확도
- ✅ 99.9% uptime
- ✅ Horizontal scaling
- ✅ $70K/month 비용

### 기술 스택
**Backend**: Go (성능 + 동시성)  
**DB**: PostgreSQL (ACID) + TimescaleDB (시계열)  
**Queue**: RabbitMQ (priority) + Kafka (events)  
**Cache**: Redis (속도)  
**Infra**: Kubernetes (오케스트레이션)

---

## 📝 화이트보드 그림 가이드

```
면접관에게 그릴 다이어그램 순서:

1단계: High-level (2분)
[Client] → [API] → [Scheduler] → [Queue] → [Workers]
                         ↓
                      [Database]

2단계: Scheduler 확대 (2분)
Time Wheel: [Slot 0][Slot 1]...[Slot 3599]
             Jobs    Jobs          Jobs

3단계: Data Flow (2분)
Create → Validate → Store → Schedule → Queue → Execute
   ↓                                              ↓
 Response                                      Result

4단계: Multi-region (1분)
    [Global LB]
   /     |      \
[US]  [EU]  [ASIA]
```

---

**면접 시간 배분**:
- 문제 이해: 2분
- Scale 추정: 3분
- High-level 설계: 5분
- Deep dive: 10분
- 트레이드오프 논의: 5분
- Q&A: 5분

**끝!** 🎉
