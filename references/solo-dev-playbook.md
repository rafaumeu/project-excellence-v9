# Solo Dev Playbook — Referencia para Project Excellence v9

Contexto: Dev solo usando Hermes (CLI AI agent) como pair programmer. Sem time, sem code review entre pares, sem stakeholders para PRR. Adaptacoes de praticas de time para a realidade solo + AI.

## Mapeamento Team -> Solo + AI

| Pratica de Time | Equivalente Solo + AI | Ferramenta |
|-----------------|----------------------|------------|
| Code review entre pares | AI review (CodeRabbit) + self-review checklist | CodeRabbit + `validate:pr` |
| PRR com stakeholders | Automated gates + manual checklist | CI + `references/prr-checklist.md` |
| Pair programming | Spec review + agent guardrails | SKILL.md + delegate_task |
| Architecture review | Spec-driven design + ADR no Obsidian | Obsidian `04-Projects/` |
| Incident on-call | Alerting + automated rollback | Vercel rollback + Uptime Kuma |
| Sprint planning | Obsidian roadmap + batch specs | Obsidian `04-Projects/` |
| Retrospectives | Hindsight memory + postmortem docs | Hindsight + Obsidian |
| Knowledge sharing | Skills + references + pitfalls | SKILL.md + `references/` |
| QA testing | Automated testing pyramid + mutation | Vitest + Stryker |
| Security audit | Pentest subagents (3 paralelos) | delegate_task |

## Self-Review Checklist (antes de todo commit)

### Funcionalidade
- [ ] Spec escrita e validada contra estado real do projeto
- [ ] Todos os criterios de aceite testados
- [ ] Edge cases cobertos (input invalido, null/undefined, falha de dependência)
- [ ] Comportamento de erro testado e user-friendly

### Qualidade
- [ ] `validate:pr` passou (biome + tsc + vitest + build)
- [ ] Coverage 100% (ou gaps documentados como provider bugs)
- [ ] Nenhum hardcoded secret (grep por `|| '`, `|| "`, fallback hardcoded)
- [ ] Imports corretos (client component nao importa server-only modules)

### Segurança
- [ ] RLS em todas tabelas com PII
- [ ] Auth check em toda API route
- [ ] Zod validation em todo input
- [ ] Error messages genericos (sem leak de internals)
- [ ] CSP headers configurados (se frontend)

### Manutenibilidade
- [ ] Código legível sem comentarios excessivos
- [ ] Nomes descritivos (variaveis, funcoes, arquivos)
- [ ] Dead code removido
- [ ] Types atualizados (sem `as any` desnecessário)

## AI Pair Programming Protocols

### Fluxo Otimizado (1 feature por vez)

```
1. MAPEAR (5-10 min)
   - Ler codigo existente que vai tocar
   - \dt no DB, ls em diretorios, grep por funcionalidades
   - Verificar se ja existe (pitfall #46)

2. OBSIDIAN SYNC (5 min)
   - Documentar estado em 04-Projects/[projeto]-status.md
   - Se ideia rapida -> append em 00-Inbox/Captura Rapida.md

3. SPEC (15-30 min)
   - Escrever TASK-XXX.md com contrato executavel
   - Spec Review Gate: testes derivados, contrato de saida, armadilhas
   - Spec Validation Loop: diff spec vs estado real

4. IMPLEMENTAR (30-60 min por TASK)
   - TDD: RED -> GREEN -> REFACTOR
   - Subagent por TASK (se paralelo, max 2)
   - validate:pr apos cada TASK

5. REVIEW (10-15 min)
   - git diff para revisar TODAS as mudancas
   - CodeRabbit review no PR
   - Self-review checklist

6. RELEASE (5 min)
   - Bump version + tag + GitHub Release
   - Sync Obsidian
```

### Cognitive Load Management

**Problema:** Dev solo troca de contexto entre planejamento, código, testes, review, deploy. Cada troca tem custo cognitivo.

**Soluções:**
1. **Batch specs, implement in parallel, review serial** — Escrever TODAS as specs antes de implementar nenhuma. Implementar em paralelo (subagents). Review em série (uma por vez, atenção total).
2. **1 feature por vez** — Não interleave features. Feature A completa (incluindo testes + review) antes de iniciar Feature B.
3. **Context recovery** — Se sessão cai (PC trava), usar `session_search` + `hindsight_recall` + Obsidian para retomar contexto. Nunca tentar "lembrar".
4. **Capture, don't switch** — Ideia no meio da implementação? Append na Captura Rapida, NÃO trocar de contexto. (Já existe na skill Obsidian)

