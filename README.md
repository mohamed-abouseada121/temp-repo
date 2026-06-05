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
