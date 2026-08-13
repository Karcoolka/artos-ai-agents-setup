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

### 6. Use Branded IDs for Critical Domain Identifiers

- Avoid raw `string` types for identifiers that must not be mixed.
- Brand tenant IDs, user IDs, invoice IDs and company IDs when passing them through domain logic.
- Prevent cross-entity mistakes at compile time instead of relying on naming discipline.

Example:

```ts
type Brand<T, Name extends string> = T & { readonly __brand: Name };

type TenantId = Brand<string, 'TenantId'>;
type InvoiceId = Brand<string, 'InvoiceId'>;

async function getInvoice(tenantId: TenantId, invoiceId: InvoiceId) {}
```

---

### 7. Use `satisfies` for Typed Configuration Objects

- Use `satisfies` when an object must match a shape while preserving precise inference.
- Prefer it for permissions, feature flags, route metadata, event maps and lookup tables.
- Avoid broad `Record<string, ...>` annotations when the valid keys are known.

Example:

```ts
type Resource = 'invoice' | 'customer';
type Action = 'read' | 'write' | 'delete';

const permissions = {
    invoice: ['read', 'write'],
    customer: ['read'],
} satisfies Record<Resource, Action[]>;
```

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

### ❌ Do not use raw strings for critical IDs

Use branded ID types when mixing IDs can cause tenant, authorization or data access bugs.

---

### ❌ Preserve inference for configuration objects

Use `satisfies` instead of broad annotations when validating known object shapes.

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
- [ ] Are critical domain IDs protected from being mixed?
- [ ] Did I use `satisfies` for typed maps or config objects where useful?

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

Example:

```ts
await invoiceService.create(input, {
    tenantId: req.user.tenantId,
    userId: req.user.id,
    requestId: req.id,
});
```

### 4. Centralize Error Handling

- Throw typed application errors from controllers and services.
- Let global error middleware convert application errors into consistent HTTP responses.
- Avoid route-level `try/catch` blocks that hide failures or return inconsistent error shapes.

Example:

```ts
if (!invoice) {
    throw new NotFoundError('Invoice not found');
}
```

### 5. Never Return Database Models Directly

- Map Prisma entities to API DTOs before returning responses.
- API contracts should not leak internal fields, tenant metadata, soft-delete flags or database schema details.
- Response DTOs protect clients from accidental schema changes.

Example:

```ts
res.json({
    id: user.id,
    email: user.email,
    displayName: user.displayName,
});
```

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

# Prisma

## Core Principles

### 1. Use Transactions for Multi-Step Writes

- Related writes must succeed or fail together.
- Use `prisma.$transaction` when creating or updating multiple records that represent one business operation.
- Audit logs, inventory movement, payments and invoice updates usually belong in the same transaction as the domain write.

Example:

```ts
await prisma.$transaction(async (tx) => {
    const invoice = await tx.invoice.create({ data: invoiceData });
    await tx.auditLog.create({ data: { action: 'invoice.created', invoiceId: invoice.id } });
});
```

### 2. Select Only the Fields Needed

- Avoid loading full records when the use case only needs a few fields.
- Use `select` to reduce data transfer, memory use and accidental exposure of internal fields.
- API list views, dropdowns and search results should be especially strict about field selection.

Example:

```ts
const users = await prisma.user.findMany({
    select: { id: true, email: true, displayName: true },
});
```

### 3. Always Scope Tenant-Owned Queries

- Tenant-owned data must include tenant isolation in the query.
- Prefer query patterns that make tenant scope obvious during review.
- Missing tenant filters are security bugs, not style issues.

Example:

```ts
const invoice = await prisma.invoice.findFirst({
    where: { id: invoiceId, tenantId },
});
```

---

## Common Code Review Comments

### Use a Transaction for Related Writes

Multiple database writes in one business operation must be atomic.

### Do Not Load Full Records Without Need

Use `select` for the fields required by the use case.

### Prove Tenant Isolation

Every tenant-owned query must include tenant scope or equivalent RLS enforcement.

---

## Daily Development Checklist

Before opening a Pull Request ask yourself:

- [ ] Are related database writes wrapped in a transaction?
- [ ] Did I select only the fields needed?
- [ ] Does every tenant-owned query prove tenant isolation?
- [ ] Could this query leak another tenant's data?
- [ ] Are database operations aligned with the business invariant?

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
- Protect domain identifiers with stronger types when mistakes are expensive.
- Treat database consistency and tenant isolation as production safety requirements.
- Query only the data the use case actually needs.

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
- ✅ Branded IDs for critical domain identifiers
- ✅ `satisfies` for typed configuration objects
- ✅ Prisma transactions for multi-step writes
- ✅ Prisma field selection with `select`
- ✅ Tenant-scoped Prisma queries