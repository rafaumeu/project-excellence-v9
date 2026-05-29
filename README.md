# Project Excellence v9 — Portal Status

> Source of truth da skill. Atualizado em 2026-05-29.

## O que é

Framework de excelência obrigatório para todo projeto de software. Spec-Driven com contrato executável, OWASP 2025 (A03 Supply Chain + A10 Exceptional Conditions), OWASP ASI Agent Security, Solo Dev Playbook, Error Budget Policy, Testing Pirâmide, DORA 2025, AI Quality Metrics.

**Regra, não guideline.** Toda feature, todo projeto, todo commit segue este pipeline.

## Branch Atual

- **main** (HEAD: `31059b5`) — v9.0.0, release estável

## Excelência — O que TEM vs o que FALTA

### CI/CD — 100% ✅

| Item | Status | Detalhe |
|------|--------|---------|
| GitHub repo privado | ✅ | github.com/rafaumeu/project-excellence-v9 |
| Estrutura completa | ✅ | SKILL.md + 17 references + 2 templates |
| README template | ✅ | Adaptado do project-status-template |

### References completas — 17/17 ✅

| Item | Status | Detalhe |
|------|--------|---------|
| ai-agent-governance.md | ✅ | OWASP ASI Top 10 + controles |
| ci-templates.md | ✅ | Snippets CI/CD |
| coverage-gap-workflow.md | ✅ | Workflow p/ fechar gaps |
| pitfalls-all.md | ✅ | 78 armadilhas categorizadas |
| solo-dev-playbook.md | ✅ | Anti-patterns solo dev |
| sre-research.md | ✅ | SLOs, Error Budget, DORA |
| supabase-quick-reference.md | ✅ | Patterns Supabase + LGPD |
| pentest-framework.md | ✅ | 8 categorias de pentest |
| v8-coverage-patterns.md | ✅ | Padrões coverage v8 |
| + mais 7 references | ✅ | Diversos tópicos |

### Faltando:

- [ ] CI workflow `.github/workflows/ci.yml` (lint + validação do SKILL.md)
- [ ] LICENSE file (MIT explícito)
- [ ] Dependabot (se aplicável) — repo documental, sem dependências

## Proximos Passos (Prioridade)

1. Adicionar CI workflow pra validar formatação dos .md
2. Adicionar arquivo LICENSE (MIT)
3. Evoluir para v10 conforme novos padrões OWASP/ASI

## Decisões

- **SKILL.md como spec principal** — formato Hermes Agent com frontmatter YAML
- **references/ como deep knowledge** — carregado sob demanda, não polui o contexto
- **README como portal status** — adaptado do project-status-template com seção "O que TEM vs FALTA"

---

**Autor:** Hermes Agent / Rafael Zendron
**Licença:** MIT
**Versão:** 9.0.0