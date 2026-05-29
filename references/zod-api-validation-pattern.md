# Zod Input Validation Pattern — Next.js App Router

## When to Use

Sec 3 checklist: "Input validation em todo endpoint (zod/joi)". Sempre que um projeto tem API routes sem validacao, aplicar este padrao.

## Architecture

```
src/lib/validation.ts          ← Central utility (parseBody, parseQuery, formatZodError)
src/lib/schemas/<domain>.ts   ← Zod schemas por dominio (auth, users, atividades, etc.)
src/app/api/*/route.ts        ← Routes import schemas + use parseBody
```

## Core Utility — `parseBody`

```typescript
// src/lib/validation.ts
import { z } from 'zod'

type ParseResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: Response; errorData: { error: string; details?: unknown } }

export async function parseBody<T extends z.ZodTypeAny>(
  req: Request,
  schema: T,
): Promise<ParseResult<z.infer<T>>> {
  try {
    const raw = await req.text()
    const json = JSON.parse(raw)
    const result = schema.safeParse(json)
    if (result.success) return { ok: true, data: result.data }
    return {
      ok: false,
      error: new Response(
        JSON.stringify(formatZodError(result.error)),
        { status: 400, headers: { 'Content-Type': 'application/json' } },
      ),
      errorData: formatZodError(result.error),
    }
  } catch (e) {
    return {
      ok: false,
      error: new Response(
        JSON.stringify({ error: 'Body invalido' }),
        { status: 400, headers: { 'Content-Type': 'application/json' } },
      ),
      errorData: { error: 'Body invalido' },
    }
  }
}
```

## Route Integration Pattern

```typescript
// BEFORE (manual validation — DELETE this)
const body = (await req.json()) as { email?: string; password?: string }
if (!body || !body.email || typeof body.email !== 'string') {
  return NextResponse.json({ success: false, message: 'Email obrigatorio' }, { status: 400 })
}

// AFTER (Zod validation — USE this)
const parsed = await parseBody(req, LoginBodySchema)
if (!parsed.ok) {
  return NextResponse.json(
    { success: false, message: parsed.errorData.error },
    { status: 400 },
  )
}
// parsed.data is fully typed
```

## Schema Organization

```typescript
// src/lib/schemas/auth.ts
import { z } from 'zod'

export const LoginBodySchema = z.object({
  email: z.string().email('Email invalido'),
  password: z.string().min(1, 'Senha obrigatoria'),
})

export const ChangePasswordBodySchema = z.object({
  old_password: z.string().min(1, 'Senha atual obrigatoria'),
  new_password: z.string().min(6, 'Senha deve ter no minimo 6 caracteres'),
})
```

Group schemas by domain: `auth.ts`, `users.ts`, `atividades.ts`, `clube.ts`, `admin.ts`.

## `parseQuery` for GET/DELETE params

```typescript
export function parseQuery<T extends z.ZodTypeAny>(
  url: string | URL,
  schema: T,
): ParseResult<z.infer<T>> {
  const params = Object.fromEntries(new URL(url).searchParams)
  const result = schema.safeParse(params)
  // same return shape as parseBody
}
```

## Key Decisions

1. **`req.text()` instead of `req.json()`** — Avoids double-read errors when body is consumed by multiple middleware
2. **Discriminated union return** — `{ ok: true, data }` vs `{ ok: false, error }` forces handling both cases
3. **`errorData` alongside `error` Response** — Routes can extract error message for custom response shapes (e.g., `{ success: false, message: ... }`)
4. **`z.coerce.number()` for query params** — URL searchParams always returns strings; coerce handles `?id=5` → `5`

## TDD Approach

1. Write validation.test.ts FIRST (16+ tests: valid input, missing field, wrong type, empty body, invalid email, etc.)
2. Confirm RED (module doesn't exist)
3. Implement validation.ts
4. Confirm GREEN
5. Integrate into routes (1 route at a time, run tests after each)

## Pitfalls

- **Routes that read body before parseBody**: If middleware/handler already called `await req.json()`, `parseBody` will throw (body already consumed). Fix: ensure parseBody is the FIRST thing that reads the body.
- **Conditional schemas**: Some endpoints have different body shapes based on `action` query param. Use `z.discriminatedUnion('action', [...])` or parse `action` separately and branch.
- **`passthrough()` on update schemas**: When the route uses dynamic `SET` builder (any field can be updated), use `.passthrough()` to allow unknown fields through Zod.
- **Response shape compatibility**: Legacy routes may return `{ success: false, message }` on error. Convert parseBody error: `{ success: false, message: parsed.errorData.error }`.
