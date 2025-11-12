# 📝 Document Management System Design (30분 인터뷰용)

> **Wikipedia/Notion/Google Docs 스타일 문서 관리 시스템 (실시간 편집 제외)**

---

## 1️⃣ 문제 정의 (2분)

### 요구사항
- ✅ 문서 생성, 수정, 삭제, 조회
- ✅ 버전 관리 (히스토리 추적)
- ✅ 권한 관리 (읽기/쓰기/공유)
- ✅ 폴더/계층 구조
- ✅ 전체 텍스트 검색
- ✅ 첨부 파일 지원
- ❌ 실시간 협업 편집 (제외)

### 비기능 요구사항
- **성능**: 문서 조회 < 200ms (p95)
- **검색**: 검색 결과 < 500ms
- **가용성**: 99.9% uptime
- **확장성**: 100M+ 문서 지원
- **저장 용량**: 최대 100MB/문서

---

## 2️⃣ 규모 추정 (3분)

```
📊 Scale
├── 사용자: 10M (DAU: 2M)
├── 문서: 100M
├── 평균 문서 크기: 50KB
├── 일일 읽기: 50M (25 reads/user)
├── 일일 쓰기: 5M (2.5 writes/user)
└── 피크 QPS: 10,000 reads/sec, 1,000 writes/sec

💾 Storage
├── 문서 본문: 100M × 50KB = 5TB
├── 버전 히스토리: 5TB × 5 versions = 25TB
├── 메타데이터: 100M × 2KB = 200GB
├── 첨부 파일: 10M × 2MB = 20TB
├── 검색 인덱스: ~5TB
└── 총: ~55TB

📈 Bandwidth
├── 읽기: 10K QPS × 50KB = 500MB/s = 4Gbps
├── 쓰기: 1K QPS × 50KB = 50MB/s = 400Mbps
└── 검색: 500 QPS × 100KB = 50MB/s

💰 비용
└── 월 ~$15,000 (compute + storage + CDN)
```

---

## 3️⃣ 핵심 엔티티 (3분)

```typescript
// User - 사용자
{
  userId: string,
  email: string,
  name: string,
  organizationId: string,
  createdAt: timestamp
}

// Document - 문서
{
  documentId: string,
  workspaceId: string,
  title: string,
  content: string,              // Markdown/Rich Text
  contentHash: string,          // 중복 방지
  version: number,              // 현재 버전
  parentId: string | null,      // 폴더 구조
  path: string,                 // "/folder1/folder2/doc"
  
  // 메타데이터
  size: number,                 // bytes
  status: 'DRAFT' | 'PUBLISHED' | 'ARCHIVED',
  
  // 권한
  ownerId: string,
  visibility: 'PRIVATE' | 'WORKSPACE' | 'PUBLIC',
  
  // 타임스탬프
  createdAt: timestamp,
  updatedAt: timestamp,
  lastViewedAt: timestamp
}

// DocumentVersion - 버전 히스토리
{
  versionId: string,
  documentId: string,
  version: number,
  content: string,              // 또는 diff
  contentHash: string,
  
  // 변경 정보
  changes: {
    linesAdded: number,
    linesRemoved: number,
    changeType: 'MAJOR' | 'MINOR'
  },
  
  // 작성자
  authorId: string,
  commitMessage: string,
  createdAt: timestamp
}

// Permission - 권한
{
  permissionId: string,
  documentId: string,
  userId: string | null,        // null = everyone
  role: 'VIEWER' | 'EDITOR' | 'ADMIN',
  grantedBy: string,
  createdAt: timestamp
}

// Attachment - 첨부 파일
{
  attachmentId: string,
  documentId: string,
  fileName: string,
  fileSize: number,
  mimeType: string,
  s3Key: string,
  uploadedBy: string,
  createdAt: timestamp
}
```

**샤딩 전략**: 
- Documents: documentId 기반 (range sharding 또는 hash)
- Versions: documentId 기반 (같은 샤드에 함께)

---

## 4️⃣ API 설계 (2분)

