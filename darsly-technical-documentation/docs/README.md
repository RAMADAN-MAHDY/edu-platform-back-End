# Darsly — Technical Documentation

## Purpose

This documentation package defines the technical baseline for the Darsly educational platform.

It is intended for:

- Backend Developers
- Frontend Developers
- Database Engineers
- QA Engineers
- DevOps Engineers
- Technical Leads

## Source of Truth

The `openapi.yaml` file is the canonical API contract for REST endpoints.

The Markdown documents explain architecture, database design, authorization, business rules, and unresolved product decisions.

## Documentation Map

| Document | Purpose |
|---|---|
| `architecture/system-architecture.md` | High-level system architecture and component boundaries |
| `architecture/authorization.md` | Roles, resource isolation, and authorization rules |
| `database/database-schema.md` | PostgreSQL schema, keys, relationships, and constraints |
| `api/openapi.yaml` | OpenAPI 3.1 REST API contract |
| `business-rules/business-rules.md` | Domain and business rules |
| `open-decisions.md` | Decisions that must be confirmed before implementation |

## Financial Model

The previous wallet model has been removed.

The final model is direct payment:

`Payment -> PaymentTransaction -> Payment Provider`

No wallet balance, wallet top-up, wallet withdrawal, or wallet transaction entity is part of the target architecture.

## API Base URL

`/api/v1`

## Standard Response Envelope

Success:

```json
{
  "success": true,
  "data": {},
  "meta": null
}
```

Error:

```json
{
  "success": false,
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Resource not found",
    "details": []
  }
}
```

## Recommended Implementation Order

1. Authentication and identity
2. Teacher and student management
3. Subjects and groups
4. Enrollment and transfers
5. Sessions and scheduling
6. Attendance and audit trail
7. Homework, exams, and grades
8. Direct payments
9. Learning materials
10. Notifications and messaging
11. WhatsApp integration
12. Reports and exports
13. Search and dashboard aggregation

## Important

Any item marked as an Open Decision must not be silently invented by an implementation team. Confirm it first and update the relevant schema and OpenAPI contract.
