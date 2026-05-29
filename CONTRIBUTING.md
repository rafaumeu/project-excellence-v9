# Contributing

This repository is a **skill repository** — a public notebook of knowledge, conventions, and deep-dives. You’re welcome to **fork, comment, and reuse** under the MIT License — **no CLA required**.

## Ways to Contribute

- ✅ **Fix typos, improve wording, correct references** – minimal friction
- ✅ **Add a missing reference**, extend an existing one, or split large files into focused ones
- ✅ **Suggest or open an issue** for missing topics or skill gaps
- ✅ **Port this skill to another repository** (rename/copy everything, remove attribution only if heavy rewrite)
- ❌ **Send executable code or runtimes** – not needed here
- ❌ **Promote unrelated content** – no external links/ads

## How

1. **Fork** → make changes → open a Pull Request from your fork
2. **Add a reference file**: create `references/<short-name>-workflow.md` or `references/<short-name>-gentle-guide.md`
3. **Update SKILL.md**: add a new section like `project-name-*` with ‚úÖ / ‚ùé checks, and link the new reference file

## Conventions

- **Filenames**: kebab-case, no spaces, end with `-workflow.md` or `-guide.md` or leave blank for deep dives
- **Tone**: direct, technical, portuguese br, "tu" informal
- **Frontmatter** (YAML, first 10 lines of SKILL.md): required fields for indexing and validation by CI

## Testing Locally

```bash
# The minimal validation (runs on every push via GH Actions)
gh workflow run validate --repo rafaumeu/project-excellence-v9

# Optional: full local lint (if node installed)
npx markdownlint-cli2@latest '*.md' 'references/**/*.md'
```

## Licensing

All content **MUST** be your own work or properly attributed. The skill and all references inherit the [LICENSE (MIT)](LICENSE) unless an individual file has a different license header on top.

## Recognition

Contributors are credited in a **thanks** table inside SKILL.md (last section).

---

Thanks to contributors (alphabetical order):

|Nome|Role|Link|
|---|---|---|
|(vazio — você pode ser o primeiro!)|Learner/Contributor|(adicione aqui se quiser crédito no README)|