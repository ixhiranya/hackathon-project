# CertiFlow -- System Architecture

## 1. Purpose and scope

CertiFlow manages the certificate lifecycle for the Pragati Skills
Mission: batch intake, validation, district review, quality review for
large batches, state approval, certificate issuance, WhatsApp delivery,
cancellation and public verification.

This document records the proposed architecture and explicitly
distinguishes problem-statement requirements from implementation
choices.

## 2. Requirements that shape the architecture

### Required by the problem statement

-   React + Vite frontend.
-   Node.js + Express + TypeScript backend.
-   PostgreSQL with Prisma or Drizzle and migrations.
-   Redis caching.
-   A queue or message broker for background jobs/events.
-   Containerized local execution with one command.
-   Automated tests for critical scenarios.
-   Secure role/data access and end-to-end traceability.
-   A simulated WhatsApp provider with failure and timeout behaviour.
-   Live updates without manual refresh.

### Proposed implementation choices

-   Prisma as ORM.
-   BullMQ backed by Redis for background jobs.
-   PostgreSQL transactional outbox to avoid losing events between
    database commits and queue publication.
-   Server-Sent Events (SSE) for one-way server-to-browser live updates.
-   Zod for request and import validation.
-   PDFKit for PDF generation and a QR-code library for verification
    links.
-   Local private volume for development certificate files; private
    object storage in production.
-   Vitest and Supertest for automated tests.

These are design decisions, not technologies mandated by the brief.

## 3. High-level architecture

``` mermaid
flowchart TB
    U[Centre / District / Quality / State Users] --> FE[React + Vite]
    P[Public / Employers] --> FE
    FE --> API[Express + TypeScript API]
    API --> AUTH[JWT Authentication + RBAC]
    AUTH --> SVC[Application Services / Workflow Rules]
    SVC <--> DB[(PostgreSQL)]
    SVC --> CACHE[(Redis Cache)]
    SVC --> OUTBOX[(Transactional Outbox)]
    OUTBOX --> PUB[Outbox Publisher]
    PUB --> Q[BullMQ Queues on Redis]
    Q --> W[Background Workers]
    W --> DB
    W --> FILES[Private Certificate File Storage]
    W --> WA[Simulated WhatsApp Provider]
    API --> SSE[SSE Endpoint]
    SSE --> FE
```

## 4. Component responsibilities

### 4.1 Frontend -- React + Vite

-   Role-aware dashboards for training centres, district officers,
    quality reviewers and state officers.
-   Batch creation, candidate import and row-level validation feedback.
-   Approval/review workspaces.
-   Certificate list, download and delivery status.
-   Public verification page that does not require login.
-   SSE client for live status updates.

The frontend is not a security boundary. Every protected operation and
data access must be checked by the backend.

### 4.2 API -- Node.js + Express + TypeScript

Responsibilities: - Authenticate users and authorize actions. - Validate
request bodies, query parameters and uploaded files. - Enforce batch
state transitions and business rules. - Execute database transactions. -
Create audit records. - Write outbox events in the same transaction as
the business change. - Expose REST endpoints and SSE updates.

Suggested internal layers: - Routes: URL and HTTP method mapping. -
Middleware: authentication, authorization, validation and error
handling. - Controllers: HTTP input/output handling. - Services:
business rules and transactions. - Repositories / Prisma client:
persistence. - Workers: asynchronous processing.

### 4.3 PostgreSQL

Stores durable system-of-record data: - Users, districts, training
centres and courses. - Batches, batch candidates and candidate
profiles. - Approval decisions and batch versions. - Certificates and
delivery attempts. - Audit logs and transactional outbox events.

Use database constraints as the final protection against duplicate
certificates and duplicate approvals. Application checks alone are not
sufficient under concurrent requests.

### 4.4 Redis and BullMQ

Redis supports: - BullMQ queue storage and worker coordination. -
Optional short-lived cache entries. - Optional rate-limit state.

BullMQ workers handle work that should not block an API request: - PDF
generation. - WhatsApp delivery and retry. - Notifications. - Outbox
event processing / downstream event dispatch. - Scheduled stuck-batch
checks.

Redis is not the source of truth for approvals, certificates or audit
history. Those remain in PostgreSQL.

### 4.5 Transactional outbox

For any state-changing transaction that must trigger asynchronous
work: 1. Update the business record in PostgreSQL. 2. Insert an outbox
event in the same PostgreSQL transaction. 3. Commit both together. 4. A
publisher reads pending outbox rows and publishes corresponding
jobs/events. 5. Mark the outbox event as published only after successful
publication.

If Redis is unavailable, the outbox event remains durable and can be
published later. The publisher must tolerate publishing the same event
more than once; consumers must be idempotent.

### 4.6 Certificate storage and verification

