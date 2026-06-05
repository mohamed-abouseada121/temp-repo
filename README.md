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

## Technical Assessment Platform - MVP

## Overview
منصة تقييم تقني لاختبار المتقدمين للوظائف عبر 3 مراحل: MCQ/Written، Linux Lab، One-Way Interview.

**Target:** 100 user/day | Single VPS | ~$25-50/mo

---

## Registration & Eligibility Flow

### القواعد:
1. المستخدم **لازم يسجل أولاً** ويدخل الرقم القومي
2. الرقم القومي **لا يتكرر** — شخص واحد = حساب واحد
3. لو فشل في امتحان → **ممنوع يدخله تاني إلا بعد 3 شهور**
4. الـ Cooldown بالامتحان مش بالمنصة (لو فشل في MCQ يقدر يدخل Lab عادي)

```mermaid
sequenceDiagram
    participant U as User
    participant API as Backend
    participant DB as PostgreSQL

    U->>API: Register (name, email, national_id, phone)
    API->>API: SHA256(national_id) → check unique
    alt National ID already exists
        API-->>U: ❌ "هذا الرقم مسجل مسبقاً"
    else New user
        API->>DB: Store user (hash + encrypted national_id)
        API-->>U: ✅ Account created + JWT
    end

    Note over U,API: ───── Later: User wants to take exam ─────

    U->>API: Request to start Exam X
    API->>DB: Check ExamSessions WHERE user_id AND exam_id AND status='Failed'
    API->>API: Last failed attempt date + 90 days > today?
    alt Cooldown active
        API-->>U: ❌ "ممنوع إعادة الامتحان قبل {date}"
    else Eligible
        API->>DB: Create new ExamSession
        API-->>U: ✅ Start exam
    end
```

### الـ Cooldown Logic (Backend):

```python
from datetime import datetime, timedelta

COOLDOWN_DAYS = 90

async def check_eligibility(user_id: str, exam_id: str, db: Session) -> dict:
    last_failed = db.query(ExamSession).filter(
        ExamSession.user_id == user_id,
        ExamSession.exam_id == exam_id,
        ExamSession.status == "Failed"
    ).order_by(ExamSession.finished_at.desc()).first()

    if last_failed:
        eligible_date = last_failed.finished_at + timedelta(days=COOLDOWN_DAYS)
        if datetime.utcnow() < eligible_date:
            return {
                "eligible": False,
                "retry_after": eligible_date.isoformat(),
                "message": f"ممنوع إعادة الامتحان قبل {eligible_date.strftime('%Y-%m-%d')}"
            }

    return {"eligible": True}
```

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
    API->>DB: Check eligibility (cooldown)
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
    alt Score >= passing grade
        API->>DB: status = "Passed"
    else Score < passing grade
        API->>DB: status = "Failed" (cooldown starts)
    end
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
    API->>DB: Check eligibility (cooldown)
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
    alt Score >= passing grade
        W->>DB: status = "Passed"
    else Score < passing grade
        W->>DB: status = "Failed" (cooldown starts)
    end
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

**Note:** الـ Interview مفيهوش Pass/Fail تلقائي — HR هو اللي بيقيّم يدوي.

---

## Database Schema