```http
# 문서 생성
POST /api/v1/documents
{
  "title": "System Design Notes",
  "content": "# Chapter 1\nIntroduction...",
  "parentId": "folder_123",
  "status": "PUBLISHED"
}
→ 201 Created { documentId: "doc_abc123" }

# 문서 조회
GET /api/v1/documents/{documentId}
→ 200 OK
{
  "documentId": "doc_abc123",
  "title": "System Design Notes",
  "content": "...",
  "version": 5,
  "owner": {...},
  "updatedAt": "2025-11-11T10:30:00Z"
}

# 문서 수정
PUT /api/v1/documents/{documentId}
{
  "content": "# Chapter 1\nUpdated content...",
  "commitMessage": "Fixed typos"
}
→ 200 OK { version: 6 }

# 버전 히스토리 조회
GET /api/v1/documents/{documentId}/versions
→ 200 OK
{
  "versions": [
    { "version": 6, "authorId": "...", "createdAt": "...", ... },
    { "version": 5, "authorId": "...", "createdAt": "...", ... }
  ]
}

# 특정 버전 조회
GET /api/v1/documents/{documentId}/versions/5
→ 200 OK { content: "...", version: 5 }

# 버전 복원
POST /api/v1/documents/{documentId}/restore
{ "version": 5 }
→ 200 OK { version: 7 } # 새 버전으로 생성

# 검색
GET /api/v1/search?q=system+design&workspace=ws_123
→ 200 OK
{
  "results": [
    {
      "documentId": "doc_abc123",
      "title": "System Design Notes",
      "snippet": "...system design principles...",
      "score": 0.95
    }
  ],
  "total": 156
}

# 권한 관리
POST /api/v1/documents/{documentId}/permissions
{
  "userId": "user_xyz",
  "role": "EDITOR"
}
→ 201 Created

# 첨부 파일 업로드
POST /api/v1/documents/{documentId}/attachments
Content-Type: multipart/form-data
→ 201 Created { attachmentId: "att_123", url: "https://..." }
```

---

## 5️⃣ 시스템 아키텍처 (10분)

### High-Level Design

```
┌─────────────┐
│   Clients   │ Web/Mobile/Desktop App
└──────┬──────┘
       │ HTTPS
       ▼
┌─────────────┐
│     CDN     │ Static assets + Cached content
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ API Gateway │ Auth, Rate Limit, Routing
└──────┬──────┘
       │
       ▼
┌────────────────────────────────────────┐
│       Application Services             │
│  ┌──────────┐  ┌──────────┐           │
│  │Document  │  │ Search   │           │
│  │Service   │  │ Service  │           │
│  └────┬─────┘  └────┬─────┘           │
│       │             │                  │
│  ┌────┴─────┐  ┌────┴─────┐          │
│  │Version   │  │Permission│          │
│  │Service   │  │ Service  │          │
│  └──────────┘  └──────────┘          │
└────────────────────────────────────────┘
       │             │          │
       ▼             ▼          ▼
┌─────────────┐ ┌─────────┐ ┌──────────┐
│ PostgreSQL  │ │Elasticsearch│ Redis   │
│  (Sharded)  │ │(Search)  │ │(Cache)  │
└─────────────┘ └─────────┘ └──────────┘
       │
       ▼
┌─────────────┐
│     S3      │ Document versions + Attachments
└─────────────┘
```

### 상세 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                    Client Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │   Web    │  │  Mobile  │  │ Desktop  │         │
│  │   SPA    │  │   App    │  │   App    │         │
│  └──────────┘  └──────────┘  └──────────┘         │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│                   CDN Layer                         │
│  Cloudflare / CloudFront                           │
│  - Static assets (JS, CSS, Images)                │
│  - Cached read-only documents                      │
│  - Edge locations worldwide                        │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│              Load Balancer (Layer 7)               │
│  NGINX / AWS ALB                                   │
└────────────────────────┬────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────┐
│                  API Gateway                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   Auth   │  │   Rate   │  │  Router  │        │
│  │ (JWT)    │  │ Limiter  │  │          │        │
│  └──────────┘  └──────────┘  └──────────┘        │
└────────────────────────┬────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Document   │ │    Search    │ │  Permission  │
│   Service    │ │   Service    │ │   Service    │
│              │ │              │ │              │
│ - CRUD ops   │ │ - Indexing   │ │ - ACL check  │
│ - Validation │ │ - Full-text  │ │ - Share mgmt │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       ▼                │                │
┌──────────────┐        │                │
│   Version    │        │                │
│   Service    │        │                │
│              │        │                │
│ - Diff calc  │        │                │
│ - History    │        │                │
└──────┬───────┘        │                │
       │                │                │
       ▼                ▼                ▼
┌────────────────────────────────────────────────────┐
│                   Cache Layer                       │
│  Redis Cluster                                      │
│  - Document metadata (5 min TTL)                   │
│  - User permissions (10 min TTL)                   │
│  - Recent documents (LRU)                          │
│  - Session data                                     │
└────────────────────────┬────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ PostgreSQL   │ │Elasticsearch │ │    S3        │
│   Cluster    │ │   Cluster    │ │   Bucket     │
│              │ │              │ │              │
│ Primary +    │ │ 3 nodes      │ │ Versioning   │
│ 2 Replicas   │ │ Sharded by   │ │ Lifecycle    │
│ Sharded      │ │ documentId   │ │ policies     │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

