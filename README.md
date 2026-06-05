# Technical Assessment Platform - System Design

## Overview
منصة تقييم تقني متكاملة لاختبار المتقدمين للوظائف عبر 3 مراحل أساسية.

---

## High-Level Architecture

```mermaid
graph TB
    subgraph Client["🖥️ Frontend / Client"]
        FE[Next.js + TypeScript]
        WS[WebSocket Connection]
    end

    subgraph Security["🛡️ Security Layer"]
        WAF[WAF - Web Application Firewall]
        LB[Load Balancer]
        RL[Rate Limiter]
    end

    subgraph Gateway["🚪 API Gateway"]
        AG[Nginx / Kong]
    end

    subgraph Services["⚙️ Microservices"]
        AUTH[Auth Service]
        EXAM[Exam Service]
        LAB[Lab Service]
        INTERVIEW[Interview Service]
    end

    subgraph DataLayer["💾 Data Layer"]
        PG[(PostgreSQL)]
        REDIS[(Redis Cache<br/>State + Timer + Auto-save)]
    end

    subgraph AsyncLayer["📨 Async Processing"]
        MQ[Message Broker<br/>RabbitMQ / SQS]
    end

    subgraph Workers["👷 Background Workers"]
        VW[Video Processing<br/>Workers]
        LW[Lab Workers<br/>K8s Jobs]
        EW[Evaluation Workers]
    end

    subgraph Isolation["🔒 Sandboxed Environments"]
        FC[Firecracker MicroVMs<br/>/ gVisor]
        NP[Network Policies<br/>No Internet / No Internal Access]
    end

    subgraph Storage["📦 Object Storage"]
        S3[AWS S3 / MinIO]
        PRE[Pre-signed URLs]
    end

    subgraph Monitoring["📊 Observability"]
        PROM[Prometheus]
        GRAF[Grafana]
        AUDIT[Audit Trail<br/>Immutable Logs]
    end

    %% Flow
    FE --> WAF
    WS --> WAF
    WAF --> LB
    LB --> RL
    RL --> AG

    AG --> AUTH
    AG --> EXAM
    AG --> LAB
    AG --> INTERVIEW

    AUTH --> PG
    EXAM --> PG
    EXAM --> REDIS
    LAB --> MQ
    INTERVIEW --> MQ

    MQ --> VW
    MQ --> LW
    MQ --> EW

    LW --> FC
    FC --> NP
    EW --> PG

    VW --> S3
    FE -.->|Direct Upload| PRE
    PRE --> S3
    S3 -.->|Webhook| INTERVIEW

    EXAM --> AUDIT
    LAB --> AUDIT
    AUTH --> AUDIT

    PROM --> GRAF
    Services --> PROM
```

---

## Assessment Phases

### Phase 1: MCQ / Written Assessment
- Multiple Choice + Written Questions
- **Server-Side Timer** عبر WebSocket (مش Client-Side عشان محدش يتلاعب)
- Full Screen Mode مع نظام Violations
- منع Copy/Paste و Right Click
- تسجيل كل أحداث الغش (Focus Lost, Tab Change, Fullscreen Exit)
- إلغاء الامتحان بعد عدد معين من المخالفات (configurable)
- **Auto-save** كل 10 ثواني للـ Redis

### Phase 2: Linux Practical Lab
- كل ممتحن يتعمله **Firecracker MicroVM** مستقلة (مش Docker عادي)
- Network Policies صارمة — لا إنترنت، لا وصول للـ Internal Network
- الـ Evaluation Script يشتغل على **Snapshot** من الـ Filesystem
- تقييم تلقائي + تخزين النتيجة كـ JSON

**Workflow:**
```mermaid
sequenceDiagram
    participant C as Candidate
    participant API as API Gateway
    participant MQ as Message Broker
    participant W as Lab Worker
    participant VM as Firecracker VM
    participant EV as Evaluation Worker
    participant DB as PostgreSQL

    C->>API: Start Lab Exam
    API->>MQ: Publish "create-lab" message
    API-->>C: 202 Accepted (preparing...)
    MQ->>W: Consume message
    W->>VM: Create MicroVM
    W-->>C: WebSocket: Lab Ready + Terminal Access
    Note over C,VM: Candidate works on tasks...
    Note over VM: Timer expires
    VM->>W: Timeout signal
    W->>VM: Snapshot filesystem
    W->>VM: Destroy VM
    W->>MQ: Publish "evaluate-lab" message
    MQ->>EV: Consume message
    EV->>EV: Run evaluation script on snapshot
    EV->>DB: Save score + report JSON
    EV-->>C: WebSocket: Results ready
```

