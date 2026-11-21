# CertiFlow -- API Documentation

## 1. Overview

This document defines the proposed REST API contract for CertiFlow.

-   Base path: `/api/v1`
-   Backend: Node.js + Express + TypeScript
-   Database: PostgreSQL via Prisma
-   Authentication: JWT bearer token
-   Request/response format: JSON, except file uploads and PDF downloads
-   Dates: ISO 8601 UTC timestamps
-   IDs: opaque string identifiers (UUID recommended)

**Important:** These are proposed contracts derived from the problem
requirements and architecture. They are not APIs specified by the
original problem statement. Keep frontend and backend implementations
aligned with this document.

## 2. Conventions

### Authentication

Protected endpoints require:

``` http
Authorization: Bearer <access_token>
```

The authenticated user ID and role must come from the verified
token/session, never from a client-supplied `actorId`, `officerId` or
`role`.

### Standard success response

``` json
{
  "success": true,
  "data": {},
  "message": "Operation completed"
}
```

### Standard error response

``` json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request contains invalid data",
    "details": []
  }
}
```

### Common HTTP status codes

  Status   Meaning
  -------- ---------------------------------------------------
  200      Successful read or action
  201      Resource created
  202      Accepted for asynchronous processing
  400      Invalid request
  401      Missing/invalid authentication
  403      Authenticated but not authorized
  404      Resource not found or not visible to caller
  409      State conflict, duplicate or concurrency conflict
  413      Upload too large
  415      Unsupported media type
  422      Semantically invalid file/data
  429      Rate limit exceeded
  500      Unexpected server error
  503      Temporary dependency/service unavailable

## 3. Roles

  -----------------------------------------------------------------------
  Role                                Access
  ----------------------------------- -----------------------------------
  `TRAINING_CENTRE`                   Manage batches belonging to the
                                      user's centre

  `DISTRICT_OFFICER`                  Review batches assigned to the
                                      officer's district

  `QUALITY_OFFICER`                   Review batches requiring quality
                                      scrutiny

  `STATE_OFFICER`                     Final approval, monitoring and
                                      certificate cancellation

  Public                              Verify certificate authenticity
                                      only; no login
  -----------------------------------------------------------------------

The demo identities are Anika (centre), Dev and Meera (district
officers), Neha (quality team), and Farah (state office).

## 4. Authentication APIs

### 4.1 Login

`POST /auth/login`

Access: Public

Request:

``` json
{
  "email": "anika@certiflow.demo",
  "password": "Demo@12345"
}
```

Response `200`:

``` json
{
  "success": true,
  "data": {
    "accessToken": "<jwt>",
    "user": {
      "id": "user_001",
      "name": "Anika",
      "email": "anika@certiflow.demo",
      "role": "TRAINING_CENTRE",
      "districtId": "district_001",
      "trainingCentreId": "centre_001"
    }
  }
}
```

Errors: `400`, `401`, `429`.

### 4.2 Current user

`GET /auth/me`

Access: Authenticated

Returns the current authenticated user's safe profile and permissions.

## 5. Batch APIs

### 5.1 Create batch

`POST /batches`

Access: `TRAINING_CENTRE`

Request:

``` json
{
  "batchName": "Web Development – November 2026",
  "courseId": "course_001",
  "districtId": "district_001",
  "trainingCentreId": "centre_001",
  "trainingStartDate": "2026-10-01",
  "trainingEndDate": "2026-11-15"
}
```

The backend must derive/validate the centre from the authenticated user.
A centre must not create a batch for another centre.

Response `201`:

``` json
{
  "success": true,
  "data": {
    "id": "batch_001",
    "batchNumber": "PF-2026-0001",
    "status": "DRAFT",
    "version": 1
  }
}
```

### 5.2 Upload candidate spreadsheet

`POST /batches/:batchId/candidates/upload`

Access: `TRAINING_CENTRE`; batch must belong to the caller and be
editable.

