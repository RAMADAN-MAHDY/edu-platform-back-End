# System Architecture

## 1. Architectural Style

The recommended implementation is a modular REST API backed by PostgreSQL.

Suggested stack:

- Backend: NestJS or Express.js with TypeScript
- Database: PostgreSQL
- ORM: Prisma or TypeORM
- Authentication: JWT access/refresh tokens + OTP where required
- Object Storage: S3-compatible storage
- Realtime: WebSocket or Server-Sent Events where required
- Background Processing: Queue-based workers for notifications, WhatsApp, exports, and asynchronous jobs
- API Contract: OpenAPI 3.1

## 2. Logical Domains

```text
Identity
├── Account
├── Teacher
├── Student
└── Registration Request

Academic
├── Subject
├── Group
├── Enrollment
├── Session
└── Scheduling

Attendance
├── Attendance Record
└── Attendance Audit

Assessment
├── Homework
├── Submission
├── Exam
├── Question
├── Exam Submission
├── Answer
└── Grade

Finance
├── Payment
└── Payment Transaction

Content
├── Learning Material
└── Material Access

Communication
├── Notification
├── Message
└── WhatsApp Message

Online
└── Online Session

System
└── User Settings
```

## 3. Payment Architecture

Wallets are explicitly removed.

```text
Student
  |
  v
POST /payments
  |
  v
Payment (PENDING)
  |
  v
Payment Provider Checkout
  |
  v
Provider Webhook
  |
  v
Signature Verification
  |
  v
PaymentTransaction
  |
  +--> SUCCESS
  +--> FAILED
  +--> PENDING
```

The frontend must never be treated as the source of truth for successful payment.

## 4. Authorization Boundary

Authorization must be enforced server-side.

```text
HTTP Request
   |
   v
Authentication
   |
   v
Role Authorization
   |
   v
Tenant/Teacher Scope
   |
   v
Resource Ownership
   |
   v
Database Query
```

The frontend may hide controls, but it is not a security boundary.

## 5. Data Ownership

The platform is modeled as a teacher-scoped system.

Every student, subject, group, session, payment, and learning material must be resolved within the authenticated teacher's scope.

## 6. Soft Deletion

Historical records must not be physically deleted where they are required for:

- Financial history
- Attendance history
- Grade history
- Audit history
- Student transfer history

Use archive/status transitions instead.

## 7. External Integrations

External providers must be hidden behind adapters:

```text
Application Domain
      |
      v
Provider Interface
      |
      +--> Payment Provider Adapter
      +--> WhatsApp Provider Adapter
      +--> Video Meeting Adapter
      +--> Object Storage Adapter
```

This prevents vendor lock-in and makes testing easier.
