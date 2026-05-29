# PITFALLS — Armadilhas Conhecidas (Complete List)

All 69 pitfalls from the project-excellence skill (v9). The main SKILL.md keeps the top 25 most critical inline; this file preserves ALL 69 in full detail.

---

## 1. RLS circular dependencies

Se tabela A referencia B na policy e B referencia A, Postgres erro 42P17 "infinite recursion". Fix: SECURITY DEFINER helper com `public.` qualified names e `SET search_path = ''`.

## 2. REVOKE FROM PUBLIC (nao so anon/authenticated)

`REVOKE ... FROM anon, authenticated` NAO e suficiente em SECURITY DEFINER functions. PUBLIC e superset. Sem `REVOKE ALL ON FUNCTION ... FROM PUBLIC`, anon executa via `/rest/v1/rpc/`.

## 3. Vercel serverless rate limiting

In-memory (`Map`) reseta a cada cold start. Aceitavel como primeiro passo com trade-off documentado. Para producao com volume real, usar Upstash Redis ou Vercel KV. Nao bloqueie o deploy por rate limiting perfeito — in-memory com 60/min e melhor que zero.

## 4. Next.js API route auth — middleware > per-file

Middleware protege pages E api routes. O padrao recomendado: middleware verifica auth e injeta `x-user-id` header, rotas leem o header. Per-file `withAuth()` wrapping e FRAGIL quando handlers tem assinaturas inconsistentes. Middleware e 1 arquivo protegendo todas as rotas. **CRITICAL:** Se o codigo le `x-user-id` header, VERIFICAR que o middleware realmente SETA esse header. Se o middleware NAO seta o header mas o handler le (confiando no header como "middleware-injected"), qualquer cliente pode bypassar auth enviando `x-user-id: <uuid>` manualmente. Descoberto como VULN-HIGH em pentest real — o comentario dizia "set by middleware" mas o middleware nunca setou o header.

## 5. Open redirect em auth callback

Parametro `next` deve rejeitar valores com `//` (protocol-relative). So permitir paths com `/`.

## 6. IDOR em rotas com entity ID

Toda query DEVE filtrar por `userId` autenticado. Sem isso, usuario acessa dados de outros.

## 7. Next.js security headers

`next.config.ts` DEVE ter: CSP, X-Frame-Options, X-Content-Type-Options, HSTS. `poweredBy: false`.

## 8. SET search_path = '' exige public. prefix

Function com `SET search_path = ''` NAO encontra tabelas sem `public.` qualificado. Causa erro 42P01.

## 9. CodeRabbit false positive em imports

Verificar com `node -e "require('pkg')"` antes de aceitar.

## 10. CI falha por quota sem logs

Jobs morrem em 3-10s. Diagnostico: `gh api repos/OWNER/REPO/check-runs/JOB_ID/annotations`.

### 8. Coverage 100% e OBRIGATORIO para SaaS
Projetos com dinheiro real exigem 100% em todas metricas. Nao e aspiracional, e regra. NAO e line coverage apenas — branches DEVEM estar em 100%.

**Branch vs Line Coverage:** 100% line coverage ≠ 100% branch coverage. Branch coverage revela gaps em:
- Ternary operators (`condition ? A : B`)
- Short-circuit evaluation (`a && b()`)
- Optional chaining (`a?.b?.c`)
- Early returns dentro de blocos `if`

Quando buscando 100%, revisar branch gaps especificamente:
```bash
# Rodar coverage e checar branch percentage
npx vitest run --coverage

# Se Statements=100% mas Branches < 100%, checar:
# 1. Operadores ternarios
# 2. Guard clauses com valias impossiveis
# 3. Type guards (`typeof`, `instanceof`)
```

### 8. Coverage 100% e OBRIGATORIO para SaaS

Projetos que processam dinheiro real (payment provider) exigem 100% em todas as metricas (statements, branches, functions, lines, perFile). Nao e aspiracional, e regra. Se o modulo e simples demais pra ter 100%, simplifique mais ou exclua do coverage.

## 12. Lock file desync

Gerar com mesma versao de Node do CI.

## 13. Honeypot nome obvio

Nao usar `_canary_tokens`. Usar nomes criveis: `user_backup_2024`.

## 14. Supabase schema drift

SEMPRE rodar query no banco REAL antes de escrever migrations. Nao confie no schema file local.

## 15. db query --linked e transaction

Migration grande que falha na linha 490 = NADA aplica. Dividir em chunks.

## 16. Spec sem ARMADILHAS

Agente repete erros. Sempre documentar falhas anteriores.

## 17. Parallel pentest pattern

3 subagents: (1) DB/RLS, (2) Auth/API, (3) Frontend/XSS+OSINT+LGPD. Consolidar em relatorio unico.

## 18. Formulario de captacao sem protecao

Leads/newsletter forms sao alvo facil de spam, SQLi e XSS. Sempre: rate limit, input validation (zod), CSP header, sanitize output no admin.

## 19. LGPD nao e so checkbox

Dados de menores exigem: consentimento dos pais (nao da crianca), criptografia em repouso, log de acesso, politica de retencao, direito de exclusao funcional.

## 20. Landing page e surface area real

