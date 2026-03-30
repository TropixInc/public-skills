# Setup Skill Index

Skill de fundacao da plataforma W3block. Use este documento PRIMEIRO antes de qualquer outro skill da KEY API (`https://api.w3block.io`) — tokens, loyalty, withdrawals.

> **Swagger (documentacao interativa):** Acesse `https://api.w3block.io/docs` para testar endpoints e ver schemas atualizados.

---

## Documentos

| # | Documento | Descricao | Status | Quando usar |
|---|-----------|-----------|--------|-------------|
| 1 | [SETUP_PROJECT_BOOTSTRAP.md](./SETUP_PROJECT_BOOTSTRAP.md) | Instalacao, .env, provider tree, NextAuth, QueryClient | ✅ Referencia | Configurar um projeto do zero ou validar setup existente |
| 2 | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) | useAxios, usePrivateQuery, usePublicQuery, useMutation, error handling, JWT | ✅ Referencia | Entender como fazer chamadas API em qualquer modulo |

---

## Guia Rapido

### Para configurar um projeto W3block do zero:

```
1. Leia: SETUP_PROJECT_BOOTSTRAP.md       → Instale pacotes e configure .env.local
2. Configure o provider tree               → W3blockUISDKGeneralConfigProvider + NextAuth
3. Configure o NextAuth route handler      → /app/api/auth/[...nextauth]/route.ts
4. Leia: SETUP_API_PATTERNS.md            → Entenda useAxios, usePrivateQuery, etc.
5. Comece a implementar features           → Use os skills de dominio (auth, checkout, etc.)
```

### Implementacao minima (4 passos):

```
1. npm install @w3block/w3block-ui-sdk @tanstack/react-query next-auth axios
2. Criar .env.local com as 6 URLs de API + NEXTAUTH_SECRET
3. Montar provider tree no layout raiz (SessionProvider > W3blockUISDKGeneralConfigProvider > ...)
4. Criar route handler NextAuth em /app/api/auth/[...nextauth]/route.ts
```

---

## Pre-requisitos

| Requisito | Versao minima | Notas |
|-----------|---------------|-------|
| Node.js | 18+ | LTS recomendado |
| Next.js | 15 | App Router (`/app`) obrigatorio |
| React | 18 | Compativel com Server Components |
| TypeScript | 5+ | Projeto usa TS estritamente |
| `@tanstack/react-query` | 5+ | Gerenciamento de estado server-side |
| `next-auth` | 4.x | Autenticacao via Credentials providers |
| `axios` | 1.x | HTTP client (usado internamente pelo SDK) |

---

## Tabela de Decisao

| Se voce precisa... | Leia este documento | Depois leia... |
|--------------------|---------------------|----------------|
| Criar projeto do zero | [SETUP_PROJECT_BOOTSTRAP.md](./SETUP_PROJECT_BOOTSTRAP.md) | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) |
| Entender como fazer chamadas API | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) | Skill do dominio especifico |
| Implementar NFTs/tokens (KEY API) | SETUP_API_PATTERNS.md (useAxios) | Skill de Tokens |
| Implementar loyalty (KEY API) | SETUP_API_PATTERNS.md (usePrivateQuery) | Skill de Loyalty |
| Implementar withdrawals (KEY API) | SETUP_API_PATTERNS.md (useMutation) | Skill de User-Profile |
| Debugar erro de API | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) (Error Handling) | - |
| Debugar token expirado | [SETUP_API_PATTERNS.md](./SETUP_API_PATTERNS.md) (JWT Validation) | - |

---

## Arquitetura: 5 Microservicos

| Servico | Enum | Base URL (producao) | Responsabilidade |
|---------|------|---------------------|------------------|
| Identity (ID) | `W3blockAPI.ID` | `https://id.w3block.io` | Auth, usuarios, KYC, perfil |
| Key (Blockchain) | `W3blockAPI.KEY` | `https://api.w3block.io` | NFTs, wallets, tokens, blockchain |
| Commerce | `W3blockAPI.COMMERCE` | `https://commerce.w3block.io` | Produtos, pedidos, pagamentos |
| Poll | `W3blockAPI.POLL` | `https://survey.w3block.io` | Enquetes, pesquisas |
| Pass | `W3blockAPI.PASS` | `https://pass.w3block.io` | Token passes, beneficios |

> **Staging:** Em ambientes de staging/dev, substitua pelas URLs correspondentes do ambiente.

---

## Dois Repositorios Principais

| Repositorio | Tipo | Descricao |
|-------------|------|-----------|
| `w3block-ui-sdk` | Biblioteca React | 100+ hooks, providers, componentes reutilizaveis. Importado como `@w3block/w3block-ui-sdk` |
| `w3block-connect-front` | Aplicacao Next.js 15 | Frontend completo que consome o SDK. App Router, NextAuth, i18n |

---

## Glossario Rapido

| Termo | Descricao |
|-------|-----------|
| `companyId` | UUID do tenant (empresa) na plataforma W3block. Usado em praticamente todas as rotas de API. Sinonimo de `tenantId` |
| `tenantId` | Sinonimo de `companyId`. Algumas rotas usam um, outras usam o outro |
| `W3blockAPI` | Enum TypeScript com os 5 microservicos: `ID`, `KEY`, `COMMERCE`, `POLL`, `PASS` |
| `Bearer token` | JWT (access token) enviado no header `Authorization: Bearer <token>`. Obtido via login no Identity API |
| `refreshToken` | Token usado para renovar o access token quando ele expira. Gerenciado automaticamente pelo NextAuth |
| `useAxios(type)` | Hook principal do SDK — retorna instancia Axios configurada (publica ou autenticada) para o microservico especificado |
| `usePrivateQuery` | Wrapper de `useQuery` que so executa quando o usuario esta autenticado (token existe) |
| `usePublicQuery` | Wrapper de `useQuery` para endpoints publicos (sem autenticacao) |
| `useCompanyConfig()` | Hook que retorna `{ companyId, logoUrl, appBaseUrl, connectProxyPass, name }` do contexto |
| `usePixwaySession()` | Hook que retorna a sessao NextAuth (`{ data, status, update }`) |
| `useProfile()` | Hook que busca o perfil do usuario logado via Identity API |
| `useToken()` | Hook que retorna o `accessToken` da sessao atual |
| `handleNetworkException` | Funcao utilitaria que normaliza erros Axios em `{ errorCode, message, statusCode }` |
| `PixwayAPIRoutes` | Enum com todas as rotas de API (127+ rotas) com placeholders `{companyId}`, `{tenantId}`, etc. |
| `CredentialProviderName` | Enum com os 6 providers NextAuth: `signInWithCompanyId`, `changePasswordAndSignIn`, `signInWithCode`, `signInAfterSignUp`, `signInWithGoogle`, `signInWithApple` |
| `W3blockUISDKGeneralConfigProvider` | Provider raiz do SDK que configura APIs, locale, companyId, QueryClient, Metamask, Socket, Cart |
| `SDKQueryProvider` | Provider interno do SDK que wrapa `QueryClientProvider` do TanStack React Query |
| `SessionUser` | Interface que extende `User` do NextAuth com `accessToken`, `refreshToken`, `roles`, `companyId` |