### Phase 3: One-Way HR Interview
- أسئلة عشوائية أو ثابتة
- تسجيل فيديو مباشر
- رفع الفيديو عبر **Pre-signed URLs** مباشرة للـ S3
- S3 Webhook يبلغ الـ Backend إن الفيديو جاهز

**Video Upload Flow:**
```mermaid
sequenceDiagram
    participant C as Candidate
    participant API as Backend
    participant S3 as Object Storage

    C->>API: Request upload permission
    API->>S3: Generate Pre-signed URL
    API-->>C: Return Pre-signed URL
    C->>S3: Direct upload video
    S3->>API: Webhook: upload complete
    API->>API: Link video to user record
```

---

## Database Schema

```mermaid
erDiagram
    Users {
        uuid id PK
        string full_name
        string national_id_hash UK
        bytea national_id_encrypted
        string phone
        string email
        timestamp created_at
    }

    Exams {
        uuid id PK
        string title
        enum type "MCQ | Written | Lab | Interview"
        int duration_minutes
        boolean active
        timestamp created_at
    }

    QuestionBank {
        uuid id PK
        uuid exam_id FK
        enum difficulty "Easy | Medium | Hard | Expert"
        enum question_type "MCQ | Written"
        text body
        jsonb options
        text answer
        int points
    }

    ExamSessions {
        uuid id PK
        uuid user_id FK
        uuid exam_id FK
        timestamp started_at
        timestamp finished_at
        int violations_count
        int score
        enum status "In_Progress | Completed | Cancelled | Violated"
    }

    ExamQuestionMapping {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        int display_order
        jsonb shuffled_options
    }

    Answers {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        text user_answer
        boolean is_correct
        int time_spent_seconds
        timestamp answered_at
    }

    ViolationEvents {
        uuid id PK
        uuid session_id FK
        enum type "Focus_Lost | Tab_Change | Fullscreen_Exit | Copy_Paste"
        timestamp occurred_at
        jsonb metadata
    }

    LabSessions {
        uuid id PK
        uuid user_id FK
        string container_id
        string microvm_id
        timestamp started_at
        timestamp ended_at
        int score
        jsonb report_json
        enum status "Running | Evaluating | Completed | Timeout"
    }

    InterviewSessions {
        uuid id PK
        uuid user_id FK
        string video_path
        int duration_seconds
        int hr_score
        timestamp recorded_at
    }

    AuditLog {
        uuid id PK
        uuid user_id FK
        string action
        jsonb payload
        timestamp created_at
    }

    Users ||--o{ ExamSessions : takes
    Users ||--o{ LabSessions : takes
    Users ||--o{ InterviewSessions : records
    Exams ||--o{ QuestionBank : contains
    Exams ||--o{ ExamSessions : has
    ExamSessions ||--o{ ExamQuestionMapping : includes
    ExamSessions ||--o{ Answers : has
    ExamSessions ||--o{ ViolationEvents : logs
    QuestionBank ||--o{ ExamQuestionMapping : mapped_to
    Users ||--o{ AuditLog : generates
```

---

## Key Design Decisions

### 1. Security - Lab Isolation
| Approach | Risk Level | Notes |
|----------|-----------|-------|
| Docker (raw) | ❌ High | Container breakout possible |
| Docker + gVisor | ✅ Medium | Kernel-level sandboxing |
| Firecracker MicroVM | ✅ Low | Full VM isolation, same as AWS Lambda |

### 2. National ID Protection
```
┌─────────────────────────────────────────────┐
│  national_id_hash   = SHA256(national_id)   │  → Unique Constraint (prevent duplicates)
│  national_id_encrypted = AES256(national_id)│  → HR can decrypt when needed
└─────────────────────────────────────────────┘
```

### 3. State Management (Exam Resilience)
```
Frontend → Auto-save every 10s → Redis
                                   ├── Current question index
                                   ├── Answers so far
                                   ├── Remaining time (server-side)
                                   └── Violation count

On reconnect → Read from Redis → Resume exactly where left off
```

