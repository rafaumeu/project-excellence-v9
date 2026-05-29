---
name: project-excellence
description: "Padrao de excelencia obrigatoria para TODO projeto. Spec-Driven com contrato executavel, Context Engineering, OWASP 2025 (A03 Supply Chain + A10 Exceptional Conditions), OWASP ASI Agent Security, Solo Dev Playbook, Error Budget Policy, Testing Piramide, DORA 2025, AI Quality Metrics. Regra, nao guideline."
version: 9.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [quality, ci, code-review, standards, excellence, security, supply-chain, spec-decomposition, testing, dora, sre, slo, feature-flags, mutation-testing, contract-testing, chaos-engineering, postmortem, rollback, agent-guardrails, ai-agent-governance, owasp-asi, context-engineering, solo-dev, error-budget]
    related_skills: [writing-plans, requesting-code-review, github-pr-workflow, subagent-driven-development, test-driven-development]
---

# Project Excellence v9 — Regra Absoluta

Toda feature, todo projeto, todo commit segue este pipeline. Sem excecao.

**Tripe:** SPEC (contrato executavel + context engineering) > SECURITY (OWASP 2025 + ASI Agent Security) > CI (gates automatizados)

## Workflow: MAPEAR > OBSIDIAN SYNC > ESPECIFICAR > IMPLEMENTAR > RELEASE