Atacantes nem chegam no login. Testam formulario de leads, /api/leads, headers, OSINT. Pentest SEMPRE comeca pela superficie publica.

## 21. Vercel preview com DB real

Preview deployments conectam ao mesmo banco. Se o branch tem rotas debug, elas ficam acessiveis no preview URL. Isolar DB por ambiente.

## 22. Pular spec e ir direto pra implementacao

Usuario corrige: "vai mapear, criar specs e so depois vai implementar". Mesmo com urgencia, seguir o pipeline: MAPEIA (entende o problema) → SPEC (documenta a solucao) → IMPLEMENTA (escreve codigo). Se o usuario pedir "faz agora", a spec pode ser 5 linhas mas TEM que existir. Pular spec = repetir erros = retrabalho.

## 23. Biome `noControlCharactersInRegex` e hardcoded

`\x00-\x1f`, `\u0000-\u001f`, `\x7f` — nenhum formato de escape passa. Nao tente 3 formatos diferentes. Use `String.charCodeAt()` em loop, ou remova a check se outras validacoes cobrem o caso.

## 24. `gh pr create --label` falha se label nao existe

O comando aborta INTEIRO (nao cria o PR). Criar labels antes com `gh label create`, ou omitir `--label`.

## 25. git SSH signing: `git commit -S SSH` e invalido

`-S` aceita GPG key ID, nao o tipo. Para SSH: `git config gpg.format ssh` + `git commit -S` (sem argumento).

## 26. CSRF `origin.includes(host)` e bypassavel por substring

`origin.includes(host)` aceita `https://evil-example.com` quando host e `example.com`. SEMPRE usar `new URL(origin).host === host` (comparacao estrita de hostname). Descoberto em pentest real — adicionar como check de review.

## 27. API response shape mismatch

Quando client e API sao escritos por agentes/PRs diferentes, o client pode extrair `json.data` mas a API retorna `{ questions, completed }` (sem campo `data`). React Query/React rejeita `undefined`. SEMPRE validar que o campo extraido pelo client existe no response real.

## 28. Husky v10+ format change

`#!/usr/bin/env sh` and `. "$(dirname -- "$0")/_/husky.sh"` are DEPRECATED in Husky v10. They WILL FAIL. New format: just write the command directly (e.g., `npx lint-staged\n`). No shebang, no source line.

## 29. validate:pr fragile pipefail + grep pattern

`set -euo pipefail` combined with `npx X 2>&1 | tail -5 | grep -q "passed"` can silently fail. The pipestatus + grep pattern is fragile. Fix: capture output in variable first (`OUTPUT=$(npx X 2>&1)`) then grep the variable. This also avoids `set -e` killing the script on grep non-match.

## 30. Husky pre-push needs explicit nvm PATH

Husky hooks run in a minimal shell that doesn't load `.bashrc`/`.nvmrc`. Pre-push hooks that call `npx` MUST set PATH explicitly: `export PATH="/home/user/.nvm/versions/node/v20.XX.X/bin:$PATH"`. Without this, `npx` not found or uses wrong Node version.

## 31. Subagents create type errors in tests

When delegating tasks to subagents, they may generate tests with wrong function signatures (e.g., passing args to a function that takes none). ALWAYS run `tsc --noEmit` after subagent work catches these. The subagent's tests may pass at runtime but fail type checking.

## 32. NUNCA ignorar testes falhando como "pre-existentes"

"CI vermelho e CI vermelho." Se 4 testes falham ANTES da sua mudanca, eles ainda precisam ser corrigidos. Regra #2 diz "CI vermelho = PR nao mergea" sem excecao. NAO classificar falhas como "pre-existentes" e seguir em frente — resolver TODOS. O usuario CORRIGIU esse comportamento explicitamente: "erros pre-existentes nao existem, existem erros que precisam ser resolvidos". Se `tsc --noEmit` tem erros, se `vitest run` tem falhas, se coverage crasha: tudo e bug ativo que precisa de fix AGORA.

## 33. Biome large-batch lint fix workflow

When committing a large batch (50+ files) and Husky blocks with 20-35 biome errors: (1) run `npx biome check --write --unsafe src/ tests/` to auto-fix ~80% of issues (unused vars, formatting, non-null assertions); (2) run `npx biome check src/ tests/` to see remaining errors; (3) fix remainder manually or via delegate_task subagents (max 2 parallel). CRITICAL: always re-run tests after biome fixes — auto-fixes can change runtime behavior (e.g., removing `!` non-null assertions changes types, replacing `<a href="#">` with `<button>` changes semantics). The test suite MUST be green after lint fixes.

## 34. Next.js `rel` attribute appends `noreferrer`

Next.js automatically adds `noreferrer` alongside `noopener` on `<a target="_blank">` elements. Tests asserting `expect(el).toHaveAttribute('rel', 'noopener')` will FAIL because the actual value is `"noreferrer noopener"`. Fix: use `expect(el.getAttribute('rel')).toContain('noopener')` or test both values.

## 35. Zod 4 `z.record()` requires 2 args

Zod 4.x changed `z.record()` to require `(keySchema, valueSchema)`. `z.record(z.unknown())` compiles in Zod 3 but fails in Zod 4 with a type error. Fix: `z.record(z.string(), z.unknown())`. Always check `package.json` for `zod` version before writing schemas. If `zod@^4.x`, all `z.record()` calls need 2 args.