### 4. Question Randomization
- أسئلة عشوائية من الـ Question Bank حسب التوزيع:
  - 5 Easy | 10 Medium | 4 Hard | 1 Expert
- Shuffle ترتيب الأسئلة
- Shuffle ترتيب الإجابات (MCQ)
- تتبع أي أسئلة اتعرضت على مين (ExamQuestionMapping)

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js, TypeScript, TailwindCSS |
| Real-time | WebSocket (Socket.IO) |
| Backend | FastAPI (Python) |
| Database | PostgreSQL |
| Cache | Redis |
| Message Broker | RabbitMQ / AWS SQS |
| Object Storage | AWS S3 / MinIO |
| Lab Isolation | Firecracker / gVisor |
| Orchestration | Kubernetes |
| Gateway | Nginx / Kong |
| Monitoring | Prometheus + Grafana |
| Security | WAF + Rate Limiter |
| Audit | Immutable Append-only Logs |

---

## Scalability Considerations

- **Horizontal Scaling**: كل Service يقدر يتعمله Scale مستقل
- **Message Broker**: يضمن إن الـ System ميقعش تحت الضغط (Spiky Traffic)
- **Pre-signed URLs**: الـ Backend مش بيتحمل bandwidth رفع الفيديوهات
- **Redis**: يقلل الـ Load على PostgreSQL للعمليات المتكررة (Timer, State)
- **K8s Jobs**: كل Lab بيتعمله Job مستقلة بتمسح نفسها بعد ما تخلص

---

## Future Enhancements
- AI Evaluation للـ Linux Lab (تقييم ذكي بدل Scripts ثابتة)
- AI Analysis للـ HR Interview (تحليل لغة الجسد والإجابات)
- توليد امتحانات جديدة تلقائياً من Question Bank
- Dashboard للإحصائيات وتقارير التوظيف
- Ranking System للممتحنين
- Video Storage Retention Policy (نقل لـ Cold Storage بعد 90 يوم)





بدايه في الاول






______________________________________________________________________________________________________________________________________________________________________________________________________________________________________________________

# Technical Assessment Platform - MVP

## Overview
منصة تقييم تقني لاختبار المتقدمين للوظائف عبر 3 مراحل: MCQ/Written، Linux Lab، One-Way Interview.

**Target:** 100 user/day | Single VPS | ~$25-50/mo

---

## Architecture

```mermaid
graph TB
    subgraph Client["🖥️ Frontend (Vercel)"]
        FE[Next.js + TypeScript + TailwindCSS]
        WS[WebSocket Client]
    end

    subgraph VPS["🖧 Single VPS (4 vCPU / 8GB RAM)"]
        NG[Nginx<br/>Reverse Proxy + Rate Limiting + SSL]
        API[FastAPI Monolith]
        CELERY[Celery Worker]
        DOCKER[Docker + gVisor<br/>Lab Containers]
        PG[(PostgreSQL)]
        REDIS[(Redis<br/>Cache + State + Timer)]
    end

    subgraph External["☁️ External Services"]
        S3[Cloudflare R2<br/>Video + File Storage]
        SENTRY[Sentry<br/>Error Tracking]
    end

    FE -->|HTTPS| NG
    WS -->|WSS| NG
    NG --> API
    API --> PG
    API --> REDIS
    API --> CELERY
    CELERY --> DOCKER
    CELERY --> S3
    FE -.->|Pre-signed Upload| S3
    API --> SENTRY
```

---

## Assessment Phases

### Phase 1: MCQ / Written Assessment

```mermaid
sequenceDiagram
    participant C as Candidate Browser
    participant API as FastAPI
    participant R as Redis
    participant DB as PostgreSQL

    C->>API: Start Exam (JWT)
    API->>DB: Create ExamSession
    API->>DB: Select random questions
    API->>R: Store state (timer, answers, violations)
    API-->>C: Questions + WebSocket Timer

    loop Every 10 seconds
        C->>R: Auto-save (current answer + position)
    end

    Note over C: Violation detected (tab switch)
    C->>API: Report violation
    API->>R: Increment violation count
    API-->>C: Warning (or cancel if max reached)

    C->>API: Submit answer
    API->>DB: Save to Answers table
    API->>R: Update state

    Note over C: Exam finished / Timer expired
    API->>DB: Calculate score + save
    API-->>C: Results
```

