---
id: FLOW_SIGNUP
title: "Auth - Fluxo de Signup"
module: auth
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-24"
authors:
  - rafaelmhp
---

# Sign Up / Sign In / Reset Password Flow

## Overview

Este documento cobre os tres fluxos de autenticacao do Weblock:

1. **Sign Up** — Cadastro de novo usuario
2. **Sign In** — Login de usuario existente
3. **Reset Password** — Recuperacao de senha

Os fluxos compartilham componentes, hooks e logica de redirect. A secao de Sign Up e a base; Sign In e Reset Password referenciam elementos ja descritos quando aplicavel.

> **Nota:** Este documento contém duas perspectivas para cada fluxo:
> 1. **React SDK Components** — Componentes e hooks do SDK original (`w3block-ui-sdk`), útil como referência de comportamento
> 2. **Implementação Real (Next.js)** — Mapeamento para os componentes e hooks da implementação Next.js App Router atual

---

## Infraestrutura Compartilhada (Next.js)

### API Layer

| Arquivo | Responsabilidade |
|---------|------------------|
| `src/lib/api/auth.ts` | Todas as funções de API de auth: `createUser`, `signInWithCredentials`, `signInWithCode`, `requestSignInCode`, `requestPasswordReset`, `resetPassword`, `signInWithGoogle`, `signInWithApple`, `refreshToken`, `getGoogleRedirectUrl`, `verifySignUp`, `requestConfirmationEmail`, `fetchTenantByHostname` |
| `src/lib/api/client.ts` | `getPublicApi()` (sem auth) e `getSecureApi(token)` (com Bearer) — instâncias Axios |
| `src/lib/api/routes.ts` | `ApiRoutes` enum com todas as rotas de API |

### Types

| Arquivo | Conteúdo |
|---------|----------|
| `src/types/auth.ts` | `CredentialProviderName` enum (6 providers), interfaces: `SignInResponse`, `SessionUser`, `SignUpPayload`, `VerifySignUpResponse`, etc. |
| `src/types/next-auth.d.ts` | Type augmentation do NextAuth: Session (accessToken, refreshToken, roles), JWT (accessTokenExpires, user) |
| `src/types/tenant.ts` | `TenantConfig` interface |

### Providers & Config

| Arquivo | Responsabilidade |
|---------|------------------|
| `src/providers/session-provider.tsx` | Wrapper SessionProvider do NextAuth |
| `src/providers/tenant-provider.tsx` | `TenantProvider` — carrega config do tenant por hostname. `useTenantContext()` hook |
| `src/config/tenant.ts` | `getTenantConfig()` — config do tenant via env vars (fallback) |
| `src/lib/auth/next-auth-config.ts` | Config NextAuth: 6 credential providers, JWT callback com refresh automático (50% lifetime), session callback |
| `src/lib/auth/session.ts` | `getServerSession()` para server-side |
| `src/middleware.ts` | Protege rotas via `withAuth`. Matcher exclui: auth, api, _next, favicon |

### Layout

| Arquivo | Responsabilidade |
|---------|------------------|
| `src/app/auth/layout.tsx` | Layout centralizado para todas as páginas de auth |

---

# PART 1: Sign Up Flow

## Overview

Fluxo de cadastro de novo usuario no Weblock. O usuario preenche um formulario com email e senha, recebe um codigo de verificacao por email, e apos verificar e automaticamente autenticado e redirecionado.

O fluxo possui **dois caminhos** dependendo da configuracao do tenant:

- **Caminho A (padrao):** Formulario com senha → Verificacao por codigo de 6 digitos → Auto sign-in → Redirect
- **Caminho B (Passwordless):** Formulario apenas com email → NextAuth sign-in → Redirect ou fallback

Existe tambem um **caminho alternativo de verificacao por link** onde o usuario clica um link no email em vez de digitar o codigo.

---

## Prerequisites

- **Autenticação:** Nenhuma (fluxo público)
- **Configurações do tenant:**
  - `companyId` (tenantId) — identificador do tenant, obtido do contexto da aplicação
  - `passwordless` — flag que determina se o tenant usa cadastro sem senha
  - `skipWallet` — flag que determina se pula a etapa de wallet externa pós-login
  - `postSigninURL` — URL customizada de redirect pós-login (definida no theme do tenant)
- **Locale:** `i18nLocale` (`'pt-BR'` ou `'en'`) — determinado pelo contexto de internacionalização

---

## Steps

### Step 1: Formulario de Cadastro

**Rota:** `/auth/signUp`

#### Screen

**Template de entrada:** `SignUpTemplateSDK` (`src/modules/auth/templates/SignUpTemplateSDK.tsx`)
→ renderiza `SignUpFormWithoutLayout` (`src/modules/auth/components/SignUpFormWithoutLayout.tsx`)

O usuario ve:
- Campo de email
- Campo de senha (oculto se tenant `passwordless`)
- Campo de confirmacao de senha (oculto se tenant `passwordless`)
- Checkbox "Aceito os Termos de Uso"
- Checkbox "Aceito a Politica de Privacidade"
- Botao "Cadastrar"
- Link para pagina de login

**Campos do formulario:**

| Campo | Tipo | Obrigatorio | Regras |
|-------|------|-------------|--------|
| `email` | string | Sim | Formato de email valido |
| `password` | string | Sim (se !passwordless) | Min 8 chars, regex: `/((?=.*\d)\|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/` |
| `confirmation` | string | Sim (se !passwordless) | Deve ser igual a `password` |
| `acceptsTermsOfUse` | boolean | Sim | Deve ser `true` |
| `acceptsPolicyTerms` | boolean | Sim | Deve ser `true` |

#### User Action

O usuario preenche o formulario e clica em "Cadastrar" (submit).

- **Caminho A (com senha):** Todos os campos sao preenchidos e validados.
- **Caminho B (passwordless):** Apenas o email e preenchido; o submit dispara `signInAfterSignUp()` via NextAuth.

#### Frontend Validation

Validacao via schema **Yup** (hook `usePasswordValidationSchema`):

- **Email:** formato valido (Yup `email()`)
- **Senha:** minimo 8 caracteres + regex que exige pelo menos:
  - 1 letra maiuscula
  - 1 letra minuscula
  - 1 digito ou caractere especial
  - Regex: `/((?=.*\d)|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/`
  - Constantes em: `src/modules/auth/utils/passwordConstants.ts`
- **Confirmacao:** `confirmation` deve ser identica a `password` (Yup `oneOf([ref('password')])`)
- **Checkboxes:** ambos devem ser `true` (Yup `isTrue()`)

#### API Call

**Caminho A (com senha):**

```
POST /auth/signup
```

**Payload:**
```json
{
  "email": "user@example.com",
  "password": "SenhaSegura1!",
  "confirmation": "SenhaSegura1!",
  "tenantId": "company-uuid-here",
  "i18nLocale": "pt-BR",
  "callbackUrl": "https://app.example.com/auth/verify-sign-up?email=user%40example.com",
  "verificationType": "Numeric",
  "utmParams": {
    "utmSource": "google",
    "utmMedium": "cpc",
    "utmCampaign": "signup-campaign"
  }
}
```

**Notas sobre o payload:**
- `tenantId` é obtido de `companyId` do contexto da aplicacao
- `callbackUrl` é construido via `resolveCallbackUrl('/auth/verify-sign-up')` + query string original
- `verificationType` é fixo como `'Numeric'` (codigo de 6 digitos)
- `utmParams` é opcional — capturado da URL de entrada se presente

**Hook:** `useSignUp()` → `POST /auth/signup`
**Arquivo do hook:** `src/modules/auth/hooks/useSignUp.ts`

**Caminho B (passwordless):**
- Nao chama `POST /auth/signup` diretamente
- Chama `signInAfterSignUp()` via NextAuth (fluxo de autenticacao sem senha)
- Se o email ja existir, redireciona para `/auth/signIn/code`

#### Response Handling

**Sucesso (Caminho A):**
- A variavel interna `step` muda de `Steps.SIGN_UP` para `Steps.SUCCESS`
- O formulario de cadastro e substituido pelo componente de verificacao por codigo (`VerifySignUpWithCodeWithoutLayout`)
- O `email` e `password` sao mantidos em estado para uso no Step 3 (auto sign-in)

**Sucesso (Caminho B — passwordless):**
- Redirect automatico via NextAuth

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Email ja cadastrado | `auth>signUpError>emailAlreadyInUse` | Mensagem exibida via `<Alert variant="error">` |
| Erro generico | `auth>signUpError>genericErrorMessage` | Mensagem exibida via `<Alert variant="error">` |

**Deteccao de erro:** O erro `"email is already in use"` e identificado pela mensagem retornada da API e traduzido via i18n.

#### State Changes

- `step`: `Steps.SIGN_UP` → `Steps.SUCCESS`
- `email` e `password` armazenados em estado local do componente (para uso posterior no auto sign-in)

---

### Step 2: Verificacao de Email (Codigo Numerico)

**Renderizado dentro de:** `SignUpFormWithoutLayout` quando `step === Steps.SUCCESS`

#### Screen

**Componente:** `VerifySignUpWithCodeWithoutLayout`
**Arquivo:** `src/modules/auth/components/VerifySignUpWithCodeWithoutLayout.tsx`

O usuario ve:
- Titulo: "Verificacao necessaria"
- Mensagem: "Enviamos um email para: fe****@example.com" (email parcialmente mascarado)
- Grid de 6 inputs numericos (`CodeInputGrid`)
- Botao "Continuar" (desabilitado ate 6 digitos completos)
- Link "Reenviar codigo" (com cooldown de 1 minuto)
- Texto informativo: "O codigo expira em 15 minutos"

**Componente de input:** `CodeInputGrid` (`src/modules/auth/components/CodeInputGrid.tsx`)
- 6 campos de input individuais
- Auto-focus no proximo campo ao digitar
- Navegacao com backspace para campo anterior
- Gerenciado pelo hook `useCodeInput()` (`src/modules/auth/hooks/useCodeInput.ts`)

#### User Action

1. O usuario abre o email e copia o codigo de 6 digitos
2. Digita o codigo nos 6 campos (auto-avanco entre campos)
3. Clica em "Continuar" (ou submete automaticamente quando 6 digitos estao preenchidos)

**Acao alternativa:** Clica em "Reenviar codigo" se nao recebeu o email.

#### Frontend Validation

- O botao "Continuar" so fica habilitado quando todos os 6 campos estao preenchidos
- Validacao de que cada campo contem exatamente 1 digito numerico

#### API Call (Verificacao)

```
GET /auth/verify-sign-up
```

**Payload:**
```json
{
  "email": "user@example.com",
  "token": "123456"
}
```

**Notas:**
- `token` e o codigo de 6 digitos concatenado dos 6 inputs
- `email` e o mesmo utilizado no Step 1

**Hook:** `useVerifySignUp()` → `sdk.api.auth.verifySignUp(payload)`
**Arquivo do hook:** `src/modules/auth/hooks/useVerifySignUp.ts`

#### API Call (Reenvio de Codigo)

```
POST /auth/request-confirmation-email
```

**Payload:**
```json
{
  "email": "user@example.com",
  "tenantId": "company-uuid-here",
  "callbackUrl": "https://app.example.com/auth/complete-profile",
  "verificationType": "numeric"
}
```

**Notas:**
- `callbackUrl` aqui aponta para `/auth/complete-profile` (diferente do Step 1)
- `verificationType` e `'numeric'` (lowercase, diferente do `'Numeric'` do Step 1)

**Hook:** `useRequestConfirmationMail()`
**Arquivo do hook:** `src/modules/auth/hooks/useRequestConfirmationMail.ts`

**Cooldown:** 1 minuto entre reenvios, gerenciado via hook `useCountdown`

#### Response Handling

**Verificacao bem-sucedida:**
- `data.data.verified === true` **E** `password` existe em estado → procede ao Step 3 (Auto Sign-In)
- `data.data.verified === true` **E** `password` nao existe (passwordless) → redirect direto

**Verificacao falhou:**
- `data.data.verified === false` ou erro na resposta → exibe mensagem de erro

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Codigo invalido ou expirado | `auth>codeVerify>invalidOrExpiredCode` | Mensagem exibida; campos de codigo limpos |

#### State Changes

- Apos verificacao com sucesso, o fluxo avanca automaticamente para o Step 3
- O codigo digitado e descartado da memoria

---

### Step 2 (Alternativo): Verificacao por Link

**Rota:** `/auth/verify-sign-up?email=X&token=Y`

#### Screen