### 주요 컴포넌트 상세

#### 1. Document Service

```go
type DocumentService struct {
    db        *PostgreSQL
    cache     *Redis
    s3        *S3Client
    search    *ElasticsearchClient
    version   *VersionService
}

func (ds *DocumentService) GetDocument(docId string, userId string) (*Document, error) {
    // 1. Check cache
    if doc := ds.cache.Get(docId); doc != nil {
        return doc, nil
    }
    
    // 2. Check permission
    if !ds.permission.CanRead(userId, docId) {
        return nil, ErrForbidden
    }
    
    // 3. Get from DB (metadata only)
    doc := ds.db.GetDocument(docId)
    
    // 4. Get content from S3 (if large)
    if doc.Size > 100KB {
        doc.Content = ds.s3.GetContent(doc.ContentKey)
    }
    
    // 5. Update cache
    ds.cache.Set(docId, doc, 5*time.Minute)
    
    // 6. Track view (async)
    go ds.trackView(docId, userId)
    
    return doc, nil
}

func (ds *DocumentService) UpdateDocument(docId string, userId string, newContent string) error {
    // 1. Check permission
    if !ds.permission.CanWrite(userId, docId) {
        return ErrForbidden
    }
    
    // 2. Get current document
    oldDoc := ds.db.GetDocument(docId)
    
    // 3. Create new version
    version := ds.version.CreateVersion(oldDoc, newContent, userId)
    
    // 4. Save to S3
    contentKey := ds.s3.SaveContent(docId, version.Version, newContent)
    
    // 5. Update DB
    tx := ds.db.Begin()
    tx.UpdateDocument(docId, version.Version, contentKey)
    tx.InsertVersion(version)
    tx.Commit()
    
    // 6. Invalidate cache
    ds.cache.Delete(docId)
    
    // 7. Update search index (async)
    go ds.search.Index(docId, newContent)
    
    return nil
}
```

#### 2. Version Service

**저장 전략 선택**:

**Option A: Full Snapshot (선택) ✅**
```
v1: "Hello World"
v2: "Hello Beautiful World"
v3: "Hello Beautiful Wonderful World"

장점:
- 조회 속도 빠름 (O(1))
- 구현 간단
- 임의 버전 접근 용이

단점:
- 저장 공간 많이 사용
```

**Option B: Delta/Diff**
```
v1: "Hello World"
v2: +Beautiful (insert at 6)
v3: +Wonderful (insert at 16)

장점:
- 저장 공간 효율적

단점:
- 조회 시 재구성 필요 (느림)
- 구현 복잡
```

**하이브리드 접근**: 
- 최근 10개 버전: Full snapshot
- 오래된 버전: Delta + 주기적 checkpoint

```go
type VersionService struct {
    s3        *S3Client
    db        *PostgreSQL
}

func (vs *VersionService) CreateVersion(oldDoc *Document, newContent string, userId string) *Version {
    // 1. Calculate diff
    diff := vs.calculateDiff(oldDoc.Content, newContent)
    
    // 2. Create version
    version := &Version{
        VersionId:   generateId(),
        DocumentId:  oldDoc.DocumentId,
        Version:     oldDoc.Version + 1,
        Content:     newContent,  // Full snapshot
        ContentHash: hash(newContent),
        Changes: Changes{
            LinesAdded:   diff.LinesAdded,
            LinesRemoved: diff.LinesRemoved,
        },
        AuthorId:    userId,
        CreatedAt:   time.Now(),
    }
    
    // 3. Save to S3
    s3Key := fmt.Sprintf("documents/%s/versions/%d", 
                         oldDoc.DocumentId, version.Version)
    vs.s3.Put(s3Key, newContent)
    
    return version
}

func (vs *VersionService) GetVersion(docId string, version int) (*Version, error) {
    // 1. Get from DB (metadata)
    v := vs.db.GetVersion(docId, version)
    
    // 2. Get content from S3
    s3Key := fmt.Sprintf("documents/%s/versions/%d", docId, version)
    content := vs.s3.Get(s3Key)
    v.Content = content
    
    return v, nil
}
```

#### 3. Search Service

