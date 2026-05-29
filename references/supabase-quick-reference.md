# Supabase — Quick Reference

## Pentest Audit Workflow

Todo projeto deve ter pelo menos uma auditoria de seguranca antes de ir pra producao.

### Fluxo de Auditoria (3 subagentes paralelos)

```
Subagent 1: DATABASE
  - RLS status de TODAS as tabelas (habilitado? policies?)
  - Anon access test (deve retornar vazio em todas)
  - Credentials hardcoded no codigo-fonte
  - SECURITY DEFINER functions executaveis por anon
  - search_path faltando em functions
  - Honeypot tables acessiveis
  - security_audit_report() resultado

Subagent 2: AUTH + API
  - OAuth callback: state parameter, code injection, open redirect
  - Auth check em TODAS as API routes (quanto % sem auth?)
  - IDOR: rotas com [id] sem verificar propriedade
  - SQL injection (concatenacao vs parameterized)
  - Rate limiting (existe? funciona em serverless?)
  - CSRF protection
  - Secrets no codigo (VAPID keys, senhas hardcoded)

Subagent 3: FRONTEND
  - dangerouslySetInnerHTML
  - localStorage/sessionStorage com dados sensiveis
  - Client-side secrets (NEXT_PUBLIC_ com segredos)
  - Security headers (CSP, HSTS, X-Frame-Options, etc.)
  - poweredBy header exposto
  - eval() / new Function()
  - Third-party scripts
  - .gitignore para .env
```

### Classificacao de Severidade

| Severidade | Criterio |
|-----------|----------|
| CRITICA   | Permite acesso nao autenticado, tomada de conta, ou dano real |
| ALTA      | Permite acesso a dados de outros usuarios ou modificacao indevida |
| MEDIA     | Hardening faltando mas sem exploit trivial |
| BAIXA     | Informacional, boas praticas |

### Output Padrao

Cada finding: `SEC-NN: titulo | severidade | arquivo:linha | descricao | fix`

### Apos Auditoria

1. Criar plano de correcao em 3 fases (Critico → Alto → Medio)
2. Decompor em specs TASK-xxx (max 200 LOC cada)
3. Mapear dependencias entre tasks
4. Executar por batch (independentes primeiro)

## Pentest Attack Vectors

| Vetor | Risco | Contra-medida |
|-------|-------|---------------|
| Anon Key Extraction | Anon key e publica | RLS em tudo + rate limiting |
| RLS Bypass | Policy mal escrita | Testar `SET ROLE anon` |
| SQL Injection | Query concatenada | Parameterized queries sempre |
| Service Role Leak | Bypass total | Nunca no frontend/bundle |
| CORS * | Qualquer origem | Whitelist de dominios |
| JWT Manipulation | Claims modificados | Validar via `auth.getUser()` |

## Comandos Uteis

```sql
-- Security audit
SELECT * FROM public.security_audit_report();

-- Tabelas sem RLS
SELECT c.relname FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public' AND c.relkind = 'r' AND c.relrowsecurity = false;

-- Testar anon access
SET ROLE anon;
SELECT count(*) FROM public.users; -- Esperado: 0
RESET ROLE;
```

## Role Hierarchy

```
anon          → acesso publico (sem auth)
authenticated → usuario logado (JWT valido)
service_role  → bypass RLS (uso interno APENAS)
```
