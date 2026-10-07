# Coding Standards

## 1. General Principles

Code should be:

- Readable
- Simple
- Testable
- Consistent
- Explicit

Prefer understandable code over clever code.

## 2. TypeScript

Use TypeScript throughout the backend, frontend, workers, and shared packages.

Avoid `any` unless there is a documented reason.

Prefer explicit types at system boundaries.

## 3. Naming

Use:

- `camelCase` for variables and functions
- `PascalCase` for classes, components, and types
- `UPPER_SNAKE_CASE` for constants when appropriate

Use names that describe the purpose of the code.

Avoid names such as:

```text
data
thing
temp
stuff
```

##  4. Functions

Functions should have one clear responsibility.

Avoid large functions that perform validation, business logic, database operations, and response formatting simultaneously.

##  5. Backend Separation

Do not place business logic inside routes.

Do not place database queries inside controllers.

Do not place HTTP-specific logic inside services.

Follow:
```text
Route → Controller → Service → Database
```
##  6. Error Handling

Errors must be handled consistently through the application's error-handling mechanism.

Do not silently ignore errors.

Do not expose internal errors, stack traces, database details, or secrets to API consumers.

##  7. Validation

Validate external input at system boundaries.

Use Zod for API input validation.

Never assume client-provided data is valid.

##  8. Comments

Comments should explain why something is done when the reason is not obvious.

Do not write comments that simply restate the code.

##  9. Code Duplication

Avoid unnecessary duplication.

However, do not create abstractions solely to eliminate a few repeated lines.

Prefer simple code until a real reuse pattern exists.

##  10. Pull Requests

Code must pass:

- Type checking
- Linting
- Tests

before merging.

Large unrelated changes should not be included in the same pull request.