```go
type SearchService struct {
    es *elasticsearch.Client
}

// 문서 인덱싱 (비동기)
func (ss *SearchService) IndexDocument(doc *Document) error {
    indexDoc := map[string]interface{}{
        "documentId": doc.DocumentId,
        "title":      doc.Title,
        "content":    doc.Content,
        "ownerId":    doc.OwnerId,
        "workspaceId": doc.WorkspaceId,
        "tags":       doc.Tags,
        "createdAt":  doc.CreatedAt,
        "updatedAt":  doc.UpdatedAt,
    }
    
    _, err := ss.es.Index(
        "documents",
        doc.DocumentId,
        indexDoc,
    )
    return err
}

// 검색
func (ss *SearchService) Search(query string, userId string, workspaceId string) ([]*SearchResult, error) {
    // Elasticsearch query
    searchQuery := map[string]interface{}{
        "query": map[string]interface{}{
            "bool": map[string]interface{}{
                "must": []map[string]interface{}{
                    {
                        "multi_match": map[string]interface{}{
                            "query":  query,
                            "fields": []string{"title^3", "content"},
                            "type":   "best_fields",
                        },
                    },
                    {
                        "term": map[string]interface{}{
                            "workspaceId": workspaceId,
                        },
                    },
                },
                "filter": []map[string]interface{}{
                    // Permission filter (can be complex)
                    {
                        "bool": map[string]interface{}{
                            "should": []map[string]interface{}{
                                {"term": {"ownerId": userId}},
                                {"term": {"visibility": "WORKSPACE"}},
                                {"term": {"visibility": "PUBLIC"}},
                            },
                        },
                    },
                },
            },
        },
        "highlight": map[string]interface{}{
            "fields": map[string]interface{}{
                "content": map[string]interface{}{
                    "fragment_size": 150,
                    "number_of_fragments": 3,
                },
            },
        },
    }
    
    res, err := ss.es.Search(
        ss.es.Search.WithIndex("documents"),
        ss.es.Search.WithBody(esutil.NewJSONReader(searchQuery)),
    )
    
    return ss.parseResults(res), err
}
```

**Elasticsearch 인덱스 구조**:
```json
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "document_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "stop", "snowball"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "documentId": { "type": "keyword" },
      "title": { 
        "type": "text", 
        "analyzer": "document_analyzer",
        "boost": 3.0
      },
      "content": { 
        "type": "text", 
        "analyzer": "document_analyzer"
      },
      "ownerId": { "type": "keyword" },
      "workspaceId": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "createdAt": { "type": "date" },
      "updatedAt": { "type": "date" }
    }
  }
}
```

#### 4. Permission Service

```go
type PermissionService struct {
    db    *PostgreSQL
    cache *Redis
}

func (ps *PermissionService) CanRead(userId string, docId string) bool {
    // 1. Check cache
    cacheKey := fmt.Sprintf("perm:%s:%s", userId, docId)
    if cached := ps.cache.Get(cacheKey); cached != nil {
        return cached.(bool)
    }
    
    // 2. Get document
    doc := ps.db.GetDocument(docId)
    
    // 3. Check visibility
    if doc.Visibility == "PUBLIC" {
        ps.cache.Set(cacheKey, true, 10*time.Minute)
        return true
    }
    
    // 4. Check ownership
    if doc.OwnerId == userId {
        ps.cache.Set(cacheKey, true, 10*time.Minute)
        return true
    }
    
    // 5. Check workspace membership
    if doc.Visibility == "WORKSPACE" {
        if ps.isWorkspaceMember(userId, doc.WorkspaceId) {
            ps.cache.Set(cacheKey, true, 10*time.Minute)
            return true
        }
    }
    
    // 6. Check explicit permission
    perm := ps.db.GetPermission(userId, docId)
    if perm != nil && (perm.Role == "VIEWER" || perm.Role == "EDITOR" || perm.Role == "ADMIN") {
        ps.cache.Set(cacheKey, true, 10*time.Minute)
        return true
    }
    
    ps.cache.Set(cacheKey, false, 1*time.Minute)
    return false
}

func (ps *PermissionService) CanWrite(userId string, docId string) bool {
    // Similar logic but stricter (EDITOR or ADMIN only)
    // ...
}
```

---

## 6️⃣ 핵심 플로우 (5분)

### 문서 생성 플로우

```
1. Client → API Gateway (JWT 검증)
2. API Gateway → Document Service
3. Document Service → Permission Service (quota 확인)
4. Document Service → Generate documentId
5. Document Service → S3 (content 저장)
6. Document Service → PostgreSQL (metadata 저장)
7. Document Service → Search Service (비동기 인덱싱)
8. Document Service → Client (201 Created)

Timeline: ~150ms
```

