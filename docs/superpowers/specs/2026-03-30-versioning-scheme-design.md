# Esquema de Versionamento — Design Spec

**Data:** 2026-03-30
**Status:** Aprovado
**Abordagem:** A — Frontmatter + Git Tags

---

## 1. Frontmatter padrão para cada skill

Todo arquivo `.md` de skill recebe um bloco YAML no topo:

```yaml
---
id: FLOW_CHECKOUT_PAYMENT_PIX
title: "Checkout - Pagamento PIX"
module: checkout
version: "1.0.0"
type: flow                    # flow | api-reference | index | setup
status: implemented           # draft | review | implemented | deprecated
last_updated: "2026-03-30"
authors:
  - rafaelmhp
tags:
  - payment
  - pix
depends_on:
  - FLOW_CHECKOUT_OVERVIEW
  - CHECKOUT_API_REFERENCE
---
```

### Campos obrigatórios

| Campo | Descrição |
|-------|-----------|
| `id` | Identificador unico da skill (nome do arquivo sem extensao) |
| `title` | Titulo legivel por humanos |
| `module` | Dominio ao qual pertence (auth, checkout, pass, tokens, setup, user-profile) |
| `version` | SemVer da skill individual |
| `type` | Tipo: `flow`, `api-reference`, `index`, `setup` |
| `status` | Estado: `draft`, `review`, `implemented`, `deprecated` |
| `last_updated` | Data ISO da ultima alteracao |
| `authors` | Lista de autores |

### Campos opcionais

| Campo | Descrição |
|-------|-----------|
| `tags` | Tags para busca |
| `depends_on` | Skills das quais esta depende |
| `required_by` | Skills que dependem desta |
| `token_estimate` | Estimativa de tokens para contexto LLM |
| `platform_version` | Versao minima da plataforma |

---

## 2. Versionamento por modulo com Git Tags

Formato de tag: `<modulo>/v<MAJOR>.<MINOR>.<PATCH>`

Exemplos: `checkout/v1.0.0`, `pass/v1.0.0`

### Regras de bump

| Acao | Bump |
|------|------|
| Novo arquivo de skill no modulo | MINOR |
| Nova secao ou endpoint em skill existente | MINOR |
| Correcao de texto, typo, exemplo errado | PATCH |
| Skill removida, renomeada ou contrato alterado | MAJOR |

### Conventional Commits

```
docs(checkout): add installment payment details    -> MINOR
fix(pass): correct QR code endpoint example        -> PATCH
breaking(auth): restructure signup flow            -> MAJOR
```

---

## 3. Versao no SKILL_INDEX de cada modulo

O SKILL_INDEX recebe frontmatter com `module_version` e a tabela inclui coluna `Versao`:

```yaml
---
id: CHECKOUT_SKILL_INDEX
title: "Checkout Skill Index"
module: checkout
module_version: "1.0.0"
type: index
last_updated: "2026-03-24"
authors:
  - rafaelmhp
---
```

---

## 4. CHANGELOG com SemVer

Headers por versao do repo, sub-headers por modulo:

```markdown
## [1.1.0] - 2026-03-30

### pass v1.0.0 — Novo modulo
- Adicionado fluxo completo de gestao de passes

## [1.0.0] - 2026-03-24

### auth v1.0.0 — Novo modulo
- Adicionado fluxo completo de signup
```

### Regras de bump do repo

| Acao | Bump |
|------|------|
| Novo modulo adicionado | MINOR |
| Correcoes em modulos existentes | PATCH |
| Mudanca breaking em qualquer modulo | MAJOR |