**Features:**
- Full Screen Mode (Fullscreen API)
- Server-Side Timer عبر WebSocket
- Violation System (focus lost, tab change, fullscreen exit, copy/paste)
- Auto-save كل 10 ثواني للـ Redis
- Random question selection + shuffling
- Auto-grading for MCQ
- لو النت فصل → يرجع من Redis لنفس النقطة

---

### Phase 2: Linux Practical Lab

```mermaid
sequenceDiagram
    participant C as Candidate
    participant API as FastAPI
    participant W as Celery Worker
    participant D as Docker Container
    participant DB as PostgreSQL

    C->>API: Start Lab
    API->>W: Task: create_lab_container
    API-->>C: "Preparing..." (202)
    W->>D: docker run --runtime=runsc --network=none --memory=512m
    W-->>C: WebSocket: Terminal Ready
    Note over C,D: Candidate works via xterm.js WebSocket
    Note over D: Timer expires (Celery scheduled task)
    W->>D: docker cp (snapshot filesystem)
    W->>D: docker stop + docker rm
    W->>W: Run evaluation script on snapshot
    W->>DB: Save score + report_json
    W-->>C: WebSocket: Results ready
```

**Security:**
```
Docker flags:
  --runtime=runsc          # gVisor kernel isolation
  --network=none           # لا إنترنت
  --memory=512m            # Max RAM
  --cpus=1                 # Max CPU
  --pids-limit=100         # Prevent fork bombs
  --read-only              # Root filesystem read-only
  --tmpfs /tmp:size=100m   # Writable tmp only
```

**Evaluation يشتغل على Snapshot مش على الـ Container مباشرة** — عشان الممتحن ميقدرش يضلل الـ Script.

---

### Phase 3: One-Way HR Interview

```mermaid
sequenceDiagram
    participant C as Candidate
    participant API as FastAPI
    participant R2 as Cloudflare R2

    C->>API: Request upload URL
    API->>R2: Generate Pre-signed URL
    API-->>C: Pre-signed URL
    Note over C: Record video (MediaRecorder API)
    C->>R2: Direct upload video
    R2-->>API: Webhook: upload complete
    API->>API: Link video to InterviewSession
```

**Features:**
- أسئلة عشوائية أو ثابتة تظهر واحد واحد
- تسجيل فيديو عبر المتصفح (MediaRecorder API)
- رفع مباشر للـ R2 بدون مرور على الـ Backend
- HR Review interface لمشاهدة وتقييم الفيديوهات

---

## Database Schema

```mermaid
erDiagram
    Users {
        uuid id PK
        string full_name
        string national_id_hash UK
        bytea national_id_encrypted
        string phone
        string email
        timestamp created_at
    }

    Exams {
        uuid id PK
        string title
        enum type "MCQ | Written | Lab | Interview"
        int duration_minutes
        boolean active
        timestamp created_at
    }

    QuestionBank {
        uuid id PK
        uuid exam_id FK
        enum difficulty "Easy | Medium | Hard | Expert"
        enum question_type "MCQ | Written"
        text body
        jsonb options
        text answer
        int points
    }

    ExamSessions {
        uuid id PK
        uuid user_id FK
        uuid exam_id FK
        timestamp started_at
        timestamp finished_at
        int violations_count
        int score
        enum status "In_Progress | Completed | Cancelled | Violated"
    }

    ExamQuestionMapping {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        int display_order
        jsonb shuffled_options
    }

    Answers {
        uuid id PK
        uuid session_id FK
        uuid question_id FK
        text user_answer
        boolean is_correct
        int time_spent_seconds
        timestamp answered_at
    }

    ViolationEvents {
        uuid id PK
        uuid session_id FK
        enum type "Focus_Lost | Tab_Change | Fullscreen_Exit | Copy_Paste"
        timestamp occurred_at
        jsonb metadata
    }

    LabSessions {
        uuid id PK
        uuid user_id FK
        string container_id
        timestamp started_at
        timestamp ended_at
        int score
        jsonb report_json
        enum status "Running | Evaluating | Completed | Timeout"
    }

    InterviewSessions {
        uuid id PK
        uuid user_id FK
        string video_path
        int duration_seconds
        int hr_score
        timestamp recorded_at
    }

    AuditLog {
        uuid id PK
        uuid user_id FK
        string action
        jsonb payload
        timestamp created_at
    }

    Users ||--o{ ExamSessions : takes
    Users ||--o{ LabSessions : takes
    Users ||--o{ InterviewSessions : records
    Exams ||--o{ QuestionBank : contains
    Exams ||--o{ ExamSessions : has
    ExamSessions ||--o{ ExamQuestionMapping : includes
    ExamSessions ||--o{ Answers : has
    ExamSessions ||--o{ ViolationEvents : logs
    QuestionBank ||--o{ ExamQuestionMapping : mapped_to
    Users ||--o{ AuditLog : generates
```