### 문서 조회 플로우 (Cache Hit)

```
1. Client → CDN (static assets)
2. Client → API Gateway → Document Service
3. Document Service → Redis (cache hit!)
4. Document Service → Permission Service (권한 확인)
5. Document Service → Client (200 OK)

Timeline: ~50ms
```

### 문서 조회 플로우 (Cache Miss)

```
1. Client → API Gateway → Document Service
2. Document Service → Redis (cache miss)
3. Document Service → PostgreSQL (metadata)
4. Document Service → S3 (content - if large)
5. Document Service → Redis (캐시 저장)
6. Document Service → Client (200 OK)

Timeline: ~200ms
```

### 문서 수정 플로우

```
1. Client → API Gateway → Document Service
2. Document Service → Permission Service (write 권한 확인)
3. Document Service → PostgreSQL (현재 버전 조회)
4. Document Service → Version Service (새 버전 생성)
5. Version Service → Calculate diff
6. Version Service → S3 (새 버전 저장)
7. Version Service → PostgreSQL (버전 메타데이터 저장)
8. Document Service → PostgreSQL (문서 업데이트)
9. Document Service → Redis (캐시 무효화)
10. Document Service → Search Service (비동기 재인덱싱)
11. Document Service → Client (200 OK)

Timeline: ~300ms
```

### 검색 플로우

```
1. Client → API Gateway → Search Service
2. Search Service → Redis (최근 검색 캐시 확인)
3. Search Service → Elasticsearch (쿼리 실행)
4. Elasticsearch → 관련 문서 반환 (with highlights)
5. Search Service → Permission Service (배치 권한 필터링)
6. Search Service → Client (200 OK)

Timeline: ~400ms
```

---

## 7️⃣ 주요 설계 결정 (3분)

### 1. 콘텐츠 저장: DB vs S3

**선택**: 하이브리드 ✅

```go
if doc.Size < 100KB {
    // PostgreSQL에 직접 저장 (빠른 조회)
    db.SaveContent(docId, content)
} else {
    // S3에 저장, DB에는 참조만
    s3Key := s3.Put(docId, content)
    db.SaveReference(docId, s3Key)
}
```

**이유**:
- 작은 문서: DB 저장으로 latency 감소
- 큰 문서: S3 저장으로 DB 부하 감소
- 대부분의 문서는 작음 (50KB 평균)

### 2. 버전 저장: Full Snapshot vs Delta

**선택**: Full Snapshot ✅

**이유**:
- 조회 속도 중요 (diff 재구성 느림)
- 저장 공간은 상대적으로 저렴
- 구현 단순
- 버전 간 비교는 on-demand로 계산

**최적화**:
```go
// 오래된 버전 압축
if version.CreatedAt < 90.Days.Ago() {
    compressedContent := gzip.Compress(version.Content)
    s3.Put(s3Key, compressedContent, metadata: {compressed: true})
}
```

### 3. 검색: PostgreSQL Full-Text vs Elasticsearch

**선택**: Elasticsearch ✅

**비교**:

| 기능 | PostgreSQL | Elasticsearch |
|------|-----------|---------------|
| 검색 속도 | 느림 (~2s) | 빠름 (<500ms) |
| 관련성 | 기본 | 고급 (BM25) |
| Highlighting | 제한적 | 강력 |
| Scale | 수직 | 수평 |
| 복잡도 | 낮음 | 높음 |

**이유**:
- 100M 문서에서 빠른 검색 필요
- 관련성 스코어링 중요
- Highlight 기능 필수
- 수평 확장 가능

### 4. 권한 관리: Row-level vs Application-level

**선택**: Application-level ✅

```go
// Application에서 권한 체크
if !permission.CanRead(userId, docId) {
    return ErrForbidden
}
doc := db.GetDocument(docId)
```

**이유**:
- 유연성 (복잡한 권한 로직)
- 캐싱 가능
- DB 부하 감소
- PostgreSQL RLS는 성능 오버헤드

### 5. 폴더 구조: Adjacency List vs Materialized Path

**선택**: Materialized Path ✅

```sql
-- Adjacency List
documentId | parentId
doc1       | null
doc2       | doc1
doc3       | doc2

-- Materialized Path (선택)
documentId | path
doc1       | /
doc2       | /doc1/
doc3       | /doc1/doc2/
```

**이유**:
- 조상 찾기: 1 query vs N queries
- 계층 구조 조회 빠름
- 검색 필터링 용이
- 이동 시 업데이트 필요 (trade-off)