**Template de entrada:** `VerifySignUpTemplateSDK` (`src/modules/auth/templates/VerifySignUpTemplateSDK.tsx`)

O usuario ve:
- Estado de loading enquanto o token e verificado automaticamente
- **Sucesso:** Mensagem de confirmacao + redirect automatico para `/auth/signIn` apos 6 segundos
- **Token expirado:** Tela com botao para reenviar email de verificacao

#### User Action

1. O usuario clica no link recebido por email
2. A pagina extrai `email` e `token` da query string
3. Verificacao automatica via `useVerifySignUp`

Nenhuma acao manual e necessaria (exceto no caso de token expirado).

#### API Call

Mesma chamada do Step 2 padrao:

```
GET /auth/verify-sign-up
```

**Payload:**
```json
{
  "email": "user@example.com",
  "token": "token-from-email-link"
}
```

#### Response Handling

- **Sucesso:** Redirect automatico para `/auth/signIn` apos 6 segundos (o usuario faz login manualmente)
- **Token expirado:** Exibe tela de reenvio de email

#### Error States

| Erro | Comportamento |
|------|---------------|
| Token expirado | Exibe tela com opcao de reenviar email |
| Token invalido | Exibe mensagem de erro |

---

### Step 3: Auto Sign-In e Redirect

**Executado automaticamente** apos verificacao com sucesso (Step 2, caminho com senha).

#### API Call

```
signIn({ email, password, companyId })
```

**Notas:**
- Utiliza as credenciais armazenadas em estado local do componente (do Step 1)
- `companyId` e o `tenantId` do contexto

#### Response Handling

- `data.error === null` → chama `redirect()` (logica de redirect abaixo)
- `data.error !== null` → exibe erro de autenticacao

#### Logica de Redirect (`useAuthRedirect`)

**Arquivo:** `src/modules/auth/hooks/useAuthRedirect.ts`

A funcao `redirect()` segue esta **ordem de prioridade** para determinar o destino:

| Prioridade | Condicao | Destino |
|------------|----------|---------|
| 1 | `query.callbackPath` existe | `pushConnect(callbackPath)` |
| 2 | `query.callbackUrl` existe | `pushConnect(callbackUrl)` |
| 3 | `query.contextSlug` existe | `pushConnect(COMPLETE_KYC)` |
| 4 | `postSigninURL` do theme existe | `pushConnect(postSigninURL)` |
| 5 | `!skipWallet` (config do tenant) | `pushConnect(CONNECT_EXTERNAL_WALLET)` |
| 6 | `redirectLink` existe | `pushConnect(redirectLink)` |
| 7 | Nenhuma condicao anterior | `pushConnect('/')` (home) |

**Constantes de rota:**
- `COMPLETE_KYC`: rota de KYC definida em `PixwayAppRoutes`
- `CONNECT_EXTERNAL_WALLET`: rota de wallet definida em `PixwayAppRoutes`
- Enums em: `src/modules/shared/enums/PixwayAppRoutes.ts`

---

### Backend Notes

> **Fase 2** — Detalhamento baseado na analise do backend e Swagger (`pixwayid.w3block.io/docs`).

#### POST /auth/signup — Criacao de Usuario

**Validacao server-side (SignupUserDto):**

| Campo | Tipo | Obrigatorio | Validacao Server |
|-------|------|-------------|------------------|
| `tenantId` | UUID | Sim | `@IsUUID()`, nao pode ser SYSTEM_TENANT_ID |
| `email` | string | Sim | `@IsEmail()`, `@MaxLength(50)`, lowercase + sanitizacao de caracteres nao-printaveis |
| `password` | string | Sim (se !passwordless) | `@IsRequiredPassword()`: 8-32 chars, regex `/((?=.*\d)\|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/` |
| `confirmation` | string | Sim (se !passwordless) | `@Match('password')` — deve ser identico a `password` |
| `name` | string | Nao | `@MinLength(1)`, `@MaxLength(250)` |
| `i18nLocale` | enum | Nao | `@IsEnum(I18nLocaleEnum)`, default: `PT_BR` |
| `callbackUrl` | string | Nao | `@IsUrl()`, `@IsAllowedHost()` — URL de callback para verificacao |
| `verificationType` | enum | Nao | `@IsEnum(VerificationType)` — `NUMERIC` (codigo 6 digitos) ou `INVISIBLE` (link hex) |
| `phone` | string | Nao | `@IsPhoneNumber()` |
| `utmParams` | objeto | Nao | `@Type(() => UTMParamsDto)` — campos: `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content` |
| `referrer` | string | Nao | Codigo de referral do usuario que indicou |

**Comportamento do backend:**

1. **Verificacao de tenant:** Valida que o tenant existe e busca suas configuracoes
2. **Validacao de referrer:** Se `signUp.requireReferrer === true` no tenant config, valida que o referrer existe
3. **Modo passwordless:** Se habilitado no tenant config, auto-gera senha internamente
4. **Verificacoes pre-criacao:** Tenant existe, email nao esta em uso, phone valido (se fornecido)
5. **Persistencia:** Cria usuario com role `User`, status `active: true`
6. **Operacoes pos-criacao:** Armazena UTM params, gera codigo de referral unico, cria relacao referrer (se aplicavel)
7. **Envio de email de verificacao:** Se configurado, envia email com codigo de 6 digitos ou link de verificacao. Subject: "Seja bem-vindo / Welcome". Inclui logo do tenant se disponivel
8. **Verificacao de phone:** Se habilitado no tenant config, envia codigo de verificacao
9. **Evento webhook:** Emite evento `USER_CREATED`
10. **Retorno:** Gera tokens JWT

**Token de verificacao:**

- **NUMERIC:** Codigo de 6 digitos + expiracao (default: **15 minutos**)
- **INVISIBLE:** Token hex de 64 caracteres + expiracao (default: **15 minutos**)
- Expiracao pode ser customizada por tenant

**UTM Params:** Armazenados como JSON. Cria ou atualiza (merge) com campos existentes.

**Swagger:** `POST /auth/signup` → retorna `201 Created` com `SignInResponseDto`

---

#### GET /auth/verify-sign-up — Verificacao de Email

**Metodo HTTP:** `GET` (nao POST — o frontend envia via query params)

**Query Params:**

| Param | Tipo | Validacao |
|-------|------|-----------|
| `email` | string | `@IsEmail()`, `@MaxLength(50)` |
| `token` | string | `@IsVerificationToken()` — formato de 6 digitos ou hex |

**Validacao do token:**

1. Busca usuario pelo email
2. Compara o token enviado com o token armazenado
3. Verifica se o token nao expirou
4. Se token expirado → erro `Token expired. Please complete the sign up process again.`
5. Se token nao encontrado → erro de usuario nao encontrado

**Acoes apos verificacao bem-sucedida:**

1. Marca email como verificado e invalida o token (uso unico)
2. Retorna status de verificacao completa

**Response (VerifySignupResponseDto):**

```json
{
  "verified": true,      // Status geral (email + phone se requerido)
  "emailVerified": true,  // Email verificado com sucesso
  "phoneVerified": false   // Status de verificacao de phone
}
```

**Efeitos colaterais:**
- Token e **invalidado** (limpo do banco) — nao pode ser reutilizado
- Conta marcada como email verificado
- **Nao cria wallet automaticamente** neste endpoint — wallet creation pode ser acionada por fluxos subsequentes dependendo da configuracao do tenant

**Swagger:** `GET /auth/verify-sign-up` → retorna `200 OK` com `VerifySignupResponseDto`

---

#### POST /auth/request-confirmation-email — Reenvio de Email

**Payload (RequestConfirmationEmailDto):**

| Campo | Tipo | Obrigatorio | Validacao |
|-------|------|-------------|-----------|
| `email` | string | Sim | `@IsEmail()` |
| `tenantId` | UUID | Nao | `@IsUUID()` |
| `callbackUrl` | string | Nao | `@IsUrl()`, `@IsAllowedHost()`. Default: `{baseFrontendUrl}/auth/complete-profile` |
| `verificationType` | enum | Nao | `@IsEnum(VerificationType)` — `NUMERIC` ou `INVISIBLE` |

**Comportamento:**
- Sempre gera novo token, sobrescrevendo o anterior
- Mesma logica de expiracao do signup
- Envia mesmo template de email do signup (link ou codigo numerico)
- Se mesmo email existe em multiplos tenants, retorna erro de conflito

**Swagger:** `POST /auth/request-confirmation-email` → retorna `204 No Content`

---

#### signIn (Auto Sign-In pos-verificacao)

**Mecanismo de token:** JWT com algoritmo **RS256** (RSA assimetrico)

**Chave publica:** Exposta via JWKS endpoint (`GET /auth/jwks.json`) — use este endpoint para validar tokens no seu frontend/backend.

**JWT Access Token — Claims:**

| Claim | Descricao |
|-------|-----------|
| `sub` | User ID (UUID) |
| `iss` | Tenant ID (issuer) |
| `aud` | Tenant ID (audience) |
| `email` | Email do usuario |
| `name` | Nome do usuario |
| `roles` | Array de roles (`['USER']`) |
| `tenantId` | Tenant ID |
| `verified` | Boolean — email (e phone se requerido) verificados |
| `emailVerified` | Boolean |
| `phoneVerified` | Boolean |
| `type` | `JwtType.User` |
| `iat` | Issued at (unix timestamp) |
| `exp` | Expiration (unix timestamp) |

**Tempos de expiracao (defaults):**

| Token | Default |
|-------|---------|
| Access Token | `1h` |
| Refresh Token | `2h` |
| Verification Token | `15m` |

**Permissoes do usuario recem-criado:**
- Roles: `[UserRoleEnum.User]` — usuario padrao
- `active: true`
- `verified`: depende se email foi verificado (no fluxo de signup com codigo, a verificacao ocorre antes do sign-in)

**Response (SignInResponseDto):**

```json
{
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIs...",
  "data": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "name": "User Name",
    "roles": ["USER"],
    "tenantId": "tenant-uuid",
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false,
    "type": "user"
  }
}
```

---

## Sequencia Completa de API

### Caminho A — Com senha (padrao)

```
1. POST /auth/signup                           → Cria usuario + envia email com codigo
2. GET /auth/verify-sign-up             → Verifica codigo de 6 digitos
3. signIn({ email, password, companyId }) → Autentica usuario
4. redirect()                            → Redireciona para destino apropriado
```

**Chamada opcional (reenvio):**
```
   POST /auth/request-confirmation-email → Reenvia codigo de verificacao
```

### Caminho B — Passwordless

```
1. signInAfterSignUp() via NextAuth      → Cadastra e autentica em uma unica acao
2. redirect() ou fallback                → /auth/signIn/code se email ja existe
```

### Caminho Alternativo — Verificacao por Link

```
1. POST /auth/signup                           → (ja executado anteriormente)
2. GET /auth/verify-sign-up             → Verificacao automatica via token do link
3. Redirect para /auth/signIn apos 6s    → Usuario faz login manualmente
```

---

## Recuperacao de Erros

| Cenario | Comportamento | Acao do Usuario |
|---------|---------------|-----------------|
| Email ja cadastrado | Mensagem `emailAlreadyInUse` no formulario | Usar outro email ou ir para login |
| Erro generico no signup | Mensagem `genericErrorMessage` | Tentar novamente |
| Codigo invalido | Mensagem `invalidOrExpiredCode` | Digitar codigo correto |
| Codigo expirado | Mensagem `invalidOrExpiredCode` | Clicar em "Reenviar codigo" |
| Token de link expirado | Tela de reenvio de email | Clicar para reenviar |
| Falha no auto sign-in | Erro de autenticacao | Ir para tela de login manualmente |
| Cooldown de reenvio ativo | Botao "Reenviar" desabilitado por 1 minuto | Aguardar cooldown |

---

## React SDK Components

### Componentes Principais

| Componente | Arquivo | Responsabilidade |
|------------|---------|------------------|
| `SignUpTemplateSDK` | `src/modules/auth/templates/SignUpTemplateSDK.tsx` | Template wrapper que aplica theme do tenant e renderiza o formulario |
| `SignUpFormWithoutLayout` | `src/modules/auth/components/SignUpFormWithoutLayout.tsx` | Formulario completo de signup + logica de steps (SIGN_UP → SUCCESS) |
| `VerifySignUpWithCodeWithoutLayout` | `src/modules/auth/components/VerifySignUpWithCodeWithoutLayout.tsx` | Tela de input do codigo de 6 digitos + reenvio |
| `CodeInputGrid` | `src/modules/auth/components/CodeInputGrid.tsx` | Grid de 6 inputs numericos com auto-focus e navegacao |
| `VerifySignUpTemplateSDK` | `src/modules/auth/templates/VerifySignUpTemplateSDK.tsx` | Fluxo alternativo de verificacao por link (token na URL) |