Content type: `multipart/form-data`

Form field: - `file`: CSV file (MVP contract). Excel support can be
added if implemented.

Proposed CSV columns:

  Column                 Required   Notes
  ---------------------- ---------- -----------------------------------------
  `registrationNumber`   Yes        Unique within the batch
  `fullName`             Yes        Candidate name
  `phone`                Yes        Include country code in normalized form
  `email`                No         Validate if supplied
  `dateOfBirth`          No         ISO date if collected
  `completionStatus`     Yes        Must indicate passed/completed

The exact template is an implementation decision; publish it as a
downloadable sample for users.

Atomic import rule: validate the entire file first. If any row is
invalid, save no candidate/batch-candidate rows.

Validation error response `422`:

``` json
{
  "success": false,
  "error": {
    "code": "IMPORT_VALIDATION_FAILED",
    "message": "The file contains invalid rows. No candidates were imported.",
    "details": [
      {
        "row": 4,
        "field": "phone",
        "code": "INVALID_PHONE",
        "message": "Phone number is invalid"
      },
      {
        "row": 7,
        "field": "registrationNumber",
        "code": "DUPLICATE_REGISTRATION_NUMBER",
        "message": "Registration number already exists in this batch"
      }
    ]
  }
}
```

Successful response `200`:

``` json
{
  "success": true,
  "data": {
    "batchId": "batch_001",
    "importedCount": 60,
    "rejectedCount": 0
  }
}
```

### 5.3 Get batch list

`GET /batches`

Access: Authenticated; results are scoped by role and organization.

Query parameters: - `status` -- optional batch status. - `districtId` --
state/authorized users only. - `trainingCentreId` -- state/authorized
users only. - `page` -- default 1. - `limit` -- default 20, capped by
server. - `search` -- batch number/name.

Response:

``` json
{
  "success": true,
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 0,
      "totalPages": 0
    }
  }
}
```

### 5.4 Get batch details

`GET /batches/:batchId`

Access: Authenticated and authorized for the batch.

Returns batch metadata, version, candidate summary, current workflow
state, required/received approval counts, review history and timestamps.
Avoid exposing candidate details to users without the appropriate
access.

### 5.5 Submit batch

`POST /batches/:batchId/submit`

Access: `TRAINING_CENTRE`

Preconditions: - Batch belongs to the centre. - Batch is `DRAFT` or
`RETURNED` and editable. - Required batch metadata and candidate data
are valid. - Candidate count is greater than zero.

For a resubmission, increment the batch version and make earlier
approvals ineligible for the new version while retaining them in
history.

Response `202` or `200` depending on whether submission triggers
asynchronous work:

``` json
{
  "success": true,
  "data": {
    "batchId": "batch_001",
    "version": 2,
    "status": "DISTRICT_REVIEW"
  }
}
```

## 6. Approval APIs

All decisions must record the authenticated actor, timestamp, batch
version and decision. A reason is required for `RETURN` and `REJECT`.

### 6.1 District decision

`POST /batches/:batchId/district-decision`

Access: `DISTRICT_OFFICER`

Request:

``` json
{
  "decision": "APPROVE",
  "remarks": "Candidate records verified"
}
```

Allowed decisions: `APPROVE`, `RETURN`, `REJECT`.

Rules: - Officer must belong to the batch's district. - Two distinct
officers must approve the same batch version. - One officer cannot count
twice. - Use a database transaction and concurrency-safe state
transition. - Duplicate decision for the same officer/version returns
`409`. - `RETURN` and `REJECT` require a non-empty reason.

Response `200`:

``` json
{
  "success": true,
  "data": {
    "batchId": "batch_001",
    "version": 1,
    "approvalCount": 1,
    "requiredApprovals": 2,
    "status": "DISTRICT_REVIEW"
  }
}
```

### 6.2 Quality decision

`POST /batches/:batchId/quality-decision`

Access: `QUALITY_OFFICER`

Request:

``` json
{
  "decision": "APPROVE",
  "remarks": "Quality checks completed"
}
```

Allowed decisions: `APPROVE`, `RETURN`, `REJECT`.

Only available when the batch requires quality review and has completed
district approval.

Response includes the quality decision and resulting batch status.

### 6.3 State decision

`POST /batches/:batchId/state-decision`

Access: `STATE_OFFICER`

Request:

``` json
{
  "decision": "APPROVE",
  "remarks": "Final approval granted"
}
```

Allowed decisions: `APPROVE`, `RETURN`, `REJECT`.

Preconditions: - All required district approvals exist for the current
version. - Quality approval exists if the batch meets the configured
threshold. - Batch is awaiting state decision.

On approval: - Record state approval. - Atomically create one logical
certificate record per eligible batch candidate, protected by unique
constraints. - Write outbox events for PDF generation and delivery
workflow. - Return promptly; PDF generation and WhatsApp delivery must
not block the API request.

Response `202`:

``` json
{
  "success": true,
  "message": "Batch approved. Certificate issuance has been queued.",
  "data": {
    "batchId": "batch_001",
    "status": "ISSUING",
    "candidateCount": 60
  }
}
```

## 7. Certificate APIs

### 7.1 List certificates

`GET /certificates`

Access: Authenticated; role and organization scoped.

Query parameters: - `batchId` - `status` - `page` - `limit` - `search`
(certificate number or permitted candidate search)

### 7.2 Download certificate

`GET /certificates/:certificateId/download`

Access: Authorized user, or candidate access mechanism if implemented.

Returns `application/pdf` as a streamed response. Do not expose private
storage paths or unrestricted storage URLs.

### 7.3 Cancel certificate

`POST /certificates/:certificateId/cancel`

Access: `STATE_OFFICER`

Request:

``` json
{
  "reason": "Certificate issued against an invalid candidate record"
}
```

Rules: - Reason is mandatory. - Cancellation is a status change, not
deletion. - Record actor, timestamp and reason in audit history. -
Public verification must report the certificate as cancelled immediately
after the transaction commits.

Response:

``` json
{
  "success": true,
  "data": {
    "certificateId": "cert_001",
    "status": "CANCELLED",
    "cancelledAt": "2026-11-21T10:00:00.000Z"
  }
}
```

## 8. Public verification API

### 8.1 Verify certificate

`GET /public/verify/:certificateNumber`

Access: Public; no JWT required.

Response for valid certificate:

``` json
{
  "success": true,
  "data": {
    "valid": true,
    "status": "ISSUED",
    "certificateNumber": "PF-CERT-2026-000001",
    "candidateDisplayName": "A. Kumar",
    "courseName": "Web Development",
    "issuedAt": "2026-11-21T10:00:00.000Z"
  }
}
```

Response for cancelled certificate:

``` json
{
  "success": true,
  "data": {
    "valid": false,
    "status": "CANCELLED",
    "certificateNumber": "PF-CERT-2026-000001",
    "message": "This certificate has been cancelled."
  }
}
```

Response for unknown certificate:

``` json
{
  "success": true,
  "data": {
    "valid": false,
    "status": "NOT_FOUND",
    "message": "Certificate could not be verified."
  }
}
```

Privacy: - Do not return phone number, email, date of birth, address or
full candidate record. - Rate-limit this public endpoint. - Return a
minimal response to avoid personal-data exposure.

## 9. Dashboard and monitoring APIs

### 9.1 Dashboard summary

`GET /dashboard/summary`

Access: Authenticated; aggregate data scoped to the caller's role.

Possible fields: - Pending batches. - Batches by status. - Average time
in current stage. - Issued certificates. - Failed/pending deliveries. -
Overdue batches.

### 9.2 Stuck batches

`GET /monitoring/stuck-batches`

Access: `STATE_OFFICER` or explicitly authorized senior role.

Returns batches exceeding the configured stage-specific SLA, with age
and current assignee. SLA values are configuration decisions because the
brief does not specify durations.