1. **MAPEAR**: Entender codigo existente, listar arquivos que vai tocar. CRITICAL: Para specs de DB schema ou API routes, primeiro verificar se ja existe:
   - `\\dt public.*` no DB para listar todas tabelas
   - `grep -rn "table_name" src/app/api/` para encontrar rotas CRUD existentes
   - `ls src/app/api/` para visualizar endpoints existentes
   Se o usuario forneceu state do projeto (stash count, PRs, bugs, testes), VERIFICAR com comandos reais antes de confiar — state informado frequentemente esta desatualizado (pitfall #26)
2. **OBSIDIAN SYNC**: Documentar no Obsidian (source of truth) — se nao esta no Obsidian, nao existe
3. **ESPECIFICAR**: Criar spec com testes derivados + contrato de saida (Spec Review Gate bloqueante). **Spec Validation Loop**: Antes de implementar, diff spec contra estado real do projeto (DB schema, rotas existentes, types). Se spec propoe criar algo que ja existe, spec volta para refinamento (pitfall #46).
4. **IMPLEMENTAR**: So DEPOIS de spec aprovada. TDD obrigatorio.

NUNCA pular direto pra implementacao. Mesmo que pareca simples. Especialmente quando parece simples.

### Rush Mode (production bugs, trivial fixes)

When the user says "rush rush" or points to a production bug with clear symptoms:
1. **Quick trace** — UI → API → SQL, find root cause (compressed MAPEAR)
2. **Skip Obsidian + spec** — for 1-line fixes with obvious root cause
3. **TDD still mandatory** — regression test FIRST, then fix
4. **Commit + PR + merge** — fast cycle, no ceremony

This is NOT for features or refactors. Only for: production bugs with clear symptoms, 1-3 line fixes, or obvious typos. The TDD gate is never skipped — even in rush mode, the regression test must exist.

5. **RELEASE**: Apos IMPLEMENTAR + CI verde + merge:
   - Bump version em `package.json` (seguir semver)
   - Commit: `chore: bump version to X.Y.Z`
   - Push
   - Merge PR (squash) via `gh pr merge`
   - Tag: `git tag -a vX.Y.Z <merge-commit> -m "vX.Y.Z — summary"`
   - Push tag: `git push origin vX.Y.Z --no-verify`
   - GitHub Release: `gh release create vX.Y.Z --title "vX.Y.Z — Title" --notes "changelog"`
   - Sync local: `git checkout main && git reset --hard origin/main`
   - Atualizar Obsidian com versao e changelog

   **Branch protection bypass for version bumps** (pitfall #35): Solo devs can't approve own PRs. For version bumps on main:
   1. `gh api repos/OWNER/REPO/branches/main/protection/enforce_admins --method DELETE`
   2. `git push origin main --no-verify`
   3. `gh api repos/OWNER/REPO/branches/main/protection/enforce_admins --method POST`
   Step 3 is CRITICAL — forgetting it leaves main unprotected.

### OBSIDIAN SYNC

Documentar DURANTE o trabalho, nao depois:
- MAPEAR completo > `04-Projects/[projeto]-status.md` (estado, decisoes, proximos passos, abordagens que falharam)
- ESPECIFICAR completo > atualizar note com specs criadas
- IMPLEMENTAR completo > atualizar note com resultado, commit hash, coverage
- Capturar ideia rapida > `00-Inbox/Captura Rapida.md`
- Template: `templates/project-status-template.md`

---

## 1. SPEC — Planejamento + Decomposicao

### Regra de Ouro

Todo feature comeca com spec em `.planning/<feature>/specs/TASK-*.md`. Sem spec, sem codigo. 1 spec = max 200 LOC = 1 subagente independente.

### Estrutura obrigatoria

    .planning/<feature>/
    ├── README.md              ← Index: problema, solucao, ordem, regras
    └── specs/
        ├── TASK-xxx.md        ← Spec 1
        └── TASK-yyy.md        ← Spec 2

### Template TASK-XXX (contrato executavel)

    # TASK-XXX: [nome]

    ## O QUE E
    [Uma frase]

    ## DEPENDENCIAS
    - TASK-xxx: [o que precisa pronto]

    ## IMPORTS
    [Imports exatos que esta task precisa]

    ## O QUE CRIAR
    Arquivo: `src/caminho/arquivo.ts`
    [Descricao + codigo copy-pasteable]

    ## TESTES DERIVADOS
    <!-- Derivados mecanicamente do criterio de aceite. Devem existir ANTES da implementacao. -->

    dado [pre-condicao do criterio de aceite]
    quando [acao descrita no criterio]
    entao [resultado esperado do criterio]
      e [side effect esperado, se houver]

    <!-- Casos de borda obrigatorios: -->
    - [ ] Happy path
    - [ ] Input invalido / edge case
    - [ ] Falha de dependencia externa (se aplicavel)

    ## CONTRATO DE SAIDA
    <!-- O que esta TASK exporta. Zod schema ou TypeScript type. -->
    <!-- Dependentes mockeiam EXATAMENTE este contrato, nao inventam um. -->

    export const TaskOutputSchema = z.object({
      id: z.string().uuid(),
      status: z.enum(['success', 'error']),
      data: z.unknown(),
    })
    export type TaskOutput = z.infer<typeof TaskOutputSchema>

    (N/A se task nao exporta nada)

    ## ARMADILHAS
    - [Edge cases conhecidos desta task]

    ## CRITERIO DE ACEITE
    - [ ] [max 5 items testaveis — cada item DEVE ter pelo menos 1 teste derivavel mecanicamente]
    - [ ] Nenhum termo ambiguo (evitar: "adequado", "rapido", "correto" sem metrica)

    ## VALIDACAO CI
    [Bloco CI — ver secao 3]

    ## ESTIMATIVA
    LOC: ~[100-200] | Tempo subagent: ~[5-15 min]

### Spec Review Gate (obrigatorio antes de implementar)

Uma spec NAO esta pronta para implementacao se qualquer item falhar:
- [ ] Criterio de aceite tem pelo menos um caso de teste derivavel mecanicamente
- [ ] `## TESTES DERIVADOS` preenchido com pseudocodigo
- [ ] `## CONTRATO DE SAIDA` preenchido (ou marcado explicitamente como N/A)
- [ ] Dependencias listadas em `## IMPORTS` com contrato de saida referenciado
- [ ] Nenhum termo ambiguo no criterio de aceite

Se qualquer item falhar: spec volta para refinamento. Agente NAO comeca implementacao.

### Execucao via Subagents

- Cada task = 1 subagente via `delegate_task`, max 2 paralelos
- Dependencia explicita: `DEPENDENCIAS: [TASK-xxx]`
- Subagente le spec + dependencias, implementa sem contexto das outras specs

### Agent Guardrails & Abort Criteria

**Regra de abort:** 3 falhas consecutivas na mesma etapa = PARA, documenta erro com contexto, escala pra humano. Nunca 4a variacao sem intervencao.

**Operacoes que exigem confirmacao humana (nunca autonomamente):** `git push --force` | `DROP TABLE` / `DELETE FROM` sem `WHERE` | deploy prod fora de janela | rotacao de secrets prod

**Checkpoint por fase (nao por task):** FASE [N] CONCLUIDA > resumo do feito > estado (branch, commit, testes?) > proxima fase > bloqueadores. Exigido ao completar cada fase do plano.

**Escopo:** Subagente nunca expande alem da TASK delegada. Trabalho fora do escopo = nova TASK proposta + para.

---

## 2. SECURITY — OWASP 2025 + Agent Security + Supply Chain

### OWASP Top 10 2025 (atualizado)

Novidades 2025 vs 2021:
- **A03:2025 Software Supply Chain Failures** — expandido de "Vulnerable Components" para cobrir todo o ecossistema: dependencias, build systems, distribuicao
- **A10:2025 Mishandling of Exceptional Conditions** — NOVO: error handling como categoria propria. Erros que fail open, privilege escalation via exception handling, stack traces expostos
- **SSRF** consolidado em A01 Broken Access Control
- A01 e A02 mantidos no topo — praticas fundamentais continuam criticas

### Checklist de Seguranca por Camada

**API:** Sem rotas debug em prod | auth em tudo | rate limit persistente | CORS whitelist | zod em todo endpoint | sem stack traces | **error handling resiliente (A10:2025)** — nunca fail open
**DB/RLS:** RLS em TODAS tabelas | FORCE RLS em PII | policies CRUD | indice nas policy cols | parameterized queries | anon test = vazio
**Frontend:** Zero hardcoded secrets | security headers (CSP/X-Frame/HSTS) | `poweredBy: false`
**Auth:** JWT HttpOnly+SameSite+Secure | `next` param rejeita `//` | IDOR: toda query filtra `userId`

### Agent Security (OWASP ASI Top 10, Dez 2025)

AI coding agents sao superficie de ataque. 30+ CVEs descobertos em Dez 2025 (incluindo CamoLeak CVSS 9.6). 15-25% do codigo gerado por AI contem vulnerabilidades.

**Principio:** Tratar AI agent como terceiro nao-confiavel. Least privilege, code review obrigatorio, audit logging.

**Riscos criticos (OWASP ASI):**
- **ASI01 Agent Goal Hijack** — instrucoes em README/comments redirecionam agent
- **ASI02 Tool Misuse** — agent usa ferramentas alem do escopo da TASK
- **ASI05 Unexpected Code Execution** — agent gera codigo com eval/exec
- **ASI06 Memory/Context Poisoning** — skills ou contexto corrompidos persistentemente
- **ASI09 Human-Agent Trust** — agent gera explicacoes confiantes porem incorretas

**Controles obrigatorios:**
- Code review humano de TODO codigo gerado por AI antes de merge
- Scope boundaries: agent NAO expande alem da TASK (regra absoluta #9)
- Input sanitization: separar instrucoes do agent de conteudo nao-confiavel
- Memory integrity: review periodico de skills (mensal) para staleness/poisoning
- Audit logging: registrar acoes autonomas (files modified, commands executed)

Referencia completa: `references/ai-agent-governance.md`

### Supply Chain & Dependency Security (A03:2025)

> OWASP 2025 elevou Supply Chain de A06 pra A03 — agora cobre todo o ecossistema: dependencias, build systems, distribuicao, dev workstations.

**Gates obrigatorios em CI:**

    npm audit --audit-level=high --omit=dev
    npx license-checker-rseidelsohn --failOn 'GPL-3.0;AGPL-3.0;GPL-2.0;SSPL-1.0;BSL-1.1'

**License check strategy:** Prefer `--failOn` (blocklist of copyleft licenses) over `--onlyAllow` (whitelist). With `--onlyAllow`, every new permissive license (BlueOak, CC0, Unlicense, Python-2.0, MPL-2.0) breaks the build until you add it to the allowlist. With `--failOn`, only genuinely problematic licenses (GPL, AGPL, SSPL, BSL) are blocked. All permissive licenses pass by default.

**Dependabot config minimo** (`.github/dependabot.yml`):

    version: 2
    updates:
      - package-ecosystem: "npm"
        directory: "/"
        schedule:
          interval: "weekly"
        open-pull-requests-limit: 5

**SBOM** (projetos com dados de menores ou compliance):

    npx @cyclonedx/cyclonedx-npm --output-file sbom.json

**Regra:** PRs com dependencias novas passam por `license-checker` antes do merge. GPL/AGPL requerem aprovacao explicita.

**Pentest Framework:** 3 subagents paralelos — (1) DB/RLS, (2) Auth/API, (3) Frontend/OSINT/LGPD. Relatorio: `SEC-NN: titulo | SEVERITY | arquivo:linha | descricao | fix`. Referencia: `references/pentest-framework.md`

**Dados de Menores (LGPD Art. 14):** PII de menores = criptografia em repouso + consentimento dos pais + log de acesso + retencao + direito exclusao. NAO permitir exportacao em massa. Ref DB: `references/supabase-quick-reference.md`

---

## 3. CI — Pipelines + Gates

### Gates obrigatorios em CI

| Gate | Comando | Bloqueia? |
|------|---------|-----------|
| Lint | `npm run lint` (Biome) | Sim |
| Test + Coverage | `npm run test:coverage` (Vitest, threshold realistic) | Sim |
| Build | `npm run build` | Sim |
| Security audit | `npm audit --audit-level=high --omit=dev` | Sim |
| License check | `npx license-checker-rseidelsohn --failOn 'GPL-3.0;AGPL-3.0;...'` (blocklist) | Sim |
| Dead code | `npx knip --no-exit-code` | Nao (legacy) / Sim (novos) |
| Type coverage | `npx type-coverage --at-least 85 --strict` | Sim |

**Coverage provider selection (CRITICAL):** Use **v8** provider. Istanbul INFLATES coverage by masking real gaps — it reported 100% on code where v8 showed 94% statements and 85% branches (confirmed on Quizoteca with 1456 tests switching from istanbul to v8). Istanbul instruments at AST level which SOUNDS more accurate, but in practice it over-counts coverage on async routes, error paths, and conditional branches. V8's source-map quirks (Pattern 7 in v8-coverage-patterns.md) are real but manageable — the alternative (istanbul hiding untested code) is worse for production SaaS.

**100% coverage IS achievable with v8** after addressing three categories: (1) dead code removal (unreachable branches after Zod parse, redundant type guards), (2) targeted test additions for error paths (ZodError catches, missing DB records), (3) honest exclusions for v8 false negatives (intervalRef in hooks, animated components with framer-motion). On Quizoteca: 132 files, 1472 tests, 100% ALL metrics (statements/branches/functions/lines perFile true) — v0.20.0.

**Realistic starting thresholds (before closing gaps):** statements 94%, branches 88%, functions 95%, lines 94%. Work up to 100% by analyzing `coverage-final.json` with Node.js: `node -e 'const d=require("./coverage/coverage-final.json"); for (const k of Object.keys(d)) { const s=d[k]; for (const id of Object.keys(s.s)) { if(s.s[id]===0) console.log("STMT #"+id+" line "+s.statementMap[id].start.line+" ("+k.split("/").pop()+")"); } for (const bid of Object.keys(s.b)) { for (let i=0;i<s.b[bid].length;i++) { if(s.b[bid][i]===0) console.log("BRANCH #"+bid+" path "+i+" line "+s.branchMap[bid].loc.start.line+" ("+k.split("/").pop()+")"); } } }'`

**Coverage exclusions for v8 (honest false negatives):** Exclude from coverage: animated components (framer-motion), hooks tested via `renderHook` where v8 can't trace refs through closures (e.g., `intervalRef.current` in useEffect cleanup), and metadata helpers consumed only by server components. These produce false 0% with v8 despite having thorough test suites. Add them to vitest.config.ts `coverage.exclude`. Document WHY each exclusion exists — future audits must distinguish "tested but untracked" from "untested".

**Coverage-gap workflow (iterative, existing codebase):** When bringing an existing codebase to 100% coverage, don't start from scratch — run coverage, find the biggest gaps, drill into specific uncovered lines, and write targeted tests. This is NOT TDD (tests-first) — it's coverage-gap-driven testing for existing code. The full iterative workflow with commands is in `references/coverage-gap-workflow.md`. Common patterns: dead code removal (unreachable branches after Zod parse), error path tests, Biome lint conflicts in test files (delete process.env → = undefined), and flaky test fixes.

Templates completos: `references/ci-templates.md`

### Dead Code & Type Coverage Gates

**knip:** Detecta exports, types e files nao utilizados. Config: `knip.json` com `entry` files + `ignoreIssues` para false positives (Zod contracts, shadcn UI). Use `ignoreExportsUsedInFile: { interface: true, type: true }` para types usados internamente. Nao bloqueia codigo legado existente, MAS bloqueia adicoes novas que adicionem dead code. Config keys validas knip 6.x: `entry`, `project`, `ignoreIssues`, `ignoreBinaries`, `ignoreDependencies`, `ignoreExportsUsedInFile`. NOT valid: `ignoreExports` (unrecognized key).

**type-coverage:** Garante tipagem real (nao so `noImplicitAny`). Meta: >= 85%, aumentar 5% por sprint ate 95%. Roda em < 30s, nao requer build completo.

Ambos como GitHub Actions job separado. Falha = PR nao mergea.

### CodeRabbit (`.coderabbit.yml`)

    language: "pt-BR"
    reviews:
      profile: "assertive"
      auto_review: { enabled: true, ignore_draft: true }
    tone_instructions: "Responda em portugues brasileiro. Direto e tecnico."

**CodeRabbit PR review gate (mandatory):** CodeRabbit is a GitHub App (NOT a standalone CLI — no public `coderabbit` CLI binary exists). It auto-reviews PRs when installed. Before merging, ensure CodeRabbit has reviewed the PR and address all findings (minimum: all `potential_issue` and `major`). Check review status via `gh api repos/OWNER/REPO/pulls/NUMBER/reviews` or the PR web UI. Document skipped findings with reason. Only merge when CodeRabbit clean + CI green. Related tools: `coderabbitai-mcp` (MCP server for agents), `coderabbit-cli-mcp` (MCP wrapper).

### Local Validation

    { "validate:pr": "biome check src/ tests/ && tsc --noEmit && vitest run" }

Chain linter + types + tests com `&&`. Falha em qualquer um para tudo.

**Pre-merge gate (OBRIGATORIO):** `validate:full` — inclui security audit + license check + validate:pr + SonarQube scan. Nenhum PR deve ser mergeado sem `validate:full` passar. Nenhum erro deve chegar em producao.
```
"validate:pr": "biome check src/ tests/ && tsc --noEmit && vitest run"
"validate:full": "npm run security:audit && npm run security:license && npm run validate:pr"
```

**Note:** `sonar:scan` removed from `validate:full` for TypeScript projects on ARM64 (SonarQube JS/TS plugin doesn't work on aarch64, and Community has no free tier). Biome + tsc + Vitest cover the same ground at zero cost. Add `&& npm run sonar:scan` back if running x86 with SonarQube available.

For TypeScript, use a scoped config:
```
"validate:types": "tsc --noEmit -p tsconfig.validate.json"
```
Create `tsconfig.validate.json` that excludes CLI entry points, scrapers, workers, and browser automation from type checking (these often have pre-existing TS errors from external APIs).

**Pitfall — SonarQube no pre-push hook:** SonarQube scan demora 3-5 min + precisa Docker up. NAO colocar no Husky pre-push. Usar como gate manual pre-merge. Workflow: `validate:pr` (rapido, pre-push) > `validate:full` (completo, antes de merge).

### PR Governance + Branch Protection

- Milestone + label required (via `pr-governance.yml`)
- Branch protection: required checks, linear history, conversation resolution, enforce admins
- Husky: pre-commit (lint-staged), pre-push (vitest run)
- Red Team Flow: implementa > validate:pr > coderabbit review > fix > push > CI > CR auto review > fix critico > merge (squash)

---

## 4. TESTING — Piramide Completa

### Piramide (obrigatorio para SaaS)

           /\ 
          /E2\       5% — fluxos criticos (dinheiro, auth)
         /----\
        /Integ\      15% — payment+DB, core+state
       /--------\
      /Contract \    10% — API response shape, webhook payload
     /------------\
    /Property-based\  5% — sanitizacao, rate limit, parsers
   /----------------\
  /   Unit + Mutation \  65% — logica pura, provada por mutation
 /--------------------\

### Regras por Tipo de Teste

**Unit (obrigatorio):** TDD RED>GREEN>REFACTOR. 100% coverage perFile. Mock SOMENTE deps externas. Cada branch = 1 teste.

**Integration (obrigatorio para billing):** DB real ou mock que simula Postgres. Testar transacoes: sucesso, falha, rollback. Nunca `vi.mock` pra DB.

**E2E (obrigatorio para fluxos de dinheiro):** Playwright. Cenarios: login>core flow>completion, pricing>checkout>webhook. DB limpo entre testes.

**Mutation (obrigatorio apos unit):** Stryker. Score >= 80%, money-handling >= 90%. Meta: 100% com equivalent mutant exclusions documentadas. Mutantes sobreviventes = teste falso.

**Contract (obrigatorio para APIs):** Zod schema compartilhado. API valida saida, frontend valida entrada. Contratos em `src/lib/contracts/`.

**Property-based (input externo):** fast-check. Sanitizacao, rate limit, parsers. Gerar inputs aleatorios.

**Regression (obrigatorio):** Cada bug corrigido = teste. Tag `@regression`. Nunca deletar.

### Pipeline de Testes por PR

    Unit (vitest --coverage, 100% threshold)
      > Integration
        > Mutation (>= 80%)
          > E2E
            > Security audit

### PRE-IMPLEMENTACAO CHECKLIST (bloqueante)

    [ ] 1. Li a spec completa
    [ ] 2. Li o codigo existente que vou testar/modificar
    [ ] 3. Mapei TODOS os branches (if/else/switch/ternary)
    [ ] 4. Identifiquei mocks necessarios (DB, API, payment)
    [ ] 5. Escrevi teste PRIMEIRO (RED) — vi FALHAR pelo motivo certo
    [ ] 6. Implementei codigo minimo (GREEN)
    [ ] 7. Refactorei se necessario
    [ ] 8. Todos testes existentes passam, coverage = 100%, validate:pr passa

---

## 5. RELIABILITY — SLOs + Error Budget + PRR + Rollback + Postmortem

### SLOs Template

    PAYMENT API:     99.9% availability → 43 min/mes
    CORE FEATURE:    99.5% availability → 3.6h/mes
    AUTH:            99.9% availability → 43 min/mes
    LATENCY:         p95 < 2s, p99 < 5s

Error budget esgotado em payment = PARAR features, focar reliabilidade.

### Error Budget Policy (Google SRE Workbook, adaptado solo dev)

**Janela:** 4 semanas (rolling window).

| Evento | Threshold | Acao |
|--------|-----------|------|
| Error budget esgotado | 100% consumido | PARAR novas features, focar reliability ate budget restaurado |
| Incidente unico | > 20% do budget | Postmortem obrigatorio em 24h |
| Classe de outage | > 20% do budget em 1 trimestre | P0 no proximo planejamento |
| SLO consistentemente excedido | 3 janelas seguidas | Review SLO (muito apertado?) |

**Consequencia pratica:** Se error budget esgotado, TODO trabalho de feature para. Apenas reliability, bug fixes e security patches ate o budget ser restaurado. Sem excecao.

**Review mensal:** SLOs revisados mensalmente. Ajustar se necessario.

### PRR Checklist (pre-launch, obrigatorio para features de dinheiro)

**RELIABILIDADE:** SLO definido, plano payment/DB/Vercel fallback, feature flags, rollback documentado, backup verificado
**SEGURANCA:** Pentest 8 categorias, RLS todas tabelas, zero rotas debug, security headers, npm audit limpo, supply chain check
**OBSERVABILIDADE:** Logger estruturado, health check, error handler, deploy log
**TESTES:** Coverage 100%, mutation >= 90%, E2E fluxos dinheiro, integration webhook+DB, regression suite
**DADOS:** LGPD compliance, consentimento, direito exclusao, criptografia menores (se aplicavel)

PRR salvo em Obsidian: `04-Projects/PRR-[feature].md`

### Rollback Playbook

**Caminho padrao (reversivel):**
1. Detectar via alertas SLO ou relatorio de incidente
2. `vercel rollback [deployment-url]` ou `git revert HEAD && git push`
3. Verificar health check e SLOs em 5 minutos
4. Abrir postmortem draft imediatamente

**Padrao expand-contract para migrations zero-downtime:** (1) add coluna nullable — deploy, (2) migrar dados em background, (3) app usa nova coluna — deploy, (4) drop coluna antiga — deploy separado. Nunca rename/drop em unica migration com trafego real.

**Decisao rollback vs forward fix:**

| Situacao | Decisao |
|---|---|
| Bug em codigo, dados intactos | Rollback |
| Schema change, dados migrados | Forward fix |
| Feature flag disponivel | Desativar flag, investigar |
| < 1% usuarios afetados | Forward fix com hotfix prioritario |
| > 10% usuarios afetados | Rollback imediato |

**Quando rollback e impossivel:** Breaking schema ja consumido > migration forward. Dados sensiveis expostos > incident response. Dependencia externa notificada > coordenar forward fix.

### Incident Response + Postmortem

    DETECTAR > ISOLAR (flag OFF ou redeploy) > DIAGNOSTICAR > CORRIGIR > VERIFICAR > POSTMORTEM (24h)

**Postmortem blameless** (Obsidian `04-Projects/Postmortems/`):
Data, Severidade, Duracao, Impacto, Timeline, Causa Raiz (5 Whys), Acoes Preventivas, Licao.

### Chaos Scenarios (obrigatorios para SaaS com dinheiro)

Cada cenario = teste integration ou E2E:
- **Payment:** webhook atrasado, duplicado, failed, provider 500, browser fecha no checkout
- **Database:** cold start 3s, query lenta 5s, pool esgota, deadlock, migration mid-request
- **Auth:** token expira mid-action, sessoes concorrentes, logout + acao simultanea
- **Concorrencia:** 2 checkouts mesmo usuario, core feature + state update paralelo
- **Frontend:** JS crash em pagamento, network offline no submit, browser back mid-fluxo

---

## 6. OPS — DORA 2025 + Feature Flags + Observabilidade

### DORA Metrics (2025 Update)

> DORA 2025 renomeado para "State of AI-assisted Software Development". IA e amplificador — reflete cultura existente. Falta metricas de qualidade do output AI.

| Metrica | Meta | Como medir |
|---------|------|-----------|
| Lead Time | < 1h | git log timestamp vs deploy timestamp |
| Deploy Frequency | >= 2/sem | git log --oneline main count |
| Change Failure Rate | < 15% | postmortems/ count |
| MTTR | < 30min | postmortem inicio > fix |

**AI Quality Metrics (DORA 2025):**

| Metrica AI | O que Mede | Meta |
|------------|-----------|------|
| AI Pass Rate | % codigo AI que passa CI sem alteracao | >= 70% |
| Regression Rate AI | % bugs causados por codigo AI | < 20% |
| Spec Accuracy | % specs que match implementacao sem rework | >= 80% |

Deploy log (`scripts/log-deploy.sh`), postmortem folder, review mensal.
Referencia: `references/sre-research.md`, `references/ai-agent-governance.md`

### Feature Flags + Lifecycle Management

    const flags = {
      PAYMENT_ENABLED: process.env.FEATURE_PAYMENT === 'true',
      FEATURE_Y_V2: process.env.FEATURE_Y_V2 !== 'false', // default ON
    } as const;
    export function isFeatureEnabled(flag: keyof typeof flags): boolean {
      return flags[flag];
    }

- Toda feature de dinheiro = feature flag (obrigatoria)
- Default OFF pra novas, liga apos validar
- Kill switch: `FEATURE_X=false` + redeploy = desligada em < 2 min
- Testar AMBOS caminhos (ON e OFF)

**Middleware integration pattern (Next.js):**

Feature flags that aren't consumed in production code are useless as kill switches. Integrate in `middleware.ts`:

1. Create `getFlagForPath(pathname: string): FeatureFlag | null` mapping URL path prefixes to their flag
2. In middleware, run flag check BEFORE auth/CSRF/rate-limit
3. Disabled API routes return 404; disabled pages redirect to dashboard
4. Paths without a flag mapping (e.g. `/api/health`, `/api/auth`, `/dashboard`) pass through ungated

```ts
// feature-flags.ts
export function getFlagForPath(pathname: string): FeatureFlag | null {
  if (pathname.startsWith("/api/billing") || pathname.startsWith("/billing") || pathname.startsWith("/pricing")) return "PAYMENT_ENABLED";
  if (pathname.startsWith("/api/survival") || pathname.startsWith("/survival")) return "SURVIVAL_MODE_ENABLED";
  // ... other mappings
  return null;
}

// middleware.ts — add BEFORE api/page route handling
const flag = getFlagForPath(request.nextUrl.pathname);
if (flag && !isFeatureEnabled(flag)) {
  if (request.nextUrl.pathname.startsWith("/api/")) {
    return NextResponse.json({ success: false, message: "..." }, { status: 404 });
  }
  return NextResponse.redirect(new URL("/dashboard", request.url));
}
```

Tests: 1 test per path mapping, 1 test for API block (404), 1 test for page redirect, 1 test for ungated passthrough.

**Lifecycle obrigatoria (novo v9):**

    CREATE > ENABLE > VALIDATE (30 dias) > HARDCODE > REMOVE FLAG

1. **CREATE:** Flag criada, default OFF, CI testa ambos caminhos
2. **ENABLE:** Flag ligada em staging, depois prod. Monitorar SLOs.
3. **VALIDATE:** Feature estavel por 30 dias sem incidente. Se incidente = voltar pra ENABLE.
4. **HARDCODE:** Remover condicional, codigo do caminho ON vira o unico caminho
5. **REMOVE FLAG:** Deletar flag do codigo e env vars

**Anti-patterns:** Flags orfãs (nunca removidas), flags como config permanente, flags com escopo amplo demais. Review mensal — se > 30 dias estável, remover.

**CI Check:** `grep -r "process.env.FEATURE_" src/ | grep -v ".test."` — alertar se flag ativa ha > 45 dias.

### Observabilidade

**3 Pilares:** Metrics (numeros), Logs (eventos com contexto), Traces (caminho do request).

**Stack free (solo dev):** Sentry (errors) + Pino (logs) + Vercel Analytics (latency) + Uptime Kuma (uptime)
**Stack free (small team):** Grafana Cloud + Sentry + BetterStack + OpenTelemetry

**Logging minimo em toda API route:** `logger.info({ event: 'api_request', method, path, userId })` antes, `logger.info({ event: 'api_response', status, durationMs })` depois. Erro: `logger.error({ event: 'api_error', path, error: String(error) })`. NUNCA stack traces em producao.

---

## 7. AI AGENT GOVERNANCE — Seguranca para Coding Agents

> "AI agents should be treated as untrusted third parties with the same security controls applied to external contractors." — OWASP ASI Top 10

### Context Engineering (Thoughtworks 2025)

**Definicao:** Prompt engineering otimiza human-LLM interaction. Context engineering otimiza agent-LLM interaction — a arte de construir o contexto certo pra que o agent gere codigo correto.

**Elementos em uso (SKILL.md ja e context engineering):**
- SKILL.md = system prompt estruturado por dominio
- references/ = conhecimento profundo acessivel on-demand
- pitfalls-all.md = experiencia negativa codificada
- Hindsight = memoria de longo prazo
- MCP servers = documentacao em tempo real

**Otimizacoes:** Token budget (carregar apenas skills relevantes), Info hierarquica (SKILL.md → references → codigo fonte), Spec como compressao de contexto.

### Agent Guardrails (atualizado v9)

**Regra de abort:** 3 falhas consecutivas = PARA, documenta, escala.
**Confirmacao humana:** `git push --force` | `DROP TABLE` sem WHERE | deploy prod fora de janela | rotacao de secrets.
**Escopo:** Subagente nunca expande alem da TASK.
**Memory Integrity (novo v9):** Review periodico de skills (mensal). Skills desatualizadas = ASI06 (Memory Poisoning).

### Spec Validation Loop (novo v9)

Antes de implementar QUALQUER spec, validar contra estado real (DB schema, rotas, types). Se propoe criar algo que ja existe, spec volta pra refinamento (pitfall #46).

### Code Review for AI-Generated Code

15-25% do codigo gerado por AI contem vulnerabilidades. Review obrigatorio: nenhum hardcoded secret, nenhum eval/exec, imports corretos, error messages genericos, RLS em tabelas novas, auth em novas routes, testes de edge cases.

Referencia: `references/ai-agent-governance.md`

---

## 8. SOLO DEV PLAYBOOK — Praticas Adaptadas

Dev solo + AI agent = time de 2. Adaptacoes sem sacrificar qualidade.

### Self-Review Checklist (antes de todo commit)

**Funcionalidade:** Spec validada | Criterios testados | Edge cases cobertos | Erro user-friendly
**Qualidade:** validate:pr passou | Coverage 100% | Nenhum hardcoded secret | Imports corretos
**Seguranca:** RLS em PII | Auth em API routes | Zod em input | Error messages genericos
**Manutenibilidade:** Nomes descritivos | Dead code removido | Types atualizados

### Cognitive Load Management

1. **Batch specs, implement in parallel, review serial**
2. **1 feature por vez** — Feature A completa antes de B
3. **Context recovery** — PC trava? session_search + hindsight_recall + Obsidian
4. **Capture, don't switch** — Ideia? Append em Captura Rapida

### Anti-Patterns Solo Dev

YOLO Coding | Trust All AI Output | Context Hoarding | Feature Interleaving | Skip Testing | Deploy on Friday | Silo Knowledge | Premature Abstraction | Scope Creep via Agent | Orphaned Feature Flags.

Referencia: `references/solo-dev-playbook.md`

---

## 9. PWA — Progressive Web App Checklist

### Minimum Viable PWA (no dependencies)

**Three required files:**
1. `public/manifest.json` — name, short_name, start_url, display: standalone, theme_color, icons (192+512)
2. `public/sw.js` — vanilla Service Worker (no serwist/next-pwa needed for simple caching)
3. Registration in providers.tsx or layout client component

**Service Worker pattern (stale-while-revalidate for static, network-first for API):**
```js
// public/sw.js — vanilla JS, no TypeScript
var CACHE_NAME = "app-v1";
self.addEventListener("install", (e) => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(["/", "/manifest.json"])).then(() => self.skipWaiting()));
});
self.addEventListener("activate", (e) => {
  e.waitUntil(caches.keys().then(keys => Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))).then(() => self.clients.claim()));
});
self.addEventListener("fetch", (e) => {
  if (e.request.method !== "GET") return;
  var url = new URL(e.request.url);
  if (url.origin !== self.location.origin) return;
  if (url.pathname.startsWith("/api/")) {
    e.respondWith(fetch(e.request).catch(() => new Response(JSON.stringify({error:"Offline"}), {status:503, headers:{"Content-Type":"application/json"}})));
    return;
  }
  e.respondWith(caches.match(e.request).then(cached => {
    var fetched = fetch(e.request).then(r => { if(r.ok){var cl=r.clone();caches.open(CACHE_NAME).then(c=>c.put(e.request,cl))} return r }).catch(() => cached || new Response("Offline",{status:503}));
    return cached || fetched;
  }));
});
```

**Registration (non-blocking, graceful degradation):**
```tsx
// In providers.tsx or root layout client component
useEffect(() => {
  if ("serviceWorker" in navigator) {
    navigator.serviceWorker.register("/sw.js").catch(() => {});
  }
}, []);
```

**Critical:** SW file in `public/` must be vanilla JS — no TypeScript, no `declare`, no ESM imports. Node.js syntax check runs on public files.

**global-error.tsx (Next.js root error boundary):** Required for PWA resilience. Must render its own `<html>` and `<body>` tags (unlike `error.tsx` which inherits layout). Captures errors that crash the root layout itself.

**Lighthouse PWA score:** manifest.json + SW + icons + HTTPS = installable. Lighthouse checks these 4 requirements.

---

## 10. SETUP — Projeto Novo (referencia rapida)

`git init` > `.gitattributes` (`* text=auto eol=lf`) > `package.json` (lint/test/build/validate:pr) > Biome+Vitest(100%) > Husky > ci.yml+pr-governance+dependabot+coderabbit > `.planning/specs/` > branch protection+CODEOWNERS > security baseline (RLS+PII+honeypot+anon test) > observabilidade > pentest > LGPD (se dados pessoais). Templates: `references/ci-templates.md`

---

## REGRAS ABSOLUTAS

1. Sem spec, sem codigo. MAPEIA > SPEC > VALIDA > IMPLEMENTA (sempre).
2. CI vermelho = PR nao mergea. Nunca.
3. `validate:pr` antes de todo push. Sem excecao.
4. Nunca push direto pra main. Branch + PR sempre.
5. Todo banco tem RLS. PII = FORCE RLS.
6. Specs < 200 LOC. 1 spec = 1 subagente.
7. Pentest antes de deploy. 8 categorias.
8. DELETE debug/admin routes antes de deploy.
9. Nunca hardcoded senhas/keys.
10. Rate limiting. In-memory como primeiro passo, Upstash para producao real.
11. JWT HttpOnly + SameSite + Secure.
12. Logging estruturado em toda API route.
13. Nunca expor stack traces em producao.
14. Testing piramide obrigatoria. Unit 65%, Integration 15%, E2E 5%, Contract 10%, Property 5%. Mutation >= 80%, meta 100% com equivalent exclusions documentadas.
15. PRE-IMPLEMENTACAO checklist e bloqueante. TDD e mecanismo, nao guideline.
16. Cada bug corrigido = teste de regressao. Nunca deletar.
17. LGPD compliance obrigatorio para todo SaaS.
18. Spec Review Gate bloqueante. Sem testes derivados, sem implementacao.
19. Agent guardrails: 3 falhas consecutivas = abort e escala.
20. Rollback playbook documentado. Migrations seguem expand-contract.
21. Supply chain: license-checker em todo PR. GPL/AGPL requer aprovacao.
22. **Agent Security (OWASP ASI):** AI agent = terceiro nao-confiavel. Code review obrigatorio de todo codigo gerado. Memory integrity review mensal.
23. **Error Budget Policy:** Budget esgotado = PARAR features, focar reliability. Sem excecao.
24. **Feature Flag Lifecycle:** CREATE > ENABLE > VALIDATE (30d) > HARDCODE > REMOVE.
25. **Spec Validation Loop:** Diff spec contra estado real antes de implementar. Duplicatas = spec volta.
### 26. **`validate:full` pre-merge gate:** `validate:full` e OBRIGATORIO antes de merge. Para TypeScript, o combo Biome (lint+format) + `tsc --noEmit` (type check) + Vitest (tests+coverage) substitui SonarQube — covers lint, type safety, dead code, and test coverage sem 1.5GB+ RAM overhead. SonarQube Community não tem free tier (apenas trial 14 dias) e o plugin JS/TS NÃO funciona em ARM64. Se o projeto tem SonarQube configurado e funcionando (x86), usar `sonar:scan` no `validate:full`. Caso contrário, o trio Biome+tsc+Vitest e suficiente.

---

## PITFALLS — Armadilhas Conhecidas

**Pitfalls completos (78 itens): `references/pitfalls-all.md`. Top 25 mais criticos abaixo. Para AI Agent Security risks (OWASP ASI), ver `references/ai-agent-governance.md`. Para solo dev anti-patterns, ver `references/solo-dev-playbook.md`.**

### 1. RLS circular dependencies
Tabela A referencia B na policy e B referencia A = erro 42P17. Fix: SECURITY DEFINER com `public.` qualified + `SET search_path = ''`.

### 2. REVOKE FROM PUBLIC (nao so anon/authenticated)
PUBLIC e superset. Sem `REVOKE ALL ON FUNCTION ... FROM PUBLIC`, anon executa via `/rest/v1/rpc/`.

### 3. Next.js API route auth — middleware > per-file
Middleware injeta `x-user-id`. **CRITICAL:** Se o codigo le `x-user-id` mas middleware NAO seta, qualquer cliente bypassa auth enviando header manualmente. Descoberto como VULN-HIGH em pentest real.

### 4. CSRF `origin.includes(host)` e bypassavel
`origin.includes('example.com')` aceita `https://evil-example.com`. SEMPRE `new URL(origin).host === host`.

### 5. IDOR em rotas com entity ID
Toda query DEVE filtrar por `userId` autenticado.

### 6. Vercel serverless rate limiting
In-memory reseta a cada cold start. Aceitavel como primeiro passo. Producao = Upstash/Vercel KV.

### 7. API response shape mismatch
Client extrai `json.data` mas API retorna `{ questions, completed }`. SEMPRE validar que campo extraido existe no response real.

### 8. Coverage 100% e OBRIGATORIO para SaaS
Projetos com dinheiro real exigem 100% em todas metricas. Nao e aspiracional, e regra. **Achievable with v8** after: (a) dead code removal (unreachable branches after Zod parse, redundant type guards defended by schema), (b) targeted tests for error paths, (c) honest exclusions for v8 false negatives documented with reason. Prefer honest 100% with documented exclusions over fake 100% with istanbul.

**ANTI-PATTERN: Excluding source files from coverage to inflate the percentage.** This is NOT acceptable. Adding source files to `coverage.exclude` to boost numbers without writing real tests is dishonest and was explicitly rejected by the user. The only valid exclusions are: type-only files, test setup, shadcn UI wrappers, and v8 false negatives — each with a documented reason. When coverage drops, the answer is ALWAYS "write more tests", never "exclude more files".

**Coverage boost strategy (order matters):**
1. **Remove dead code FIRST** — Before writing any new tests, scan for dead code: methods never called (`grep -rn "methodName" src/ | grep -v test | grep -v "private.*methodName"` → only the declaration = dead), modules never imported (`grep -rn "import.*moduleName" src/ | grep -v test` → 0 results = dead), type-only exports. Remove them. This is honest coverage improvement — writing tests for dead code is dishonest.
2. **Exclude type-only files** — Files that only export TypeScript types/interfaces (0% stmts in v8) should be excluded in `coverage.exclude`. They have no executable code.
3. **Then write targeted tests** — Only for uncovered branches on LIVE code paths.
4. **Document honest exclusions** — Every `coverage.exclude` entry needs a comment explaining WHY.

### 9. SET search_path = '' exige public. prefix
Function com `SET search_path = ''` nao encontra tabelas sem `public.` qualificado.

### 10. Open redirect em auth callback
Parametro `next` deve rejeitar `//` (protocol-relative).

### 11. Husky v10+ format change
`#!/usr/bin/env sh` e `. "$(dirname -- "$0")/_/husky.sh"` SAO DEPRECATED. Novo formato: comando direto, sem shebang, sem source.

### 12. validate:pr fragile pipefail + grep
`set -euo pipefail` + `grep -q "passed"` pode falhar silenciosamente. Capturar output em variavel primeiro.

### 13. Subagents create type errors in tests
Assinaturas erradas passam em runtime mas falham em `tsc --noEmit`. SEMPRE rodar tsc apos subagent work.

**Fix arquitetural:** Separar `tsconfig.app.json` (build, exclui testes) de `tsconfig.test.json` (validação, inclui tudo). O `tsc -b` usa `tsconfig.app.json` — se ele inclui test files, erros de tipo em testes gerados por subagents quebram o build de produção. Config:
```json
// tsconfig.app.json
{ "extends": "./tsconfig.json", "include": ["src"], "exclude": ["src/**/*.test.ts", "src/**/*.test.tsx"] }

// tsconfig.test.json
{ "extends": "./tsconfig.app.json", "include": ["src"], "exclude": [] }
```
Build usa `tsc -b` (app config). Validação de testes usa `tsc --noEmit -p tsconfig.test.json` separadamente.

### 14. NUNCA ignorar testes falhando como "pre-existentes"
CI vermelho e CI vermelho. Regra #2 sem excecao. "Tolerancia zero" — se um teste falha, corre ou remove. Opcoes em ordem:
1. **Corrigir o teste** — se o teste tem um bug real (timezone mismatch, mock desatualizado, assertion errada)
2. **Corrigir o codigo** — se o teste esta certo e o codigo quebrou
3. **Corrigir o infrastructure** — se o teste depende de Docker/DB que existe mas .env.test aponta pro DB errado (ex: `tesouros_test` vs `tesouros_jau_test`). Corrigir env e re-incluir no vitest config
4. **Excluir do vitest config** — ULTIMO RECURSO. Somente se o teste e genuinamente codigo morto. Adicionar ao `exclude` com comentario explicando PORQUE
5. NUNCA simplesmente ignorar e seguir em frente

Quando disser "pre-existente", obrigatoriamente corrigir ou excluir. Nao existe "vou deixar pra depois". Rodar `NODE_ENV=test npx vitest run` pra garantir que .env.test e carregado pra integration tests.

### 15. Biome large-batch lint fix
`npx biome check --write --unsafe` auto-fixa ~80%. CRITICAL: re-rodar testes apos — auto-fixes mudam runtime behavior.

### 15a. Biome 1.9.4 overrides: `include` NOT `includes`
Key correta em `biome.json` overrides e `include` (singular). `includes` causa `Found an unknown key` e Biome recusa-se a rodar. Ja quebrou Husky pre-commit em producao.

### 15b. Biome `biome-ignore` placement: line BEFORE, not inline trailing
`biome-ignore` no final da linha (inline) e SILENTLY IGNORED pelo Biome. Precisa ficar na LINHA ANTERIOR a declaracao:
```ts
// biome-ignore lint/style/useConst: reason  ← CORRECT
let x = 1;
```
```ts
let x = 1; // biome-ignore lint/style/useConst: reason  ← WRONG (silently ignored)
```

### 15c. Biome CSS parse errors with Tailwind v4 — restrict files.include
Biome 2.x doesn't natively support `@custom-variant`, `@theme`, and `@apply` directives from Tailwind v4 CSS. These cause parse errors. **Fix:** Add `files.include` to `biome.json` scoping to TS/TSX only:
```json
{
  "files": {
    "include": ["src/**/*.ts", "src/**/*.tsx"]
  }
}
```
This skips all CSS files entirely. After adding, verify with `npx biome check src/` — only TS/TSX files should be checked. The `includes` key may be auto-converted by `biome migrate`. The glob format differs between versions: `"**/src/**/*.ts"` (Vite-style) vs `"src/**/*.ts"` (relative) — let `biome migrate` rewrite it correctly.

### 15d. Vitest `globals: true` + TSC — must reference vitest types
Setting `globals: true` in `vitest.config.ts` makes `describe`, `it`, `expect`, `beforeEach`, `vi` available without imports at runtime, but TypeScript still complains about undeclared globals. **Fix:** Create `src/test/globals.d.ts`:
```ts
/// <reference types="vitest/globals" />
```
This file must be included in `tsconfig.json`'s `include` (the default `**/*.ts` catches it). Do NOT put this in `skipLibCheck`-gated configs if you want type safety on test globals.

### 15e. Biome `--unsafe` DELETES "unused" private methods
`noUnusedPrivateClassMembers` + `--unsafe` will DELETE private methods only accessed via `(obj as any).method()` in tests. Biome sees them as unused because `as any` bypasses static analysis. Even `// biome-ignore` comments may not protect them with `--unsafe`. **NEVER run `biome check --write --unsafe` on source files in projects using `as any` to access private members.** Safe pattern: (a) `biome check --write --unsafe` only on test files, (b) `git checkout -- src/` after to restore any source changes, (c) run full test suite, (d) only keep changes that pass tests.

### 16. Next.js `rel` attribute appends `noreferrer`
Tests com `expect(el).toHaveAttribute('rel', 'noopener')` FALHAM. Usar `.toContain('noopener')`.

### 17. Zod 4 `z.record()` requires 2 args
`z.record(z.unknown())` falha em Zod 4. Fix: `z.record(z.string(), z.unknown())`.

### 18. Biome `noNonNullAssertion` auto-fix breaks hook deps
`auth!.clubeId` > `auth?.clubeId` em hook deps causa warning. Fix: `auth` inteiro nos deps, `auth?.clubeId` no callback.

### 19. Module-level env constants + test isolation
`const X = process.env.Y.split(',')` no top-level NAO e recalculado. Fix: lazy init `function getX() { return process.env.Y?.split(',') ?? fallback }`.

### 20. Write tests AFTER reading source
Assumptions sobre return types causam 2-3 rewrite cycles. Pattern: ler source > identificar types/errors/defaults > depois escrever tests.

### 21. Discriminated union return needs ok-check
`ParseResult<T>` requer `if (result.ok)` antes de `.data`. Acesso direto = undefined.

### 22. Push to protected branch (solo dev cannot approve own PR)
**For version bumps and solo-dev-init workarounds:** 3-step API dance:
1. `gh api repos/OWNER/REPO/branches/main/protection/enforce_admins --method DELETE`
2. `git push origin main --no-verify` (or `--force-with-lease` if force-push)
3. `gh api repos/OWNER/REPO/branches/main/protection/enforce_admins --method POST`

Step 3 is CRITICAL — forgetting it leaves main unprotected. Use `git push --no-verify` to bypass Husky pre-push (optional) but note that CI will still run remotely.

### 23. API route tests must assert body, not just status
Stryker `ObjectLiteral`/`StringLiteral` mutants sobrevivem se so asserta status. ALWAYS: `expect(data.error).toBe("Unauthorized")`.

### 24. Pentest fix leaves tests validating INSECURE behavior
Apos fix de seguranca, buscar e remover/atualizar tests que validam o comportamento inseguro antigo.

### 25. Subagent pentest reads WRONG project
Incluir `pwd && head -5 package.json` como primeiro comando do subagent. Cross-check que arquivos reportados existem no target.

### 26. Husky 9.x — core.hooksPath breaks after removing _/ directory
When removing the deprecated `_/.husky/_/` directory from Husky 9.x projects, `git config core.hooksPath` may still point to the old path (e.g. `--version/_`). After removing `_/,` always verify with `git config core.hooksPath` and reset to `.husky` if needed: `git config core.hooksPath .husky`. Also ensure hook files have executable permission: `chmod +x .husky/pre-commit .husky/pre-push`.

### 27. Git commit messages with `--` break pre-commit hooks
Em-dash (`—`) or double-dash (`--`) in commit message body can cause `sh: 0: Illegal option --` when passed through husky hooks. Fix: use a file for the commit message (`git commit -F /tmp/msg.txt`) or avoid `--` in the message body.

### 28. Vitest `--pool=threads` OOM on large test suites
`vitest run --pool=threads` can cause worker OOM crashes (unhandled errors, worker dies) on projects with 700+ tests or heavy file I/O tests. Removing `--pool=threads` (runs single-threaded) eliminates the crashes at the cost of speed. For `validate:pr`, use `vitest run` without pool flag. Keep `--pool=threads` only for interactive `test:watch`.

### 29. Biome auto-fix needs two passes
`npx biome check --write --unsafe` fixes lint issues but can create formatting inconsistencies. After the unsafe pass, run `npx biome check --write src/ tests/` (without `--unsafe`) to fix formatting. Both passes are needed for a clean check.

### 30. oklch light colors fail WCAG AA with white text
`oklch(0.72 0.22 142)` (L=0.72) with white foreground = contrast ~2.3:1 (needs 4.5:1). Fix: darken L channel only. `oklch(0.52 0.20 142)` gives ~4.5:1. Rule of thumb: L <= 0.55 for AA, L <= 0.40 for AAA. Check BOTH light and dark mode themes. Verify with Lighthouse `color-contrast` audit.

### 31. Knip false positives on API contracts + UI components + server component imports
Knip can't trace Zod schema usage across route boundaries (`z.infer<typeof Schema>` doesn't count as "using" the exported type) or shadcn UI component usage in pages. Fix: use `ignoreIssues` in `knip.json` with file patterns mapped to issue types (NOT `ignoreExports` — that key doesn't exist in knip 6.x). Also useful: `ignoreExportsUsedInFile: { interface: true, type: true }` for types used only within their own file, and `ignoreBinaries`/`ignoreDependencies` for false-positive deps. NOT a fix: removing the exports (they ARE used, just not statically traceable).

**Important:** Knip reports three categories: `exports`, `types`, AND `files`. Files imported by Next.js server components via `generateMetadata` (e.g., `src/lib/metadata.ts`) show as "Unused files" because KNIP can't trace the import through layout entry points. The fix is adding `"files"` to the ignoreIssues array: `"src/lib/metadata.ts": ["exports", "files"]`. If you only add `"exports"`, KNIP still reports "Unused files (1)".

**Working knip 6.x config example:**
```json
{
  "$schema": "https://unpkg.com/knip@6/schema.json",
  "entry": ["src/app/**/page.tsx", "src/app/**/layout.tsx", "src/app/**/route.ts"],
  "project": ["src/**/*.ts", "src/**/*.tsx"],
  "ignoreExportsUsedInFile": { "interface": true, "type": true },
  "ignoreIssues": {
    "src/lib/contracts/**": ["exports", "types"],
    "src/components/ui/**": ["exports"]
  },
  "ignoreBinaries": ["knip"],
  "ignoreDependencies": ["shadcn", "tw-animate-css"]
}
```
Key: `ignoreIssues` (file pattern → issue types), NOT `ignoreExports` (rejected as unrecognized key in knip 6.x).

### 32. Vitest fileParallelism causes race conditions with shared filesystem
When tests write to a shared directory (e.g., `data/cv-tailored/`, `data/pdfs/`), running test files in parallel causes race conditions. Symptom: tests pass in isolation (300ms) but timeout (5000ms+) or fail with file corruption when run as a suite. Fix: set `fileParallelism: false` in `vitest.config.ts`. This runs test files sequentially while still running individual tests within each file in parallel. Different from pitfall #28 (pool=threads OOM) — this is about file-level parallelism, not thread pools.

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    fileParallelism: false, // when tests share filesystem state
  },
});
```

### 33. Biome organizeImports strips vitest globals
`npx biome check --write --unsafe` may remove `beforeEach`, `afterAll` from vitest imports. Symptom: `tsc --noEmit` shows `Cannot find name 'beforeEach'`. Fix: always re-run `tsc --noEmit` after Biome auto-fix and restore stripped imports.

### 34. Squash merge CRLF + Husky hook corruption
GitHub squash merge can introduce CRLF across all files (pitfall #78). Husky hooks with CRLF crash `npx lint-staged` with cryptic "undefined@lint-staged" (pitfall #79). **Prevention:** `.gitattributes` with `* text=auto eol=lf` added at project init. After squash merge, run `npx biome check --write src/ tests/` immediately. Write hooks with `printf` not editors.

### 35. Next.js client components can't export metadata
Pages with `"use client"` at the top cannot export `metadata` or `generateMetadata`. The build silently ignores it. Fix: create a `layout.tsx` in the same folder as the client page — layouts are always server components and can export metadata. Pattern:
```tsx
// src/app/quiz/layout.tsx (server component — no "use client")
import type { Metadata } from "next";
export const metadata: Metadata = { title: "Quiz", description: "..." };
export default function QuizLayout({ children }: { children: React.ReactNode }) { return children; }

// src/app/quiz/page.tsx (client component)
"use client";
export default function QuizPage() { /* ... */ }
```
For server component pages, add `export const metadata` directly. Audit with: `grep -L "export.*metadata" $(find src/app -name "page.tsx") | while read f; do head -1 "$f"; echo "$f"; done`

### 36. `replace_all=true` with patch tool can mangle files
When using `replace_all=true`, the `old_string` pattern may match in unintended locations. Example: replacing `],\n    ]);` matched inside array literals containing similar endings, corrupting mock data arrays into ZodError issue objects. The tool replaced 11 occurrences across the file when only 1 was intended. **Prevention:** Use `replace_all` ONLY when the pattern is truly unique or when you INTEND all occurrences. For single replacements, always verify the pattern is unique first. If the file gets corrupted: `git checkout -- <file>` and retry with a more specific pattern. Do NOT attempt manual cleanup of a mangled file — restore and redo.

### 37. Zod v4 `$ZodIssueInvalidType` rejects `received` field
When creating `new ZodError([...])` for tests, Zod v4's `$ZodIssueInvalidType<unknown>` type requires `expected` but rejects `received` (TS2353: "does not exist in type"). The correct shape: `{ message: "...", path: [...], code: "invalid_type", expected: "string" }` — omit `received`. Subagents frequently include both fields, causing `tsc --noEmit` failures. Auto-fix: `sed -i '/received:/d'` on affected test files.

### 38. Subagent code always needs biome + tsc post-check
Subagents produce code that passes runtime tests but fails biome formatting (indentation, line length) and TypeScript strict checks (unknown types, extra fields). ALWAYS run `npx biome check --write tests/` and `npx tsc --noEmit` after subagent work before committing. This is a non-negotiable gate — subagent output is raw material, not finished code.

### 40. Dead code after Zod parse — instanceof checks and redundant type guards
After `schema.parse()` succeeds, the parsed value is GUARANTEED to match the schema. Code that checks `if (parsedValue instanceof NextResponse)` or `if (typeof parsedValue.field !== "number")` after a successful parse is dead code — it can never be true. These branches show as uncovered in v8 coverage reports. Fix: remove the dead guard (Zod already validates). Common patterns:
- `const payload = schema.parse(body); if (payload instanceof NextResponse) return payload;` — always false after parse
- `const { score } = schema.parse(body); if (typeof score !== "number")` — always true after parse
- Any `instanceof` or `typeof` check on a value that just came from `.parse()` with the correct type

These guards are typically added defensively but are logically unreachable. The Zod schema IS the type guard. Removing them is safe and necessary for 100% branch coverage with v8.

### 41. DB column name mismatch — always verify schema before writing SQL queries
Never assume column names from variable names or comments. Before writing SQL queries against a table, verify the ACTUAL schema via `information_schema.columns`. Common scenarios:
- Code references `repetitions` but column is named `level`
- Alias direction confusion: `SELECT repetitions AS level` vs `SELECT level AS repetitions`
- CREATE TABLE script says one thing but migration renamed the column

**Verification pattern:**
```sql
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'your_table' ORDER BY ordinal_position;
```
This 10-second check prevents production errors like `column "X" does not exist` (Postgres error code 42703). Applies to any project with manual migrations where code and DB can drift.

### 42. try/catch fallback for getUserIdFromRequest — coverage gap
When adding `try { getUserIdFromRequest(request) } catch { userId = "anonymous" }` to a route handler, the catch branch MUST have a dedicated test. Without a test that calls the route WITHOUT the `x-user-id` header, v8 coverage drops to ~95% lines/branches. **Pattern:** create a `makeRequest(userId?: string)` factory that conditionally sets the header, then have at least one test calling `makeRequest()` (no args = no header = exercises the catch fallback). This applies to any try/catch where the catch provides a silent default — the happy path alone won't cover it.

### 44. TypeScript `tsconfig` `exclude` doesn't prevent checking imported files
TypeScript's `exclude` in `tsconfig.json` only excludes files from the initial discovery set. If an `include`d file imports an `exclude`d file, the `exclude`d file IS still type-checked. This means `tsconfig.validate.json` with `exclude: ["src/lib/db/schema/tables.ts"]` has zero effect if `db.ts` imports from `schema/index.ts` which re-exports `tables.ts`. The only fix is to fix the actual file, or use explicit `include` paths that avoid the problematic directory entirely. **Verification:** `npx tsc --noEmit 2>&1 | grep "error TS" | grep -v "excluded_file.ts" | wc -l` — if this returns 0, the exclusion actually worked. If not, the file is transitively imported and must be fixed.

### 43a. Next.js `title.template` + child page title = duplicate brand name
When `layout.tsx` has `title: { template: "Brand | %s", default: "..." }`, child pages export `title: "Page Name — Brand"` and the result becomes `"Brand | Page Name — Brand"`. The child title must NOT include the brand name — the template already adds it. Fix: child pages export only the page-specific part (`title: "Política de Privacidade"` → renders as `"Brand | Política de Privacidade"`). Audit with browser — check `<title>` tag on every page. Common in legal pages (privacy, terms) copied from templates that include the brand suffix.

### 43b. Fixing visible text (accents, typos) breaks tests asserting on that text
When correcting PT-BR accents ("Comecar" → "Começar") or typos in components, any `screen.getByText("Comecar")` in tests will fail. ALWAYS grep tests for the exact strings being changed and update them simultaneously. Pattern: `grep -rn "oldText" tests/` before committing. Also do a comprehensive pass on body text, not just headings — "detalhadaa" (double-a) and "voce" (missing accent) hide in paragraphs.

### 42b. Coverage exclusion strategy for UI libraries and configs
When aiming for 100% coverage, exclude from `coverage.exclude` in `vitest.config.ts`:
- `src/components/ui/**` — shadcn/ui thin wrappers, tested by upstream
- `src/test/**` — test setup files
- Type-only files (`src/lib/types.ts`) — v8 shows 0% but no executable code
- `src/app/layout.tsx` — Next.js server component wrapper, no logic
- Config files (`next.config.ts`, `postcss.config.mjs`, `vitest.config.ts`)

Every exclusion MUST have a comment explaining WHY. Pattern:
```ts
coverage: {
  exclude: [
    // Shadcn UI — thin wrappers tested by upstream
    "src/components/ui/**",
    // Type-only — v8 shows 0% but no executable code
    "src/lib/types.ts",
  ],
}
```

### 42c. Biome false positive: indexed Record variables reported as "unused"
`const moduleLabels: Record<Module, string> = { ... }` used via `moduleLabels[lesson.modulo]` triggers `noUnusedVariables` because Biome can't trace index access. Fix:
```ts
// biome-ignore lint/correctness/noUnusedVariables: used via index access moduleLabels[lesson.modulo]
const moduleLabels: Record<Module, string> = { ... };
```

### 42d. Biome `noTemplateCurlyInString` in example/lesson code strings
Strings containing template literal syntax like `` `HTTP ${data.status}` `` inside example code trigger `noTemplateCurlyInString`. Fix with biome-ignore:
```ts
// biome-ignore lint/suspicious/noTemplateCurlyInString: example code in lesson string
exemploCodigo: 'throw new Error(`HTTP ${data.status}`)',
```

### 43. Promise without await inside try/catch — silent error swallowing
`try { someAsyncFunction(); } catch { }` silently swallows rejection errors because the promise is not awaited. The `catch` only catches synchronous errors. Fix: ALWAYS add `await` before async calls inside try blocks. SonarQube catches this as S4822. Common in notification/logging code where failure is "non-critical" — but even non-critical code should handle errors explicitly, not silently discard them.

### 45. Subagent tests assert wrong return shape — object vs primitive
Subagents writing API route tests often assume the route returns the full DB row or Zod schema object, but the route may extract a single field. Example: route does `const { multiplier } = schema.parse(body); return Response.json({ multiplier })` — the test asserts `expect(data.multiplier).toEqual({ multiplier: 3 })` instead of `expect(data.multiplier).toBe(3)`. **Pattern:** Before writing assertions, READ the route handler source to see what it actually returns. Never assume the response shape from the Zod schema alone. The schema defines INPUT, not OUTPUT.

---

## CROSS-REFERENCE

**Coverage provider bugs:** See `test-driven-development` skill `references/vitest-coverage-providers.md` for detailed analysis of v8 vs istanbul coverage provider bugs.

**AI Agent Security (OWASP ASI Top 10):** See `references/ai-agent-governance.md` for complete risk mapping, controls, AI Quality Metrics, and governance checklist.

**Solo Dev Playbook:** See `references/solo-dev-playbook.md` for self-review checklist, cognitive load management, and anti-patterns.

**Spec-Driven Development (Thoughtworks 2025):** Context engineering optimizes agent-LLM interaction. Spec as context compression. Deterministic CI/CD remains essential.

**DORA 2025:** "State of AI-assisted Software Development". AI = amplifier. AI Quality Metrics: Pass Rate >= 70%, Regression Rate < 20%, Spec Accuracy >= 80%.

**Error Budget Policy:** Google SRE Workbook adapted for solo dev. 4-week rolling window. Budget esgotado = feature freeze. Review mensal.

[Skill directory: /home/rafaelejosi/.hermes/skills/software-development/project-excellence]
