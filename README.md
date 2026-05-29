# Project Excellence v9

Padrão obrigatório de excelência para TODO projeto. Spec-Driven com contrato executável, Context Engineering, OWASP 2025 (A03 Supply Chain + A10 Exceptional Conditions), OWASP ASI Agent Security, Solo Dev Playbook, Error Budget Policy, Testing Pirâmide, DORA 2025, AI Quality Metrics. **Regra, não guideline.**

## Estrutura

```
.
├── SKILL.md                      # Skill principal (Hermes Agent format)
├── README.md                     # Este arquivo
├── references/                   # Conhecimento aprofundado
│   ├── ai-agent-governance.md    # OWASP ASI Top 10 + controles
│   ├── ci-templates.md           # Snippets CI/CD reutilizáveis
│   ├── coverage-gap-workflow.md  # Workflow para fechar gaps de coverage
│   ├── pitfalls-all.md           # 78 armadilhas conhecidas
│   ├── solo-dev-playbook.md      # Anti-patterns + práticas solo dev
│   ├── sre-research.md           # SLOs, Error Budget, DORA
│   ├── supabase-quick-reference.md
│   ├── pentest-framework.md
│   ├── v8-coverage-patterns.md
│   └── ...
└── templates/
    ├── biome-template.json
    └── project-status-template.md
```

## Tripé

1. **SPEC** — Contrato executável + Context Engineering
2. **SECURITY** — OWASP 2025 + ASI Agent Security
3. **CI** — Gates automatizados (Biome + Vitest 100% + tsc + audit)

## Regras Absolutas

1. Sem spec, sem código
2. CI vermelho nunca mergea
3. validate:pr antes de todo push
4. Nunca push direto pra main
5. Todo banco tem RLS. PII = FORCE RLS
6. Specs < 200 LOC
7. Pentest antes de deploy. 8 categorias
8. DELETE debug/admin routes antes de deploy
9. Nunca hardcoded secrets/keys
10. Testing pirâmide + mutation >= 85%

---

**Autor:** Hermes Agent / Rafael Zendron
**Licença:** MIT
**Versão:** 9.0.0