## 36. Biome `noNonNullAssertion` auto-fix breaks hook deps

When Biome `--unsafe` auto-fixes `auth!.clubeId` to `auth?.clubeId` inside React hooks, the `useExhaustiveDependencies` rule then complains: (a) `auth` is used but not in deps, (b) `auth?.clubeId` in deps is "more than necessary". Correct fix: replace `auth!.clubeId` in deps array with `auth` (the whole object), and use `auth?.clubeId` in the callback body. Do NOT put `auth?.clubeId` in the deps array.

## 37. Stryker config stale after framework migration

When migrating from one framework to another (e.g. Vite/React SPA → Next.js App Router), the Stryker `mutate` array still points to old directories (`lib/`) instead of new ones (`src/`). Result: mutation testing silently tests zero production code. ALWAYS update `stryker.config.json` mutate paths after any directory restructuring. Check with `npx stryker run --dryRun` to verify mutants are being generated for the right files.

## 38. Module-level env constants + test isolation

`const X = process.env.Y.split(',')` no top-level de um modulo NAO e recalculado quando testes mutam `process.env.Y` com `beforeEach`. Node.js cacheia o modulo. Symptom: teste seta `process.env.ALLOWED_ORIGIN = '...'` mas a funcao usa valor antigo. Fix: lazy initialization — mover pra `function getX(): string[] { return process.env.Y?.split(',') ?? fallback }` chamada a cada invocacao. Alternativa: `vi.resetModules()` + `await import()` dinamico em cada teste (mais fragil). Preferir lazy init.

## 39. `require()` bypasses Vite alias resolution in tests

`require('@/lib/logger')` em testes Vitest NAO resolve aliases do `tsconfig.json` (paths como `@/*`). Retorna `Cannot find module '@/lib/logger'`. Fix: usar `vi.resetModules()` + `await import()` dynamic import pattern para cada teste que precisa de modulo fresco com env mutado. Pattern: `const { logger } = await import('@/lib/logger')` dentro de cada `it()` ou `test()`, precedido por `vi.resetModules()` no `beforeEach`. Nunca `require()` com aliases.

## 61. Dual lock files (package-lock + pnpm-lock) break Vercel build

When both `package-lock.json` and `pnpm-lock.yaml` exist in repo root, Vercel detects package manager change from npm to pnpm. Since pnpm is not configured (no corepack, no `.npmrc`), `pnpm install` exits with code 1. Error: "Skipping build cache since Package Manager changed from 'npm' to 'pnpm'" → "Command 'pnpm install' exited with 1". Fix: `git rm pnpm-lock.yaml`. Prevention: only use the project's designated package manager. Detection: `ls package-lock.json pnpm-lock.yaml yarn.lock bun.lockb`.

## 40. Biome `noNonNullAssertion` — use `as Type` cast

Biome proibe non-null assertions (`env[key]!`, `user!.name`). Substituir por type assertion: `env[key] as string`, `user as User`. Nao tentar `env[key]!` nem `user!` — o linter rejeita no pre-commit. Este e o erro #1 em commits rejeitados pelo Husky pre-commit hook.

## 41. `next build` timeout em hardware modesto

`next build` pode demorar >180s em maquinas com hardware modesto (AMD 4600G, 16GB RAM). Se o agente usa timeout de 180s, o build sera killed com exit code 124. Usar timeout >= 300s (600s e seguro). Isso vale para `npm run build` executado via terminal com timeout.

## 42. Write tests AFTER reading source, not before

When creating contract/unit tests for existing code, ALWAYS read the full source file first. Assumptions about return types (e.g., "parseBody returns parsed data directly" vs "returns ParseResult discriminated union") or defaults (e.g., "error defaults to 400" vs "defaults to 500") cause 2-3 rewrite cycles. The pattern: (1) read source, (2) identify return types + error paths + default values, (3) THEN write tests. This is the #1 cause of wasted test-writing effort.

## 43. Discriminated union return patterns need ok-check in tests

When a function returns `ParseResult<T> = { ok: true, data: T } | { ok: false, error: Response }`, tests MUST check `result.ok` before accessing `.data`. TypeScript narrowing requires `if (result.ok)`. Tests that do `result.data` directly will have undefined values and fail. Always read the type signature before writing test assertions.

## 44. `corsJson()` vs `corsHeaders()` — different purposes

In projects with CORS helpers, `corsJson(data)` returns a full Response with JSON body, while `corsHeaders()` returns only headers. Using `corsJson()` as a headers argument (0 args) causes TSC errors. Pattern: use `corsHeaders()` for header-only cases, `corsJson(data)` for full responses.

## 45. `git checkout --ours` silently loses main features

During merge conflict resolution, `git checkout --ours .` accepts HEAD for ALL conflicting files. If main had real changes (new types, imports, API signatures), they are discarded silently. ALWAYS run `tsc --noEmit` + full test suite after. Symptom: 0 conflicts reported but type errors appear from missing imports or changed signatures. Fix: restore specific source files from HEAD after merge, or resolve each conflict individually.

## 46. GitHub API branch protection PUT — boolean fields are literals