---

## Tech Stack

| Layer | Technology | Cost |
|-------|-----------|------|
| Frontend | Next.js + TypeScript + TailwindCSS (Vercel Free) | $0 |
| Backend | FastAPI (Python) | included |
| Task Queue | Celery + Redis as broker | included |
| Database | PostgreSQL 16 | included |
| Cache/State | Redis 7 | included |
| Lab Runtime | Docker + gVisor (runsc) | included |
| Web Terminal | xterm.js + WebSocket | included |
| Storage | Cloudflare R2 (10GB Free) | $0-5/mo |
| Reverse Proxy | Nginx + Let's Encrypt | included |
| Server | Single VPS (4 vCPU, 8GB RAM) | ~$20-40/mo |
| Error Tracking | Sentry Free Tier | $0 |
| Domain | Cloudflare | ~$10/yr |
| **Total** | | **~$25-50/mo** |

---

## Infrastructure

### docker-compose.yml

```yaml
version: "3.8"

services:
  api:
    build: ./backend
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/assessments
      - REDIS_URL=redis://redis:6379
      - S3_ENDPOINT=https://xxx.r2.cloudflarestorage.com
      - S3_ACCESS_KEY=${S3_ACCESS_KEY}
      - S3_SECRET_KEY=${S3_SECRET_KEY}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - db
      - redis
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

  celery-worker:
    build: ./backend
    command: celery -A app.worker worker --loglevel=info --concurrency=4
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/assessments
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    restart: unless-stopped

  celery-beat:
    build: ./backend
    command: celery -A app.worker beat --loglevel=info
    environment:
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=assessments
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - api
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

### nginx.conf (Key Parts)

```nginx
upstream api {
    server api:8000;
}