**최적화**:
```sql
-- 전체 폴더 트리 조회
SELECT * FROM documents 
WHERE path LIKE '/folder1/%'
ORDER BY path;

-- 인덱스
CREATE INDEX idx_documents_path ON documents(path);
```

---

## 8️⃣ 확장성 & 최적화 (2분)

### Database Sharding

**전략**: Range-based sharding by documentId

```
Shard 0: doc_000000 ~ doc_1fffff (0-33%)
Shard 1: doc_200000 ~ doc_3fffff (33-66%)
Shard 2: doc_400000 ~ doc_5fffff (66-100%)

Router:
shardId = (hash(documentId) / maxHash) * numShards
```

**장점**:
- 균등 분산
- Hot shard 방지
- 확장 용이

### Caching 전략

```
Redis 캐싱:
├── L1: Document metadata (5분 TTL, 10GB)
│   └── Key: "doc:{docId}"
│
├── L2: User permissions (10분 TTL, 5GB)
│   └── Key: "perm:{userId}:{docId}"
│
├── L3: Search results (5분 TTL, 2GB)
│   └── Key: "search:{hash(query)}"
│
└── L4: Popular documents (LRU, 20GB)
    └── Key: "hot:doc:{docId}"

CDN 캐싱:
├── Public documents (24시간 TTL)
├── Static assets (1주일 TTL)
└── 150+ PoP locations
```

### 읽기 최적화

```go
// Read-through cache
func GetDocument(docId string) (*Document, error) {
    // 1. Try cache
    if doc := cache.Get(docId); doc != nil {
        return doc, nil
    }
    
    // 2. Get from DB
    doc := db.GetDocument(docId)
    
    // 3. Warm cache
    cache.Set(docId, doc, 5*time.Minute)
    
    return doc, nil
}

// Read replica for queries
func SearchDocuments(query string) ([]*Document, error) {
    // Use read replica to reduce primary load
    return readReplica.Search(query)
}
```

### 쓰기 최적화

```go
// Write-behind cache
func UpdateDocument(docId string, content string) error {
    // 1. Invalidate cache immediately
    cache.Delete(docId)
    
    // 2. Update DB (async if possible)
    go db.UpdateDocument(docId, content)
    
    // 3. Re-index (async)
    go search.ReIndex(docId, content)
    
    return nil
}

// Batch updates for analytics
func TrackViews(views []View) {
    // Buffer and batch insert every 1 minute
    buffer.Add(views)
    if buffer.Size() > 1000 || time.Since(lastFlush) > 1*time.Minute {
        db.BatchInsert(buffer.Drain())
    }
}
```

---

## 9️⃣ 데이터 일관성 (추가 질문 대비)

### 문제: 동시 수정

**시나리오**:
```
User A: 문서 조회 (v5)
User B: 문서 조회 (v5)
User A: 수정 → v6 생성
User B: 수정 → v6 생성 (충돌!)
```

**해결책**: Optimistic Locking ✅

```go
func UpdateDocument(docId string, expectedVersion int, newContent string) error {
    tx := db.Begin()
    
    // 1. Lock row
    doc := tx.LockDocument(docId)
    
    // 2. Check version
    if doc.Version != expectedVersion {
        return ErrVersionConflict  // 409 Conflict
    }
    
    // 3. Create new version
    newVersion := doc.Version + 1
    tx.UpdateDocument(docId, newVersion)
    tx.InsertVersion(newVersion, newContent)
    
    tx.Commit()
    return nil
}
```

**클라이언트 처리**:
```javascript
// Client-side conflict resolution
try {
    await updateDocument(docId, currentVersion, newContent);
} catch (error) {
    if (error.code === 'VERSION_CONFLICT') {
        // Show conflict dialog
        const resolution = await showConflictDialog(
            theirVersion,
            myChanges
        );
        
        // Retry with merged content
        await updateDocument(docId, theirVersion, resolution);
    }
}
```

### 캐시 일관성

**문제**: 캐시 무효화 실패 시 stale data

**해결책**:
1. **TTL 기반**: 모든 캐시에 TTL (최대 5분 stale)
2. **Version in Cache**: 캐시에 버전 포함
3. **Cache-aside Pattern**: 항상 DB가 source of truth

