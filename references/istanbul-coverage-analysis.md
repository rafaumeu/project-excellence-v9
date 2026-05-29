# Istanbul Coverage Gap Analysis Technique

## When to Use

When you need to identify EXACTLY which functions, branches, or lines are not covered by tests. The terminal coverage report truncates file paths and only shows percentages — you can't tell which specific `onError` callback or `onKeyDown` handler is uncovered.

## The Problem

`npx vitest run --coverage` shows a table with truncated paths and aggregate percentages. You see "92.6% branches" but can't tell which branch. The HTML report requires a browser. You need machine-parseable data.

## Solution: Parse `coverage/coverage-final.json`

After running `npx vitest run --coverage`, Istanbul writes `coverage/coverage-final.json` with per-function, per-branch, per-statement execution counts.

### Script 1: Find all files below 100% any metric

Save as `/tmp/cov-gaps.py` and run from project root:

```python
#!/usr/bin/env python3
import json

data = json.load(open('coverage/coverage-final.json'))
results = []
for k, v in data.items():
    if '.test.' in k or not k.endswith(('.tsx', '.ts')):
        continue
    s_map = v.get('s', {})
    b_map = v.get('b', {})
    f_map = v.get('f', {})
    total_s = len(s_map)
    covered_s = sum(1 for c in s_map.values() if c > 0)
    total_b = sum(len(branch) for branch in b_map.values())
    covered_b = sum(sum(1 for c in branch if c > 0) for branch in b_map.values())
    total_f = len(f_map)
    covered_f = sum(1 for c in f_map.values() if c > 0)
    if total_s == 0:
        continue
    sp = covered_s / total_s * 100
    bp = covered_b / total_b * 100 if total_b > 0 else 100
    fp = covered_f / total_f * 100
    if sp < 100 or bp < 100 or fp < 100:
        results.append((k.split('/')[-1], sp, bp, fp))
results.sort(key=lambda x: x[1])
for name, s, b, f in results:
    print(f'{name}  S={s:.1f}%  B={b:.1f}%  F={f:.1f}%')
```

### Script 2: Find uncovered functions by name and line

Save as `/tmp/uncov-detail.py`:

```python
#!/usr/bin/env python3
import json

data = json.load(open('coverage/coverage-final.json'))

# Replace with component names you want to inspect
for target in ['YourComponent', 'AnotherComponent']:
    for k, v in data.items():
        if target in k and k.endswith('.tsx'):
            fn_map = v.get('fnMap', {})
            f_map = v.get('f', {})
            print(f"\n=== {k.split('/')[-1]} ===")
            for fid, info in fn_map.items():
                count = f_map.get(fid, -1)
                name = info.get('name', '?')
                line = info.get('loc', {}).get('start', {}).get('line', '?')
                status = "COVERED" if count > 0 else "NOT COVERED"
                print(f"  fid={fid} line={line} name='{name}' {status}")
```

### Istanbul JSON Structure Quick Reference

```json
{
  "src/components/Foo.tsx": {
    "s": { "0": 5, "1": 3, "2": 0 },      // statements: id → execution count
    "b": { "0": [3, 0], "1": [5, 5] },     // branches: id → [true_count, false_count]
    "f": { "0": 10, "1": 0, "2": 5 },      // functions: id → execution count
    "fnMap": {                              // function metadata
      "0": { "name": "Foo", "loc": { "start": { "line": 10 } } },
      "1": { "name": "handleClick", "loc": { "start": { "line": 25 } } }
    },
    "branchMap": {                          // branch metadata
      "0": { "type": "if", "loc": { "start": { "line": 12 } } }
    },
    "statementMap": {                       // statement metadata
      "0": { "start": { "line": 10 }, "end": { "line": 10 } }
    }
  }
}
```

## Key Insights

- `f` (functions) map is the most actionable — tells you which named functions are never called
- `b` (branches) with `[N, 0]` means the false branch was never taken; `[0, N]` means true branch
- `fnMap` gives you the function NAME and line number — cross-reference with source to write targeted tests
- Count of 0 = never executed. Count > 0 = covered.