-   Store certificate metadata and a private file key in PostgreSQL.
-   Store PDF bytes in private local storage for development.
-   Do not expose filesystem paths or public bucket URLs.
-   The download endpoint authorizes access before streaming the PDF.
-   The public verification endpoint returns only the minimum
    information needed to establish authenticity.
-   Verification reads current certificate status from PostgreSQL (or
    uses a cache with strict invalidation); cancellation must be
    reflected immediately. For the MVP, prefer a database read to avoid
    stale "valid" cache results.

### 4.7 Simulated WhatsApp provider

Create an adapter interface around the stand-in provider. It should
simulate: - Successful delivery. - Explicit provider failure. -
Timeout. - Retryable and non-retryable errors.

Persist each delivery attempt. Use bounded exponential backoff with
jitter for retryable failures. Do not generate a new certificate when a
message delivery fails.

## 5. Core domain model

Recommended entities:

  Entity             Purpose
  ------------------ -------------------------------------------------
  User               Login identity and role
  District           Geographic assignment
  TrainingCentre     Centre submitting batches
  Course             Skill/course definition
  Batch              A submitted group of candidates
  Candidate          Person receiving certification
  BatchCandidate     Candidate's participation in a particular batch
  DistrictApproval   One officer's decision for a batch version
  QualityReview      Quality team's decision for a batch version
  StateApproval      State office's final decision
  Certificate        Issued credential and current status
  DeliveryAttempt    Each WhatsApp send attempt
  AuditLog           Append-only record of important actions
  OutboxEvent        Durable event awaiting publication

### Key relationships

-   District 1---many TrainingCentres.
-   District 1---many Users (district officers).
-   TrainingCentre 1---many Batches.
-   Course 1---many Batches.
-   Batch 1---many BatchCandidates.
-   Candidate 1---many BatchCandidates over time.
-   Batch 1---many DistrictApprovals, QualityReviews and StateApprovals
    across versions.
-   BatchCandidate 1---zero or one Certificate.
-   Certificate 1---many DeliveryAttempts.
-   User 1---many AuditLogs.
-   OutboxEvent stores a domain event and its publication state.

### Important constraints

-   Unique user email.
-   Unique batch number.
-   Unique certificate number.
-   Unique `(batch_candidate_id)` in certificates: at most one
    certificate per candidate's batch participation.
-   Unique `(batch_id, version, officer_id)` in district approvals.
-   Foreign keys for all relationships.
-   Check constraints for valid enum/status values where practical.

## 6. Batch workflow and state transitions

Proposed states:

``` text
DRAFT
  -> SUBMITTED
  -> DISTRICT_REVIEW
  -> QUALITY_REVIEW (only if threshold requires it)
  -> STATE_REVIEW
  -> APPROVED
  -> ISSUING
  -> ISSUED
```

Alternative transitions: - District or quality reviewer may return a
batch with a reason: `RETURNED`. - A permitted rejection moves the batch
to `REJECTED`. - A returned batch can be corrected and resubmitted as a
new version. - An overdue batch is flagged/escalated; its approval state
need not change.

The exact state names are an implementation choice. The business rules
must prevent skipping required approvals.

### District approval policy

-   At least two distinct district officers must approve the same
    submitted batch version.
-   An officer cannot approve the same version twice.
-   An officer cannot approve a batch outside their assigned district.
-   The approval operation uses a database transaction and
    concurrency-safe state check/locking.
-   A rejection or return requires a reason.

### Large-batch quality policy

The brief requires additional scrutiny for large batches but does not
define the threshold. Make it configurable, for example
`QUALITY_REVIEW_THRESHOLD=50`, and clearly label 50 as an MVP
assumption.

### Resubmission policy

Proposed policy: increment `batch.version`; keep old decisions for
audit, but do not count them toward the new version. All required
approvals are collected again for the new version.

## 7. Certificate issuance and delivery guarantees

When the state office gives final approval: 1. In a transaction, verify
the batch is at the correct version and all required approvals exist. 2.
Move the batch to `APPROVED` / `ISSUING` as defined by the
implementation. 3. Create one certificate record per eligible batch
candidate, protected by a unique constraint. 4. Create outbox events for
certificate generation and delivery workflow. 5. Workers generate PDF
files and store them privately. 6. Workers submit WhatsApp messages
through the simulated provider. 7. Persist every attempt and provider
result. 8. Mark the batch `ISSUED` when issuance processing reaches the
defined completion condition; delivery failures remain visible and
retryable.

"Exactly once" external message delivery cannot be guaranteed by the
application alone unless the provider supports idempotency. The system
should guarantee one logical certificate per eligible batch candidate,
deduplicate jobs and use provider idempotency keys when available.

