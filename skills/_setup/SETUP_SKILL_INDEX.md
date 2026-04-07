---
id: SETUP_SKILL_INDEX
title: "Setup Skill Index"
module: setup
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
---

# Setup Skill Index

Skill de fundação da plataforma W3Block. Use este documento PRIMEIRO antes de qualquer outro skill da KEY API (`https://api.w3block.io`) — tokens, loyalty, withdrawals.

> **Swagger (documentação interativa):** Acesse `https://api.w3block.io/docs` para testar endpoints e ver schemas atualizados.

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [SETUP_PROJECT_BOOTSTRAP.md](./SETUP_PROJECT_BOOTSTRAP.md) | 1.0.0 | Instalação, .env, provider tree, NextAuth, QueryClient | ✅ Referência | Configurar um projeto do zero ou validar setup existente |
| 2 | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) | 1.0.0 | useAxios, usePrivateQuery, usePublicQuery, useMutation, error handling, JWT, paginação | ✅ Referência | Entender como fazer chamadas API em qualquer módulo |
| 3 | [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md) | 1.0.0 | Diagrama de serviços, URLs base, mapa de módulos, fluxo de integração completo | ✅ Referência | Entender a arquitetura geral antes de integrar |
| 4 | [GLOSSARY.md](./GLOSSARY.md) | 1.0.0 | Definições de termos do ecossistema W3Block | ✅ Referência | Esclarecer termos como tenant, edition, collection, context, etc. |

---

## Guia Rápido

### Para configurar um projeto W3Block do zero:

```
1. Leia: ARCHITECTURE_OVERVIEW.md         → Entenda os serviços e dependências
2. Leia: SETUP_PROJECT_BOOTSTRAP.md       → Instale pacotes e configure .env.local
3. Configure o provider tree               → W3blockUISDKGeneralConfigProvider + NextAuth
4. Configure o NextAuth route handler      → /app/api/auth/[...nextauth]/route.ts
5. Leia: SETUP_API_PATTERNS.md            → Entenda useAxios, usePrivateQuery, etc.
6. Comece a implementar features           → Use os skills de domínio (auth, checkout, etc.)
```

### Implementação mínima (4 passos):

```
1. npm install @w3block/w3block-ui-sdk @tanstack/react-query next-auth axios
2. Criar .env.local com as 6 URLs de API + NEXTAUTH_SECRET
3. Montar provider tree no layout raiz (SessionProvider > W3blockUISDKGeneralConfigProvider > ...)
4. Criar route handler NextAuth em /app/api/auth/[...nextauth]/route.ts
```

---

## Pré-requisitos

| Requisito | Versão mínima | Notas |
|-----------|---------------|-------|
| Node.js | 18+ | LTS recomendado |
| Next.js | 15 | App Router (`/app`) obrigatório |
| React | 18 | Compatível com Server Components |
| TypeScript | 5+ | Projeto usa TS estritamente |
| `@tanstack/react-query` | 5+ | Gerenciamento de estado server-side |
| `next-auth` | 4.x | Autenticação via Credentials providers |
| `axios` | 1.x | HTTP client (usado internamente pelo SDK) |

---

## Tabela de Decisão

| Se você precisa... | Leia este documento | Depois leia... |
|--------------------|---------------------|----------------|
| Entender a arquitetura geral | [ARCHITECTURE_OVERVIEW.md](./ARCHITECTURE_OVERVIEW.md) | Skill do domínio específico |
| Criar projeto do zero | [SETUP_PROJECT_BOOTSTRAP.md](./SETUP_PROJECT_BOOTSTRAP.md) | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) |
| Entender como fazer chamadas API | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) | Skill do domínio específico |
| Entender termos do ecossistema | [GLOSSARY.md](./GLOSSARY.md) | - |
| Implementar NFTs/tokens (KEY API) | SETUP_API_PATTERNS.md (useAxios) | Skill de Tokens |
| Implementar loyalty (KEY API) | SETUP_API_PATTERNS.md (usePrivateQuery) | Skill de Loyalty |
| Implementar withdrawals (KEY API) | SETUP_API_PATTERNS.md (useMutation) | Skill de User-Profile |
| Debugar erro de API | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) (Error Handling) | [COMMON_ERROR_REFERENCE.md](../references/COMMON_ERROR_REFERENCE.md) |
| Debugar token expirado | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) (JWT Validation) | [AUTH_FLOW_INTEGRATION.md](../references/AUTH_FLOW_INTEGRATION.md) |

