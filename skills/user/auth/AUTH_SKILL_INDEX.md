---
id: USER_AUTH_SKILL_INDEX
title: "Índice de Skills de Autenticação (Usuário)"
module: user/auth
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
---

# Índice de Skills de Autenticação — Perspectiva do Usuário

Fluxos de autenticação do ponto de vista do usuário final: cadastro, login e recuperação de senha.

**Serviço:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

> Para implementação via SDK React (Next.js), os fluxos abaixo incluem mapeamento para hooks e componentes.
> Para integração direta via API (sem SDK), consulte [AUTH_FLOW_INTEGRATION.md](../../references/AUTH_FLOW_INTEGRATION.md).

---

## Documentos

| # | Documento | Descrição | Quando usar |
|---|-----------|-----------|-------------|
| 1 | [FLOW_SIGNUP.md](./FLOW_SIGNUP.md) | Cadastro de usuário, login (senha, OTP, magic token, OAuth) e recuperação de senha | Implementar qualquer fluxo de autenticação do usuário |

---

## Referências Relacionadas

| Documento | Descrição |
|-----------|-----------|
| [AUTH_API_REFERENCE.md](../../references/auth/AUTH_API_REFERENCE.md) | Todos os endpoints, DTOs, enums e schemas de autenticação |
| [AUTH_FLOW_INTEGRATION.md](../../references/AUTH_FLOW_INTEGRATION.md) | Guia de integração sem SDK (headers, refresh, rate limits) |
| [admin/auth/AUTH_SKILL_INDEX.md](../../admin/auth/AUTH_SKILL_INDEX.md) | Skills de auth para administradores (OAuth config, tenant onboarding) |

---

## Guia Rápido

```
Cadastro de usuário   → FLOW_SIGNUP.md § Sign Up
Login por senha       → FLOW_SIGNUP.md § Sign In (senha)
Login por OTP         → FLOW_SIGNUP.md § Sign In (código)
Login Google/Apple    → FLOW_SIGNUP.md § OAuth
Recuperar senha       → FLOW_SIGNUP.md § Reset Password
```

---

## Armadilhas Comuns

| Armadilha | Solução |
|-----------|---------|
| Não verificar email após cadastro | Exibir tela de verificação e chamar `/auth/request-confirmation-email` se o código expirar |
| Tentar login antes de verificar email | Verificar `emailVerified` no JWT. Se `false`, redirecionar para verificação |
| OAuth redirect URL não configurada | Cadastrar URL de callback no painel do tenant antes de ativar OAuth |
