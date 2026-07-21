# Authorization and Access Control

## Roles

### TEACHER

Full management access within their own scope.

### STUDENT

Read access only to resources owned by or assigned to the authenticated student.

### PARENT

No platform account and no login session in the first release. Parent is a notification recipient, primarily through WhatsApp, associated with a student record.

## Resource Isolation

A student request must always be scoped to the authenticated student identity.

Bad:

```sql
SELECT * FROM students WHERE id = :studentId;
```

Required:

```sql
SELECT s.*
FROM students s
JOIN accounts a ON a.id = s.account_id
WHERE s.id = :studentId
  AND a.id = :authenticatedAccountId;
```

For teacher resources, enforce:

```text
resource.teacher_id == authenticatedTeacher.id
```

## Access Matrix

| Resource | Teacher | Student | Parent |
|---|---|---|---|
| Students | CRUD | Own record | No |
| Subjects | CRUD | Read assigned | No |
| Groups | CRUD | Read assigned | No |
| Sessions | CRUD | Read assigned | No |
| Attendance | CRUD | Read own | WhatsApp only |
| Homework | CRUD | Read/submit own | WhatsApp only |
| Exams | CRUD | Read/submit assigned | WhatsApp only |
| Grades | CRUD | Read own | WhatsApp only |
| Payments | CRUD | Read own | WhatsApp only |
| Materials | CRUD | Read/download if entitled | No |
| Notifications | Own | Own | No |
| Messages | Teacher ↔ Student | Teacher ↔ Student | No |
| Reports | Read/export | No | No |
| Settings | Own | Own | No |

## Security Rules

- Never trust `studentId` from the client without ownership verification.
- Never expose another student's records through search.
- Payment success must be confirmed through a verified provider callback.
- Webhook endpoints must validate provider signatures.
- File download authorization must be checked on every request.
- Use rate limiting for authentication, OTP, search, and public registration.
- Store passwords using a strong adaptive password hash.
- Refresh tokens should be revocable.