### Time Management Patterns

**Batch Processing (recomendado para solo dev):**

```
Segunda:   MAPEAR (todas as features do sprint) + OBSIDIAN SYNC
Terça:     SPEC (todas as specs, Spec Review Gate)
Quarta:    IMPLEMENTAR TASK-001, TASK-002 (paralelo via subagents)
Quinta:    IMPLEMENTAR TASK-003, TASK-004 (paralelo)
Sexta:     REVIEW + RELEASE + RETROSPECTIVE
```

**Flow State Protection:**
- Desativar notificações durante IMPLEMENTAR
- Usar `--no-verify` apenas se `validate:pr` acabou de passar localmente
- PC trava? Não entra em panico — Hindsight + Obsidian recuperam tudo

## Automated PRR Gates (Solo Dev)

Sem stakeholders, PRR vira checklist automatizado + verificação manual:

### Automated (CI verifica)

```yaml
# .github/workflows/pr.yml
pr-checks:
  - lint (biome check)
  - type-check (tsc --noEmit)
  - test (vitest run --coverage)
  - build (next build)
  - security (npm audit --audit-level=high)
  - license (license-checker-rseidelsohn)
  - dead-code (knip)
```

### Manual (dev verifica antes de release)

- [ ] SLOs definidos para a feature
- [ ] Rollback playbook documentado (Vercel rollback ou git revert)
- [ ] Error budget policy entendida (se budget esgotado, parar features)
- [ ] Backup do DB verificado (se migration envolve schema change)
- [ ] Feature flag configurada (se feature de dinheiro ou alta prioridade)
- [ ] Pentest das 8 categorias feito (se SaaS com dados pessoais)
- [ ] LGPD compliance verificado (se coleta dados pessoais)
- [ ] Sentry configurado com `sendDefaultPii: false`

## Decision Log Pattern

Sem time pra discutir, decisões ficam no Obsidian:

```markdown
## Decisão: [Título]
- **Data:** YYYY-MM-DD
- **Contexto:** [Por que precisei decidir]
- **Opções consideradas:**
  1. [Opção A] — [prós/contras]
  2. [Opção B] — [prós/contras]
- **Decisão:** [O que escolhi]
- **Por que:** [Justificativa]
- **Consequências:** [O que muda]
```

## Anti-Patterns Solo Dev

1. **YOLO Coding** — Pular spec e ir direto no código. AI facilita isso. Resistir.
2. **Trust All AI Output** — Aceitar código gerado sem review. 15-25% tem vulns.
3. **Context Hoarding** — Tentar carregar tudo na cabeça. Usar Obsidian + Hindsight.
4. **Feature Interleaving** — Trabalhar em 3 features ao mesmo tempo. 1 por vez.
5. **Skip Testing** — "Funciona no meu". Não, não funciona. TDD sempre.
6. **Deploy on Friday** — Sexta é review + release cedo. Nunca deploy grande sexta à noite.
7. **Silo Knowledge** — Não documentar. Se não tá no Obsidian, não existe.
8. **Premature Abstraction** — Criar abstrações antes de ter 3+ implementações concretas.
9. **Scope Creep via Agent** — Agent expande além da TASK. Spec Review Gate previne.
10. **Orphaned Feature Flags** — Flags que nunca são removidas. Lifecycle management.

## Error Budget Policy — Solo Dev

Baseado em Google SRE Workbook, adaptado para dev solo:

| SLO | Error Budget (4 semanas) | Threshold | Ação |
|-----|--------------------------|-----------|------|
| Payment API 99.9% | 43 min downtime | Budget esgotado | PARAR features, focar reliability |
| Core Feature 99.5% | 3.6h downtime | Budget esgotado | PARAR features, focar reliability |
| Auth 99.9% | 43 min downtime | Budget esgotado | PARAR features, focar reliability |
| Qualquer serviço | Incidente > 20% budget | Postmortem obrigatório | Documentar em 04-Projects/Postmortems/ |

**Janela:** 4 semanas (rolling window).
**Consequência:** Se error budget esgotado, NÃO iniciar novas features. Trabalhar exclusivamente em reliability até budget ser restaurado.
**Postmortem:** Incidente único que consome >20% do budget = postmortem em 24h (blameless).
**Review mensal:** SLOs são revisados mensalmente. Ajustar se necessário.
