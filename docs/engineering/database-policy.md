
---

# `database-policy.md`

This one needs to be strict because you're building payroll software.

```md
# Database Policy

## 1. Database

PostgreSQL is the system of record for application data.

Prisma is used as the application's ORM/database access layer.

## 2. Schema Ownership

The Prisma schema is shared project infrastructure.

Changes affecting database structure must be reviewed.

## 3. Migrations

Database schema changes must use Prisma migrations.

Do not manually modify the database schema in a way that is not represented by a migration.

## 4. Development Database

Developers should use the project's Docker-based PostgreSQL environment.

This keeps development environments consistent.

## 5. Data Integrity

Use database constraints where appropriate.

Examples:

- Unique constraints
- Foreign keys
- Required fields
- Appropriate indexes

Do not rely exclusively on application-level validation for database integrity.

## 6. Money

Payroll amounts must not use JavaScript floating-point arithmetic for financial calculations.

Use the project's agreed decimal/money representation.

Calculations must preserve monetary precision.

## 7. Payroll Data

Finalized payroll records must be treated as immutable.

Corrections should use an appropriate correction or adjustment mechanism rather than silently modifying finalized payroll results.

## 8. Sensitive Data

Do not store secrets or credentials in the database unless there is a clear requirement and appropriate protection.

Do not log sensitive employee information unnecessarily.

## 9. Database Changes

Before changing the schema, consider:

- Existing relations
- Existing data
- Migration safety
- Indexes
- Constraints
- Other modules depending on the model

## 10. Production

Production database changes must be reviewed and tested before deployment.

Never run destructive database operations against production without explicit authorization and a recovery plan.