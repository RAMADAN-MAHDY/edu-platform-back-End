# Database Schema

Target database: PostgreSQL.

All primary keys are UUID unless explicitly stated.

## 1. accounts

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| phone | VARCHAR(20) | Conditional | UNIQUE |
| email | VARCHAR(255) | Conditional | UNIQUE |
| password_hash | TEXT | Yes | |
| role | ENUM | Yes | |
| status | ENUM | Yes | |
| last_login_at | TIMESTAMPTZ | No | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

Constraint: at least one of phone or email must exist.

## 2. teachers

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| account_id | UUID | Yes | FK accounts.id UNIQUE |
| full_name | VARCHAR(150) | Yes | |
| phone | VARCHAR(20) | No | |
| email | VARCHAR(255) | No | |
| profile_image_url | TEXT | No | |
| bio | TEXT | No | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

## 3. students

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| account_id | UUID | Yes | FK accounts.id UNIQUE |
| teacher_id | UUID | Yes | FK teachers.id |
| student_code | VARCHAR(50) | Yes | UNIQUE within teacher |
| full_name | VARCHAR(150) | Yes | |
| phone | VARCHAR(20) | Yes | |
| email | VARCHAR(255) | No | |
| parent_phone | VARCHAR(20) | No | |
| education_level | VARCHAR(100) | No | |
| school_name | VARCHAR(150) | No | |
| profile_image_url | TEXT | No | |
| status | ENUM | Yes | |
| qr_token | UUID | Yes | UNIQUE |
| archived_at | TIMESTAMPTZ | No | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

## 4. student_registration_requests

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| account_id | UUID | Yes | FK accounts.id |
| status | ENUM | Yes | |
| submitted_at | TIMESTAMPTZ | Yes | |
| reviewed_at | TIMESTAMPTZ | No | |
| reviewed_by | UUID | No | FK accounts.id |
| rejection_reason | TEXT | No | |

## 5. subjects

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| name | VARCHAR(150) | Yes | |
| description | TEXT | No | |
| default_price | NUMERIC(12,2) | No | |
| currency | CHAR(3) | Yes | |
| status | ENUM | Yes | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

## 6. groups

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| subject_id | UUID | Yes | FK subjects.id |
| name | VARCHAR(150) | Yes | |
| education_level | VARCHAR(100) | No | |
| price | NUMERIC(12,2) | No | |
| status | ENUM | Yes | |
| created_at | TIMESTAMPTZ | Yes | |

## 7. student_group_enrollments

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| student_id | UUID | Yes | FK students.id |
| group_id | UUID | Yes | FK groups.id |
| started_at | TIMESTAMPTZ | Yes | |
| ended_at | TIMESTAMPTZ | No | |
| status | ENUM | Yes | |
| created_at | TIMESTAMPTZ | Yes | |

Recommended partial unique index: one ACTIVE enrollment per student/group policy as confirmed by product.

## 8. recurring_schedule_rules

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| group_id | UUID | Yes | FK groups.id |
| day_of_week | SMALLINT | Yes | |
| start_time | TIME | Yes | |
| end_time | TIME | Yes | |
| timezone | VARCHAR(50) | Yes | |
| starts_on | DATE | Yes | |
| ends_on | DATE | No | |
| status | ENUM | Yes | |

## 9. sessions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| group_id | UUID | Conditional | FK groups.id |
| student_id | UUID | Conditional | FK students.id |
| recurring_rule_id | UUID | No | FK recurring_schedule_rules.id |
| title | VARCHAR(150) | Yes | |
| starts_at | TIMESTAMPTZ | Yes | |
| ends_at | TIMESTAMPTZ | Yes | |
| status | ENUM | Yes | |
| platform | ENUM | No | |
| meeting_url | TEXT | No | |
| notes | TEXT | No | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

Constraint: group_id XOR student_id.

## 10. attendance_records

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| session_id | UUID | Yes | FK sessions.id |
| student_id | UUID | Yes | FK students.id |
| status | ENUM | Yes | |
| recorded_by | UUID | Yes | FK accounts.id |
| recorded_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

## 11. attendance_audit_trail_entries

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| attendance_record_id | UUID | Yes | FK attendance_records.id |
| changed_by | UUID | Yes | FK accounts.id |
| old_status | ENUM | Yes | |
| new_status | ENUM | Yes | |
| reason | TEXT | No | |
| changed_at | TIMESTAMPTZ | Yes | |

## 12. homeworks

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| group_id | UUID | Conditional | FK groups.id |
| student_id | UUID | Conditional | FK students.id |
| title | VARCHAR(200) | Yes | |
| instructions | TEXT | No | |
| due_at | TIMESTAMPTZ | No | |
| created_at | TIMESTAMPTZ | Yes | |

Constraint: group_id XOR student_id.

## 13. homework_submissions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| homework_id | UUID | Yes | FK homeworks.id |
| student_id | UUID | Yes | FK students.id |
| content | TEXT | No | |
| attachment_url | TEXT | No | |
| submitted_at | TIMESTAMPTZ | Yes | |
| status | ENUM | Yes | |
| reviewed_at | TIMESTAMPTZ | No | |
| feedback | TEXT | No | |

## 14. exams

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| subject_id | UUID | No | FK subjects.id |
| group_id | UUID | No | FK groups.id |
| title | VARCHAR(200) | Yes | |
| description | TEXT | No | |
| starts_at | TIMESTAMPTZ | No | |
| ends_at | TIMESTAMPTZ | No | |
| grading_mode | ENUM | Yes | |
| status | ENUM | Yes | |