server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate     /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;

    # API
    location /api/ {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # WebSocket
    location /ws/ {
        proxy_pass http://api;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## Security (Day 1)

```
✅ HTTPS (Let's Encrypt via Certbot)
✅ JWT with refresh tokens (short-lived access tokens)
✅ National ID: SHA256 Hash (unique) + AES-256 encrypted (readable)
✅ Rate limiting: 10 req/s per IP (Nginx)
✅ Lab containers: --runtime=runsc (gVisor)
✅ Lab containers: --network=none (no internet access)
✅ Lab containers: --memory=512m --cpus=1 --pids-limit=100
✅ Input validation: Pydantic models on every endpoint
✅ SQL injection: SQLAlchemy ORM (no raw queries)
✅ CORS: whitelist frontend domain only
✅ Secrets: environment variables (never in code)
✅ Evaluation: runs on filesystem snapshot (not live container)
```

---

## Key Design Decisions

### National ID Protection
```
┌─────────────────────────────────────────────┐
│  national_id_hash   = SHA256(id)            │  → Unique Constraint
│  national_id_encrypted = AES256(id)         │  → HR can decrypt
└─────────────────────────────────────────────┘
```

### State Management (Disconnect Resilience)
```
Frontend → Auto-save every 10s → Redis
                                   ├── Current question index
                                   ├── Answers submitted
                                   ├── Remaining time (server-authoritative)
                                   └── Violation count

Internet drops → Reconnect → Read Redis → Resume same point
```

### Question Randomization
- توزيع: 5 Easy | 10 Medium | 4 Hard | 1 Expert
- Shuffle ترتيب الأسئلة
- Shuffle ترتيب الإجابات (MCQ)
- ExamQuestionMapping يتتبع أي أسئلة اتعرضت على مين

---

## Project Structure

```
project/
├── docker-compose.yml
├── .env                         # Secrets (not in git)
├── .env.example                 # Template
├── nginx/
│   ├── nginx.conf
│   └── certs/
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   ├── alembic/                 # DB migrations
│   │   └── versions/
│   ├── app/
│   │   ├── main.py              # FastAPI app + WebSocket
│   │   ├── config.py            # Settings (from env)
│   │   ├── dependencies.py      # DI (db session, current user)
│   │   ├── models/              # SQLAlchemy models
│   │   │   ├── user.py
│   │   │   ├── exam.py
│   │   │   ├── question.py
│   │   │   └── session.py
│   │   ├── schemas/             # Pydantic schemas
│   │   │   ├── auth.py
│   │   │   ├── exam.py
│   │   │   └── lab.py
│   │   ├── api/                 # Route handlers
│   │   │   ├── auth.py
│   │   │   ├── exams.py
│   │   │   ├── labs.py
│   │   │   ├── interviews.py
│   │   │   └── admin.py
│   │   ├── services/            # Business logic
│   │   │   ├── auth_service.py
│   │   │   ├── exam_service.py
│   │   │   ├── lab_service.py
│   │   │   └── evaluation.py
│   │   ├── worker.py            # Celery tasks
│   │   └── websocket.py         # WS handlers (timer, terminal)
│   ├── evaluation_scripts/
│   │   ├── linux_basics.sh
│   │   └── nginx_setup.sh
│   └── tests/
│       ├── test_auth.py
│       ├── test_exam.py
│       └── test_lab.py
├── frontend/
│   ├── package.json
│   ├── next.config.js
│   ├── src/
│   │   ├── app/                 # Next.js App Router
│   │   │   ├── page.tsx         # Landing
│   │   │   ├── login/
│   │   │   ├── register/
│   │   │   ├── exam/
│   │   │   ├── lab/
│   │   │   ├── interview/
│   │   │   └── admin/
│   │   ├── components/
│   │   │   ├── ExamScreen.tsx   # Fullscreen exam UI
│   │   │   ├── Terminal.tsx     # xterm.js wrapper
│   │   │   ├── VideoRecorder.tsx
│   │   │   └── Timer.tsx
│   │   └── lib/
│   │       ├── api.ts           # Axios/fetch wrapper
│   │       ├── websocket.ts     # WS connection manager
│   │       └── auth.ts          # JWT handling
│   └── public/
└── lab-images/
    ├── linux-basics/
    │   └── Dockerfile
    └── nginx-setup/
        └── Dockerfile
```

---

## Sprint Plan

### Sprint 1 (Week 1-2): Auth + Admin
- [ ] User Registration + Login (JWT + Refresh Token)
- [ ] National ID validation (Hash + AES-256)
- [ ] Admin panel لإدارة الأسئلة والامتحانات
- [ ] Question Bank CRUD
- [ ] Docker Compose setup + deployment script

### Sprint 2 (Week 3-4): MCQ/Written Exam
- [ ] Full Screen Exam Mode
- [ ] Server-side Timer (WebSocket)
- [ ] Violation Detection + Violation System
- [ ] Auto-save to Redis (every 10s)
- [ ] Random question selection + shuffling
- [ ] Auto-grading for MCQ
- [ ] Reconnection resilience

### Sprint 3 (Week 5-6): Linux Lab
- [ ] Docker + gVisor container creation
- [ ] Web terminal (xterm.js + WebSocket)
- [ ] Timer + auto-destroy (Celery Beat)
- [ ] Filesystem snapshot + evaluation script
- [ ] Score calculation + JSON report
- [ ] Build 2-3 lab images

### Sprint 4 (Week 7-8): One-Way Interview
- [ ] Video recording UI (MediaRecorder API)
- [ ] Pre-signed URL upload to R2
- [ ] Link video to user session
- [ ] HR review interface (watch + score)
- [ ] Question display (one at a time)

### Sprint 5 (Week 9-10): Polish + Launch
- [ ] Results dashboard (candidate + admin views)
- [ ] Email notifications (exam invite, results)
- [ ] Basic analytics (pass rate, avg score)
- [ ] Security audit + hardening
- [ ] Load testing (simulate 100 users)
- [ ] Deploy to production VPS

---

## Scaling Roadmap

لما المنصة تكبر، هتحتاج تضيف حاجات تدريجياً:

| Signal | Action |
|--------|--------|
| > 500 users/day | أضف Load Balancer + سيرفر ثاني |
| > 50 concurrent labs | انقل Labs لسيرفر مستقل (8+ vCPU) |
| > 2000 users/day | فصّل Microservices |
| > 100 concurrent labs | انتقل لـ Kubernetes + Firecracker |
| Revenue > $5K/mo | أضف WAF + Full Monitoring (Prometheus/Grafana) |
| Enterprise clients | SOC2 Compliance + Audit Logs |

---

## Deployment (First Time)

```bash
# 1. SSH to VPS
ssh root@your-server

# 2. Install Docker + Docker Compose
curl -fsSL https://get.docker.com | sh
apt install docker-compose-plugin

# 3. Install gVisor
curl -fsSL https://gvisor.dev/archive.key | gpg --dearmor -o /usr/share/keyrings/gvisor-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/gvisor-archive-keyring.gpg] https://storage.googleapis.com/gvisor/releases release main" | tee /etc/apt/sources.list.d/gvisor.list
apt update && apt install -y runsc
# Configure Docker to use gVisor
cat > /etc/docker/daemon.json << 'EOF'
{
  "runtimes": {
    "runsc": {
      "path": "/usr/bin/runsc"
    }
  }
}
EOF
systemctl restart docker

# 4. Clone project + setup
git clone your-repo /opt/assessment-platform
cd /opt/assessment-platform
cp .env.example .env
# Edit .env with real secrets

# 5. Build lab images
docker build -t lab-linux-basics ./lab-images/linux-basics/

# 6. Start everything
docker compose up -d

# 7. Run migrations
docker compose exec api alembic upgrade head

# 8. Create admin user
docker compose exec api python -m app.create_admin
```

---

## Sample Lab Image (Dockerfile)

```dockerfile
# lab-images/linux-basics/Dockerfile
FROM ubuntu:22.04

RUN apt-get update && apt-get install -y \
    nginx \
    vim \
    curl \
    net-tools \
    systemctl \
    && rm -rf /var/lib/apt/lists/*

# Setup tasks description
COPY tasks.md /home/candidate/TASKS.md
COPY .bashrc /root/.bashrc

WORKDIR /home/candidate
CMD ["/bin/bash"]
```

## Sample Evaluation Script

```bash
#!/bin/bash
# evaluation_scripts/linux_basics.sh
# Runs on filesystem snapshot, not live container

SNAPSHOT_PATH=$1
score=0
total=100
report=""

# Task 1: Nginx installed and configured (20 points)
if [ -f "$SNAPSHOT_PATH/etc/nginx/nginx.conf" ]; then
    score=$((score + 10))
    report="$report\n✅ Nginx config exists (+10)"
fi
if grep -q "proxy_pass" "$SNAPSHOT_PATH/etc/nginx/conf.d/default.conf" 2>/dev/null; then
    score=$((score + 10))
    report="$report\n✅ Proxy pass configured (+10)"
fi

# Task 2: Backup created (20 points)
if [ -f "$SNAPSHOT_PATH/backup/data.tar.gz" ]; then
    score=$((score + 20))
    report="$report\n✅ Backup file created (+20)"
fi

# Task 3: User created (20 points)
if grep -q "devops" "$SNAPSHOT_PATH/etc/passwd" 2>/dev/null; then
    score=$((score + 20))
    report="$report\n✅ User 'devops' created (+20)"
fi

# Task 4: Cron job (20 points)
if [ -f "$SNAPSHOT_PATH/var/spool/cron/crontabs/root" ]; then
    score=$((score + 20))
    report="$report\n✅ Cron job configured (+20)"
fi

# Task 5: Firewall rules (20 points)
if [ -f "$SNAPSHOT_PATH/etc/iptables/rules.v4" ]; then
    score=$((score + 20))
    report="$report\n✅ Firewall rules saved (+20)"
fi

# Output JSON
echo "{\"score\": $score, \"total\": $total, \"report\": \"$report\"}"
```

---
