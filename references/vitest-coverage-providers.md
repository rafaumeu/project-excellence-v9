# Vitest Coverage Providers

## Overview

O projeto usa **istanbul** como provider default (recomendado pela equipe Vitest) para `vitest --coverage`. O provider **v8** foi rejeitado durante o Coverage Sprint 2026-05-28 por inconsistências na contagem de branches.

## Providers Suportados

| Provider | Uso | Quando Escolher |
|----------|-----|-----------------|
| istanbul | Default e padrão de fábrica | Quase sempre. Lida melhor com branches complexas e JSX/TSX |
| v8 | Nativo (mais rápido) | Projetos 100% TypeScript puro sem transpilação pesada. Pode sub-reportar branches |

## Configuração Padrão (istanbul)

```ts
// vitest.config.ts
export default defineConfig({
  test: {
    coverage: {
      provider: 'istanbul',
      reportsDirectory: './coverage',
      reporter: ['text', 'json-summary'],
      thresholds: {
        statements: 90,
        branches: 90,
        functions: 90,
        lines: 90,
      },
    },
  },
});
```

## Referências

- [Vitest Coverage Docs](https://vitest.dev/guide/coverage.html)
- [istanbul.js](https://istanbul.js.org/)
- [v8 coverage](https://v8.dev/blog/javascript-code-coverage)