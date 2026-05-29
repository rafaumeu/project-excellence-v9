# Coverage-Gap-Driven Testing Workflow

Workflow for reaching 100% coverage on an existing codebase (NOT TDD — this is for closing gaps after initial implementation).

Use when the user says "we need 100% coverage" or when CI gates block on thresholds.

## Phase 1: Find the Gaps

```bash
# Fast: run coverage excluding slow integration tests
npx vitest run --exclude='tests/integration/**' --coverage 2>&1 | tee /tmp/cov1.txt

# Parse coverage summary to find files below 100%
cd coverage
python3 -c "
import json
d = json.load(open('coverage-summary.json'))
for f in sorted(d):
    s = d[f]
    if (s.get('statements',{}).get('pct',100) < 100 or
        s.get('branches',{}).get('pct',100) < 100 or
        s.get('functions',{}).get('pct',100) < 100):
        print(f'{f.split(\"/\")[-1]:40s} stmts={s[\"statements\"][\"pct\"]:5.1f}  branches={s[\"branches\"][\"pct\"]:5.1f}  funcs={s[\"functions\"][\"pct\"]:5.1f}  lines={s[\"lines\"][\"pct\"]:5.1f}')
"
```

**Better: sort by biggest gap first:**
```bash
cd coverage
python3 -c "
import json
d = json.load(open('coverage-summary.json'))
files = [(k, v['statements']['pct'], v['branches']['pct'], v['functions']['pct']) for k,v in d.items()]
files.sort(key=lambda x: x[1])  # sort by stmts ASC
for fname, st, br, fn in files:
    if st < 100 or br < 100:
        print(f'{st:5.1f}% stmts  {br:5.1f}% br  {fn:5.1f}% fn  {fname.split(\"/\")[-1]:40s}')
"
```

## Phase 2: Drill Into a Specific File

```bash
# Find ALL uncovered statements and branches in a file
node -e "
const cov = require('./coverage/coverage-final.json');
const target = process.argv[1] || '';
for (const [key, data] of Object.entries(cov)) {
  if (!key.includes(target.replace('.ts','').replace('.tsx',''))) continue;

  // Statements
  for (const [sid, hits] of Object.entries(data.s)) {
    if (hits === 0) {
      const loc = data.statementMap[sid];
      console.log('STMT #' + sid + ' line=' + loc.start.line + ':' + loc.start.column + ' ~ ' + loc.end.line + ':' + loc.end.column + ' (' + key.split('/').pop() + ')');
    }
  }

  // Branches
  for (const [bid, vals] of Object.entries(data.b)) {
    if (Array.isArray(vals) && vals.includes(0)) {
      const loc = data.branchMap[bid];
      console.log('BRANCH #' + bid + ' type=' + loc.type + ' line=' + loc.start.line + ':' + loc.start.column + ' vals=' + JSON.stringify(vals) + ' (' + key.split('/').pop() + ')');
    }
  }

  // Functions
  for (const [fid, hits] of Object.entries(data.f)) {
    if (hits === 0) {
      const loc = data.fnMap[fid];
      console.log('FUNC ' + (loc?.name || 'anon') + ' line=' + (loc?.loc?.start?.line || '?') + ' (' + key.split('/').pop() + ')');
    }
  }
}
" YOUR_FILE
```

## Phase 3: Fix Patterns by Category

### A. Dead Code

Code that is logically guaranteed to be unreachable:

- `instanceof` checks after Zod parse — schema.parse() guarantees type; `if (val instanceof NextResponse)` after parse = dead
- `typeof` guards on parsed values — `typeof val !== "number"` after number schema = dead
- `?? ""` on constrained array lookups — `DAY_NAMES[date.getDay()] ?? ""` where getDay() returns 0-6 = dead
- Unused functions — defined but never called. Remove it or if exported, add a test.

**Fix:** Remove the dead code. Not "exclude from coverage" — actually delete.

### B. Error Paths

Code paths that only run on failures:

- try/catch blocks with silent handlers — add a test that triggers the error
- ZodError catches in routes — add a test with bad input
- "Not found" branches (`if (!record) return 404`) — add a test with non-existent ID
- Auth guard clauses (`if (!userId) return 401`) — add a test without auth header

**Fix:** One targeted test per error branch.

### C. False Negatives (v8 source-map bugs)

See `references/v8-coverage-patterns.md` (Patterns 7, 8, 10). Characteristic: reported line/col doesn't match actual code.

### D. Biome Lint Conflicts in Test Files

| Issue | Error | Fix |
|-------|-------|-----|
| `delete process.env.X` | biome `noDelete` | `process.env.X = undefined` |
| `Function` type | biome `noExplicitAny` | Use explicit: `(...args: string[]) => void` |
| Unused variables | biome `noUnusedVariables` | Remove or use `vi.fn()` assignment |
| `any` casts | biome `noExplicitAny` | `as unknown as T` or biome-ignore |

### E. Flaky Test Patterns

| Pattern | Symptom | Fix |
|---------|---------|-----|
| `shuffleArray()` in component | `findByText` intermittent | Use regex: `/text|alt text/` |
| `waitFor` default timeout 1000ms | Timeout on async data | `{ timeout: 5000 }` or verify mock data matches component |
| `userEvent.setup()` unused | biome warning | Remove if only using `fireEvent` |
| `fireEvent.click` on `disabled` button | Click won't fire | Change to `aria-disabled` (see v8 Pattern 8) |
| `vi.useFakeTimers` + `userEvent` | Infinite hang | Never combine. Use `waitFor` with real timers |

## Phase 4: Iterate (Fast Loop)

1. Fix the biggest gap → run coverage → check if gap closed
2. If yes → pick next biggest gap
3. If no → drill into the file to find remaining uncovered lines
4. Repeat

Use `--exclude='tests/integration/**'` for fast iterations (slow integration tests are for the final pass only).

## Phase 5: Final Verification

```bash
npx vitest run --coverage
grep thresholds vitest.config.ts
```

## Key Rule

**Never exclude source files from coverage to inflate numbers.** Writing tests is always the answer. The only valid exclusions: type-only files, shadcn UI wrappers, test setup, and documented v8 false negatives.