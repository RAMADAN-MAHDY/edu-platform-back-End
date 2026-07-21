# Business Rules

## Identity

- A user must have at least one login identifier: phone or email.
- Teacher accounts are active after successful registration/verification according to the final authentication policy.
- Student self-registration is pending until teacher approval.
- Rejected registration requests remain stored for audit/history.
- Archived students are not physically deleted.

## Student Scope

- A student can only access their own academic, attendance, payment, material, and messaging records.
- A student cannot access another student's resource by changing a URL or ID.

## Groups

- Student transfers must preserve historical enrollment.
- Closing an enrollment must set `ended_at`.
- A new enrollment is created for the target group.
- The system must not rewrite historical attendance or grades when a student transfers.

## Sessions

- A session belongs to either a group or an individual student.
- Session overlap rules must be enforced server-side.
- Rescheduling must preserve an audit trail if historical scheduling data is required.

## Attendance

- Attendance statuses: PRESENT, ABSENT, LATE, EXCUSED.
- Every attendance modification creates an audit entry.
- Attendance history must not be silently overwritten.

## Homework

- Homework may target a group or an individual student.
- A student may submit only for homework they are entitled to access.
- Review feedback is visible to the relevant student.

## Exams

- Exam questions may be manually authored or imported.
- Automatic grading is supported where the question type allows it.
- Manual grading remains available for non-automatic questions.

## Grades

- Score must not exceed max_score.
- Percentage is calculated server-side.
- Student grade access is restricted to the authenticated student's records.

## Payments

- Wallet functionality is removed.
- Payments are direct.
- Payment status is controlled by the backend and verified provider events.
- The frontend cannot mark a payment as successful.
- Provider webhooks must be idempotent.
- Duplicate provider callbacks must not create duplicate successful transactions.
- A successful payment may unlock a paid learning material when the material is linked to that payment.

## Learning Materials

- Paid materials require a successful payment before download/access.
- View-only materials cannot be downloaded through a direct storage URL.
- Download authorization is enforced by the API.
- Storage URLs should preferably be signed and short-lived.

## Notifications

- System notifications are for teachers and students.
- Parents are not platform notification users in the first release.
- Parent communication is handled through WhatsApp events.

## Messaging

- Messaging is one-to-one between teacher and student.
- Group chat is not part of the first release.
- New messages can trigger an in-app notification.

## WhatsApp

- WhatsApp is an outbound communication channel.
- Message status must be tracked.
- Template variables must be stored in the outgoing payload for traceability.
- Retry operations must be idempotent where supported by the provider.

## Reports

- Reports are teacher-only in the first release.
- Reports support filtering.
- Reports support PDF and Excel export.
- Large exports should be asynchronous.

## Localization

- Arabic and English are first-class languages.
- Arabic switches the application to RTL layout.
- Dates, numbers, and currency formatting should respect locale.