### Hooks

| Hook | Arquivo | Responsabilidade |
|------|---------|------------------|
| `useSignUp` | `src/modules/auth/hooks/useSignUp.ts` | Mutation de criacao de usuario (`POST /auth/signup`) |
| `useVerifySignUp` | `src/modules/auth/hooks/useVerifySignUp.ts` | Mutation de verificacao de email (`GET /auth/verify-sign-up`) |
| `useRequestConfirmationMail` | `src/modules/auth/hooks/useRequestConfirmationMail.ts` | Mutation de reenvio de email de verificacao |
| `useAuthRedirect` | `src/modules/auth/hooks/useAuthRedirect.ts` | Logica de redirect pos-login com prioridade de destinos |
| `useCodeInput` | `src/modules/auth/hooks/useCodeInput.ts` | Gerenciamento dos 6 campos de input do codigo |
| `usePasswordValidationSchema` | `src/modules/auth/hooks/usePasswordValidationSchema.ts` | Schema Yup para validacao de senha |

### Utilitarios e Constantes

| Arquivo | Conteudo |
|---------|----------|
| `src/modules/auth/utils/passwordConstants.ts` | Regex de validacao de senha e tamanho minimo |
| `src/modules/shared/enums/PixwayAPIRoutes.ts` | Enum com todas as rotas de API |
| `src/modules/shared/enums/PixwayAppRoutes.ts` | Enum com todas as rotas da aplicacao (COMPLETE_KYC, CONNECT_EXTERNAL_WALLET, etc.) |

### Templates (pontos de entrada do SDK)

| Template | Arquivo | Responsabilidade |
|----------|---------|------------------|
| `SignUpTemplateSDK` | `src/modules/auth/templates/SignUpTemplateSDK.tsx` | Template wrapper que aplica theme do tenant e renderiza o formulario de signup |
| `VerifySignUpTemplateSDK` | `src/modules/auth/templates/VerifySignUpTemplateSDK.tsx` | Template para fluxo de verificacao por link (token na URL) |

---

## Implementação Real (Next.js) — Sign Up

> As seções acima descrevem os componentes do SDK React original. Abaixo está o mapeamento para a implementação Next.js App Router atual.

### Componentes

| Componente Next.js | Arquivo | Equivalente SDK | Responsabilidade |
|---------------------|---------|-----------------|------------------|
| `SignUpForm` | `src/components/auth/sign-up-form.tsx` | `SignUpFormWithoutLayout` | Formulário de cadastro com validação de senha, steps (SIGN_UP → SUCCESS), UTM tracking |
| `CodeVerification` | `src/components/auth/code-verification.tsx` | `VerifySignUpWithCodeWithoutLayout` + `CodeInputGrid` | Input OTP 6 dígitos com slots visuais, reenvio com cooldown 60s, auto-clear em erro |
| `PasswordStrengthIndicator` | `src/components/auth/password-strength-indicator.tsx` | `AuthPasswordTips` + `PasswordValidationList` | Indicador visual de requisitos de senha em tempo real |
| `AuthCardWrapper` | `src/components/auth/auth-card-wrapper.tsx` | (Templates SDK) | Container reusável para páginas de auth (title, description, children, footer) |
| `AuthFooter` | `src/components/auth/auth-footer.tsx` | — | Footer de navegação entre sign-in e sign-up |
| `OAuthButtons` | `src/components/auth/oauth-buttons.tsx` | (inline no `SignInWithoutLayout`) | Botões Google/Apple condicionais via `useCompanyConfig` |

> **Nota:** Templates do SDK (`*TemplateSDK`) não existem no Next.js — as pages do App Router (`src/app/auth/*/page.tsx`) fazem esse papel.

### Hooks

| Hook Next.js | Arquivo | Equivalente SDK | Diferenças |
|--------------|---------|-----------------|------------|
| `useSignUp()` | `src/hooks/use-sign-up.ts` | `useSignUp` | Mesmo conceito. Usa `useCompanyConfig()` e `useUtmParams()` |
| `useVerifySignUp()` | `src/hooks/use-verify-sign-up.ts` | `useVerifySignUp` | Mesmo conceito |
| `useRequestConfirmationMail()` | `src/hooks/use-request-confirmation-mail.ts` | `useRequestConfirmationMail` | Mesmo conceito. Hardcoded: `verificationType: 'numeric'` |
| `useAuthRedirect()` | `src/hooks/use-auth-redirect.ts` | `useAuthRedirect` | Prioridade: callbackPath → callbackUrl → contextSlug → postSigninURL → connect-wallet → redirectLink → `/` |
| `usePasswordValidation()` | `src/hooks/use-password-validation.ts` | `usePasswordValidationSchema` + `usePasswordMeetsCriteria` | Combinado em um hook: retorna `{ hasMinLength, hasUppercase, hasLowercase, hasNumberOrSpecial, isValid }` |
| `useEmailMask()` | `src/hooks/use-email-mask.ts` | — | Mascara email para privacidade (ex: `jo****@example.com`) |
| `useCountdown()` | `src/hooks/use-countdown.ts` | `useCountdown` (SDK) | Timer reusável para cooldowns (60s para código, 180s para reset) |
| `useUtmParams()` | `src/hooks/use-utm-params.ts` | — | Captura UTM params da URL e persiste em localStorage |
| `useCompanyConfig()` | `src/hooks/use-company-config.ts` | Contexto do tenant no SDK | Retorna config do tenant: `companyId`, `isPasswordless`, `haveGoogleSignIn`, `haveAppleSignIn`, `skipWallet`, `postSigninURL`, `termsUrl`, `privacyUrl` |
| `useTenantBaseUrl()` | `src/hooks/use-tenant-base-url.ts` | — | Resolve base URL do tenant para callbacks. Prioridade: main host → env var → `window.location.origin` |

> **Hooks SDK que NÃO existem no Next.js:**
> - `useCodeInput` — Input OTP é gerenciado inline no componente `CodeVerification`
> - `usePasswordValidationSchema` — Substituído por `usePasswordValidation()` (Zod em vez de Yup)

### Utilitários

| Arquivo | Equivalente SDK | Conteúdo |
|---------|-----------------|----------|
| `src/lib/utils/password-constants.ts` | `src/modules/auth/utils/passwordConstants.ts` | Regex de validação de senha e tamanho mínimo (8 chars) |
| `src/lib/utils/email-mask.ts` | — | Função `maskEmail()` para mascarar email |
| `src/lib/utils/token-validation.ts` | — | Função `validateResetToken()` para validar token de reset (formato `{token};{expiration}`) |

### Rotas

| Rota Next.js | Arquivo | Step |
|-------------|---------|------|
| `/auth/signUp` | `src/app/auth/signUp/page.tsx` | Formulário de cadastro |
| `/auth/verify-sign-up` | `src/app/auth/verify-sign-up/page.tsx` | Verificação por link (auto-verify via query params) |

---
---

# PART 2: Sign In Flow

## Overview

Fluxo de login de usuario existente no Weblock. Suporta **tres metodos de autenticacao** dependendo da configuracao do tenant:

- **Caminho A (padrao):** Email + Senha → Autenticacao → Redirect
- **Caminho B (Passwordless / Code):** Email → Codigo de 6 digitos por email → Autenticacao → Redirect
- **Caminho C (OAuth):** Google ou Apple → Callback OAuth → Autenticacao → Redirect

Apos autenticacao bem-sucedida, o usuario e redirecionado seguindo a mesma logica de prioridade descrita no Sign Up (ver `useAuthRedirect`).

---

## Prerequisites

- **Autenticacao:** Nenhuma (fluxo publico)
- **Configuracoes do tenant:**
  - `companyId` (tenantId) — identificador do tenant, obtido do contexto da aplicacao
  - `passwordless` — flag que determina se o tenant usa login sem senha (apenas codigo)
  - `googleSignIn` — flag que habilita login com Google
  - `appleSignIn` — flag que habilita login com Apple
  - `skipWallet` — flag que determina se pula a etapa de wallet externa pos-login
  - `postSigninURL` — URL customizada de redirect pos-login (definida no theme do tenant)

---

## Steps

### Step 1: Formulario de Login

**Rota:** `/auth/signIn`

#### Screen

**Template de entrada:** `SignInTemplateSDK` (`src/modules/auth/templates/SignInTemplateSDK.tsx`)
→ renderiza `SignInWithoutLayout` (`src/modules/auth/components/SignInWithoutLayout.tsx`)

O usuario ve:
- Campo de email
- Campo de senha (oculto se tenant `passwordless`)
- Link "Esqueceu a senha?" (redireciona para `/auth/changePassword/request`)
- Botao "Entrar"
- Link para pagina de cadastro ("Nao tem conta? Cadastre-se")
- Botao de login com Google (condicional: `googleSignIn` habilitado)
- Botao de login com Apple (condicional: `appleSignIn` habilitado)

**Campos do formulario:**

| Campo | Tipo | Obrigatorio | Regras |
|-------|------|-------------|--------|
| `email` | string | Sim | Formato de email valido |
| `password` | string | Sim (se !passwordless) | Validado contra schema de senha |
| `twoFactor` | string | Nao | Campo opcional para 2FA (se habilitado) |
| `companyId` | string | Auto | Preenchido automaticamente do contexto |

#### User Action

- **Caminho A (com senha):** Usuario preenche email e senha, clica em "Entrar"
- **Caminho B (passwordless):** Usuario preenche apenas email, clica em "Continuar" → redirecionado para `/auth/signIn/code`
- **Caminho C (OAuth):** Usuario clica no botao Google ou Apple

#### Frontend Validation

Validacao via schema **Yup**:

- **Email:** formato valido (Yup `email()`) — erro: `shared>invalidEmail`
- **Senha (modo padrao):** validada contra schema de senha — erro: `companyAuth>signIn>invalidPasswordFeedback`
- **Senha (modo passwordless):** campo opcional, nao validado

Modo de validacao: `onTouched` (valida quando o campo perde o foco)

#### API Call (Caminho A — Email/Senha)

```
POST auth/signin
```

**Payload:**
```json
{
  "email": "user@example.com",
  "password": "SenhaSegura1!",
  "tenantId": "company-uuid-here"
}
```

**Notas:**
- `tenantId` e obtido de `companyId` do contexto da aplicacao
- A chamada e feita via NextAuth Credentials Provider (`SIGNIN_WITH_COMPANY_ID`)

**Hook (SDK):** `usePixwayAuthentication()` → `signIn(payload)`
**Hook (Next.js):** `useSignIn()` → `signInWithCredentials({ email, password })` via NextAuth provider `SIGNIN_WITH_COMPANY_ID`

#### API Call (Caminho B — Passwordless)

Quando o tenant esta configurado como passwordless, o submit do email redireciona para a rota `/auth/signIn/code?email=user@example.com`, onde o fluxo de codigo e iniciado (ver Step 1B).

#### Response Handling

**Sucesso:**
- Sessao criada via NextAuth (JWT + refresh token armazenados)
- Dados da sessao disponiveis via `usePixwaySession()` (SDK) ou `useSession()` do next-auth (Next.js)
- Redirect automatico via `useAuthRedirect` (mesma logica de prioridade do Sign Up — ver Step 3 do Sign Up)

**Response format:**
```json
{
  "token": "jwt-access-token",
  "refreshToken": "refresh-token-string",
  "data": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "roles": ["USER"],
    "name": "User Name",
    "verified": true,
    "companyId": "company-uuid"
  }
}
```

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Credenciais invalidas | `companyAuth>signIn>loginFailedError` | Alerta vermelho exibido por 6 segundos |
| Email invalido | `shared>invalidEmail` | Validacao inline no campo |
| Senha invalida | `companyAuth>signIn>invalidPasswordFeedback` | Validacao inline no campo |
| Timeout de loading | `auth>signIn>loadingTimeout` | Mensagem exibida apos 15 segundos com opcao de reload |

#### State Changes

- Sessao criada e armazenada no contexto NextAuth
- Token JWT e refresh token persistidos
- Profile do usuario carregado via `useProfile()`

---

### Step 1B: Sign In com Codigo (Passwordless)

**Rota:** `/auth/signIn/code`

#### Screen

**Template de entrada:** `SignInWithCodeTemplateSDK` (`src/modules/auth/templates/SignInWithCodeTemplateSDK.tsx`)
→ renderiza `SignInWithCodeWithoutLayout` (`src/modules/auth/components/SignInWithCodeWithoutLayout.tsx`)