## Script 3: Find exact uncovered branches with line numbers (Node.js)

V8 Istanbul uses offset-based locations internally, but `branchMap`/`statementMap` have `start.line`. Use Node.js to parse:

```bash
node -e "
const cov = require('./coverage/coverage-final.json');
for (const [key, data] of Object.entries(cov)) {
  if (key.includes('TARGET_FILENAME')) {
    const bmap = data.branchMap;
    const b = data.b;
    const smap = data.statementMap;
    const s = data.s;
    for (const [bid, vals] of Object.entries(b)) {
      if (Array.isArray(vals) && vals.includes(0)) {
        const loc = bmap[bid];
        const start = loc.loc ? loc.loc.start : (loc.start || {});
        console.log('branch', bid, 'type=' + (loc.type || '?'),
          'L' + (start.line || '?'), 'C' + (start.column || '?'),
          'vals=' + JSON.stringify(vals));
      }
    }
    for (const [sid, count] of Object.entries(s)) {
      if (count === 0) {
        const loc = smap[sid];
        const start = loc.start || {};
        console.log('stmt', sid, 'L' + (start.line || '?'),
          'C' + (start.column || '?'), 'count=' + count);
      }
    }
  }
}
"
```

**vals interpretation:**
- `if` type: `vals=[true_count, false_count]`
- `binary-expr` type: `vals=[left_count, right_count]`
- `cond-expr` type (ternary): `vals=[true_count, false_count]`

## Common Patterns of Uncovered Code

| Pattern | Example | Fix |
|---------|---------|-----|
| Event handlers | `onError`, `onKeyDown`, `onInvalid` | `fireEvent.error(el)`, `fireEvent.keyDown(el, { key: 'Escape' })` |
| Error catch blocks | `catch { showToast('error') }` | Mock fetch to reject |
| Default/fallback branches | `spec.color \|\| '#fbbt24'` | Pass input without the optional field |
| Early returns | `if (!auth) return` | Render with null auth |
| Optional chaining | `auth?.clubeId` | Pass null/undefined |

## Unreachable Branches: Simplify Source, Don't Write Impossible Tests

When V8 reports a branch as uncovered but analysis shows the branch is **structurally unreachable**, simplify the source code instead of contorting tests. Common patterns:

### Pattern 1: `?? ""` on indexed array access
```ts
// BEFORE — branch always has value (array index always valid)
DAY_NAMES[date.getDay()] ?? ""
// AFTER — non-null assertion
DAY_NAMES[date.getDay()]!
```

### Pattern 2: `hm.index ?? 0` after successful regex match
```ts
// BEFORE — index always exists when match succeeds
const hm = str.match(/pattern/);
if (!hm) continue;
afterBlock = block.substring((hm.index ?? 0) + hm[0].length);
// AFTER — non-null assertion
afterBlock = block.substring(hm.index! + hm[0].length);
```

### Pattern 3: `answers[qId] \|\| ""` when qId comes from `Object.keys(answers)`
```ts
// BEFORE — qId is always a valid key
const questionIds = Object.keys(answers);
for (const qId of questionIds) {
  const selectedUuid = answers[qId] \|\| "";
// AFTER — non-null assertion
  const selectedUuid = answers[qId]!;
```

### Pattern 4: `typeof x === "number"` after `Number()` construction
```ts
// BEFORE — Map values built with Number(), always numbers
const xpByDate = new Map<string, number>();
const xp = typeof xpVal === "number" ? xpVal : 0;
// AFTER — direct cast (add `as number` if Map generic is `unknown`)
const xp = (xpByDate.get(dateStr) as number) ?? 0;
```

### Decision Framework
1. **Can I write a test that hits this branch?** → Yes: write the test.
2. **Is the branch structurally unreachable?** → Simplify source code.
3. **Will removing the branch break TypeScript?** → Use `!` or `as Type` cast.
4. **Not sure?** → Leave it and add a test. Don't guess.

**NEVER** remove error handling or validation logic. Only remove branches where the input domain guarantees the condition.