## 8. Concurrency, idempotency and failure handling

  -----------------------------------------------------------------------
  Risk                                Design
  ----------------------------------- -----------------------------------
  Two officers approve concurrently   Transaction + row lock or
                                      conditional update; unique approval
                                      constraint

  Duplicate request                   Idempotency key for critical
                                      create/issue operations where
                                      appropriate

  Duplicate worker execution          Idempotent handlers and database
                                      uniqueness constraints

  Redis restart                       Outbox remains in PostgreSQL;
                                      publisher resumes

  Worker crash                        BullMQ retries/recovery; job
                                      handler safe to repeat

  WhatsApp timeout                    Persist attempt; retry with bounded
                                      backoff

  WhatsApp outage                     Queue backlog, alerting and
                                      controlled retry

  Slow database                       Connection pool limits, query/index
                                      review, timeouts and health
                                      monitoring

  Cancellation                        Update status in PostgreSQL; public
                                      verification checks current status

  Partial spreadsheet import          Validate full file before
                                      transaction; rollback all writes on
                                      failure
  -----------------------------------------------------------------------

## 9. Security and privacy

-   Hash passwords with bcrypt or Argon2.
-   Use short-lived JWT access tokens; never accept role or actor
    identity from request bodies.
-   Enforce role-based access in backend middleware and service-level
    ownership checks.
-   Enforce district scoping for district officers.
-   Validate and limit uploaded file size/type; parse safely.
-   Use parameterized database access through Prisma.
-   Rate-limit login and public verification endpoints.
-   Do not expose phone numbers, email addresses or full candidate
    records publicly.
-   Keep certificate files private and authorize downloads.
-   Append audit events for decisions, resubmissions, issuance and
    cancellation.
-   Store secrets in environment variables, not source control.

## 10. Real-time updates

Use Server-Sent Events (SSE) for dashboard updates: - Client connects to
an authenticated stream. - Server emits batch status, approval,
certificate and delivery status changes. - Reconnects should be
supported. - The client should refetch authoritative details after
reconnect. - SSE is a notification mechanism; PostgreSQL remains the
source of truth.

## 11. Observability

Log structured fields where applicable: - `requestId` - `actorId` -
`batchId` - `batchVersion` - `certificateId` - `eventId` - `jobId` -
`providerMessageId`

Monitor: - API error rate and latency. - Queue depth and oldest job
age. - Failed/retried jobs. - Outbox backlog and oldest unpublished
event. - Batch age by workflow state. - Certificate generation
failures. - WhatsApp success/failure rate. - Public verification
latency. - Database connection saturation.

## 12. Suggested repository layout

``` text
certiflow/
├── frontend/
│   └── (React + Vite application)
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── src/
│   │   ├── config/
│   │   ├── middleware/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── batches/
│   │   │   ├── approvals/
│   │   │   ├── certificates/
│   │   │   ├── verification/
│   │   │   ├── notifications/
│   │   │   └── audit/
│   │   ├── events/
│   │   ├── queues/
│   │   ├── workers/
│   │   ├── services/
│   │   ├── utils/
│   │   └── app.ts
│   ├── tests/
│   ├── Dockerfile
│   └── package.json
├── docs/
│   ├── architecture.md
│   └── api-documentation.md
├── docker-compose.yml
├── README.md
└── DEPLOYMENT.md
```

This layout is a proposal. Keep the existing repository structure if it
already differs, and place the documentation in the agreed location.

## 13. Production scale plan (summary)

The brief's target is 100,000 registered users, around 10,000 daily
active users, and peak issuance of approximately 30,000 PDFs and 30,000
WhatsApp messages in an hour after 500 batch approvals.

A production deployment should: - Run stateless API instances behind a
load balancer. - Scale PDF and notification workers independently based
on queue depth and age. - Use managed PostgreSQL with backups,
connection pooling and read replicas where justified. - Use managed
Redis with high availability appropriate to queue durability needs. -
Store PDFs in private object storage with encryption and short-lived
authorized download links. - Apply provider rate limits and control
worker concurrency. - Retain outbox records until safely published and
provide replay/recovery procedures. - Alert on queue backlog, worker
failure, provider outage, database latency and stuck batches.

A detailed provider/hosting selection and monthly cost estimate belongs
in `DEPLOYMENT.md`; costs should be estimated using the selected cloud
region and actual service sizing.

## 14. Open decisions to confirm

-   Exact CSV/Excel template and mandatory columns.
-   Large-batch quality threshold.
-   Whether district rejection is terminal or can be returned for
    correction.
-   SLA duration and escalation recipients.
-   Certificate numbering and visual template.
-   Candidate access model (download link versus authenticated candidate
    portal).
-   WhatsApp provider retry and idempotency capabilities.
-   Retention policy for candidate data, PDFs and audit records.