O usuario ve:
- Titulo: "Voce ja esteve aqui antes"
- Subtitulo: "Digite o codigo de confirmacao que enviamos para [email mascarado]"
- Grid de 6 inputs numericos (`CodeInputGrid`)
- Botao "Continuar" (desabilitado ate 6 digitos completos)
- Link "Reenviar codigo" (com cooldown de 1 minuto)
- Timer de cooldown exibido (ex: "Voce pode solicitar um novo codigo em 1m 30s")

**Componente de input:** `CodeInputGrid` (`src/modules/auth/components/CodeInputGrid.tsx`)
- 6 campos de input individuais (type="tel")
- Auto-focus no proximo campo ao digitar
- Navegacao com backspace para campo anterior
- Dimensoes: 40x40px (mobile) / 50x50px (desktop)

#### User Action

1. **Solicitacao automatica de codigo:** Ao carregar a pagina, o codigo e enviado automaticamente para o email do usuario
2. O usuario abre o email e copia o codigo de 6 digitos
3. Digita o codigo nos 6 campos (auto-avanco entre campos)
4. Clica em "Continuar"

**Acao alternativa:** Clica em "Reenviar codigo" se nao recebeu o email (cooldown de 1 minuto).

#### API Call (Solicitacao de Codigo)

```
POST /auth/signin/request-code
```

**Payload:**
```json
{
  "email": "user@example.com",
  "tenantId": "company-uuid-here"
}
```

**Hook:** `useRequestSignInCode()`
**Arquivo do hook:** `src/modules/auth/hooks/useRequestSignInCode.ts`

**Cooldown:** 1 minuto entre reenvios, gerenciado via hook `useCountdown`

#### API Call (Verificacao de Codigo)

```
POST /auth/signin/code
```

**Payload:**
```json
{
  "email": "user@example.com",
  "code": "123456"
}
```

**Notas:**
- `code` e o codigo de 6 digitos concatenado dos 6 inputs
- `email` e obtido da query string da URL

**Hook:** `useSignInWithCode()`
**Arquivo do hook:** `src/modules/auth/hooks/useSignInWithCode.ts`

#### Response Handling

**Sucesso:**
- Chama `signInWithCode()` do contexto de autenticacao
- Provider NextAuth: `SIGN_IN_WITH_CODE`
- Sessao criada (JWT + refresh token)
- Redirect via `useAuthRedirect` (mesma logica de prioridade)

**Falha:**
- Exibe mensagem de erro + campos de codigo limpos

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Codigo invalido | `auth>codeVerify>invalidCode` | Mensagem exibida; campos limpos |
| Codigo expirado | `auth>codeVerify>invalidOrExpiredCode` | Mensagem exibida; campos limpos |
| Cooldown ativo | `auth>setCode>cooldownTimeMessage` | Timer exibido; botao de reenvio desabilitado |

---

### Step 1C: Sign In com OAuth (Google / Apple)

#### Screen

Botoes de OAuth exibidos no formulario de login principal (`/auth/signIn`):
- Botao "Continuar com Google" (condicional)
- Botao "Continuar com Apple" (condicional)

#### User Action (Google)

1. Usuario clica em "Continuar com Google"
2. Redirect para URL de OAuth do Google (obtida via `useGetGoogleRedirectLink`)
3. Usuario autoriza no Google
4. Google redireciona de volta para `/auth/signIn?code=AUTH_CODE&scope=googleapis...`
5. Hook `useOAuthSignIn` detecta o `code` + flag `isGoogleSignIn`

#### API Call (Google)

```
GET /auth/{companyId}/signin/google/code?code={authCode}&referrer={callbackUrl}
```

**Notas:**
- `companyId` e o tenantId do contexto
- `code` e o authorization code retornado pelo Google
- `referrer` e a URL de callback

**Hook:** `useOAuthSignIn()` → `signInWithGoogle(payload)`

#### API Call (Apple)

```
POST /auth/{companyId}/signin/apple/code
```

**Fluxo identico ao Google**, com:
- Provider NextAuth: `SIGNIN_WITH_APPLE`
- Hook: `useOAuthSignIn()` → `signInWithApple(payload)`
- Rota de callback: `/auth/signIn/apple`

#### Response Handling (OAuth)

**Sucesso:**
- Sessao criada via NextAuth
- Redirect via `useAuthRedirect`

**Falha:**
- Exibe alerta: `auth>signWithoutLayout>notRegistration` (usuario nao cadastrado)
- Usuario pode tentar outro metodo de login

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Usuario nao registrado (Google) | `auth>signWithoutLayout>notRegistration` | Alerta de aviso exibido |
| Usuario nao registrado (Apple) | `auth>signWithoutLayout>notRegistration` | Alerta de aviso exibido |
| Erro generico OAuth | — | Redirect de volta ao formulario de login |

---

### Step 2: Redirect Pos-Login

**Executado automaticamente** apos autenticacao bem-sucedida (qualquer caminho).

A logica de redirect e **identica** a descrita no Sign Up Step 3 (ver `useAuthRedirect`).

**Verificacoes adicionais pos-login:**

| Verificacao | Destino |
|-------------|---------|
| Usuario nao verificado | Redirect para pagina de verificacao |
| KYC pendente | Redirect para `COMPLETE_KYC` |
| Wallet nao conectada (e `!skipWallet`) | Redirect para `CONNECT_EXTERNAL_WALLET` |
| Callback customizado | Redirect para `callbackPath` ou `callbackUrl` |
| URL pos-login do theme | Redirect para `postSigninURL` |
| Padrao | Redirect para home (`/`) |

**Timeout de seguranca:** Se o redirect nao ocorrer em 15 segundos, exibe mensagem com opcao de reload da pagina.

---

### Backend Notes (Sign In)

> **Fase 2** — Detalhamento baseado na analise do backend e Swagger (`pixwayid.w3block.io/docs`).

#### POST /auth/signin — Login com Email/Senha

**Payload (LoginUserDto):**

| Campo | Tipo | Obrigatorio | Validacao |
|-------|------|-------------|-----------|
| `email` | string | Sim | Email valido, max 50 chars, convertido para lowercase |
| `password` | string | Sim | 8-32 caracteres |
| `tenantId` | UUID | Nao | UUID valido — se omitido, busca em todos os tenants |

**Comportamento:**

1. Busca usuario pelo email (e tenant, se fornecido)
2. Verifica senha
3. Se multiplos tenants tem o mesmo email sem `tenantId` especificado, retorna erro de conflito
4. Retorna tokens JWT (access + refresh)

**Claims do access token:** Mesma estrutura descrita na secao de Sign Up (ver JWT Access Token — Claims).

| Claim | Valor |
|-------|-------|
| `sub` | User ID |
| `iss` | Tenant ID |
| `aud` | Tenant ID |
| `tenantId` | Tenant ID |
| `tokenHash` | SHA256 hash do access token (validacao cruzada) |
| `type` | `JwtType.User` |

**Response (SignInResponseDto):**

```json
{
  "token": "eyJhbGciOiJSUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJSUzI1NiIs...",
  "data": {
    "sub": "user-uuid",
    "email": "user@example.com",
    "name": "User Name",
    "roles": ["USER"],
    "tenantId": "tenant-uuid",
    "verified": true,
    "emailVerified": true,
    "phoneVerified": false,
    "type": "user"
  }
}
```

**Swagger:** `POST /auth/signin` → `201 Created` com `SignInResponseDto`. Erros: `401 Unauthorized`, `429 Too Many Requests`

---

#### POST /auth/signin/code — Login com Codigo

**Payload (LoginUserWithCodeDto):**

| Campo | Tipo | Obrigatorio | Validacao |
|-------|------|-------------|-----------|
| `email` | string | Sim | `@IsEmail()`, `@MaxLength(50)` |
| `code` | string | Sim | Exatamente 6 digitos numericos |
| `tenantId` | UUID | Nao | `@IsUUID()` |

**Comportamento:**

1. Valida o codigo de 6 digitos enviado para o email do usuario
2. Se codigo invalido ou expirado → erro de autenticacao
3. Se usuario nao tinha email verificado, marca como verificado automaticamente
4. Retorna tokens JWT

**Expiracao do codigo:** Default: 30 minutos (configuravel por tenant)

**Swagger:** `POST /auth/signin/code` → `201 Created` com `SignInResponseDto`. Erros: `401`, `429`

---

#### POST /auth/signin/request-code — Solicitar Codigo de Login

**Payload (RequestCodeEmailDto):**

| Campo | Tipo | Obrigatorio | Validacao |
|-------|------|-------------|-----------|
| `email` | string | Sim | Email valido, convertido para lowercase |
| `tenantId` | UUID | Sim | UUID valido |

**Comportamento:**

1. Busca usuario por email + tenantId
2. Gera codigo de 6 digitos (ou reutiliza codigo valido existente)
3. Expiracao default: 30 minutos (configuravel por tenant)

**Email enviado:**
- **Subject:** "Codigo de autenticacao / Authentication code"
- **Conteudo bilingue (PT/EN):**
  - "Aqui esta seu codigo de autenticacao de conta: **[CODE]**"
  - "Here is your account authentication code: **[CODE]**"
  - "Your code expires in: [expiresAt formatado como MM/DD/YYYY HH:mm]"
- Inclui logo do tenant se disponivel

**Swagger:** `POST /auth/signin/request-code` → `200 OK` (sem body). Erros: `401`, `429`

---

#### GET /auth/{tenantId}/signin/google/code — OAuth Google

**Query Params:**

| Param | Tipo | Obrigatorio |
|-------|------|-------------|
| `code` | string | Sim | Authorization code do Google |
| `referrer` | string | Nao | Codigo de referral |

**Path Param:** `tenantId` (UUID)

**Fluxo:**

1. O backend troca o authorization code por um ID token do Google
2. Extrai dados do usuario: `email`, `name`, `locale`
3. Requer que o email seja verificado no Google (`email_verified === true`)

**Criacao/vinculacao de conta:**
- **Usuario novo:** Cria conta com email verificado automaticamente
- **Usuario existente (mesmo email):** Vincula conta Google ao usuario existente

**Configuracao OAuth do tenant:** Deve ser configurada com `clientId`, `clientSecret` e opcionalmente `callbackUri` nas configuracoes do tenant.

**Response:** `SignInResponseDto` com flag `isNewUser: true` se usuario recem-criado

**Swagger:** `GET /auth/{tenantId}/signin/google/code` → `201 Created` com `SignInResponseDto`. Erros: `401`, `429`

---

#### POST /auth/{tenantId}/signin/apple/code — OAuth Apple

**Fluxo identico ao Google**, com diferencas:
- Detecta `isPrivateEmail` (relay addresses da Apple)

**Configuracao OAuth do tenant:** Deve ser configurada com `clientId`, `teamId`, `keyId`, chave privada e opcionalmente `callbackUri` nas configuracoes do tenant.

**Scope:** `['email', 'name']`

**Response:** `SignInResponseDto` com flags `isNewUser` e `isPrivateEmail`

**Swagger:** `POST /auth/{tenantId}/signin/apple/code` → `201 Created` com `SignInResponseDto`. Erros: `401`, `429`

---

#### POST /auth/refresh-token — Renovacao de Token

**Payload (RefreshTokenDto):**

| Campo | Tipo | Obrigatorio |
|-------|------|-------------|
| `refreshToken` | string (JWT) | Sim |

**Comportamento:**

1. Verifica assinatura e validade do refresh token
2. Se token invalido ou expirado → erro
3. Gera novo par de tokens (access + refresh)
4. Invalida o refresh token antigo (token rotation)

**Response (RefreshTokenResponseDto):**

```json
{
  "token": "novo-access-token",
  "refreshToken": "novo-refresh-token"
}
```

**Nota:** O campo `data` (payload decodificado) **nao e retornado** no refresh, diferente do signin.

**Swagger:** `POST /auth/refresh-token` → `201 Created` com `RefreshTokenResponseDto`. Erros: `403 Forbidden`

---

#### Validacao de Tokens

Para validar tokens JWT emitidos pela API:

1. Envie requests autenticados com header `Authorization: Bearer {token}`
2. Para validar tokens no seu backend, use o JWKS endpoint: `GET /auth/jwks.json`
   - Retorna chaves publicas no formato JWK: `{ keys: [{ kid, kty, alg, n, e }] }`
   - Algoritmo: `RS256`

---

## Sequencia Completa de API (Sign In)

### Caminho A — Email/Senha (padrao)