When calling `PUT /repos/{owner}/{repo}/branches/{branch}/protection`, fields like `enforce_admins`, `allow_force_pushes`, `allow_deletions` must be JSON boolean literals (`true`/`false`), NOT objects (`{"enabled": true}`). The API returns a confusing 422 validation error. Correct: `"enforce_admins": true`. Wrong: `"enforce_admins": {"enabled": true}`.

## 47. Force push to protected branch — 3-step API dance

When a PR has merge conflicts GitHub can't resolve (`mergeable: CONFLICTING`, `mergeStateStatus: DIRTY`), neither `--squash` nor `--admin` nor `--merge` works. Workaround: (1) Temporarily relax protection via API (`enforce_admins: false`, `allow_force_pushes: true`, `required_linear_history: false`), (2) `git push --force-with-lease`, (3) IMMEDIATELY restore protection to original state. Forgetting step 3 leaves main unprotected. This is a last resort — prefer clean merge when possible.

## 48. Auth wrapper changes handler signature in tests

When API routes use a `withAuth()` (or similar) wrapper that injects `userId`, the exported handler signature changes from `(request) => Response` to `(request, context) => Response`. Tests calling `handler(req)` with 1 arg get TS2554 "Expected 2 arguments, but got 1". Fix: `handler(req, {})` — pass empty object as context mock. This is the #1 type error after merging branches with different auth patterns.

## 49. Stryker sandbox accumulation

`.stryker-tmp/sandbox-*` directories accumulate across runs (each can be 100MB+). After multiple sessions, this eats disk space. Clean up: `rm -rf .stryker-tmp/`. Not a build issue but a disk space issue, especially on machines with limited storage.

## 50. API route tests must assert body, not just status

Tests that only check `res.status` miss `ObjectLiteral` and `StringLiteral` mutants (Stryker removes error message properties or empties strings). ALWAYS assert response body: `expect(data.error).toBe("Unauthorized")` not just `expect(res.status).toBe(401)`. This is the #1 source of surviving mutants in API route tests.

## 51. Coverage report references renamed/removed files

When `npm run test:coverage` reports gaps in filenames that don't exist on disk, the file was likely renamed, moved, or deleted since the last coverage baseline. Before writing tests, use `find src/ -type f | grep -i <partial>` to locate the actual file. Common in projects with active refactoring (e.g., `ReuniaoChecklist.tsx` → `RequirementsChecklist.tsx`, `AnoBiblicoYearRich.tsx` → `BibleYearRich.tsx`). Writing tests for a non-existent filename wastes a full cycle.

## 52. Subagent pentest/audit reads WRONG project

