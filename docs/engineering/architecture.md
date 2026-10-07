# Architecture

## 1. Overview

The Payroll Management System is a modular monolith organized as a monorepo.

The system consists of:

- Backend API
- Frontend application
- Background workers
- Shared packages
- PostgreSQL
- Redis
- External services

## 2. Repository Structure

```text
apps/
├── backend/
├── frontend/
└── workers/

packages/
└── shared packages

docs/
└── engineering documentation

The request flow is: 

```text 

Route
  ↓
Middleware
  ↓
Controller
  ↓
Service
  ↓
Prisma / External Services
  ↓
Database / External System

```


### Routes

Routes define HTTP endpoints and connect them to controllers.
Routes must not contain business logic.

### Controllers

Controllers handle HTTP concerns.

They are responsible for:

- Reading request data
- Calling services
- Returning HTTP responses
- Passing errors to the error middleware

Controllers must not contain database queries or complex business rules.

### Services

Services contain business logic.

Services are responsible for:

- Business rules
- Database operations through Prisma
- Coordinating other services
- Enforcing application-level rules

### Schemas

Schemas validate incoming data using Zod.
Validation should happen before business logic executes.

## 3. Database access

Prisma is the application's database access layer.
Application modules must not create raw database connections independently.
Database access should remain behind the backend/service layer.

##  4. Workers 

Background jobs run separately from HTTP request handling.
Workers are responsible for tasks such as:

- Payroll processing
- Payslip generation
- Email delivery
- Scheduled jobs

Long-running or asynchronous work should not block API requests

## 5. Shared packages 

Shared packages may contain code genuinely required by multiple applications.

Examples:

- API contracts
- Shared validation schemas
- Shared TypeScript types
- Shared configuration

Backend-specific business logic must remain inside the backend.

## 6. Frontend 

The frontend communicates with the backend through defined APIs.
The frontend must not access Prisma or PostgreSQL directly.

## Architctural Rules

When adding a feature, developers should first identify:

Which domain/module owns the feature?
1 What data does it require?
2.What business rules apply?
3.Is the operation synchronous or asynchronous?
4.Does it belong in the backend, worker, frontend, or shared package?

Architecture decisions that affect multiple modules must be reviewed before implementation.