```
1. POST auth/signin                     → Autentica usuario com email/senha
2. redirect()                           → Redireciona para destino apropriado
```

### Caminho B — Passwordless (Codigo)

```
1. POST /auth/signin/request-code       → Envia codigo de 6 digitos por email
2. POST /auth/signin/code               → Verifica codigo e autentica
3. redirect()                           → Redireciona para destino apropriado
```

**Chamada opcional (reenvio):**
```
   POST /auth/signin/request-code       → Reenvia codigo de verificacao
```

### Caminho C — OAuth (Google)

```
1. GET /auth/{companyId}/signin/google/code?code=X  → Backend troca code por perfil Google
2. redirect()                                        → Redireciona para destino apropriado
```

### Caminho C — OAuth (Apple)

```
1. POST /auth/{companyId}/signin/apple/code           → Backend troca code por perfil Apple
2. redirect()                                        → Redireciona para destino apropriado
```

### Refresh Token (automatico)

```
POST auth/refresh-token                → Renova JWT quando expirando (< 50% do tempo restante)
```

---

## Recuperacao de Erros (Sign In)

| Cenario | Comportamento | Acao do Usuario |
|---------|---------------|-----------------|
| Credenciais invalidas | Alerta vermelho por 6 segundos | Corrigir email/senha |
| Codigo invalido | Mensagem de erro; campos limpos | Digitar codigo correto |
| Codigo expirado | Mensagem de erro | Clicar em "Reenviar codigo" |
| Cooldown de reenvio ativo | Botao desabilitado + timer | Aguardar cooldown (1 minuto) |
| OAuth falhou (nao registrado) | Alerta de aviso | Cadastrar-se primeiro ou usar email/senha |
| Timeout de loading (15s) | Mensagem com opcao de reload | Recarregar a pagina |
| Refresh token expirado | Sessao encerrada | Fazer login novamente |

---

## React SDK Components (Sign In)

### Componentes Principais

| Componente | Arquivo | Responsabilidade |
|------------|---------|------------------|
| `SignInTemplateSDK` | `src/modules/auth/templates/SignInTemplateSDK.tsx` | Template wrapper que aplica theme e renderiza o formulario de login |
| `SignInWithoutLayout` | `src/modules/auth/components/SignInWithoutLayout.tsx` | Formulario completo de login (email/senha + OAuth) |
| `SignInWithCodeTemplateSDK` | `src/modules/auth/templates/SignInWithCodeTemplateSDK.tsx` | Template para fluxo de login com codigo |
| `SignInWithCodeWithoutLayout` | `src/modules/auth/components/SignInWithCodeWithoutLayout.tsx` | Tela de input do codigo de 6 digitos para login |
| `CodeInputGrid` | `src/modules/auth/components/CodeInputGrid.tsx` | Grid de 6 inputs numericos (compartilhado com Sign Up) |

### Hooks

| Hook | Arquivo | Responsabilidade |
|------|---------|------------------|
| `usePixwayAuthentication` | `src/modules/auth/hooks/usePixwayAuthentication.ts` | Contexto de autenticacao (signIn, signInWithCode, signInWithGoogle, signInWithApple) |
| `useOAuthSignIn` | `src/modules/auth/hooks/useOAuthSignIn.ts` | Detecta e processa callbacks OAuth (Google/Apple) |
| `useSignInWithCode` | `src/modules/auth/hooks/useSignInWithCode.ts` | Mutation de verificacao de codigo de login |
| `useRequestSignInCode` | `src/modules/auth/hooks/useRequestSignInCode.ts` | Mutation de solicitacao de codigo de login |
| `useGetGoogleRedirectLink` | `src/modules/auth/hooks/useGetGoogleRedirectLink.ts` | Obtem URL de redirect para OAuth Google |
| `useAuthRedirect` | `src/modules/auth/hooks/useAuthRedirect.ts` | Logica de redirect pos-login (compartilhado com Sign Up) |
| `useCodeInput` | `src/modules/auth/hooks/useCodeInput.ts` | Gerenciamento dos campos de input do codigo (compartilhado) |

### API Routes (Sign In)

| Rota | Enum | Descricao |
|------|------|-----------|
| `auth/signin` | `PixwayAPIRoutes.SIGN_IN` | Login com email/senha |
| `/auth/signin/code` | `PixwayAPIRoutes.SIGNIN_WITH_CODE` | Verificacao de codigo de login |
| `/auth/signin/request-code` | `PixwayAPIRoutes.REQUEST_SIGNIN_CODE` | Solicitacao de codigo de login |
| `auth/refresh-token` | `PixwayAPIRoutes.REFRESH_TOKEN` | Renovacao de token JWT |
| `/auth/{companyId}/signin/google/code` | `PixwayAPIRoutes.SIGNIN_WITH_GOOGLE` | Callback OAuth Google |
| `/auth/{companyId}/signin/apple/code` | `PixwayAPIRoutes.SIGNIN_WITH_APPLE` | Callback OAuth Apple |

---

## Gestao de Sessao

### Configuracao NextAuth

- **Estrategia de sessao:** JWT
- **Tempo maximo de sessao:** 120 minutos (ou variavel `NEXT_PUBLIC_SESSION_EXPIRES_IN_SECONDS`)
- **Refresh automatico:** Quando token JWT tem menos de 50% do tempo restante

### Providers de Credenciais

| Provider | Uso |
|----------|-----|
| `SIGNIN_WITH_COMPANY_ID` | Login padrao com email/senha |
| `SIGN_IN_WITH_CODE` | Login com codigo de 6 digitos |
| `SIGNIN_AFTER_SIGNUP` | Login automatico apos cadastro |
| `SIGNIN_WITH_GOOGLE` | Login via OAuth Google |
| `SIGNIN_WITH_APPLE` | Login via OAuth Apple |

### Hooks de Sessao

| Hook | Responsabilidade |
|------|------------------|
| `usePixwaySession()` | Obtem sessao atual do NextAuth |
| `useProfile()` | Obtem dados do perfil do usuario autenticado |

---

## Implementação Real (Next.js) — Sign In

> As seções acima descrevem os componentes do SDK React original. Abaixo está o mapeamento para a implementação Next.js App Router atual.

### Componentes

| Componente Next.js | Arquivo | Equivalente SDK | Responsabilidade |
|---------------------|---------|-----------------|------------------|
| `SignInForm` | `src/components/auth/sign-in-form.tsx` | `SignInWithoutLayout` | Formulário de login com steps (FORM → CODE). Suporta password e passwordless. Integra `OAuthButtons` e `CodeVerification` |
| `CodeVerification` | `src/components/auth/code-verification.tsx` | `SignInWithCodeWithoutLayout` + `CodeInputGrid` | Input OTP 6 dígitos (compartilhado com sign-up). Props: `email`, `onVerified`, `type: 'signup' \| 'signin'` |
| `OAuthButtons` | `src/components/auth/oauth-buttons.tsx` | (inline no `SignInWithoutLayout`) | Botões Google/Apple condicionais via `useCompanyConfig`. Google usa `getGoogleRedirectUrl()`, Apple usa rota API estática |

### Hooks

| Hook Next.js | Arquivo | Equivalente SDK | Diferenças |
|--------------|---------|-----------------|------------|
| `useSignIn()` | `src/hooks/use-sign-in.ts` | `usePixwayAuthentication.signIn()` | Provider: `SIGNIN_WITH_COMPANY_ID`. Retorna `{ signInWithCredentials, isPending, error }` |
| `useSignInWithCode()` | `src/hooks/use-sign-in-with-code.ts` | `usePixwayAuthentication.signInWithCode()` | Provider: `SIGN_IN_WITH_CODE`. Retorna `{ signInWithCode, isPending, error }` |
| `useRequestSignInCode()` | `src/hooks/use-request-sign-in-code.ts` | `useRequestSignInCode` | Mesmo conceito. Usa `useCompanyConfig()` para tenantId |
| `useOAuthSignIn()` | `src/hooks/use-oauth-sign-in.ts` | `useOAuthSignIn` | Auto-detecta provider (Google vs Apple) pelo `scope` query param. Providers: `SIGNIN_WITH_GOOGLE`, `SIGNIN_WITH_APPLE` |

> **Hooks SDK que NÃO existem no Next.js:**
> - `usePixwayAuthentication` — Não existe. Funcionalidades divididas em hooks separados: `useSignIn`, `useSignInWithCode`, `useOAuthSignIn`
> - `useGetGoogleRedirectLink` — Não é hook. É uma API function `getGoogleRedirectUrl()` em `src/lib/api/auth.ts`
> - `usePixwaySession` — Usa `useSession()` direto do `next-auth/react`
> - `useProfile` — Não implementado como hook separado
> - `useCodeInput` — Input OTP gerenciado inline no `CodeVerification`

### Providers NextAuth

| Provider | Uso |
|----------|-----|
| `SIGNIN_WITH_COMPANY_ID` | Login padrão com email/senha |
| `SIGN_IN_WITH_CODE` | Login com código de 6 dígitos |
| `SIGNIN_AFTER_SIGNUP` | Login automático após cadastro (passwordless) |
| `SIGNIN_WITH_GOOGLE` | Login via OAuth Google |
| `SIGNIN_WITH_APPLE` | Login via OAuth Apple |
| `CHANGE_PASSWORD_AND_SIGNIN` | Reset de senha + auto sign-in |

> **Nota:** O doc original lista 5 providers, mas a implementação tem 6 — inclui `CHANGE_PASSWORD_AND_SIGNIN`.

### Configuração NextAuth

| Arquivo | Responsabilidade |
|---------|------------------|
| `src/lib/auth/next-auth-config.ts` | Configuração completa: 6 credential providers, JWT callback com refresh, session callback |
| `src/lib/auth/session.ts` | `getServerSession()` para server-side |
| `src/app/api/auth/[...nextauth]/route.ts` | Route handler do NextAuth |
| `src/middleware.ts` | Protege rotas (exceto auth, api, _next). Redireciona não autenticados para `/auth/signIn` |

### Rotas

| Rota Next.js | Arquivo | Step |
|-------------|---------|------|
| `/auth/signIn` | `src/app/auth/signIn/page.tsx` | Formulário de login (password + OAuth) |
| `/auth/signIn/code` | `src/app/auth/signIn/code/page.tsx` | Input de código para passwordless |

---
---

# PART 3: Reset Password Flow

## Overview

Fluxo de recuperacao de senha do Weblock. O usuario solicita a troca de senha informando seu email, recebe um link por email com token de verificacao, e ao clicar no link define uma nova senha. Apos a troca, o usuario e **automaticamente autenticado** e redirecionado.

O fluxo possui **duas fases**:

1. **Solicitacao:** Usuario informa email → Recebe link por email
2. **Redefinicao:** Usuario clica no link → Define nova senha → Auto sign-in → Redirect

---

## Prerequisites

- **Autenticacao:** Nenhuma (fluxo publico)
- **Configuracoes do tenant:**
  - `companyId` (tenantId) — identificador do tenant, obtido do contexto da aplicacao
- **Acesso ao email:** O usuario precisa acessar o email cadastrado para receber o link de redefinicao

---

## Steps

### Step 1: Solicitacao de Troca de Senha

**Rota:** `/auth/changePassword/request`

#### Screen

**Template de entrada:** `RequestChangePasswordTemplateSDK` (`src/modules/auth/templates/RequestChangePasswordTemplateSDK.tsx`)
→ renderiza `RequestPasswordChangeWithoutLayout` (`src/modules/auth/components/RequestPasswordChangeWithoutLayout.tsx`)

O usuario ve:
- Titulo da pagina: `auth>requestPasswordChange>pageTitle`
- Titulo do formulario: `companyAuth>requestPasswordChange>formTitle`
- Campo de email com label: `home>contactModal>email`
- Placeholder do campo: `companyAuth>newPassword>enterYourEmail`
- Botao "Avancar": `components>genericMessages>advance`

**Campos do formulario:**

| Campo | Tipo | Obrigatorio | Regras |
|-------|------|-------------|--------|
| `email` | string | Sim | Formato de email valido |

#### User Action

O usuario preenche o email cadastrado e clica em "Avancar".

#### Frontend Validation

Validacao via schema **Yup**:

- **Email:** campo obrigatorio — erro: `components>form>requiredFieldValidation`
- **Email:** formato valido — erro: `companyAuth>requestPasswordChange>invalidEmailError`

Modo de validacao: `onChange` (valida em tempo real enquanto o usuario digita)

#### API Call

```
POST auth/request-password-reset
```