```go
type CachedDocument struct {
    Document *Document
    Version  int
    CachedAt time.Time
}

func GetDocument(docId string) (*Document, error) {
    // 1. Get from cache
    cached := cache.Get(docId)
    
    // 2. Verify version with DB (lightweight query)
    latestVersion := db.GetDocumentVersion(docId)
    
    // 3. If version mismatch, invalidate
    if cached != nil && cached.Version != latestVersion {
        cache.Delete(docId)
        cached = nil
    }
    
    // 4. If no cache, get from DB
    if cached == nil {
        doc := db.GetDocument(docId)
        cache.Set(docId, doc)
        return doc, nil
    }
    
    return cached.Document, nil
}
```

---

## 🔟 추가 기능 (시간 남으면)

### 1. 문서 공유 링크

```go
type ShareLink struct {
    LinkId     string
    DocumentId string
    Token      string  // Random secure token
    ExpiresAt  time.Time
    MaxViews   int
    ViewCount  int
    CreatedBy  string
}

// Generate share link
POST /api/v1/documents/{docId}/share
{
  "expiresIn": 7200,  // 2 hours
  "maxViews": 100
}
→ { 
  "shareUrl": "https://docs.com/s/abc123xyz",
  "expiresAt": "2025-11-11T12:30:00Z"
}

// Access via share link
GET /s/{token}
→ Return document (no auth required)
```

### 2. 문서 템플릿

```go
type Template struct {
    TemplateId  string
    Name        string
    Description string
    Content     string
    Category    string
    IsPublic    bool
}

// Create from template
POST /api/v1/documents/from-template
{
  "templateId": "tpl_meeting_notes"
}
→ Creates new document with template content
```

### 3. 댓글 & 주석

```go
type Comment struct {
    CommentId   string
    DocumentId  string
    AuthorId    string
    Content     string
    Position    int  // Character offset or line number
    Resolved    bool
    CreatedAt   time.Time
}

POST /api/v1/documents/{docId}/comments
{
  "content": "This section needs clarification",
  "position": 1234
}
```

---

## 1️⃣1️⃣ 모니터링 (1분)

### 핵심 메트릭

```
Golden Signals:
├── Latency: 
│   ├── Document read p95 < 200ms
│   ├── Document write p95 < 300ms
│   └── Search p95 < 500ms
│
├── Traffic: 
│   ├── 10K reads/sec
│   └── 1K writes/sec
│
├── Errors:
│   ├── 4xx rate < 1%
│   └── 5xx rate < 0.1%
│
└── Saturation:
    ├── DB connections < 80%
    ├── Cache hit rate > 90%
    └── Disk usage < 80%

비즈니스 메트릭:
├── Documents created/day
├── Search queries/day
├── Active users
└── Storage growth rate
```

### 알람

```yaml
Critical (PagerDuty):
  - API error rate > 5%
  - Database down
  - Elasticsearch cluster red
  - Cache hit rate < 50%

Warning (Slack):
  - Document read latency p95 > 500ms
  - Search latency p95 > 1s
  - Disk usage > 80%
  - Version creation failed
```

---

## 📊 트레이드오프 정리

| 측면 | 선택 | 트레이드오프 |
|------|------|-------------|
| **버전 저장** | Full snapshot | 저장공간 ↑, 조회속도 ↑ |
| **검색** | Elasticsearch | 복잡도 ↑, 성능 ↑ |
| **콘텐츠 저장** | Hybrid (DB+S3) | 로직 복잡, 최적 성능 |
| **권한** | Application-level | 유연성 ↑, DB 부하 ↓ |
| **폴더 구조** | Materialized Path | 조회 빠름, 이동 느림 |
| **일관성** | Eventual | 가용성 ↑, 약간 stale 가능 |

---

## 🎯 추가 질문 대비

### Q1: 1TB 문서를 지원해야 한다면?

```
해결책:
1. Chunking: 문서를 10MB 청크로 분할
2. S3 Multipart Upload
3. 메타데이터만 DB에, 실제 콘텐츠는 S3
4. Lazy loading: 필요한 청크만 로드
5. Streaming: 클라이언트로 스트리밍

저장:
POST /api/v1/documents/{docId}/chunks/{chunkNum}
GET /api/v1/documents/{docId}/chunks/{chunkNum}
```

### Q2: 실시간 협업 편집을 추가한다면?

```
기술:
1. WebSocket 연결
2. Operational Transformation (OT) 또는 CRDT
3. Y.js / Automerge 라이브러리
4. Redis Pub/Sub for real-time sync

추가 컴포넌트:
├── Collaboration Service (WebSocket server)
├── Conflict Resolution Engine
└── Presence Service (who's online)

도전 과제:
├── 충돌 해결 (complex!)
├── Network partition 처리
├── 동시 편집자 수 제한
└── 서버 비용 증가 (WebSocket connections)
```