## 15. exam_questions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| exam_id | UUID | Yes | FK exams.id |
| question_type | ENUM | Yes | |
| question_text | TEXT | Yes | |
| options | JSONB | No | |
| correct_answer | JSONB | No | |
| points | NUMERIC(8,2) | Yes | |
| sort_order | INT | Yes | |

## 16. exam_submissions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| exam_id | UUID | Yes | FK exams.id |
| student_id | UUID | Yes | FK students.id |
| submitted_at | TIMESTAMPTZ | No | |
| score | NUMERIC(8,2) | No | |
| status | ENUM | Yes | |

## 17. exam_answers

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| submission_id | UUID | Yes | FK exam_submissions.id |
| question_id | UUID | Yes | FK exam_questions.id |
| answer | JSONB | No | |
| score | NUMERIC(8,2) | No | |

## 18. grade_entries

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| student_id | UUID | Yes | FK students.id |
| group_id | UUID | No | FK groups.id |
| subject_id | UUID | No | FK subjects.id |
| exam_id | UUID | No | FK exams.id |
| homework_id | UUID | No | FK homeworks.id |
| type | ENUM | Yes | |
| score | NUMERIC(8,2) | Yes | |
| max_score | NUMERIC(8,2) | Yes | |
| notes | TEXT | No | |
| graded_at | TIMESTAMPTZ | Yes | |

## 19. payments

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| student_id | UUID | Yes | FK students.id |
| subject_id | UUID | No | FK subjects.id |
| group_id | UUID | No | FK groups.id |
| session_id | UUID | No | FK sessions.id |
| learning_material_id | UUID | No | FK learning_materials.id |
| amount | NUMERIC(12,2) | Yes | |
| currency | CHAR(3) | Yes | |
| payment_method | ENUM | Yes | |
| status | ENUM | Yes | |
| due_at | TIMESTAMPTZ | No | |
| paid_at | TIMESTAMPTZ | No | |
| description | TEXT | No | |
| created_at | TIMESTAMPTZ | Yes | |

## 20. payment_transactions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| payment_id | UUID | Yes | FK payments.id |
| provider | VARCHAR(50) | Yes | |
| provider_transaction_id | VARCHAR(255) | No | |
| provider_reference | VARCHAR(255) | No | |
| amount | NUMERIC(12,2) | Yes | |
| status | ENUM | Yes | |
| raw_response | JSONB | No | |
| processed_at | TIMESTAMPTZ | No | |

## 21. learning_materials

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| subject_id | UUID | No | FK subjects.id |
| group_id | UUID | No | FK groups.id |
| student_id | UUID | No | FK students.id |
| title | VARCHAR(200) | Yes | |
| description | TEXT | No | |
| type | ENUM | Yes | |
| url | TEXT | Yes | |
| access_mode | ENUM | Yes | |
| pricing_mode | ENUM | Yes | |
| price | NUMERIC(12,2) | No | |
| created_at | TIMESTAMPTZ | Yes | |

## 22. notifications

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| recipient_account_id | UUID | Yes | FK accounts.id |
| type | ENUM | Yes | |
| title | VARCHAR(200) | Yes | |
| body | TEXT | Yes | |
| entity_type | VARCHAR(50) | No | |
| entity_id | UUID | No | |
| is_read | BOOLEAN | Yes | |
| created_at | TIMESTAMPTZ | Yes | |

## 23. messages

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| student_id | UUID | Yes | FK students.id |
| sender_account_id | UUID | Yes | FK accounts.id |
| content | TEXT | Yes | |
| is_read | BOOLEAN | Yes | |
| created_at | TIMESTAMPTZ | Yes | |

## 24. whatsapp_messages

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| teacher_id | UUID | Yes | FK teachers.id |
| student_id | UUID | No | FK students.id |
| recipient_phone | VARCHAR(20) | Yes | |
| template_name | VARCHAR(100) | Yes | |
| payload | JSONB | Yes | |
| provider_message_id | VARCHAR(255) | No | |
| status | ENUM | Yes | |
| sent_at | TIMESTAMPTZ | No | |

## 25. online_sessions

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| session_id | UUID | Yes | FK sessions.id UNIQUE |
| provider | ENUM | Yes | |
| meeting_url | TEXT | Yes | |
| host_url | TEXT | No | |
| external_meeting_id | VARCHAR(255) | No | |
| created_at | TIMESTAMPTZ | Yes | |

## 26. user_settings

| Field | Type | Required | Key |
|---|---|---|---|
| id | UUID | Yes | PK |
| account_id | UUID | Yes | FK accounts.id UNIQUE |
| language | ENUM | Yes | |
| timezone | VARCHAR(50) | Yes | |
| email_notifications | BOOLEAN | Yes | |
| whatsapp_notifications | BOOLEAN | Yes | |
| created_at | TIMESTAMPTZ | Yes | |
| updated_at | TIMESTAMPTZ | Yes | |

## Core Relationships

```text
Account 1:1 Teacher
Account 1:1 Student
Teacher 1:N Student
Teacher 1:N Subject
Subject 1:N Group
Student N:M Group via StudentGroupEnrollment
Group 1:N Session
Session 1:N AttendanceRecord
AttendanceRecord 1:N AttendanceAuditTrailEntry
Homework 1:N HomeworkSubmission
Exam 1:N ExamQuestion
Exam 1:N ExamSubmission
ExamSubmission 1:N ExamAnswer
Student 1:N GradeEntry
Student 1:N Payment
Payment 1:N PaymentTransaction
Session 1:0..1 OnlineSession
Teacher 1:N LearningMaterial
Account 1:N Notification
Teacher ↔ Student via Message
```

## Wallet Removal

The following entities must not be created:

- Wallet
- WalletBalance
- WalletTransaction
- WalletTopUp
- WalletWithdrawal