---

## Arquitetura: 5 Microsserviços

| Serviço | Enum | Base URL (produção) | Responsabilidade |
|---------|------|---------------------|------------------|
| Identity (ID) | `W3blockAPI.ID` | `https://pixwayid.w3block.io` | Auth, usuários, KYC, perfil, settings |
| Key (Blockchain) | `W3blockAPI.KEY` | `https://api.w3block.io` | NFTs, wallets, tokens, blockchain, loyalty |
| Commerce | `W3blockAPI.COMMERCE` | `https://commerce.w3block.io` | Produtos, pedidos, pagamentos |
| Poll | `W3blockAPI.POLL` | `https://survey.w3block.io` | Enquetes, pesquisas |
| Pass | `W3blockAPI.PASS` | `https://pass.w3block.io` | Token passes, benefícios |

> **Staging:** Em ambientes de staging/dev, substitua pelas URLs correspondentes do ambiente.

---

## Dois Repositórios Principais

| Repositório | Tipo | Descrição |
|-------------|------|-----------|
| `w3block-ui-sdk` | Biblioteca React | 100+ hooks, providers, componentes reutilizáveis. Importado como `@w3block/w3block-ui-sdk` |
| `w3block-connect-front` | Aplicação Next.js 15 | Frontend completo que consome o SDK. App Router, NextAuth, i18n |

---

## Glossário Rápido

> Para definições completas, consulte [GLOSSARY.md](./GLOSSARY.md).

| Termo | Descrição |
|-------|-----------|
| `companyId` | UUID do tenant (empresa) na plataforma W3Block. Usado em praticamente todas as rotas de API. Sinônimo de `tenantId` |
| `tenantId` | Sinônimo de `companyId`. Algumas rotas usam um, outras usam o outro |
| `W3blockAPI` | Enum TypeScript com os 5 microsserviços: `ID`, `KEY`, `COMMERCE`, `POLL`, `PASS` |
| `Bearer token` | JWT (access token) enviado no header `Authorization: Bearer <token>`. Obtido via login no Identity API |
| `refreshToken` | Token usado para renovar o access token quando ele expira. Gerenciado automaticamente pelo NextAuth |
| `useAxios(type)` | Hook principal do SDK — retorna instância Axios configurada (pública ou autenticada) para o microsserviço especificado |
| `usePrivateQuery` | Wrapper de `useQuery` que só executa quando o usuário está autenticado (token existe) |
| `usePublicQuery` | Wrapper de `useQuery` para endpoints públicos (sem autenticação) |
| `useCompanyConfig()` | Hook que retorna `{ companyId, logoUrl, appBaseUrl, connectProxyPass, name }` do contexto |
| `usePixwaySession()` | Hook que retorna a sessão NextAuth (`{ data, status, update }`) |
| `useProfile()` | Hook que busca o perfil do usuário logado via Identity API |
| `useToken()` | Hook que retorna o `accessToken` da sessão atual |
| `handleNetworkException` | Função utilitária que normaliza erros Axios em `{ errorCode, message, statusCode }` |
| `PixwayAPIRoutes` | Enum com todas as rotas de API (127+ rotas) com placeholders `{companyId}`, `{tenantId}`, etc. |
| `W3blockUISDKGeneralConfigProvider` | Provider raiz do SDK que configura APIs, locale, companyId, QueryClient, Metamask, Socket, Cart |
