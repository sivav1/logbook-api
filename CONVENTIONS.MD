# CONVENTIONS.md — Aider Coding Rules

> Aider-specific instructions. Loaded every session via `read: CONVENTIONS.md`.
> For architecture, stack, templates, and examples, see AGENTS.md.

---

## Behaviour

- Before writing code for a new route, confirm the resource name and HTTP methods needed
- If a task is ambiguous, ask one clarifying question — do not guess and proceed
- Do not install new npm packages without asking first
- Do not rename or move existing files unless explicitly asked
- Do not change `prisma/schema.prisma` or `middleware.ts` without explicit instruction
- Never add `console.log` — use `lib/logger.ts` if logging is needed

---

## TypeScript

- Strict mode — no `any`, no non-null assertions (`!`), no `require()`
- Use `unknown` and narrow with type guards or Zod instead of `any`
- Prefer `type` over `interface` unless declaration merging is needed
- Max function length: 40 lines — extract helpers if longer
- Prefer early returns over deeply nested `if` blocks

---

## Code Generation Order

When adding a new resource, always generate in this exact order — never skip steps:

1. `lib/validators/<resource>.ts` — Zod schemas + exported inferred types
2. `app/api/v1/<resource>/route.ts` — collection endpoints (GET, POST)
3. `app/api/v1/<resource>/[id]/route.ts` — item endpoints (GET, PATCH, DELETE)
4. `tests/<resource>/route.test.ts` — full test coverage
5. Remind to run `npm run swagger:build`

If only some endpoints are needed, confirm which before generating.

---

## Route Rules

- Every handler wrapped with `withAuth()` — no unprotected routes
- Every POST/PATCH wrapped with `withValidation(Schema, handler)`
- Schemas imported from `lib/validators/` — never defined inline in route files
- All DB calls inside try/catch — use `errorResponse(err)` in every catch block
- Return `NextResponse.json(data, { status: CODE })` — never return raw objects

---

## Zod Rules

- Schemas in `lib/validators/<resource>.ts` — export schema and inferred type
- String fields: always `.trim().min(1)` to reject blank strings
- Path param IDs: validate with `z.string().cuid()` or `z.string().uuid()`
- Use `.optional()` not `.nullable()` unless the DB column allows null

---

## Swagger Rules

- Every exported route handler needs a `@openapi` JSDoc block
- Always include both `BearerAuth` and `SessionAuth` under `security`
- Always include 400, 401, and 500 responses, even if brief
- Tags must match the resource name capitalised: `[Users]`, `[Posts]`

---

## Testing Rules

- Test files in `tests/<resource>/route.test.ts`
- Every test file must cover: happy path, 400 validation, 401 unauth, 500 DB error
- Add 403 forbidden case wherever role checks exist
- Mock Prisma with `vi.mock('@/lib/db')` — never hit the real DB
- Mock `withAuth` to inject a fake `req.user` for authenticated route tests
- Throw a real `Error` object for DB error simulation — not a string

---

## Error Rules

- Always use `errorResponse(err)` from `lib/errors.ts` in catch blocks
- For known errors (404, 409): return `NextResponse.json({ error, code }, { status })`
- Never expose raw Prisma error messages in responses
- Detect unique constraint violations via `PrismaClientKnownRequestError` code `P2002` → 409

---

## Git / Branching Rules

- Check the current branch before starting any task
- If on `main`, stop — propose a branch name before writing any code
- Never commit directly to `main`
- Branch naming: `feat/<resource>-<desc>`, `fix/<scope>-<desc>`, `test/...`, `chore/...`
- Never run `git push --force` — use `--force-with-lease` only if explicitly asked
- Never merge `main` into a feature branch — rebase: `git fetch origin && git rebase origin/main`
- After completing a task, remind the user to push and open a PR — do not push automatically

---

## What NOT to Generate

- No frontend pages, React components, or UI code
- No `.env` files or secrets
- No `console.log` or `console.error`
- No new npm scripts without asking