### 9.3 Audit history

`GET /audit/:entityType/:entityId`

Access: Authenticated and authorized; sensitive metadata must be
filtered by role.

Supported entity types may include `BATCH` and `CERTIFICATE`.

Returns actor, action, timestamp and relevant change metadata.

## 10. Real-time events API

### 10.1 Subscribe to updates

`GET /notifications/stream`

Access: Authenticated SSE connection.

Headers:

``` http
Accept: text/event-stream
Authorization: Bearer <access_token>
```

Example events:

``` text
event: batch.status.updated
data: {"batchId":"batch_001","status":"STATE_REVIEW","version":1}

event: certificate.issued
data: {"certificateId":"cert_001","batchId":"batch_001"}

event: delivery.status.updated
data: {"certificateId":"cert_001","status":"DELIVERED"}
```

SSE is a notification channel, not the system of record. After
reconnecting, the client should fetch current resource state through the
REST API.

## 11. Background job status

### 11.1 Get job status

`GET /jobs/:jobId`

Access: Authenticated and authorized for the associated
batch/certificate.

Example:

``` json
{
  "success": true,
  "data": {
    "jobId": "job_001",
    "type": "CERTIFICATE_PDF_GENERATION",
    "status": "COMPLETED",
    "attemptsMade": 1
  }
}
```

Do not expose queue internals or another user's job data.

## 12. Idempotency and concurrency requirements

For critical operations, consider an `Idempotency-Key` header:

``` http
Idempotency-Key: 9f52c2f4-unique-client-operation
```

Use it for batch submission and other retryable client operations where
duplicate requests could create side effects.

Mandatory safeguards: - Unique certificate number constraint. - Unique
certificate per batch-candidate constraint. - Unique district approval
per batch/version/officer. - Transactional approval/state transitions. -
Idempotent outbox consumers and worker handlers. - Provider idempotency
key for WhatsApp if supported.

A request that conflicts with the current batch state should return
`409 Conflict`, not silently apply a second transition.

## 13. Suggested error codes

  Code                           Meaning
  ------------------------------ ------------------------------------------
  `VALIDATION_ERROR`             Invalid request fields
  `IMPORT_VALIDATION_FAILED`     One or more spreadsheet rows invalid
  `UNAUTHORIZED`                 Missing or invalid authentication
  `FORBIDDEN`                    Role, district or ownership restriction
  `BATCH_NOT_FOUND`              Batch unavailable to caller
  `INVALID_BATCH_STATE`          Operation not allowed in current state
  `DUPLICATE_APPROVAL`           Officer already decided for this version
  `INSUFFICIENT_APPROVALS`       Required approvals not yet complete
  `QUALITY_REVIEW_REQUIRED`      Quality step not completed
  `CERTIFICATE_ALREADY_ISSUED`   Certificate already exists
  `CERTIFICATE_CANCELLED`        Certificate is cancelled
  `CONCURRENCY_CONFLICT`         Another request changed the resource
  `RATE_LIMITED`                 Too many requests
  `DEPENDENCY_UNAVAILABLE`       Temporary dependency failure

## 14. API implementation checklist

-   [ ] Validate all request bodies and query parameters.
-   [ ] Authenticate protected endpoints.
-   [ ] Enforce role, district and centre ownership on the server.
-   [ ] Never trust actor identity supplied by the client.
-   [ ] Make CSV import atomic.
-   [ ] Enforce approval rules by batch version.
-   [ ] Protect concurrent approval and issuance operations.
-   [ ] Use database uniqueness constraints.
-   [ ] Persist audit events.
-   [ ] Use outbox events for asynchronous work.
-   [ ] Keep PDF files private.
-   [ ] Make public verification minimal and rate-limited.
-   [ ] Test cancelled-certificate verification.
-   [ ] Test WhatsApp failure and retry behaviour.
-   [ ] Document SSE reconnection behaviour.
