# Session 2026-05-17 — Production Fixes v2.1.3

## Pitfall: New role added to system but role-based checks not updated everywhere

When a new role is added to the auth system (e.g., `lider_aventureiro`, `lider_desbravador`), ALL existing role-based access checks must be updated. Missing checks cause features to silently disappear for that role.

**Pattern:** After adding a new role:
1. `grep -rn "includes(auth.role)" src/` — find ALL role-based checks
2. `grep -rn "isLeader\|isAdmin\|role.*===" src/` — find hardcoded role arrays
3. Update every array that should include the new role
4. Test with the new role in production

**Concrete example:** `aventureiro/page.tsx` had `isLeader={['admin', 'diretoria', 'instrutor', 'conselheiro'].includes(auth.role)}` in TWO places (BibliotecaCard and RankingMDA). `lider_aventureiro` and `lider_desbravador` were missing — leaders couldn't see the leader library or ranking details.

**General rule:** Role-based features = shared arrays or utility function. Never duplicate role lists inline. If you must inline, grep for ALL instances after adding a new role.
