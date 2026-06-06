# AGENTS.md — Next.js Backend API Project

> Cross-tool standard (read by Aider, Cursor, Copilot, Gemini CLI, etc.)
> Keep this as the single source of truth for all AI agents on this project.

---

## Project Overview

A pure backend API built with Next.js App Router. No frontend pages — only
API routes consumed by web and mobile clients. All routes are versioned under
`/api/v1/`.

---

## Stack

- **Runtime**: Node.js 20 LTS
- **Framework**: Next.js 16 (App Router, API routes only)
- **Language**: TypeScript 5.x — strict mode, no `any`
- **ORM**: Prisma + PostgreSQL
- **Auth**: NextAuth.js v5 (web session cookies) + custom JWT Bearer (mobile)
- **Validation**: Zod — all request bodies and query params
- **API Spec**: swagger-jsdoc + swagger-ui-next (OpenAPI 3.0)
- **Testing**: Vitest + Supertest (integration tests against real routes)
- **Linting**: ESLint + tsc --noEmit

---

## Directory Structure

```
app/
  api/
    auth/
      [...nextauth]/route.ts   # NextAuth handler
      mobile/
        token/route.ts         # Issue JWT for mobile clients
        refresh/route.ts       # Refresh JWT
    v1/
      users/
        route.ts               # GET /api/v1/users, POST /api/v1/users
        [id]/route.ts          # GET /api/v1/users/:id, PATCH, DELETE
      <resource>/
        route.ts
        [id]/route.ts
    swagger/
      route.ts                 # Serves swagger.json
      ui/route.ts              # Serves Swagger UI

lib/
  auth.ts                      # NextAuth config + JWT helpers
  db.ts                        # Prisma singleton
  swagger.ts                   # swagger-jsdoc spec builder
  middleware/
    withAuth.ts                # Auth guard (handles both session + JWT)
    withValidation.ts          # Zod request validation wrapper
    rateLimit.ts               # Redis-based rate limiting
  validators/
    users.ts                   # Zod schemas for user routes
    <resource>.ts
  errors.ts                    # Shared error response helpers

prisma/
  schema.prisma

tests/
  <resource>/
    route.test.ts              # Integration tests per resource
```

---

## API Route Conventions

### Every route file MUST have:

1. **Zod schema** — validate all inputs before touching the DB
2. **JSDoc `@openapi` annotation** — for Swagger spec generation
3. **Auth guard** — via `withAuth()` middleware
4. **A corresponding test file** in `tests/<resource>/route.test.ts`

### Route handler template:

```ts
import { NextRequest, NextResponse } from 'next/server'
import { withAuth } from '@/lib/middleware/withAuth'
import { withValidation } from '@/lib/middleware/withValidation'
import { z } from 'zod'
import { db } from '@/lib/db'
import { errorResponse } from '@/lib/errors'

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
})

/**
 * @openapi
 * /api/v1/users:
 *   post:
 *     summary: Create a user
 *     tags: [Users]
 *     security:
 *       - BearerAuth: []
 *       - SessionAuth: []
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             $ref: '#/components/schemas/CreateUser'
 *     responses:
 *       201:
 *         description: User created
 *       400:
 *         description: Validation error
 *       401:
 *         description: Unauthorized
 */
export const POST = withAuth(
  withValidation(CreateUserSchema, async (req: NextRequest, body) => {
    try {
      const user = await db.user.create({ data: body })
      return NextResponse.json(user, { status: 201 })
    } catch (err) {
      return errorResponse(err)
    }
  })
)
```

---

## Authentication

### Web clients (browser)
- Use NextAuth.js v5 session cookies
- `withAuth()` checks `auth()` from NextAuth first

### Mobile clients
- Use Bearer JWT in `Authorization: Bearer <token>` header
- Issue tokens via `POST /api/auth/mobile/token`
- `withAuth()` falls back to JWT verification if no session found
- JWTs expire in 15 minutes; refresh tokens expire in 30 days

### `withAuth` middleware behaviour:
1. Try `auth()` — valid NextAuth session → pass through with `req.user`
2. Try `Authorization: Bearer` header — valid JWT → pass through with `req.user`
3. Neither → return `401 Unauthorized`

---

## Validation Rules