**Payload:**
```json
{
  "email": "user@example.com",
  "tenantId": "company-uuid-here",
  "verificationType": "invisible",
  "callbackUrl": "https://app.example.com/auth/changePassword/newPassword"
}
```

**Notas:**
- `tenantId` e obtido de `companyId` do contexto da aplicacao
- `callbackUrl` aponta para a rota `PixwayAppRoutes.RESET_PASSWORD` (`/auth/changePassword/newPassword`)
- `verificationType` e `'invisible'` por padrao (link no email, nao codigo numerico)

**Hook:** `useRequestPasswordChange()`
**Arquivo do hook:** `src/modules/auth/hooks/useRequestPasswordChange.ts`

#### Response Handling

**Sucesso:**
- URL atualizada com `?step=2`
- Componente muda para `PasswordChangeEmailSentWithoutLayout` (tela de email enviado)

**Erro:**
- Se o email nao existe no sistema: erro no campo com mensagem `companyAuth>requestPasswordChange>emailDoesntExistError`
- Link para cadastro exibido: `changePasswordPage>emailDontExistErrorTip>signUpLink`

#### Error States

| Erro | Codigo i18n | Comportamento |
|------|-------------|---------------|
| Email nao cadastrado | `companyAuth>requestPasswordChange>emailDoesntExistError` | Erro inline no campo + link para Sign Up |
| Email invalido | `companyAuth>requestPasswordChange>invalidEmailError` | Validacao inline no campo |
| Campo obrigatorio vazio | `components>form>requiredFieldValidation` | Validacao inline no campo |

#### State Changes

- URL query: `step` → `2`
- Componente renderizado muda de formulario para tela de confirmacao

---

### Step 2: Email de Redefinicao Enviado

**Renderizado quando:** `step === 2` na URL

#### Screen

**Componente:** `PasswordChangeEmailSentWithoutLayout`
**Arquivo:** `src/modules/auth/components/PasswordChangeEmailSentWithoutLayout.tsx`

O usuario ve:
- Titulo: `auth>passwordChangeMailStep>formTitle`
- Mensagem: `companyAuth>requestPaswordChange>linkSentToMail` com email parcialmente mascarado (ex: "us**@example.com")
- Icone de chave em cor primaria do tenant
- Botao "Reenviar link": `auth>mailStep>resentLinkButton`
- Timer de cooldown: `companyAuth>sendMailToChangePassword>cooldownTimeMessage` (formato mm:ss)
- Texto informativo: `auth>mailStep>linkExpirationMessage`
- Botao "Continuar": `components>advanceButton>continue`

#### User Action

1. O usuario abre o email e clica no link de redefinicao
2. **Acao alternativa:** Clica em "Reenviar link" se nao recebeu o email

#### Reenvio de Email

- **API Call:** Mesma chamada do Step 1 (`POST auth/request-password-reset`)
- **Cooldown:** 3 minutos entre reenvios
- **Persistencia:** Timestamp do cooldown salvo em `localStorage` (chave: `PASSWORD_LINK_CONFIRMATION_COUNTDOWN_DATE`)
- **Gerenciado via:** hook `useCountdown`
- Botao "Reenviar" desabilitado durante cooldown ou enquanto request esta pendente

#### State Changes

- Cooldown timer iniciado apos envio/reenvio
- Timestamp salvo em localStorage para persistir entre reloads

---

### Step 3: Redefinicao de Senha

**Rota:** `/auth/changePassword/newPassword?email={email}&token={token}`

#### Validacao na Montagem

Ao carregar a pagina, o componente executa validacoes automaticas:

1. **Token presente e formato valido:**
   - Se `token` ausente na URL → Redirect para HOME
   - Se token nao contem separador `;` → Redirect para HOME
2. **Token nao expirado:**
   - Extrai timestamp de expiracao do token (formato: `{tokenString};{expirationTimestamp}`)
   - Valida com `isValid()` e `isAfter()` do `date-fns`
   - Se expirado → Exibe tela de token expirado (Step 3B)
3. **Email presente:**
   - Se `email` ausente na URL → Redirect para HOME

#### Screen (Token Valido)

**Template de entrada:** `ResetPasswordTemplateSDK` (`src/modules/auth/templates/ResetPasswordTemplateSDK.tsx`)
→ renderiza `ResetPasswordWithoutLayout` (`src/modules/auth/components/ResetPasswordWithoutLayout.tsx`)

O usuario ve:
- Titulo do formulario: `auth>changePasswordForm>title`
- Campo de nova senha:
  - Label: `companyAuth>newPassword>passwordFieldLabel`
  - Placeholder: `companyAuth>changePassword>passworldFieldPlaceholder`
  - Tipo: `password`
- Campo de confirmacao de senha:
  - Label: `companyAuth>newPassword>passwordConfirmationFieldLabel`
  - Tipo: `password`
- Componente de dicas de senha (`AuthPasswordTips`) com validacao em tempo real
- Botao "Avancar": `components>genericMessages>advance`

**Campos do formulario:**

| Campo | Tipo | Obrigatorio | Regras |
|-------|------|-------------|--------|
| `password` | string | Sim | Min 8 chars, regex de senha (mesma do Sign Up) |
| `confirmation` | string | Sim | Deve ser igual a `password` |

#### Frontend Validation

Validacao via schema **Yup** (hook `usePasswordValidationSchema`):

- **Senha:** mesmas regras do Sign Up:
  - Minimo 8 caracteres — erro: `auth>passwordValidation>minCharacters`
  - Regex: `/((?=.*\d)|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/` — erro: `auth>passwordErrorFeedback>genericInvalidMessage`
- **Confirmacao:** deve ser identica a `password` — erro: `signUpForm>passwordConfirmationValidation>passwordDoesntMatch`

**Dicas visuais de senha (`PasswordValidationList`):**

Cada requisito e exibido como item de lista com indicador visual:
- Checkmark verde = requisito atendido
- Circulo vermelho = requisito nao atendido
- Sem icone = campo ainda nao tocado

| Requisito | Codigo i18n |
|-----------|-------------|
| Minimo 8 caracteres | `companyAuth>newPasswordTips>passwordMeetsMinimumCharactersQuantity` |
| Contem letra maiuscula | `companyAuth>newPasswordTips>passwordContainsUppercaseLetter` |
| Contem letra minuscula | `companyAuth>newPasswordTips>passwordContainsLowercaseLetter` |
| Contem numero ou caractere especial | `companyAuth>newPasswordTips>passwordContainsNumbers` |

**Constantes de validacao:** `src/modules/auth/utils/passwordConstants.ts`
```
PASSWORD_MIN_LENGTH = 8
PASSWORD_REGEX = /((?=.*\d)|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/
PASSWORD_HAS_NUMBER = /\d/g
PASSWORD_HAS_CAPITALIZED_LETTER = /[A-Z]/g
PASSWORD_HAS_UNCAPITALIZED_LETTER = /[a-z]/g
```

Modo de validacao: `onChange`

#### User Action

O usuario define a nova senha, confirma, e clica em "Avancar".

#### API Call

```
POST auth/reset-password
```

**Payload:**
```json
{
  "email": "user@example.com",
  "token": "decoded-token-from-url",
  "password": "NovaSenha1!",
  "confirmation": "NovaSenha1!"
}
```

**Notas:**
- `token` e decodificado da URL com `decodeURIComponent()` (sem o timestamp de expiracao)
- `email` e obtido da query string da URL

**Hook:** `useChangePasswordAndSignIn()`
**Arquivo do hook:** `src/modules/auth/hooks/useChangePasswordAndSignIn.ts`

Este hook combina duas operacoes:
1. Chama `sdk.api.auth.resetPassword(payload)` para trocar a senha
2. Em caso de sucesso, chama `changePasswordAndSignIn()` do contexto de autenticacao para auto-login

#### Response Handling

**Sucesso:**
- `step` muda para `Steps.PASSWORD_CHANGED` (valor: `2`)
- Componente muda para `AuthPasswordChangedWithoutLayout`
- Usuario automaticamente autenticado

**Token expirado (durante submit):**
- Status HTTP `410` **OU** mensagem de erro contem `"expired"`
- `step` muda para `Steps.EXPIRED_PASSWORD` (valor: `4`)
- Componente muda para `ExpiredTokenWithoutLayout`

**Erro generico:**
- `step` muda para `Steps.ERROR` (valor: `5`)
- Componente muda para `AuthErrorChagingPassword`

#### Error States

| Erro | Condicao | Comportamento |
|------|----------|---------------|
| Token expirado (na montagem) | Timestamp do token ja passou | Exibe tela de token expirado com opcao de reenviar |
| Token expirado (no submit) | API retorna 410 ou "expired" | Exibe tela de token expirado com opcao de reenviar |
| Token invalido | Token ausente ou formato incorreto | Redirect para HOME |
| Erro generico | Qualquer outro erro na API | Exibe tela de erro com opcoes de cancelar ou tentar novamente |

#### State Changes (Steps Enum)

```
Steps {
  PASSWORD_CHANGED = 2,   → Tela de sucesso
  EMAIL_SENT = 3,         → Tela de email reenviado
  EXPIRED_PASSWORD = 4,   → Tela de token expirado
  ERROR = 5               → Tela de erro
}
```

---

### Step 3B: Token Expirado

**Renderizado quando:** Token expirado (validacao na montagem ou resposta da API)

#### Screen

**Componente:** `ExpiredTokenWithoutLayout`
**Arquivo:** `src/modules/auth/components/ExpiredTokenWithoutLayout.tsx`

O usuario ve:
- Titulo: `auth>expiredLink>stepTitle`
- Mensagem: `auth>expiredLink>linkNotValidatedMessage`
- Icone de erro de email (187x187px)
- Botao "Reenviar link": `auth>expiredLink>resendCodeButton`

#### User Action

O usuario clica em "Reenviar link" para receber um novo email de redefinicao.

#### API Call

Mesma chamada do Step 1:

```
POST auth/request-password-reset
```

**Payload:**
```json
{
  "email": "user@example.com",
  "tenantId": "company-uuid-here",
  "verificationType": "invisible",
  "callbackUrl": "https://app.example.com/auth/changePassword/newPassword"
}
```

#### Response Handling

- **Sucesso:** URL atualizada com `step=EMAIL_SENT` (valor: `3`), exibe tela de email enviado (Step 2)
- **Erro:** Exibe mensagem de erro

---

### Step 3C: Erro na Troca de Senha

**Renderizado quando:** `step === Steps.ERROR`

#### Screen

**Componente:** `AuthErrorChagingPassword`
**Arquivo:** `src/modules/auth/components/AuthErrorChagingPassword.tsx`

O usuario ve:
- Icone de erro (vermelho, 36x36px)
- Titulo: `changePassword>error>errorChangingPassword`
- Botao "Cancelar" (cinza): `components>cancelMessage>cancel` → Redirect para HOME
- Botao "Continuar" (cor primaria): `components>advanceButton>continue` → Volta ao formulario de senha (Step 3)

#### User Action

- **Cancelar:** Redireciona para a home
- **Tentar novamente:** Volta ao formulario de nova senha

---

### Step 4: Senha Alterada com Sucesso

**Renderizado quando:** `step === Steps.PASSWORD_CHANGED`

#### Screen

**Componente:** `AuthPasswordChangedWithoutLayout`
**Arquivo:** `src/modules/auth/components/AuthPasswordChangedWithoutLayout.tsx`

O usuario ve:
- Icone de sucesso (checkmark verde, 36x36px)
- Titulo: `companyAuth>resetPassword>passwordChangedSuccessfully`
- Botao "Continuar": `components>advanceButton>continue`

#### User Action

O usuario clica em "Continuar" para ser redirecionado.

#### Response Handling

- Redirect para `PixwayAppRoutes.HOME` (`/`) ou `PixwayAppRoutes.TOKENS` (`/tokens`) dependendo do template
- O usuario ja esta autenticado (auto sign-in executado no Step 3)

---

### Backend Notes (Reset Password)

> **Fase 2** — Detalhamento baseado na analise do backend e Swagger (`pixwayid.w3block.io/docs`).

#### POST /auth/request-password-reset — Solicitar Redefinicao

**Payload (RequestPasswordResetDto):**

| Campo | Tipo | Obrigatorio | Validacao |
|-------|------|-------------|-----------|
| `email` | string | Sim | `@IsEmail()` |
| `tenantId` | UUID | Nao | `@IsUUID()` |
| `callbackUrl` | string | Nao | `@IsUrl()`. Default: `{baseFrontendUrl}/auth/changePassword/newPassword` |
| `verificationType` | enum | Nao | `@IsEnum(VerificationType)`. Default: `INVISIBLE` (link) |

