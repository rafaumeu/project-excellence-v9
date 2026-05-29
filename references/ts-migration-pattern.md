# TS Migration Pattern — Large JS Pages Router → App Router Sub-routes

## When to Use

Migrating large JS files from Next.js Pages Router (`pages/api/*.js` or Vite functions `lib/api/routes/*.js`) to App Router (`src/app/api/*/route.ts`). Typical for projects evolving from prototype to production.

## Decomposition Strategies

### Strategy A: Entity-based (single entity with actions)

**Example**: `users.js` (1291 lines) — CRUD for users + special actions (link_child, sync_adventurers)

```
users.js →
  users/route.ts         (GET list/single, POST create, DELETE)
  users/update/route.ts  (PUT — dynamic field update)
  users/link/route.ts    (POST link, DELETE unlink)
  users/sync/route.ts    (POST bulk sync)
  criancas/route.ts      (separate entity — GET/POST/PUT/DELETE)
```

### Strategy B: Type-dispatch (mega-switch on query param)

**Example**: `atividades.js` (2390 lines) — 23 types in a coreHandler switch

1. **Group types by domain**: XP/achievements (award_xp, achievements, achievement_feed, manual_achievement), SRS (srs, boosts), Bible (bible_year, sabbath_school, atividades_extras), Social (indication, reaction, activity_feed), Profile (profile, public_profile, streak, daily_missions, weekly_stats)
2. **Extract shared helpers**: `addXP()`, `checkAchievements()`, `getActiveMultiplier()`, `grantAchievement()` → `src/lib/atividades/` modules
3. **One route.ts per domain group**: `src/app/api/atividades/xp/route.ts`, `src/app/api/atividades/srs/route.ts`, etc.

### Strategy C: Simple 1:1 (small routes <200 lines)

Just rewrite in place with same endpoints. Add types, fix formatting.

## Step-by-step Process

1. **Read the JS original completely** — understand all branches, edge cases, auth patterns
2. **Map all handlers** — method + route + query params → table of all endpoints
3. **Choose decomposition strategy** (A, B, or C above)
4. **Write shared helpers first** — if Strategy B, extract to `src/lib/` modules
5. **Write route files** — follow the pattern below
6. **Run `npx tsc --noEmit --project tsconfig.json`** — fix type errors
7. **Run `npx biome check --write --unsafe src/app/api/path/`** — fix formatting
8. **Run `npx vitest run`** — verify no regressions
9. **Commit** — keep JS original (dual-build coexistence)

## Route Handler Pattern

```typescript
import { corsJson, corsOptions } from '@/lib/cors';
import { withDb } from '@/lib/db';
import { handleError } from '@/lib/middleware/error-handler';
import { withClube } from '@/lib/middleware/with-clube';
import type { AuthPayload } from '@/lib/types';
import { type NextRequest, NextResponse } from 'next/server';

export async function OPTIONS() {
  return corsOptions();
}

export async function GET(req: NextRequest) {
  try {
    const auth = await withClube(req);
    if (auth.error) {
      return NextResponse.json(
        { success: false, message: auth.error },
        { status: auth.status ?? 401 },
      );
    }
    const { payload } = auth as { payload: AuthPayload };

    // Role check
    // ...

    return withDb(async (client) => {
      // Parameterized queries only: $1, $2, etc.
      // Multi-tenancy: WHERE clube_id = $N on EVERY query
      const { rows } = await client.query(
        'SELECT * FROM table WHERE clube_id = $1',
        [payload.clubeId],
      );
      return corsJson({ success: true, data: rows });
    });
  } catch (error) {
    return handleError(error);
  }
}
```

## Mandatory Rules

1. **Multi-tenancy**: `WHERE clube_id = $N` on EVERY query. No exceptions.
2. **Parameterized queries**: Never concatenate user input. Always `$1, $2, ...`.
3. **Auth on every route**: `withClube(req)` for authenticated, or explicit public route.
4. **Types from `@/lib/types`**: No `any`. Use interfaces from central types file.
5. **Biome clean**: Run `biome check --write --unsafe` before committing.
6. **Don't delete JS original**: Dual-build means both codebases coexist until cutover.

## Common Pitfalls

- `corsJson()` does NOT accept a status code second parameter
- `sanitizeBody()` returns `T | null` — always null-coalesce with `?? {}`
- `hashPassword` is in `@/lib/auth-utils`, not in route-specific files
- `git commit -S ssh` is wrong — use `git commit -S -m "msg"` (just -S flag)
- Dynamic SET builder: use array of updates + parameterized `$${idx++}` pattern
- **userId type mismatch**: `addXP()`, `grantAchievement()`, `checkAchievements()` expect `userId: number`, but `payload.userId` is `string`. ALWAYS wrap with `Number(payload.userId)`. This hit EVERY single atividades route.
- **`getActiveMultiplier()` union return**: Returns `number | MultiplierResult`. When `returnDetails=true`, returns object. Use type guard: `typeof result === 'object' ? result.multiplier : Number(result)`. Accessing `.multiplier` directly on the union is a TSC error.
- **Biome `noNonNullAssertion`**: `!` operator is forbidden by Biome lint. Use `?? ''` or `?? defaultValue` instead. `achievementName!` → `achievementName ?? ''`.
- **`instanceof Date` in pg rows**: Postgres driver returns `Date` objects for timestamp columns, even when your type annotation says `string`. Annotate as `string | Date` or handle both: `val instanceof Date ? val.toISOString().slice(0,10) : String(val).slice(0,10)`.
- **Cross-tenant queries missing clube_id**: When migrating JS routes that INSERT/DELETE on junction tables (e.g., `pais_criancas`), the original JS may NOT filter by `clube_id`. ALWAYS add `clube_id` to INSERT values and WHERE clauses. Code review caught a real cross-tenant link manipulation bug this way — a user could link/unlink parents to children in OTHER clubs.
- **CI free tier exhausted — merge workflow**: When GitHub Actions minutes are depleted, use `gh pr merge NNN --squash --admin --delete-branch`. The local `git pull` after merge fails if you have unstaged changes — `git stash && git pull --rebase origin main && git stash pop` is the safe sequence.
- **`calculateSRSResult` signature mismatch**: Helper may expect `(currentLevel: number, isCorrect: boolean)` but JS passes `(cardId, correct, activityType)`. When helpers don't match JS signatures, either inline the logic from JS or fix the helper call — don't blindly pass wrong args.
- **Biome double-quote preference**: Biome format wants `''` over `""` for string literals. `sed -i` with escaped quotes can cause format mismatches. Run `biome check --write --unsafe` after any `sed` operation.

## Verification Checklist (every migration batch)

```bash
# 1. TypeScript check — ONLY your new files
npx tsc --noEmit --project tsconfig.json 2>&1 | grep "your-path" | grep "error"

# 2. Biome format + lint
npx biome check --write --unsafe src/app/api/your-path/

# 3. Full test suite (catches regressions)
npx vitest run

# 4. Commit (GPG signed)
git add src/app/api/your-path/
git -c commit.gpgsign=true commit -S -m "feat(api): description"

# 5. Push (bypass GITHUB_TOKEN env var)
GITHUB_TOKEN= git push -u origin branch-name

# 6. PR + admin merge (CI free tier workaround)
GITHUB_TOKEN= gh pr create --base main --head branch-name --title "title" --body "body"
GITHUB_TOKEN= gh pr merge NNN --squash --admin --delete-branch
git stash && git pull --rebase origin main
```