When delegating a pentest or code audit via `delegate_task`, the subagent may read files from a different repo than intended (e.g., reading `~/projetos/tesouros-portal/` when the task is about `~/estacio-prep/`). This generates false findings (routes, schemas, files that don't exist in the target project). **Prevention:** Always include `pwd && head -5 package.json` as the first subagent command to verify the correct repo. When reviewing subagent findings, cross-check that reported files actually exist in the target project before acting on them.

## 53. Pentest fix leaves tests validating INSECURE behavior

When you remove or fix insecure code (e.g., removing `x-user-id` header trust as auth bypass), the existing tests that assert the INSECURE behavior still exist. They either: (a) fail because the code no longer does the insecure thing, or (b) pass but validate a vulnerability. Both are wrong. **Action:** After every pentest fix, search test files for strings related to the finding (header names, bypass patterns, insecure defaults) and remove/update the tests that validate the old insecure behavior. This is the security equivalent of "each bug fix = regression test" — each security fix = remove test for the vulnerability.

## 54. `patch replace_all` is nuclear — collateral damage across entire file

Using `replace_all: true` with `skill_manage(action='patch')` replaces EVERY occurrence in the file. When the old_string is a short pattern like `, "user-1")` → `)`, it hits ALL occurrences including ones in different describe blocks that need different treatment. Result: mock chains break silently — tests still pass at runtime but route handlers lose their context params, causing coverage to DROP because the handler never reaches the main logic path. The test suite stays green (645/645) while coverage silently regresses. **Rule:** Only use `replace_all` when you're CERTAIN every occurrence should change identically. For selective changes, do targeted patches one at a time with enough surrounding context for uniqueness. Always re-run `--coverage` after bulk replacements to catch silent regressions.

## 55. V8 coverage: unreachable branches → simplify source, not force tests

When V8 Istanbul reports a branch with `vals=[N,0]` (one side never hit), it often means the branch is logically unreachable. Common patterns: (a) `?? ""` on array lookups with always-valid indices (DAY_NAMES, MONTHS), (b) `?? 0` on values already transformed by `.map()`, (c) `else if (lastMatchGroup)` when it's the only remaining option in a regex alternation, (d) `|| {}` / `?? {}` fallback that never activates. Forcing a test to hit these branches creates fragile, contrived test code that may not even be possible. Instead, simplify the source: replace `?? ""` with `!` non-null assertion, remove redundant fallbacks, convert `else if (lastOption)` to `else`. Each simplification adds a `// biome-ignore lint/style/noNonNullAssertion: reason` comment on the line before. This approach is faster, cleaner, and more maintainable. Full pattern catalog: `references/v8-coverage-patterns.md`.

## 56. biome-ignore for `noThenProperty` must go before the property line, not the object literal

Biome's `lint/suspicious/noThenProperty` checks the `then:` property definition, not the object literal that contains it. A `biome-ignore` comment placed before `const thenable = {` produces "Suppression comment has no effect." The comment must be on the line immediately before `then:`, with correct indentation matching the property level. Additionally, the indentation must match exactly — 2 spaces off = "suppression has no effect."

## 57. Vitest constructor mocks need real classes, not arrow functions

`vi.fn().mockImplementation(() => ({ ... }))` creates an arrow function that CANNOT be called with `new`. Code like `new ImapFlow()` or `new Database()` will throw `TypeError: ... is not a constructor`. This is the #1 cause of "not a constructor" test failures. Fix: use a real `class MockFoo` in the mock factory. For ESM default exports (`import Database from 'better-sqlite3'`), add `__esModule: true` alongside `default:` in the factory return. Never rely on `mock.results[0].value` to access constructor instances — it's unreliable in vitest. Full pattern catalog: career-ops skill `references/vitest-mock-patterns.md`.

## 58. Test assertions must match what code ACTUALLY does, not the filename

When writing tests for scrapers, API clients, or any module that calls external services, NEVER assume the URL/method/headers based on the module name. `wellfound-scraper.ts` might use `jsearch.p.rapidapi.com`, not `api.wellfound.com`. The test mock resolves (no runtime error) but the assertion fails with "Number of calls: 0" — meaning the test never exercised the real code path. Always read the actual source code FIRST, then write assertions that match reality. When env vars gate module behavior (e.g., `RAPIDAPI_JSEARCH_KEY`), use `vi.resetModules()` + `await import()` to ensure the module reads the env var set in the test.

## 59. User-provided project state is often stale or wrong

When the user (or context handoff) provides detailed project state (stash count, open PRs, bug status, test counts, CI status), VERIFY everything with actual commands BEFORE making decisions. Common mismatches: stashes already cleaned, bugs already closed, PRs already merged, test counts from a different session. Run `git stash list`, `gh pr list`, `npm run test`, `npx tsc --noEmit` in parallel — takes 15 seconds and prevents bad decisions on stale state.

## 60. Biome 2.x schema migration breaks existing configs

Biome 2.x has breaking schema changes from 1.9.x. Key changes:
- `organizeImports` at top level → `assist.actions.source.organizeImports.level: "on"`
- `noVar` rule removed (use `useConst` instead)
- `files.ignore` → removed. Available keys: `maxSize`, `ignoreUnknown`, `includes`, `experimentalScannerIgnores`. Use targeted file paths in commands instead of ignore patterns.
- `overrides` key: `include` (1.x) → `includes` (2.x)
- Always run `npx biome --version` first to determine schema version. Use `npx biome migrate` to auto-migrate config, but verify output — it doesn't always catch everything.

## 61. Vitest vi.mock fails for DB drivers in monorepo-like setups

When a module (e.g., `web/src/lib/db.ts`) imports `better-sqlite3`, and the test file is in `tests/unit/`, `vi.mock('better-sqlite3')` may NOT intercept the import because Node resolves the package from different `node_modules` directories (root vs web/). The mock silently fails and the real DB driver is loaded, causing "Cannot open database" errors.
**Fix:** Mock the wrapper module that exports the DB-accessing function, NOT the DB driver itself:
```ts
vi.mock("../../web/src/lib/db.ts", () => ({
  getDb: vi.fn().mockReturnValue({ /* mock DB object */ }),
  getTechTrends: vi.fn(() => [/* inline test data */]),
}));
```
Full module mock with inline implementation is more reliable than `vi.importActual` + partial override — the partial override still executes the real module code during import.

## 62. Replacing hardcoded component tables with shared utility — ALL tests break at once

When a component has local hardcoded tables (e.g., `getLevelFromXP`, `getXpForLevel`) and you replace them with a shared utility function (e.g., `getLevelInfo` from another module), EVERY test that asserts on values from the old table fails simultaneously. The number of failures can be 5-15+ tests in one file.

**Pattern:**
1. BEFORE modifying the component, use `npx tsx -e "import { fn } from './path'; [inputs].forEach(x => console.log(JSON.stringify({x, ...fn(x)})))"` to dump actual outputs from the new utility for ALL test case inputs
2. Rewrite the entire test file in one batch with correct values — don't fix one test at a time, the values are all different
3. Add new tests for the shared utility itself (pure function tests are fast and reliable)

**Technique:** `npx tsx -e` with inline import + forEach dump is the fastest way to verify pure function outputs before writing a single line of test code. Example:
```bash
npx tsx -e "
import { getLevelInfo } from './src/app/atividades/hooks/use-srs.ts';
[0, 300, 700, 1000, 3000, 10000, 34252, 100000].forEach(xp => {
  const info = getLevelInfo(xp);
  console.log(JSON.stringify({ xp, level: info.level, title: info.title, progress: +info.progress.toFixed(4) }));
});
"
```

**Real example:** ProfileCard had 3 local hardcoded functions (`getLevelFromXP`, `getXpForLevel`, `getXpForNextLevel`) with a CAP at level 10. Replaced with `getLevelInfo()` (progressive formula 100*N^1.5, no CAP). 9 tests failed simultaneously — all expecting old table values (level 8 at 6000 XP → now level 7, level 10 at 10000 XP → now level 9, CAP at 15000 XP → now level 11). Fix: dump all actual values with `npx tsx -e`, rewrite all 43 tests in one batch.

## 63. ProfileCard `profile.level` override vs computed progress — UX mismatch

When `level = profile.level ?? info.level` but `pct = info.progress * 100`, the displayed level can differ from what `getLevelInfo(xp)` computes. Example: DB has `level=2` but `getLevelInfo(5000)` returns level 7 — the badge shows "2" while the progress bar reflects level 7's progress. This is acceptable when DB level is authoritative (from a legacy system or manual adjustment), but document the design decision. If full consistency is required, derive level purely from `getLevelInfo(xp)` and ignore `profile.level`.

## 64. URLSearchParams.append vs set — dynamic key creates wrong query param

`url.searchParams.append(String(someVar), '')` uses the VALUE of the variable as the KEY. If `someVar = 'srs'`, this creates `?srs=` instead of `?type=srs`. The server reads `req.query.type` as `undefined`.

**Fix:** `url.searchParams.set('type', String(someVar))` — fixed key name, dynamic value.

**Pattern:** When building query params from dynamic values, ALWAYS use a fixed string as the key. `append` is for multi-value params (e.g., `?tag=a&tag=b`). `set` is for single-value params. NEVER pass the variable itself as the key. This is extremely hard to debug because the URL "looks right" (`?srs=`) but the server can't find the expected param.

## 65. Fire-and-forget API calls fail silently

When a function is called without `await` (fire-and-forget), any HTTP error (400, 401, 500, network failure) is swallowed. The UI shows the action succeeded (local state updates), but the server never received or processed the request.

**Debugging pattern:** When a user reports "action doesn't persist" but the UI shows success:
1. Check if the API call is `await`ed
2. Add `console.log` in the fetch wrapper to see actual URL and response
3. Query the DB directly before/after the action
4. The fix may need `await` OR the error may be elsewhere — but silent failures always start with a missing `await`

**Cost of fire-and-forget:** Makes debugging extremely hard. Consider adding `.catch(console.error)` at minimum if not converting to `await`.

## 66. Three-bug cascade pattern — single symptom, multiple independent root causes

When a single user-visible symptom ("missions don't complete") has multiple independent bugs stacked, fixing only one reveals the next failure. Must fix ALL before the feature works.

**Real example:**
1. **Bug 1 (client URL):** `searchParams.append(String(params.type), '')` → wrong query param key → server gets `type=undefined` → returns 400 "Operação inválida" silently (no `await` on the call)
2. **Bug 2 (server body fallback):** Legacy handler reads `type` only from `req.query`, but `apiFetch` sends it in body for POST → `type` still undefined → fix: `req.query.type || req.body?.type`
3. **Bug 3 (wrong handler):** `completeDailyMission()` added to App Router which is DEAD CODE → fix: add SQL inline in legacy handler

**Pattern:** When a fix doesn't work, don't assume the fix was wrong — there may be ANOTHER independent bug further down the chain. Systematically verify each layer: URL construction → server routing → handler logic → DB persistence.

## 67. Dev server restart required for legacy/Pages Router API files

Changes to `lib/api/routes/*.js` or `api/[...route].js` (Pages Router catch-all) do NOT hot-reload in Next.js dev mode. The dev server must be killed and restarted. This is different from `src/` files (App Router) which have HMR.

**Debugging \"frontend doesn't reflect backend changes\":**
1. Restart dev server after `lib/` or `api/` changes
2. Hard refresh browser (Ctrl+Shift+R) for client JS changes
3. Query DB directly to verify server-side state
4. Add `console.log` in fetch wrapper for client-side debugging

## 68. Auth guard checking client-side state when auth is server-side (cookies/JWT)

When a client-side API helper (e.g., `apiFetch`) reads auth state from `localStorage.getItem('user')` or `window.__authUser` to decide whether to make the request, but the REAL auth mechanism is a server-side JWT cookie (`credentials: 'include'`), the guard ALWAYS fails silently if nothing ever writes to that client-side storage. The result: every API call aborts with `return null` before the request is even made. No error, no console output, no network activity.

**Detection pattern:** When user reports "action doesn't persist" or "data doesn't save" but UI shows the action completing:
1. Check if the fetch wrapper has an auth guard reading client state
2. Verify the client state is actually SET somewhere (`grep -rn "localStorage.setItem('user'" src/`)
3. If nothing sets it, the guard is dead code that silently blocks ALL requests
4. Use Chrome DevTools MCP: `evaluate_script` to check `localStorage.getItem('user')` and `window.__authUser`

**Fix:** Remove the client-side auth guard from the fetch wrapper. Let the server handle auth via the cookie. If you need user info client-side, fetch it from a `/api/auth/me` endpoint on mount (which is what `useAuthGuard` already does).

**Real example:** `use-srs.ts` had `const user = getAuthUser(); if (!user?.email) return null;` in `apiFetch()`. `getAuthUser()` reads `localStorage('user')` — NEVER written anywhere. Auth is JWT cookie via `withClube` middleware. Every POST aborted silently for the entire lifetime of the quiz feature.

## 69. App Router sub-routes shadow legacy catch-all router

When a Next.js project has BOTH a legacy catch-all (`api/[...route].js`) and App Router sub-routes (`src/app/api/atividades/srs/route.ts`, `src/app/api/atividades/social/route.ts`, etc.), the App Router intercepts requests to the parent path (`/api/atividades`) BEFORE the catch-all. If there's no `route.ts` at the parent level (`src/app/api/atividades/route.ts`), the App Router returns 404 with HTML — the catch-all never fires.

**Symptom:** `curl` (without cookies, no RSC headers) returns 401 JSON (catch-all fires, auth rejects), but browser returns 404 HTML (App Router intercepts first). The difference: browser sends RSC/prefetch headers that trigger App Router routing priority.

**Fix options:**
1. (Preferred) Client fetches directly to the App Router sub-route: `/api/atividades/srs` instead of `/api/atividades?type=srs`
2. Create a passthrough `route.ts` at the parent level
3. Remove the catch-all handler and migrate everything to App Router

**Detection:** When debugging a 404 that shouldn't happen, check if `src/app/api/<path>/` has sub-directories with `route.ts` files. If yes, the App Router claims the namespace.

**Real example:** Quiz client POSTs to `/api/atividades?type=srs`. `src/app/api/atividades/srs/route.ts` exists. App Router intercepts `/api/atividades` → no `route.ts` at that level → 404. `curl` worked (no RSC headers) but browser got 404. Fix: client changed to POST directly to `/api/atividades/srs`.

Changes to `lib/api/routes/*.js` or `api/[...route].js` (Pages Router catch-all) do NOT hot-reload in Next.js dev mode. The dev server must be killed and restarted. This is different from `src/` files (App Router) which have HMR.

**Debugging "frontend doesn't reflect backend changes":**
1. Restart dev server after `lib/` or `api/` changes
2. Hard refresh browser (Ctrl+Shift+R) for client JS changes
3. Query DB directly to verify server-side state
4. Add `console.log` in fetch wrapper for client-side debugging


### 70. Knip false positives with CSS-imported packages

Knip does NOT analyze CSS `@import` statements. Packages like `shadcn` (imported as `@import "shadcn/tailwind.css"` in `globals.css`), `tw-animate-css`, and `tailwindcss` (used via PostCSS config, not JS imports) are flagged as "Unused dependencies" even though they are required at runtime. **CRITICAL:** Do NOT uninstall these without verifying — removing `shadcn` breaks the build with `Can't resolve 'shadcn/tailwind.css'`. Before removing any dep flagged by knip, grep CSS files: `grep -r "package-name" src/ --include="*.css"`. If found, it's a false positive — add to knip ignore or leave as-is.

### 70a. Knip "Unused files" vs "Unused exports" — different ignoreIssues keys

Knip reports THREE categories that need separate ignoreIssues keys: `"exports"`, `"types"`, and `"files"`. A file like `src/lib/metadata.ts` imported by Next.js server components via `generateMetadata` shows as "Unused files (1)" because KNIP can't trace imports through layout entry points. Adding only `["exports"]` to ignoreIssues fixes the export warning but NOT the "Unused files" warning. The fix requires `["exports", "files"]`. Always check which category KNIP reports and add all applicable keys.

```json
// knip.json — WRONG (still reports "Unused files")
"src/lib/metadata.ts": ["exports"]

// knip.json — CORRECT
"src/lib/metadata.ts": ["exports", "files"]
```

### 71. TSC readonly `process.env.NODE_ENV` in tests

`process.env.NODE_ENV` is typed as `string | undefined` (readonly) in `@types/node`. Tests that do `process.env.NODE_ENV = "production"` get TSC error TS2540 "Cannot assign to 'NODE_ENV' because it is a read-only property." Fix: cast via `Record<string, string | undefined>`: `(process.env as Record<string, string | undefined>).NODE_ENV = "production"`. Delete also fails with TS2704 — same cast works. Combine with `vi.resetModules()` + `await import()` for module-level constants that cache env at import time.

### 72. type-coverage requires local devDep install

`npx type-coverage` fails with `Cannot find module 'typescript'` when run from the NPX cache, even if typescript is installed locally. The NPX-cached `type-coverage-core` resolves modules from its own location, not the project. Fix: `npm install --save-dev type-coverage` to install locally, then run via `npx type-coverage` (which resolves the local copy first). This is needed for any CI or validate:pr gate that checks type coverage.

### 77. Disk full during file write causes data loss
When the disk reaches 100% capacity during a write operation (especially with `patch` or `write_file` tools), the file can be truncated to 0 bytes or partially written. This happened with SKILL.md (88KB → 0 bytes) during v9 upgrade. **Fix:** Monitor disk usage before large file operations: `df -h /`. Keep at least 5GB free. If write fails with ENOSPC, check the file immediately — it may be corrupted. Always have backups in multiple locations (profile copies, git, Obsidian). **Prevention:** Regular cleanup of caches (yarn, npm, playwright, inkscape) that accumulate silently.

### 78. Squash merge introduces CRLF across entire codebase
When merging a PR via `gh pr merge --squash`, GitHub's squash commit can introduce CRLF line endings across ALL changed files, even if every source file was LF. This causes Biome to report 50+ format errors on the very next commit attempt. The pre-commit hook (lint-staged → biome) then blocks all work.
**Fix:** Add `.gitattributes` at repo root:
```
* text=auto eol=lf
*.bat text eol=crlf
```
Then renormalize: `git rm --cached -r . && git reset --hard HEAD` (careful: this resets working tree). After `.gitattributes` is committed, all future checkouts enforce LF.
**Prevention:** Add `.gitattributes` as part of initial project setup (PE FASE 1). Run `npx biome check --write src/ tests/` after any squash merge to catch and fix line-ending drift immediately.

### 79. Husky hook files with CRLF cause cryptic npx failures
When Husky hook files (`.husky/pre-commit`, `.husky/pre-push`) contain CRLF (`\r\n`) line endings, `npx lint-staged` fails with: `npm error notarget No matching version found for undefined@lint-staged`. The `\r` character corrupts the command parsing.
**Fix:** Write hooks using `printf` (which outputs LF-only) instead of editors that may add CRLF:
```bash
printf 'npx lint-staged\n' > .husky/pre-commit
printf 'npx vitest run\n' > .husky/pre-push
chmod +x .husky/pre-commit .husky/pre-push
```
Verify: `cat -A .husky/pre-commit` should show `$` at end of line, not `^M$`. If `.gitattributes` enforces LF (pitfall #78), this problem is prevented automatically after the first commit.

### 80. .mjs imports with unicode crash Vitest SSR parser
Test files that import `.mjs` scripts (e.g., `import { runDedup } from "../../../scripts/pipeline/dedup.mjs"`) and contain unicode characters (em-dash `—`, smart quotes, etc.) in `describe()` strings cause `SyntaxError: Invalid or unexpected token` in Vitest. The Vite SSR parser cannot handle the combination. Tests pass zero, suite reports as failed with no individual test results.
**Fix:** Either (a) exclude these test files from Vitest (`exclude: ["tests/unit/pipeline/*.test.ts"]` in `vitest.config.ts`) or (b) replace unicode in describe strings with ASCII equivalents (`" - "` instead of `"—"`).
**When to exclude:** If the `.mjs` scripts are legacy tooling (pipeline scripts, build helpers) rather than core business logic, excluding their tests is pragmatic. Document the exclusion reason in vitest.config.ts.

### 81. GitHub Actions billing failure blocks PR merge when CI is required
When branch protection requires CI checks to pass but GitHub Actions fails due to billing/spending limits ("recent account payments have failed or your spending limit needs to be increased"), ALL PRs are blocked from merging. The CI status shows `FAILURE` but with zero test steps executed — the job dies in 2-3 seconds before any step runs.
**Fix:** Remove required status checks from branch protection (`gh api repos/OWNER/REPO/branches/main/protection -X PUT` with `"required_status_checks": null`). Quality is still guaranteed by LOCAL gates: Husky pre-push runs `validate:pr` (biome + vitest + tsc) before any push reaches GitHub. CI becomes informational, not blocking.
**Philosophy:** For solo dev projects on free GitHub tiers, local quality gates (Husky + validate:pr) are the real safety net. CI is a bonus, not a gate. Don't let billing issues block development.

### 82. License checker --onlyAllow breaks on every new permissive license

Using `npx license-checker-rseidelsohn --onlyAllow "MIT;ISC;Apache-2.0;BSD-2-Clause;BSD-3-Clause"` causes build failures every time a dependency uses a permissive license not in the allowlist (e.g., BlueOak-1.0.0, CC0-1.0, Unlicense, Python-2.0, MPL-2.0, CC-BY-4.0, CC-BY-3.0, LGPL-3.0-or-later). Real example: Quizoteca had 13 permissive licenses across deps, requiring 13 additions to the allowlist.

**Fix:** Use `--failOn` instead (blocklist approach):
```bash
npx license-checker-rseidelsohn --failOn 'GPL-3.0;AGPL-3.0;GPL-2.0;SSPL-1.0;BSL-1.1'
```
This only blocks genuinely problematic licenses (copyleft, source-available). All permissive licenses pass by default. New deps with unusual but permissive licenses don't break the build.

### 83. Next.js `title.template` + child page title = duplicate brand name

When `layout.tsx` has `title: { template: "Brand | %s", default: "..." }`, child pages that export `title: "Page Name — Brand"` render as `"Brand | Page Name — Brand"`. The template already adds the brand prefix — child pages must export ONLY the page-specific part. Example: `title: "Política de Privacidade"` renders as `"Brand | Política de Privacidade"`. Common in legal pages copied from templates that include the brand suffix. Audit by checking `<title>` tag on every page in the browser.

### 84. Fixing visible text (accents, typos) breaks tests asserting on that text

When correcting PT-BR accents ("Comecar" → "Começar") or typos in components, `screen.getByText("Comecar")` in tests will fail. ALWAYS grep tests for exact strings being changed and update simultaneously. Pattern: `grep -rn "oldText" tests/`. Also do a comprehensive pass on body text, not just headings — "detalhadaa" (double-a) and "voce" (missing accent) hide in paragraphs. The test update is mechanical but must not be skipped.