**Comportamento:**

1. Busca usuario pelo email (e tenant, se fornecido)
2. Se email nao existe → erro
3. Verifica conflito de email entre tenants
4. Se `callbackUrl` nao fornecida, usa default: `{baseFrontendUrl}/auth/changePassword/newPassword`
5. Gera token de verificacao (default: 15 minutos de expiracao, configuravel por tenant)
6. Envia email de redefinicao

**Query params incluidos no link do email:**

| Param | Descricao |
|-------|-----------|
| `token` | Token completo (`{code};{timestamp}`) |
| `code` | Apenas a parte do codigo (para display) |
| `expire` | Timestamp de expiracao |
| `email` | Email do usuario |
| `tenantId` | ID do tenant |

**Email enviado:**
- **Subject:** "Resetar senha / Reset password"
- **Conteudo bilingue (PT/EN):** Link de redefinicao ou codigo numerico, dependendo do `verificationType`
- Inclui logo do tenant se disponivel

**Nota:** Permite reset mesmo para usuarios com email nao verificado.

**Swagger:** `POST /auth/request-password-reset` → `204 No Content`. Erros: `429 Too Many Requests`

---

#### POST /auth/reset-password — Redefinir Senha

**Payload (ResetPasswordDto):**

| Campo | Tipo | Obrigatorio | Validacao Server |
|-------|------|-------------|------------------|
| `email` | string | Sim | `@IsEmail()` |
| `token` | string | Sim | `@IsVerificationToken()` — formato `{code};{timestamp}` |
| `password` | string | Sim | `@IsRequiredPassword()`: 8-32 chars, regex `/((?=.*\d)\|(?=.*\W+))(?![.\n])(?=.*[A-Z])(?=.*[a-z]).*$/` |
| `confirmation` | string | Sim | `@Match('password')` — identico ao `password` |

**Validacao server-side da senha (`@IsRequiredPassword`):**

1. Minimo 8 caracteres, maximo 32
2. Pelo menos 1 letra maiuscula (`[A-Z]`)
3. Pelo menos 1 letra minuscula (`[a-z]`)
4. Pelo menos 1 digito (`\d`) OU 1 caractere especial (`\W`)
5. Se tenant `passwordless.enabled === true`, senha pode ser omitida

**Comportamento:**

1. Valida o token enviado (verifica match e expiracao)
2. Se token expirado → erro `Token expired`
3. Se token nao encontrado → erro de usuario nao encontrado
4. Atualiza senha do usuario (hash seguro)
5. Marca email como verificado e invalida o token (uso unico)
6. Retorna tokens JWT (auto sign-in)

**HTTP Status codes:**

| Status | Condicao |
|--------|----------|
| `200 OK` | Senha redefinida com sucesso + retorna SignInResponseDto |
| `400 Bad Request` | Token expirado: `"Token expired. Please complete the sign up process again."` |
| `404 Not Found` | Email ou token nao encontrado (`UserNotFoundByEmailException`) |
| `429 Too Many Requests` | Rate limit excedido |

**Nota:** O backend retorna `400 Bad Request` para token expirado (com mensagem contendo "expired").

**Efeitos colaterais:**
- Senha alterada
- Email marcado como verificado
- Token de verificacao invalidado (uso unico)
- Retorna novo par de tokens JWT (auto sign-in)

**Swagger:** `POST /auth/reset-password` → `200 OK` com `SignInResponseDto`. Erros: `400 Bad Request`, `429 Too Many Requests`

---

#### Tempos de Expiracao (Referencia)

| Token | Default |
|-------|---------|
| Access Token | 1h |
| Refresh Token | 2h |
| Token de verificacao/reset | 15m |

**Nota:** A expiracao de tokens de verificacao pode ser customizada por tenant.

---

#### Emails Enviados pela API (Referencia)

| Fluxo | Subject |
|-------|---------|
| Signup (verificacao) | "Seja bem-vindo / Welcome" |
| Login (codigo passwordless) | "Codigo de autenticacao / Authentication code" |
| Reset de senha | "Resetar senha / Reset password" |

Todos os emails sao bilingues (PT/EN) e incluem logo do tenant quando disponivel.

---

#### Seguranca

- Senhas sao armazenadas com hash seguro (bcrypt)
- JWT com algoritmo RS256 (assimetrico) — chaves publicas disponiveis via JWKS endpoint
- Tokens de verificacao sao invalidados apos uso (uso unico)
- Refresh token rotation implementado (token antigo invalidado ao gerar novo)
- Rate limiting por endpoint
- CORS e headers de seguranca habilitados

---

## Sequencia Completa de API (Reset Password)

### Fluxo Padrao

```
1. POST auth/request-password-reset         → Envia email com link de redefinicao
2. [Usuario clica no link no email]
3. POST auth/reset-password                  → Redefine senha com token do link
4. changePasswordAndSignIn()                 → Auto sign-in apos troca
5. redirect()                               → Redireciona para destino apropriado
```

**Chamada opcional (reenvio):**
```
   POST auth/request-password-reset         → Reenvia email de redefinicao (cooldown 3 min)
```

### Fluxo com Token Expirado

```
1. POST auth/request-password-reset         → Email original enviado
2. [Token expira antes do usuario clicar]
3. [Pagina detecta expiracao na montagem]
4. POST auth/request-password-reset         → Reenvio de email
5. [Usuario clica no novo link]
6. POST auth/reset-password                 → Redefine senha com novo token
7. changePasswordAndSignIn()                → Auto sign-in
8. redirect()                               → Redirect
```

---

## Recuperacao de Erros (Reset Password)

| Cenario | Comportamento | Acao do Usuario |
|---------|---------------|-----------------|
| Email nao cadastrado | Erro inline + link para Sign Up | Cadastrar-se ou corrigir email |
| Token expirado (na montagem) | Tela de token expirado | Clicar em "Reenviar link" |
| Token expirado (no submit) | Tela de token expirado (API 410) | Clicar em "Reenviar link" |
| Token invalido/ausente | Redirect para HOME | Solicitar novo link |
| Erro generico no reset | Tela de erro com opcoes | Cancelar ou tentar novamente |
| Cooldown de reenvio ativo | Botao desabilitado + timer (3 min) | Aguardar cooldown |
| Senhas nao coincidem | Validacao inline no campo | Corrigir confirmacao |
| Senha fraca | Dicas visuais em vermelho | Atender requisitos de senha |

---

## React SDK Components (Reset Password)

### Componentes Principais

| Componente | Arquivo | Responsabilidade |
|------------|---------|------------------|
| `RequestChangePasswordTemplateSDK` | `src/modules/auth/templates/RequestChangePasswordTemplateSDK.tsx` | Template wrapper para formulario de solicitacao |
| `RequestPasswordChangeWithoutLayout` | `src/modules/auth/components/RequestPasswordChangeWithoutLayout.tsx` | Formulario de solicitacao de troca de senha |
| `PasswordChangeEmailSentWithoutLayout` | `src/modules/auth/components/PasswordChangeEmailSentWithoutLayout.tsx` | Tela de confirmacao de email enviado |
| `ResetPasswordTemplateSDK` | `src/modules/auth/templates/ResetPasswordTemplateSDK.tsx` | Template wrapper para formulario de nova senha |
| `ResetPasswordWithoutLayout` | `src/modules/auth/components/ResetPasswordWithoutLayout.tsx` | Formulario de definicao de nova senha + logica de steps |
| `ExpiredTokenWithoutLayout` | `src/modules/auth/components/ExpiredTokenWithoutLayout.tsx` | Tela de token expirado com opcao de reenvio |
| `AuthErrorChagingPassword` | `src/modules/auth/components/AuthErrorChagingPassword.tsx` | Tela de erro com opcoes de cancelar ou retry |
| `AuthPasswordChangedWithoutLayout` | `src/modules/auth/components/AuthPasswordChangedWithoutLayout.tsx` | Tela de sucesso apos troca de senha |
| `AuthPasswordTips` | `src/modules/auth/components/AuthPasswordTips.tsx` | Wrapper de dicas de validacao de senha |
| `PasswordValidationList` | `src/modules/auth/components/PasswordValidationList.tsx` | Lista visual de requisitos de senha com indicadores |

### Hooks

| Hook | Arquivo | Responsabilidade |
|------|---------|------------------|
| `useRequestPasswordChange` | `src/modules/auth/hooks/useRequestPasswordChange.ts` | Mutation de solicitacao de email de redefinicao (`POST auth/request-password-reset`) |
| `useChangePasswordAndSignIn` | `src/modules/auth/hooks/useChangePasswordAndSignIn.ts` | Mutation combinada: troca senha + auto sign-in |
| `useChangePassword` | `src/modules/auth/hooks/useChangePassword.ts` | Mutation de baixo nivel para troca de senha |
| `usePasswordValidationSchema` | `src/modules/auth/hooks/usePasswordValidationSchema.ts` | Schema Yup para validacao de senha (compartilhado com Sign Up) |
| `usePasswordMeetsCriteria` | `src/modules/auth/hooks/usePasswordMeetsCriteria.ts` | Retorna flags booleanas para cada requisito de senha (usado nas dicas visuais) |

### API Routes (Reset Password)

| Rota | Enum | Descricao |
|------|------|-----------|
| `auth/request-password-reset` | `PixwayAPIRoutes.REQUEST_PASSWORD_CHANGE` | Solicita email de redefinicao |
| `auth/reset-password` | `PixwayAPIRoutes.RESET_PASSWORD` | Redefine senha com token |

### App Routes (Reset Password)

| Rota | Enum | Descricao |
|------|------|-----------|
| `/auth/changePassword/request` | `PixwayAppRoutes.REQUEST_PASSWORD_CHANGE` | Pagina de solicitacao |
| `/auth/changePassword/newPassword` | `PixwayAppRoutes.RESET_PASSWORD` | Pagina de nova senha |

---

## Implementação Real (Next.js) — Reset Password

> As seções acima descrevem os componentes do SDK React original. Abaixo está o mapeamento para a implementação Next.js App Router atual.

### Componentes

| Componente Next.js | Arquivo | Equivalente SDK | Responsabilidade |
|---------------------|---------|-----------------|------------------|
| `PasswordResetRequestForm` | `src/components/auth/password-reset-request-form.tsx` | `RequestPasswordChangeWithoutLayout` + `PasswordChangeEmailSentWithoutLayout` | Formulário de email + tela de email enviado (steps: FORM → EMAIL_SENT). Cooldown 3 min. Email masking via `useEmailMask` |
| `PasswordResetForm` | `src/components/auth/password-reset-form.tsx` | `ResetPasswordWithoutLayout` | Formulário de nova senha + lógica de steps (FORM → PASSWORD_CHANGED → EXPIRED_PASSWORD → ERROR). Validação de token na montagem via `validateResetToken()` |
| `ExpiredTokenMessage` | `src/components/auth/expired-token-message.tsx` | `ExpiredTokenWithoutLayout` | Mensagem de token expirado com botão "Reenviar link" |
| `ErrorMessage` | `src/components/auth/error-message.tsx` | `AuthErrorChagingPassword` | Diálogo de erro com opções retry e cancel |
| `PasswordChangedSuccess` | `src/components/auth/password-changed-success.tsx` | `AuthPasswordChangedWithoutLayout` | Tela de sucesso com ícone e botão "Continuar" |
| `PasswordStrengthIndicator` | `src/components/auth/password-strength-indicator.tsx` | `AuthPasswordTips` + `PasswordValidationList` | Indicador visual de requisitos (min chars, uppercase, lowercase, number/special) usando CheckCircle2/XCircle |

### Hooks

| Hook Next.js | Arquivo | Equivalente SDK | Diferenças |
|--------------|---------|-----------------|------------|
| `useRequestPasswordChange()` | `src/hooks/use-request-password-change.ts` | `useRequestPasswordChange` | Mesmo conceito. Hardcoded: `verificationType: 'invisible'`. Callback URL: `${tenantBaseUrl}/auth/changePassword/newPassword` |
| `useChangePasswordAndSignIn()` | `src/hooks/use-change-password-and-sign-in.ts` | `useChangePasswordAndSignIn` | Provider: `CHANGE_PASSWORD_AND_SIGNIN`. Detecta token expirado na mensagem de erro |
| `usePasswordValidation()` | `src/hooks/use-password-validation.ts` | `usePasswordMeetsCriteria` | Retorna `{ hasMinLength, hasUppercase, hasLowercase, hasNumberOrSpecial, isValid }`. Memoizado |

