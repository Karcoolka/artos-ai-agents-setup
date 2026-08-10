# Claudeartos.md
> Enterprise Engineering Handbook in .cursor/rules/engineering-mentor.md
>
> Purpose:
> This document defines the engineering principles, coding standards and best practices for this project.
> Every AI-generated solution should follow these guidelines unless there is a documented reason not to.

---

# Artos ERP – Tech Stack

## Monorepo & Tooling
- **pnpm** – Workspaces / monorepo
- **Turborepo** – Build orchestration
- **Node.js + TypeScript** – Strict mode, no `any`
- **ESLint + Prettier** – Linting & formatting
- **Vitest** – Testing
- **Zod** – Shared validation

## Backend
- **Express** – REST API framework
- **Prisma** – ORM (database access, migrations, type safety)
- **PostgreSQL** – Database + Row Level Security (RLS) multitenancy
- **RabbitMQ** – Edge messaging / AMQP
- **Redis** – Pub/Sub + SSE backend
- **Puppeteer** – PDF generation service
- **Handlebars** – Templates for PDFs

## Frontend (3 SPAs)
- **React** – Client-side SPA
- **Vite** – Dev server + bundler
- **TanStack Query** – Server state management
- **Tailwind CSS** – Styling
- **SSE (Server-Sent Events)** – Realtime push to client
- **i18n** – Czech / Slovak localization

## Infrastructure
- **Docker** – Compose + multi-stage builds
- **Hetzner** – VPS hosting
- **nginx** – Reverse proxy + static files
- **Traefik** – Edge routing + TLS
- **Cloudflare** – DNS + R2 storage
- **GitHub Actions** – CI/CD

## Development Workflow & Collaboration
- **Linear** – Issue tracking
- **GitHub** – Source code + ghcr registry
- **Microsoft Teams** – Communication
- **Claude Code** – AI pair programming

---

# TypeScript

## Core Principles

### 1. Never use `any`
- Always prefer `unknown` for external input.
- Narrow types using Zod or type guards.
- `any` is only acceptable when wrapping an untyped third-party library and must be isolated.

**Why**

`unknown` forces validation before use and prevents runtime errors.

---

### 2. Keep a Single Source of Truth for Types

Never duplicate types.

Preferred order:

```
Database
    ↓
Prisma
    ↓
TypeScript
    ↓
API DTO
    ↓
Frontend
```

Use:

- Prisma generated types
- `z.infer<>`
- utility types

Avoid handwritten interfaces that duplicate existing models.

---

### 3. Validate Every External Boundary

Everything entering the application is untrusted.

Examples:

- Express Request
- RabbitMQ Message
- SSE Event
- External API
- JSON files

Validate using Zod before business logic executes.

Business logic should never receive invalid data.

---

### 4. Make Impossible States Impossible

Avoid boolean-based state.

❌

```
{
    success: boolean,
    error?: string
}
```

Prefer discriminated unions.

```
{
    status: "success",
    data: User
}

{
    status: "error",
    message: string
}
```

Every object should represent exactly one valid state.

---

### 5. Keep TypeScript Strict

Required compiler options:

- strict
- noImplicitAny
- strictNullChecks
- noUncheckedIndexedAccess
- exactOptionalPropertyTypes
- noUnusedLocals
- noUnusedParameters

Compiler errors should be treated as design feedback.

---

## Common Code Review Comments

### ❌ Never use `any`

Replace with:

- unknown
- generic
- union
- inferred type

---

### ❌ Never expose Prisma entities directly

Always map database entities to DTOs.

Database models are internal implementation details.

---

### ❌ Never trust external input

Every request, event or payload must be validated.

---

### ❌ Don't duplicate types

Infer them.

Examples:

- Prisma
- Zod
- ReturnType
- typeof

---

### ❌ Don't fight the compiler

If TypeScript complains, investigate why.

Avoid:

- `@ts-ignore`
- unsafe casting
- disabling strict mode

---

## Daily Development Checklist

Before opening a Pull Request ask yourself:

- [ ] Did I introduce any `any`?
- [ ] Is every external input validated?
- [ ] Did I reuse existing types instead of creating new ones?
- [ ] Am I exposing database entities?
- [ ] Can TypeScript infer this instead of me writing the type?
- [ ] Are impossible states prevented?
- [ ] Did I avoid `@ts-ignore`?
- [ ] Does this compile with strict mode without warnings?

---

# Express API Architecture

## Core Principles

### 1. Keep Controllers Thin

- Controllers should only handle HTTP concerns: parse request, call services and return responses.
- Business logic belongs in service-layer functions that are easy to test and reuse.
- Avoid database writes, authorization decisions, event publishing and audit logging directly inside route handlers.

### 2. Validate Before Business Logic

- Validate all request bodies, params, query strings, webhooks, imports and queue payloads before they reach services.
- Prefer Zod schemas at the application boundary.
- Business logic should receive trusted, typed input.

### 3. Use Request-Scoped Context

- Pass explicit context objects into services instead of passing raw Express request objects.
- Context should include values such as `tenantId`, `userId`, `requestId`, permissions and locale.
- Services must not depend on Express-specific APIs.

### 4. Centralize Error Handling

- Throw typed application errors from controllers and services.
- Let global error middleware convert application errors into consistent HTTP responses.
- Avoid route-level `try/catch` blocks that hide failures or return inconsistent error shapes.

### 5. Never Return Database Models Directly

- Map Prisma entities to API DTOs before returning responses.
- API contracts should not leak internal fields, tenant metadata, soft-delete flags or database schema details.
- Response DTOs protect clients from accidental schema changes.

---

## Common Code Review Comments

### Controllers Should Stay Thin

Move business logic out of the controller and into a service.

### Validate Input At The Boundary

Validate request data before calling business logic.

### Do Not Pass Express Requests Into Services

Create a request-scoped context object with only the values the service needs.

### Use Centralized Error Handling

Throw typed errors and let global middleware format the API response.

### Do Not Expose Prisma Entities Directly

Map database models to explicit response DTOs.

---

## Daily Development Checklist

Before opening a Pull Request ask yourself:

- [ ] Is this controller limited to HTTP concerns?
- [ ] Is every request payload validated before the service layer?
- [ ] Did I pass explicit context instead of `req`?
- [ ] Are errors handled by shared error middleware?
- [ ] Did I map Prisma entities to API DTOs?

---

## Staff Engineer Principles

- Let the compiler prevent bugs.
- Prefer explicitness over cleverness.
- Validate at the boundaries.
- Business logic should never receive invalid data.
- One source of truth for every type.
- Keep APIs independent from the database schema.
- Make illegal states impossible.
- Optimize for maintainability over short-term speed.
- Keep controllers thin and business logic reusable.
- Pass explicit request context into services.
- Return stable API DTOs instead of database entities.

---

## Knowledge Progress

### Topics Covered

- ✅ Strict TypeScript fundamentals
- ✅ unknown vs any
- ✅ Type inference
- ✅ DTO pattern
- ✅ Discriminated unions
- ✅ Strict compiler configuration
- ✅ Thin Express controllers
- ✅ Boundary validation before business logic
- ✅ Request-scoped service context
- ✅ Centralized API error handling
- ✅ API DTO mapping for database entities