### Q3: 문서를 다른 형식으로 export한다면?

```
지원 형식:
├── Markdown → PDF (Pandoc)
├── Markdown → DOCX (Pandoc)
├── Markdown → HTML (markdown-it)
└── HTML → PDF (Puppeteer)

아키텍처:
┌──────────────┐
│   Client     │
└──────┬───────┘
       │ POST /export?format=pdf
       ▼
┌──────────────┐
│Export Service│
└──────┬───────┘
       │ Push to queue
       ▼
┌──────────────┐
│  Job Queue   │
└──────┬───────┘
       │ Worker pulls
       ▼
┌──────────────┐
│Export Worker │ Pandoc/Puppeteer
└──────┬───────┘
       │ Upload result
       ▼
┌──────────────┐
│      S3      │
└──────────────┘

비동기 처리:
1. 요청 → Job 생성 (202 Accepted)
2. Worker가 변환 수행
3. 완료 시 Webhook 또는 이메일 알림
4. Download link 제공 (24시간 유효)
```

### Q4: 스팸/악성 콘텐츠 방지는?

```
전략:
1. Rate limiting (문서 생성 100/일)
2. Content moderation:
   ├── Automated: ML 기반 필터링
   └── Manual: 신고 시스템
3. File scanning:
   ├── VirusTotal API
   └── ClamAV
4. Upload limits:
   ├── 파일 크기: 100MB
   ├── 첨부 파일 수: 10개
   └── Workspace quota
5. Captcha on suspicious activity

구현:
┌──────────────┐
│   Upload     │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│Content Filter│ ML model
└──────┬───────┘
       │ Pass/Block
       ▼
┌──────────────┐
│ Virus Scan   │ ClamAV
└──────┬───────┘
       │
       ▼
┌──────────────┐
│   Storage    │
└──────────────┘
```

---

## ✅ 요약 (1분)

### 핵심 설계

1. **Hybrid Storage**: 작은 문서 DB, 큰 문서 S3
2. **Full Snapshot Versioning**: 빠른 조회
3. **Elasticsearch**: 강력한 전체 텍스트 검색
4. **Application-level Permissions**: 유연성
5. **Materialized Path**: 효율적 계층 구조

### 달성 목표

- ✅ 100M+ 문서 지원
- ✅ < 200ms 문서 조회
- ✅ < 500ms 검색
- ✅ 버전 관리
- ✅ 99.9% uptime

### 기술 스택

**Backend**: Go (성능 + 동시성)  
**DB**: PostgreSQL (ACID) + Sharding  
**Search**: Elasticsearch (강력한 검색)  
**Storage**: S3 (저렴한 대용량)  
**Cache**: Redis (속도)  
**CDN**: CloudFlare (글로벌 배포)

### 확장 계획

```
Phase 1 (현재):
├── 기본 CRUD + 버전 관리
├── 검색 + 권한 관리
└── 10M 사용자

Phase 2 (+6개월):
├── 실시간 협업 편집
├── 고급 템플릿
├── API integrations
└── 50M 사용자

Phase 3 (+1년):
├── AI 기반 제안
├── 고급 analytics
├── Multi-language
└── 100M 사용자
```

---

## 📝 화이트보드 그림 가이드

```
면접 시 그릴 다이어그램 순서:

1단계: High-level (3분)
[Client] → [CDN] → [API Gateway] → [Services]
                                        ↓
                    [PostgreSQL] + [Elasticsearch] + [S3]

2단계: Document Service 확대 (2분)
Request → Auth → Permission Check → DB/S3 → Response
             ↓
           Cache (Redis)

3단계: Version 저장 전략 (2분)
v1: [Full content in S3]
v2: [Full content in S3]  ← Full snapshot 방식
v3: [Full content in S3]

4단계: Search Flow (2분)
Query → [Elasticsearch] → Results → [Permission Filter] → Response
            ↓
      [Index shards]

5단계: 권한 모델 (1분)
Document → Permissions
   ↓          ↓
Owner     Workspace
Public    Explicit shares
```

---

**면접 시간 배분**:
- 문제 이해: 2분
- Scale 추정: 3분  
- High-level 설계: 5분
- Deep dive (버전, 검색, 권한): 10분
- 트레이드오프 논의: 5분
- Q&A: 5분

**핵심 강조 포인트**:
1. **버전 관리**: Full snapshot vs Delta 비교
2. **검색**: Elasticsearch 선택 이유
3. **확장성**: Sharding + Caching 전략
4. **권한**: Application-level ACL

**끝!** 🎉