> **Hooks SDK que NÃO existem no Next.js:**
> - `useChangePassword` — Não existe separado. Apenas `useChangePasswordAndSignIn` (combinado)
> - `usePasswordValidationSchema` — Substituído por validação Zod inline nos componentes

### Utilitários

| Arquivo | Responsabilidade |
|---------|------------------|
| `src/lib/utils/token-validation.ts` | `validateResetToken(token)` — Valida formato `{token};{expiration}` e verifica expiração via `date-fns` |

### Rotas

| Rota Next.js | Arquivo | Step |
|-------------|---------|------|
| `/auth/changePassword/request` | `src/app/auth/changePassword/request/page.tsx` | Formulário de solicitação de email |
| `/auth/changePassword/newPassword` | `src/app/auth/changePassword/newPassword/page.tsx` | Formulário de nova senha com validação de token |

---
---

# PART 4: Criacao de Tenant e Host (via API — sem frontend)

## Overview

Estas operacoes sao realizadas exclusivamente via API (Swagger ou chamadas HTTP diretas). Nao existe frontend para a criacao de tenants e hosts. Um **tenant** representa uma empresa/organizacao na plataforma W3block, e um **host** representa um dominio (hostname) associado a esse tenant.

O fluxo tipico e:

1. **Criar o Tenant** — registra a empresa com nome, documento e hostname principal
2. **(Opcional) Criar Hosts adicionais** — associa dominios extras ao tenant
3. **(Opcional) Configurar o Tenant** — define configuracoes como passwordless, KYC, etc.
4. **Processos automaticos (background)** — criacao de client (API key) e setup nos servicos Key e Commerce

**Autenticacao necessaria:** Todas as rotas exigem bearer token de um usuario com role `SuperAdmin` ou `Integration`.

**Swagger:** `https://pixwayid.w3block.io/docs`

---

## Step 1: Criacao do Tenant

### API Call

```
POST /tenant
```

**Guards:** Requer autenticacao com role `SuperAdmin` ou `Integration`.
**Roles permitidas:** `SuperAdmin`, `Integration`

### Request Body (CreateTenantDto)

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome da empresa/organizacao |
| `document` | string | Sim | Documento da empresa (ex: CNPJ, EIN) |
| `countryCode` | enum (CountryCodeEnum) | Sim | Codigo ISO 3166-1 alpha-3 do pais (ex: `BRA`, `USA`) |
| `hostname` | string | Sim | Hostname principal do tenant (ex: `meuprojeto.w3block.io`) |

**Exemplo de payload:**

```json
{
  "name": "Minha Empresa LTDA",
  "document": "12345678000199",
  "countryCode": "BRA",
  "hostname": "meuprojeto.w3block.io"
}
```

**Transformacoes no payload:**
- `hostname` e convertido para lowercase e caracteres nao-printaveis sao removidos

### Comportamento

1. Valida que nao existe outro tenant com o mesmo `document` + `countryCode`
2. Cria o tenant com os dados fornecidos
3. Cria automaticamente o host principal com o `hostname` fornecido

### Response (TenantEntityDto)

```json
{
  "id": "uuid-do-tenant",
  "name": "Minha Empresa LTDA",
  "document": "12345678000199",
  "countryCode": "BRA",
  "roles": ["APPLICATION"],
  "wallets": [],
  "info": {},
  "clientId": null,
  "createdAt": "2026-03-16T00:00:00.000Z",
  "updatedAt": "2026-03-16T00:00:00.000Z"
}
```

**Status:** `201 Created`

### Processos Automaticos Pos-Criacao

Apos a criacao do tenant, processos automaticos em background sao disparados:

1. **Criacao do Client (API Key):** Uma API key e credenciais sao geradas automaticamente para o tenant. O campo `clientId` do tenant sera preenchido quando o processo completar.
2. **Setup nos servicos:** O tenant e habilitado e configurado nos servicos internos da plataforma.

### Error States

| Erro | Causa | Resposta |
|------|-------|----------|
| `400 Bad Request` | Documento ja existe para o mesmo `countryCode` | `"document X already exists, for this country code Y"` |
| `400 Bad Request` | Campos obrigatorios faltando ou invalidos | Mensagem de validacao do class-validator |
| `401 Unauthorized` | Token ausente ou invalido | — |
| `403 Forbidden` | Role insuficiente (precisa ser SuperAdmin ou Integration) | — |

---

## Step 2: Criacao de Host Adicional

### API Call

```
POST /tenant-hosts/:tenantId
```

**Controller:** `src/modules/tenant/tenant-host/tenant-host.controller.ts`
**Service:** `src/modules/tenant/tenant-host/tenant-host.service.ts`

**Decorator:** `@ControllerTenantDependent('tenant-hosts')` — gera rota `tenant-hosts/:tenantId`
**Guard:** `TenantBearerGuard`
**Roles permitidas:** `Admin`

### Request Body (CreateTenantHostDto)

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `hostname` | string | Sim | Dominio a ser associado ao tenant (ex: `loja.meusite.com.br`) |
| `isMain` | boolean | Nao | Se `true`, este host se torna o principal (desmarca o anterior) |
| `paths` | object | Nao | Paths customizados para rotas do frontend |
| `paths.fillProfileForm` | string | Nao | Path para pagina de completar perfil (default: `/auth/complete-profile/`) |
| `paths.userSignIn` | string | Nao | Path para pagina de login (default: `/auth/signIn/`) |
| `paths.nftCertificate` | string | Nao | Path para certificado de NFT (default: `/token/{{contractAddress}}/{{chainId}}/{{tokenId}}`) |

**Nota:** O campo `tenantId` e preenchido automaticamente a partir do parametro da URL (`:tenantId`), nao precisa ser enviado no body.

**Exemplo de payload:**

```json
{
  "hostname": "loja.meusite.com.br",
  "isMain": false,
  "paths": {
    "fillProfileForm": "/perfil/completar",
    "userSignIn": "/login"
  }
}
```

**Transformacoes no payload:**
- `hostname` e convertido para lowercase e caracteres nao-printaveis sao removidos
- Valores de `paths` que forem URLs completas sao convertidos para apenas o pathname

### Logica de Criacao (TenantHostService.create)

1. **Inicia transacao** no banco de dados
2. **Valida hostname:** `TenantHostService.isValidHostname()` — verifica:
   - E uma string
   - Contem apenas caracteres `a-zA-Z0-9-.`
   - Maximo 253 caracteres
   - Cada label (entre pontos) tem maximo 63 caracteres
   - Labels nao comecam nem terminam com hifen
3. **Verifica duplicidade:** Se ja existe um host com o mesmo `hostname` para o mesmo `tenantId`, lanca erro
4. **Se `isMain: true`:** Desmarca todos os hosts existentes do tenant como `isMain: false`
5. **Salva TenantHostEntity** com os dados fornecidos
6. **Commit da transacao**

### Response (TenantHostResponseDto)

```json
{
  "id": "uuid-do-host",
  "hostname": "loja.meusite.com.br",
  "tenantId": "uuid-do-tenant",
  "isMain": false,
  "paths": {
    "fillProfileForm": "/perfil/completar",
    "userSignIn": "/login"
  },
  "routes": {
    "fillProfileForm": "https://loja.meusite.com.br/perfil/completar",
    "userSignIn": "https://loja.meusite.com.br/login",
    "nftCertificate": "https://loja.meusite.com.br/token/{{contractAddress}}/{{chainId}}/{{tokenId}}"
  }
}
```

**Status:** `201 Created`

**Nota sobre `routes`:** O campo `routes` e computado automaticamente (via `@AfterLoad`) combinando o `hostname` com os `paths` configurados (ou os defaults de `FrontendPathBase`).

### Endpoints Adicionais de Host

| Metodo | Rota | Descricao |
|--------|------|-----------|
| `GET /tenant-hosts/:tenantId` | Lista todos os hosts do tenant (paginado) |
| `GET /tenant-hosts/:tenantId/main-host` | Retorna o host principal do tenant |
| `GET /tenant-hosts/:tenantId/:id` | Retorna um host especifico por ID |
| `PATCH /tenant-hosts/:tenantId/:id` | Atualiza hostname, isMain e/ou paths de um host |

### Error States

| Erro | Causa | Resposta |
|------|-------|----------|
| `422 Unprocessable Entity` | Hostname invalido (caracteres proibidos, muito longo, etc.) | `"Invalid hostname: X"` |
| `422 Unprocessable Entity` | Hostname ja existe para o tenant | `"Hostname already exists: X"` |
| `401 Unauthorized` | Token ausente ou invalido | — |
| `403 Forbidden` | Role insuficiente (precisa ser Admin) | — |

---

## Step 3 (Opcional): Configurar o Tenant

### API Call

```
POST /tenant/configurations/:tenantId
```

**Controller:** `src/modules/tenant/tenant.controller.ts`
**Service:** `src/modules/tenant/tenant-configurations.service.ts`

**Roles permitidas:** `Admin`, `SuperAdmin`, `Integration`

### Request Body (TenantConfigurationsDto)

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `passwordless` | object | Nao | Configuracoes de autenticacao sem senha |
| `passwordless.enabled` | boolean | Sim (se passwordless) | Habilita modo passwordless |
| `passwordless.sendEmailWhenRegister` | boolean | Sim (se passwordless) | Se envia email ao registrar em modo passwordless |
| `passwordless.expirationInMinutes` | number | Nao | Tempo de expiracao do token passwordless (minutos) |
| `signUp` | object | Nao | Configuracoes de signup |
| `signUp.requireReferrer` | boolean | Nao | Se exige codigo de referrer para cadastro |
| `signUp.referrerBlacklistId` | UUID | Nao | ID da blacklist de referrers |
| `kyc` | object | Nao | Configuracoes de KYC |
| `kyc.isUniqueCPF` | boolean | Nao | Se CPF deve ser unico entre usuarios do tenant |
| `clearSale` | object | Nao | Configuracoes de integracao ClearSale (consulte a documentacao para detalhes) |

**Exemplo de payload:**

```json
{
  "passwordless": {
    "enabled": false,
    "sendEmailWhenRegister": true
  },
  "signUp": {
    "requireReferrer": false
  },
  "kyc": {
    "isUniqueCPF": true
  }
}
```

**Nota:** Este endpoint faz **upsert** — cria as configuracoes se nao existirem, ou atualiza se ja existirem.

### Response (TenantConfigurationsResponseDto)

**Status:** `200 OK`

---

## Sequencia Completa de Setup de Tenant (via API)

```
1. POST /tenant                                → Cria tenant + host principal
   ↳ (automatico) Criacao de TenantClient      → API key gerada via Pixchain
   ↳ (automatico) Setup em Key e Commerce      → Tenant habilitado nos servicos

2. POST /tenant-hosts/:tenantId                → (Opcional) Adiciona hosts extras
3. POST /tenant/configurations/:tenantId       → (Opcional) Configura passwordless, KYC, etc.
```

### Endpoints Uteis para Consulta

| Metodo | Rota | Autenticacao | Descricao |
|--------|------|--------------|-----------|
| `GET /tenant/:tenantId` | Bearer (SuperAdmin/Admin/Integration) | Retorna dados do tenant |
| `GET /tenant` | Bearer (SuperAdmin/Integration) | Lista todos os tenants (paginado) |
| `GET /tenant/client/:tenantId` | Bearer (Integration) | Retorna dados do client do tenant |
| `GET /tenant/configurations/:tenantId` | Bearer (Admin/SuperAdmin/Integration) | Retorna configuracoes do tenant |
| `GET /public-tenant/by-hostname?hostname=X` | Publica | Retorna dados publicos do tenant por hostname |
| `GET /public-tenant/by-id?tenantId=X` | Publica | Retorna dados publicos do tenant por ID |
| `PUT /tenant/:tenantId` | Bearer (SuperAdmin/Integration) | Atualiza name, document, countryCode |
| `PUT /tenant/profile/:tenantId` | Bearer (SuperAdmin/Integration/Admin) | Atualiza name e logo do tenant |
| `DELETE /tenant/:tenantId` | Bearer (SuperAdmin/Integration) | Soft delete do tenant |

### Recursos Criados

| Recurso | Momento | Descricao |
|---------|---------|-----------|
| Tenant | POST /tenant | Empresa/organizacao principal |
| Host | POST /tenant (host principal) ou POST /tenant-hosts (extras) | Dominio associado ao tenant |
| Client (API Key) | Automatico (background) | Credenciais de API do tenant |
| Configuracoes | POST /tenant/configurations | Passwordless, KYC, signup configs |