```mermaid
erDiagram
    Users {
        uuid id PK
        string full_name
        string national_id_hash UK
        bytea national_id_encrypted
        string phone UK
        string email UK
        string password_hash
        boolean is_admin
        timestamp created_at
    }

    Exams {
        uuid id PK
        string title
        enum type "MCQ | Written | Lab | Interview"
        int duration_minutes
        int passing_score
        int cooldown_days "default 90"
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
        enum status "In_Progress | Passed | Failed | Cancelled | Violated"
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
        uuid exam_id FK
        string container_id
        timestamp started_at
        timestamp ended_at
        int score
        jsonb report_json
        enum status "Running | Evaluating | Passed | Failed | Timeout"
    }

    InterviewSessions {
        uuid id PK
        uuid user_id FK
        uuid exam_id FK
        string video_path
        int duration_seconds
        int hr_score
        enum status "Recorded | Under_Review | Reviewed"
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
    Exams ||--o{ LabSessions : has
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

## Infrastructure (docker-compose.yml)

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
      - S3_ENDPOINT=${S3_ENDPOINT}
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

---

## Security (Day 1)

```
✅ HTTPS (Let's Encrypt via Certbot)
✅ JWT with refresh tokens
✅ National ID: SHA256 Hash (unique constraint) + AES-256 encrypted
✅ Rate limiting: 10 req/s per IP (Nginx)
✅ Lab containers: --runtime=runsc (gVisor)
✅ Lab containers: --network=none
✅ Lab containers: --memory=512m --cpus=1 --pids-limit=100
✅ Input validation: Pydantic models
✅ SQL injection: SQLAlchemy ORM
✅ CORS: whitelist frontend domain only
✅ Secrets: .env file (never committed)
✅ Evaluation: runs on snapshot not live container
✅ Cooldown enforcement: server-side (can't bypass from frontend)
```

---

## Key Design Decisions

### National ID Protection
```
Registration:
  national_id → SHA256(national_id) → store as national_id_hash (UNIQUE)
  national_id → AES256(national_id) → store as national_id_encrypted
  
Duplicate check: compare hash (fast, O(1) with index)
HR needs original: decrypt with ENCRYPTION_KEY
```

### Exam Cooldown (3 Months)
```
ExamSessions table:
  status = "Failed" + finished_at = "2026-03-01"
  
User tries again on 2026-04-15:
  eligible_date = 2026-03-01 + 90 days = 2026-05-30
  today (2026-04-15) < eligible_date → ❌ BLOCKED

User tries again on 2026-06-05:
  today (2026-06-05) > eligible_date → ✅ ALLOWED
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
├── .env.example
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
│   │   │   ├── auth.py          # Register + Login + National ID check
│   │   │   ├── exams.py         # Start exam + eligibility check
│   │   │   ├── labs.py
│   │   │   ├── interviews.py
│   │   │   └── admin.py
│   │   ├── services/
│   │   │   ├── auth_service.py
│   │   │   ├── eligibility.py   # Cooldown logic
│   │   │   ├── exam_service.py
│   │   │   ├── lab_service.py
│   │   │   └── evaluation.py
│   │   ├── worker.py            # Celery tasks
│   │   └── websocket.py         # Timer + Terminal
│   ├── evaluation_scripts/
│   │   ├── linux_basics.sh
│   │   └── nginx_setup.sh
│   └── tests/
├── frontend/
│   ├── package.json
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx
│   │   │   ├── register/
│   │   │   ├── login/
│   │   │   ├── exam/
│   │   │   ├── lab/
│   │   │   ├── interview/
│   │   │   └── admin/
│   │   ├── components/
│   │   │   ├── ExamScreen.tsx
│   │   │   ├── Terminal.tsx
│   │   │   ├── VideoRecorder.tsx
│   │   │   └── CooldownNotice.tsx
│   │   └── lib/
│   │       ├── api.ts
│   │       ├── websocket.ts
│   │       └── auth.ts
│   └── public/
└── lab-images/
    ├── linux-basics/Dockerfile
    └── nginx-setup/Dockerfile
```

---

## Sprint Plan

### Sprint 1 (Week 1-2): Registration + Auth
- [ ] User Registration (name, email, phone, national_id)
- [ ] National ID: Hash + AES-256 + Unique check
- [ ] Login (JWT + Refresh Token)
- [ ] Eligibility service (cooldown logic)
- [ ] Admin panel لإدارة الأسئلة والامتحانات
- [ ] Question Bank CRUD
- [ ] Docker Compose setup

### Sprint 2 (Week 3-4): MCQ/Written Exam
- [ ] Eligibility check before starting
- [ ] Full Screen Exam Mode
- [ ] Server-side Timer (WebSocket)
- [ ] Violation Detection + System
- [ ] Auto-save to Redis
- [ ] Random question selection + shuffling
- [ ] Auto-grading → Pass/Fail + Cooldown trigger

### Sprint 3 (Week 5-6): Linux Lab
- [ ] Eligibility check before starting
- [ ] Docker + gVisor container creation
- [ ] Web terminal (xterm.js + WebSocket)
- [ ] Timer + auto-destroy (Celery Beat)
- [ ] Filesystem snapshot + evaluation
- [ ] Score → Pass/Fail + Cooldown trigger

### Sprint 4 (Week 7-8): One-Way Interview
- [ ] Video recording UI (MediaRecorder API)
- [ ] Pre-signed URL upload to R2
- [ ] Link video to user session
- [ ] HR review interface (watch + score)

### Sprint 5 (Week 9-10): Polish + Launch
- [ ] Results dashboard + cooldown status display
- [ ] Email notifications (invite, results, cooldown expiry)
- [ ] Basic analytics
- [ ] Security audit
- [ ] Load testing (100 users)
- [ ] Deploy to production

---

## API Endpoints (Key)

```
POST   /api/auth/register          # Register + National ID check
POST   /api/auth/login             # Login → JWT
POST   /api/auth/refresh           # Refresh token

GET    /api/exams                  # List available exams
GET    /api/exams/{id}/eligibility # Check if user can take exam
POST   /api/exams/{id}/start       # Start exam (checks cooldown)
POST   /api/exams/{id}/answer      # Submit answer
POST   /api/exams/{id}/finish      # End exam

POST   /api/labs/{id}/start        # Start lab (checks cooldown)
GET    /api/labs/{id}/status       # Lab status
POST   /api/labs/{id}/stop         # Manual stop

POST   /api/interviews/{id}/upload-url   # Get pre-signed URL
POST   /api/interviews/{id}/complete     # Mark as recorded

GET    /api/admin/users            # List users
GET    /api/admin/results          # All results
GET    /api/admin/results/{user}   # User results + cooldown info
```

---

## Scaling Roadmap

| Signal | Action |
|--------|--------|
| > 500 users/day | أضف Load Balancer + سيرفر ثاني |
| > 50 concurrent labs | انقل Labs لسيرفر مستقل |
| > 2000 users/day | فصّل Microservices |
| > 100 concurrent labs | Kubernetes + Firecracker |
| Revenue > $5K/mo | WAF + Full Monitoring |

