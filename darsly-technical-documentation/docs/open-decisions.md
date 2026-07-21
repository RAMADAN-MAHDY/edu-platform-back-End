# Open Decisions

These items should be confirmed before production implementation because they affect the database schema, API contracts, or business logic.

## 1. Authentication

- Is OTP mandatory for every login or only for sensitive operations?
- Which OTP provider will be used?
- OTP expiry duration?
- Maximum verification attempts?
- Refresh token lifetime?

## 2. Registration

- Can a student self-register without a teacher invitation?
- Is the teacher identified by a public slug, invite code, QR code, or URL?
- What happens to the student's Account after rejection?

## 3. Pricing

- Priority of pricing: session vs group vs subject?
- Can a student have multiple active paid subscriptions?
- Is payment per session, group, subject, material, or arbitrary invoice?

## 4. Payments

- Final payment provider?
- Supported methods?
- Refund policy?
- Partial payments?
- Invoice generation?
- Payment expiration?
- Currency is currently modeled as ISO 4217 code.

## 5. Student Transfers

- Can one student belong to multiple active groups simultaneously?
- Does transfer affect future sessions only?
- Are outstanding payments transferred to the new group?

## 6. Scheduling

- Are recurring sessions materialized as actual Session rows in advance?
- What is the teacher timezone?
- Can a teacher have overlapping individual and group sessions?
- Is a cancelled session retained permanently?

## 7. Exams

- Which question types are supported?
- Is automatic grading generated into GradeEntry immediately?
- Can students retake exams?
- Is there a time limit per submission?

## 8. File Uploads

- Maximum file size?
- Allowed MIME types?
- Storage provider?
- Retention policy?
- Virus scanning requirement?

## 9. Notifications

- Retention policy for read/unread notifications?
- Are email notifications required in V1?
- Which events are mandatory vs optional?

## 10. WhatsApp

- Provider?
- Approved templates?
- Retry policy?
- Message retention period?
- Parent phone validation rules?

## 11. Reports

- Maximum export date range?
- Synchronous or asynchronous export threshold?
- PDF branding requirements?

## 12. Search

- PostgreSQL full-text search or external search engine?
- Which entities are searchable?
- Minimum query length?

## 13. Realtime

- WebSocket or Server-Sent Events?
- Which events must be realtime?
- Reconnect and replay strategy?

## 14. Final Recommendation

Do not block the entire project on all open decisions.

Freeze the MVP decisions first for:

1. Authentication
2. Payment provider
3. Pricing model
4. Active group enrollment rule
5. Scheduling model
6. File storage

Then version the OpenAPI contract and database migrations.