- Use Zod for ALL input validation — request body, query params, path params
- Schemas live in `lib/validators/<resource>.ts`
- `withValidation(Schema, handler)` wrapper parses and injects typed body
- On validation failure: return `400` with structured errors:

```json
{
  "error": "Validation failed",
  "details": [{ "field": "email", "message": "Invalid email" }]
}
```

---

## Error Response Format

All error responses follow this shape — use `errorResponse()` from `lib/errors.ts`:

```json
{
  "error": "Human-readable message",
  "code": "MACHINE_READABLE_CODE",
  "details": {}
}
```

Standard HTTP status codes:
- `400` — validation error
- `401` — unauthenticated
- `403` — forbidden (authenticated but not authorized)
- `404` — resource not found
- `409` — conflict (e.g. duplicate email)
- `500` — internal server error (never expose raw DB errors)

---

## Swagger / OpenAPI

- Every route handler must have a `@openapi` JSDoc block
- Run `npm run swagger:build` to regenerate `public/swagger.json`
- Spec is served at `GET /api/swagger` (JSON) and `GET /api/swagger/ui` (UI)
- OpenAPI 3.0.0, two security schemes: `BearerAuth` and `SessionAuth`

---

## Testing Requirements

- **Framework**: Vitest + Supertest
- **Coverage target**: 100% lines, functions, branches on `app/api/` and `lib/`
- Every route test file covers:
  - Happy path (200/201)
  - Validation failure (400)
  - Unauthenticated request (401)
  - Forbidden (403) where applicable
  - DB error handling (500 returns generic message, not raw error)

```bash
npm run test            # run all tests
npm run test:coverage   # coverage report (fails if under threshold)
```

---

## Git Branching Strategy

Strategy: **trunk-based development** with short-lived feature branches.
All branches are cut from `main` and merged back to `main` via PR.
Branches should live hours to days — never weeks.

### Protected branches

| Branch | Purpose | Who can push |
|--------|---------|--------------|
| `main` | Production-ready code | PRs only (CI must pass) |

### Branch naming

```
feat/<scope>-<short-description>    # new endpoints or features
fix/<scope>-<short-description>     # bug fixes
test/<scope>-<short-description>    # adding or fixing tests only
refactor/<scope>-<description>      # internal changes, no behaviour change
chore/<description>                 # deps, config, tooling
docs/<description>                  # documentation only
hotfix/<description>                # urgent production fix (branch from main, PR fast-track)
```

Examples:
```
feat/users-list-endpoint
feat/mobile-jwt-refresh
fix/auth-cookie-expiry
test/users-coverage-gaps
chore/upgrade-prisma-5
hotfix/token-validation-bypass
```

### Branch lifecycle

```bash
# 1. Always start from a fresh main
git checkout main && git pull origin main

# 2. Create a feature branch
git checkout -b feat/users-list-endpoint

# 3. Work and commit (Aider auto-commits during sessions)

# 4. Before opening a PR — rebase onto latest main (never merge main into feature)
git fetch origin
git rebase origin/main

# 5. Push and open PR
git push origin feat/users-list-endpoint

# 6. After PR is merged — delete the branch
git branch -d feat/users-list-endpoint
git push origin --delete feat/users-list-endpoint
```

### PR rules

- CI must pass (typecheck + lint + full test suite) before merge
- Squash and merge into `main` — one clean commit per feature
- PR title must follow conventional commit format: `feat(users): add list endpoint`
- Delete branch after merge

### Commit conventions

Format: `<type>(<scope>): <description>`

Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `hotfix`
Scope: the resource or module changed — e.g. `users`, `auth`, `swagger`, `prisma`

```
feat(users): add paginated list endpoint
fix(auth): correct JWT expiry calculation
test(posts): add missing 403 coverage
chore(deps): upgrade prisma to 5.14
```

Rules:
- Description under 72 characters, imperative mood, no trailing period
- Never commit `.env` or any file containing secrets
- Never commit directly to `main`
- Run `npm run typecheck && npm run lint` before pushing

---

## Commands

```bash
npm run dev              # start dev server
npm run build            # production build
npm run typecheck        # tsc --noEmit
npm run lint             # eslint
npm run test             # vitest
npm run test:coverage    # vitest with coverage
npm run swagger:build    # regenerate swagger.json
```