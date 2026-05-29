# CI Templates — Project Excellence

> Referencia completa dos templates de CI extraidos da secao 5 do SKILL.md.
> O SKILL.md mantem apenas descricoes breves; este arquivo contem os templates integrais.

---

## 1. GitHub Actions CI (ci.yml)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run lint

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run test:coverage

  build:
    runs-on: ubuntu-latest
    needs: [lint, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run build

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm audit --audit-level=high
```

---

## 2. PR Governance (pr-governance.yml)

```yaml
# .github/workflows/pr-governance.yml
name: PR Governance
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  check-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check PR size
        run: |
          FILES=$(git diff --name-only origin/main...HEAD | wc -l)
          LINES=$(git diff origin/main...HEAD | wc -l)
          echo "Files changed: $FILES"
          echo "Lines changed: $LINES"
          if [ "$FILES" -gt 20 ]; then
            echo "::warning::PR too large ($FILES files). Consider splitting."
          fi
      - name: Check for forbidden patterns
        run: |
          git diff origin/main...HEAD | grep -iE '^\+.*(console\.log|debugger|TODO|FIXME|\.skip\()' && \
          echo "::error::Forbidden pattern found" && exit 1 || true
      - name: Validate conventional commits
        run: |
          git log origin/main..HEAD --pretty=format:"%s" | while read -r msg; do
            if ! echo "$msg" | grep -qE '^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?: .+'; then
              echo "::error::Invalid commit message: $msg"
              exit 1
            fi
          done
```

---

## 3. CodeRabbit Config (.coderabbit.yml)

```yaml
# .coderabbit.yml
language: pt-BR
profile:
  name: assertive
reviews:
  profile: assertive
  request_changes_workflow: false
  high_level_summary: true
  poem: false
  auto_review:
    enabled: true
    ignore_draft: true
    base_branches: [main, develop]
```

---

## 4. Local Validation Script (validate-pr.sh)

```bash
#!/bin/bash
# scripts/validate-pr.sh — Rodar ANTES de cada push
# Usage: bash scripts/validate-pr.sh

set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "========================================="
echo " VALIDATE PR — Project Excellence Gate"
echo "========================================="

# Gate 1: Linter
echo ""
echo -e "${YELLOW}[1/4] Running linter...${NC}"
if npm run lint; then
  echo -e "${GREEN}[PASS] Linter OK${NC}"
else
  echo -e "${RED}[FAIL] Linter failed. Fix before push.${NC}"
  exit 1
fi

# Gate 2: Tests + Coverage
echo ""
echo -e "${YELLOW}[2/4] Running tests with coverage...${NC}"
if npm run test:coverage; then
  echo -e "${GREEN}[PASS] Tests OK${NC}"
else
  echo -e "${RED}[FAIL] Tests failed. Fix before push.${NC}"
  exit 1
fi

# Gate 3: Build
echo ""
echo -e "${YELLOW}[3/4] Running build...${NC}"
if npm run build; then
  echo -e "${GREEN}[PASS] Build OK${NC}"
else
  echo -e "${RED}[FAIL] Build failed. Fix before push.${NC}"
  exit 1
fi

# Gate 4: Security
echo ""
echo -e "${YELLOW}[4/4] Running security audit...${NC}"
if npm audit --audit-level=high; then
  echo -e "${GREEN}[PASS] Security audit OK${NC}"
else
  echo -e "${RED}[FAIL] Security vulnerabilities found. Fix before push.${NC}"
  exit 1
fi

echo ""
echo "========================================="
echo -e "${GREEN} ALL GATES PASSED — Ready to push!${NC}"
echo "========================================="
```

**npm script (package.json):**

```json
{
  "scripts": {
    "validate:pr": "npm run lint && npm run test:coverage && npm run build && npm audit --audit-level=high"
  }
}
```

---

## 5. Branch Protection Setup

Configurar via GitHub Settings > Branches > Branch protection rules > `main`:

```
[ ] Require a pull request before merging
    [ ] Require approvals: 1 (se time) ou 0 (se solo)
    [ ] Dismiss stale reviews on new commits
[ ] Require status checks to pass before merging
    Required checks:
      - lint
      - test
      - build
      - security
    [ ] Require branches to be up to date before merging
[ ] Require conversation resolution before merging
[ ] Do not allow force pushes
[ ] Do not allow deletions
```

---

## 6. Husky Setup

```bash
# Instalar husky
npm install -D husky lint-staged
npx husky init

# Criar hooks
echo 'npx lint-staged' > .husky/pre-commit
echo 'npm run validate:pr' > .husky/pre-push
```

**lint-staged config (package.json):**

```json
{
  "lint-staged": {
    "*.{ts,tsx}": ["biome check --write", "vitest related --run"],
    "*.{json,md}": ["biome format --write"]
  }
}
```

**Hooks:**

`.husky/pre-commit`:
```
npx lint-staged
```

`.husky/pre-push`:
```
npm run validate:pr
```

---

## 7. SonarQube Local CI (free alternative to GitHub Actions)

For projects where GitHub Actions is not an option (no budget, privacy, local-only), SonarQube Community Edition runs locally via Docker. Zero cost, zero cloud dependency.

**docker-compose.sonarqube.yml:**
```yaml
services:
  sonarqube:
    image: sonarqube:community
    container_name: project-sonarqube
    ports:
      - "9000:9000"
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_logs:/opt/sonarqube/logs
      - sonarqube_extensions:/opt/sonarqube/extensions
    environment:
      - SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true
    restart: unless-stopped

volumes:
  sonarqube_data:
  sonarqube_logs:
  sonarqube_extensions:
```

**sonar-project.properties:**
```properties
sonar.projectKey=PROJECT_KEY
sonar.projectName=Project Name
sonar.sources=src
sonar.tests=tests
sonar.javascript.lcov.reportPaths=coverage/lcov.info
sonar.exclusions=**/node_modules/**,**/dist/**,**/coverage/**,**/web/**,.next/**,.reports/**
sonar.test.inclusions=**/*.test.ts
sonar.host.url=http://localhost:9000
sonar.token=SCANNER_TOKEN
```

**Vitest LCOV reporter** (required for SonarQube coverage):
```ts
// vitest.config.ts
coverage: {
  reporter: ["text", "html", "lcov"],  // "lcov" must be explicit
}
```

**scripts/sonar-scan.sh:**
```bash
#!/bin/bash
set -euo pipefail
npx vitest run --pool=threads --coverage 2>&1 | tail -5
SCANNER=/tmp/sonar-scanner-5.0.1.3006-linux/bin/sonar-scanner
if [ ! -f "$SCANNER" ]; then
  cd /tmp && curl -L "https://repo1.maven.org/maven2/org/sonarsource/scanner/cli/sonar-scanner-cli/5.0.1.3006/sonar-scanner-cli-5.0.1.3006-linux.zip" -o sonar-scanner.zip && unzip -qo sonar-scanner.zip && cd -
fi
"$SCANNER" -Dsonar.javascript.node.maxspace=4096
echo "Dashboard: http://localhost:9000/dashboard?id=PROJECT_KEY"
```

**package.json scripts:**
```json
{
  "sonar:scan": "bash scripts/sonar-scan.sh",
  "validate:full": "npm run validate:pr && npm run sonar:scan"
}
```

**Pipeline order (local):**
1. `git commit` → Husky pre-commit → lint-staged (Biome on staged files)
2. `git push` → Husky pre-push → `vitest run`
3. `npm run validate:full` → Biome + tsc + vitest + SonarQube scan (~3 min)
4. Dashboard: `http://localhost:9000`

**Key notes:**
- Do NOT use `npx sonar-scanner` — Permission denied on Linux. Download CLI from Maven Central.
- Do NOT put SonarQube scan in Husky pre-commit (too slow, 3+ min).
- First scan takes 1-2 min for SonarQube startup + 2-3 min for analysis.
- Full setup guide: `sonarqube-local` skill.
