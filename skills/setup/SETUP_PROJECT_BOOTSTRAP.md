# Setup Project Bootstrap

Como configurar um projeto Next.js 15 para consumir a plataforma W3block usando o SDK (`@w3block/w3block-ui-sdk`) e NextAuth.

---

## 1. Instalacao de Pacotes

```bash
# Pacotes principais
npm install @w3block/w3block-ui-sdk @w3block/sdk-id next-auth axios

# TanStack React Query (obrigatorio — SDK depende internamente)
npm install @tanstack/react-query

# Dependencias auxiliares usadas pelo frontend
npm install react-use logrocket @bugsnag/js jwt-decode

# Tipos (se nao incluidos)
npm install -D @types/node
```

> **Nota:** O `@w3block/w3block-ui-sdk` exporta hooks, providers e componentes. O `@w3block/sdk-id` exporta tipos como `UserRoleEnum`.

---

## 2. Variaveis de Ambiente (.env.local)

Crie `.env.local` na raiz do projeto com todas as variaveis necessarias:

```env
# === App Config ===
APP_NAME=my-w3block-app
PORT=3000
NEXT_PUBLIC_ENVIROMENT=development
NEXT_PUBLIC_BASE_URL=http://localhost:3000/

# === URLs das APIs (6 microservicos) ===
NEXT_PUBLIC_PIXWAY_ID_API_URL=https://id.w3block.io/
NEXT_PUBLIC_PIXWAY_KEY_API_URL=https://api.w3block.io/
NEXT_PUBLIC_COMMERCE_API_URL=https://commerce.w3block.io/
NEXT_PUBLIC_PDF_API_URL=https://pdf.w3block.io/
NEXT_PUBLIC_POLL_API_URL=https://survey.w3block.io/
NEXT_PUBLIC_PASS_API_URL=https://pass.w3block.io/

# === NextAuth ===
NEXTAUTH_URL=http://localhost:3000/
NEXT_PUBLIC_NEXTAUTH_SECRET=<gere-um-secret-seguro>

# === Opcional ===
NEXT_PUBLIC_BUILD_PATH=/
NEXT_PUBLIC_SESSION_EXPIRES_IN_SECONDS=7200
NEXT_UTM_EXPIRATION=3600000
```

### Mapeamento ENV -> Microservico

| Variavel de ambiente | Microservico | Enum `W3blockAPI` |
|---------------------|--------------|-------------------|
| `NEXT_PUBLIC_PIXWAY_ID_API_URL` | Identity (ID) | `W3blockAPI.ID` |
| `NEXT_PUBLIC_PIXWAY_KEY_API_URL` | Key (Blockchain) | `W3blockAPI.KEY` |
| `NEXT_PUBLIC_COMMERCE_API_URL` | Commerce | `W3blockAPI.COMMERCE` |
| `NEXT_PUBLIC_PDF_API_URL` | PDF (servico auxiliar) | N/A (passado direto) |
| `NEXT_PUBLIC_POLL_API_URL` | Poll | `W3blockAPI.POLL` |
| `NEXT_PUBLIC_PASS_API_URL` | Pass | `W3blockAPI.PASS` |

### URLs de Producao

| Microservico | Producao |
|--------------|----------|
| Identity (ID) | `https://id.w3block.io/` |
| Key | `https://api.w3block.io/` |
| Commerce | `https://commerce.w3block.io/` |
| PDF | `https://pdf.w3block.io/` |
| Poll | `https://survey.w3block.io/` |
| Pass | `https://pass.w3block.io/` |

---

## 3. Provider Tree

O frontend usa uma arvore de providers aninhados. A ordem importa.

### 3.1 — QueryClient (TanStack React Query)

O SDK aceita um `QueryClient` externo. Se nao receber, cria um interno com defaults:

```typescript
// src/lib/queryClient.ts
'use client';
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutos
      retry: 1,
    },
  },
});
```

> **Importante:** Sempre passe seu proprio `QueryClient` para compartilhar cache entre o SDK e seu codigo.

### 3.2 — Montando o Provider Tree (W3blockUISDKProvider)

Abaixo esta um exemplo de como montar o provider tree no seu projeto. O provider raiz combina NextAuth session, SDK config, auth adapter, router adapter e mais:

```typescript
// src/providers/W3blockUISDKProvider.tsx
'use client';
import { ReactNode, Suspense, useMemo } from 'react';
import { signIn, signOut, useSession } from 'next-auth/react';
import { useRouter, useParams, useSearchParams } from 'next/navigation';
import { QueryClient } from '@tanstack/react-query';

import {
  IW3blockAuthenticationContext,
  W3blockAuthenticationProvider,
  PixwaySDKNextRouterAdapter,
  PixwaySessionContext,
  UiSDKUtmProvider,
  W3blockUISDKGeneralConfigProvider,
  OnboardProvider,
} from '@w3block/w3block-ui-sdk';
import { UserWalletsProvider } from '@w3block/w3block-ui-sdk/shared';
import { CredentialProviderName } from '../configs/nextAuthConfig';

interface Props {
  children: ReactNode;
  client: QueryClient;
  companyId: string;
  tenantName?: string;
  logoUrl?: string;
  theme?: any;
}

export default function W3blockUISdkProvider({
  children, client, companyId, tenantName, logoUrl, theme,
}: Props) {
  const session = useSession();
  const router = useRouter();
  const queryParams = useParams();
  const searchParams = useSearchParams();
  const queryStringParams = Object.fromEntries(searchParams.entries());
  const hostname = typeof window !== 'undefined' ? window.location.hostname : '';

  // Monta o objeto de autenticacao que mapeia cada acao do SDK
  // para o CredentialsProvider correspondente do NextAuth
  const authValue = useMemo<IW3blockAuthenticationContext>(
    () => ({
      signIn: (payload: any) =>
        signIn(CredentialProviderName.SIGNIN_WITH_COMPANY_ID, {
          ...payload, redirect: false, callbackUrl: undefined,
        }),
      changePasswordAndSignIn: (payload: any) =>
        signIn(CredentialProviderName.CHANGE_PASSWORD_AND_SIGNIN, {
          ...payload, redirect: false, callbackUrl: undefined,
        }),
      signInWithCode: (payload: any) =>
        signIn(CredentialProviderName.SIGN_IN_WITH_CODE, {
          ...payload, redirect: false, callbackUrl: undefined,
        }),
      signInAfterSignUp: (payload: any) =>
        signIn(CredentialProviderName.SIGNIN_AFTER_SIGNUP, {
          ...payload, redirect: false, callbackUrl: undefined,
        }),
      signOut: (payload: any) =>
        signOut({ redirect: false, callbackUrl: payload?.callbackUrl ?? undefined }),
      signInWithGoogle: (payload: any) =>
        signIn(CredentialProviderName.SIGNIN_WITH_GOOGLE, {
          ...payload, redirect: false,
        }),
      signInWithApple: (payload: any) =>
        signIn(CredentialProviderName.SIGNIN_WITH_APPLE, {
          ...payload, redirect: false,
        }),
    }),
    []
  );

  return (
    <Suspense fallback={<div>Loading...</div>}>
      <PixwaySessionContext.Provider value={session}>
        <W3blockUISDKGeneralConfigProvider
          client={client}
          name={tenantName ?? ''}
          connectProxyPass={''}
          isProduction={process.env.NEXT_PUBLIC_ENVIROMENT === 'production'}
          api={{
            idUrl: process.env.NEXT_PUBLIC_PIXWAY_ID_API_URL ?? '',
            keyUrl: process.env.NEXT_PUBLIC_PIXWAY_KEY_API_URL ?? '',
            commerceUrl: process.env.NEXT_PUBLIC_COMMERCE_API_URL ?? '',
            pdfUrl: process.env.NEXT_PUBLIC_PDF_API_URL ?? '',
            pollUrl: process.env.NEXT_PUBLIC_POLL_API_URL ?? '',
            passUrl: process.env.NEXT_PUBLIC_PASS_API_URL ?? '',
          }}
          companyId={companyId}
          locale={'pt-BR'}
          logoUrl={logoUrl}
          appBaseUrl={'https://' + hostname}
        >
          <W3blockAuthenticationProvider value={authValue}>
            <PixwaySDKNextRouterAdapter
              router={{
                ...router,
                query: { ...queryParams, ...queryStringParams },
              } as any}
            >
              <UserWalletsProvider>
                <UiSDKUtmProvider expiration={3600000}>
                  <OnboardProvider theme={theme}>
                    {children}
                  </OnboardProvider>
                </UiSDKUtmProvider>
              </UserWalletsProvider>
            </PixwaySDKNextRouterAdapter>
          </W3blockAuthenticationProvider>
        </W3blockUISDKGeneralConfigProvider>
      </PixwaySessionContext.Provider>
    </Suspense>
  );
}
```

**Pontos importantes:**

- `PixwaySessionContext.Provider` recebe a session do NextAuth — sem ele, hooks do SDK nao tem acesso ao token.
- `W3blockAuthenticationProvider` mapeia cada acao de auth do SDK para o `CredentialsProvider` correspondente no NextAuth (veja secao 4).
- `PixwaySDKNextRouterAdapter` adapta o router do Next.js 15 para o formato esperado pelo SDK.
- `UserWalletsProvider`, `UiSDKUtmProvider` e `OnboardProvider` sao opcionais dependendo do uso.

### Props do W3blockUISDKGeneralConfigProvider

| Prop | Tipo | Obrigatorio | Descricao |
|------|------|-------------|-----------|
| `client` | `QueryClient` | Sim | Instancia do TanStack React Query |
| `api` | `{ idUrl, keyUrl, commerceUrl, pdfUrl, pollUrl?, passUrl }` | Sim | URLs dos microservicos |
| `companyId` | `string` | Sim | UUID do tenant |
| `locale` | `PixwayUISdkLocale` | Sim | `'pt-BR'`, `'en'`, etc. |
| `isProduction` | `boolean` | Sim | Controla comportamentos de prod vs dev |
| `appBaseUrl` | `string` | Sim | URL base da aplicacao (ex: `https://meusite.com`) |
| `logoUrl` | `string \| undefined` | Nao | URL do logo do tenant |
| `connectProxyPass` | `string` | Nao | Proxy pass (default `''`) |
| `name` | `string` | Nao | Nome do tenant |
| `logError` | `(error, extra?) => void` | Nao | Callback de erro (Bugsnag, Sentry, etc.) |
| `gtag` | `(event, params?) => void` | Nao | Callback do Google Analytics |

---

## 4. NextAuth Route Handler

### 4.1 — Arquivo de rota

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import NextAuth, { AuthOptions } from 'next-auth';
import getNextAuthConfig from '../../functions/onGetNextAuthConfig';

export const authOptions = getNextAuthConfig({
  baseURL: process.env.NEXT_PUBLIC_PIXWAY_ID_API_URL ?? '',
  secret: process.env.NEXT_PUBLIC_NEXTAUTH_SECRET ?? '',
});

const handler = NextAuth(authOptions as AuthOptions);
export { handler as GET, handler as POST };
```

### 4.2 — CredentialProviderName Enum

O NextAuth e configurado com multiplos `CredentialsProvider`, cada um mapeado a um fluxo de autenticacao diferente. O enum abaixo define os nomes de cada provider:

```typescript
// src/configs/nextAuthConfig.ts

export enum CredentialProviderName {
  SIGNIN_WITH_COMPANY_ID = 'signInWithCompanyId',
  CHANGE_PASSWORD_AND_SIGNIN = 'changePasswordAndSignIn',
  SIGN_IN_WITH_CODE = 'signInWithCode',
  SIGNIN_AFTER_SIGNUP = 'signInAfterSignUp',
  SIGNIN_WITH_GOOGLE = 'signInWithGoogle',
  SIGNIN_WITH_APPLE = 'signInWithApple',
}
```

### 4.3 — Padrao dos CredentialsProviders

Todos os providers seguem o mesmo padrao. Abaixo um exemplo com `signInWithCompanyId` (login com email/senha):

```typescript
// src/app/api/functions/onGetNextAuthConfig.ts
import CredentialsProvider from 'next-auth/providers/credentials';
import jwtDecode from 'jwt-decode';

export default function getNextAuthConfig({ baseURL, secret }: { baseURL: string; secret: string }) {
  return {
    secret,
    providers: [
      // Exemplo: login com email/senha
      CredentialsProvider({
        id: CredentialProviderName.SIGNIN_WITH_COMPANY_ID,
        credentials: {
          email: { type: 'email' },
          password: { type: 'password' },
          companyId: { type: 'string' },
        },
        authorize: async (payload) => {
          const response = await fetch(`${baseURL}/auth/signin`, {
            method: 'POST',
            body: JSON.stringify({
              tenantId: payload?.companyId ?? '',
              email: payload?.email ?? '',
              password: payload?.password ?? '',
            }),
            headers: { 'Content-type': 'application/json' },
          });
          const responseAsJSON = await response.json();
          if (responseAsJSON.statusCode >= 300) {
            throw new Error(responseAsJSON.message);
          }
          return mapSignInReponseToSessionUser(responseAsJSON);
        },
      }),

      // Os demais providers seguem o mesmo padrao, mudando:
      // - id: o valor do enum CredentialProviderName correspondente
      // - credentials: os campos necessarios para o fluxo
      // - authorize: a rota da API chamada
      //
      // Rotas da API por provider:
      //   SIGNIN_WITH_COMPANY_ID       -> POST /auth/signin
      //   CHANGE_PASSWORD_AND_SIGNIN   -> POST /auth/reset-password
      //   SIGN_IN_WITH_CODE            -> POST /auth/signin/code
      //   SIGNIN_AFTER_SIGNUP          -> POST /auth/signup
      //   SIGNIN_WITH_GOOGLE           -> GET  /auth/{companyId}/signin/google/code?code=...
      //   SIGNIN_WITH_APPLE            -> GET  /auth/{companyId}/signin/apple/code?code=...
    ],

    // ... callbacks (veja secao 4.4)
  };
}
```

> **Nota:** Cada `authorize` faz fetch na API Identity (`baseURL`), recebe um `SignInResponse` e o converte via `mapSignInReponseToSessionUser` (veja secao 4.6).

### 4.4 — Callbacks JWT e Session

Os callbacks controlam como tokens sao persistidos e renovados:

```typescript
callbacks: {
  async jwt({ token, user, account }) {
    // Login inicial — salva tokens
    if (account && user?.accessToken) {
      const apiToken = user?.accessToken as string;
      return {
        sub: token?.sub,
        picture: token?.picture,
        name: token?.name,
        email: token?.email,
        error: undefined,
        accessToken: apiToken,
        accessTokenExpires: getTokenExpires(apiToken, -BEFORE_TOKEN_EXPIRES),
        refreshToken: user?.refreshToken,
        user,
      };
    }
    // Token ainda valido — retorna sem alterar
    if (Date.now() < (token.accessTokenExpires as number)) {
      return token;
    }
    // Token expirado — tenta refresh
    return refreshAccessToken(token, baseURL);
  },
  async session({ session, token }) {
    const user: User = token.user as User;
    session.user = user;
    session.id = user.id;
    session.sub = token.sub;
    session.accessToken = token.accessToken;
    session.refreshToken = token.refreshToken;
    session.error = token.error;
    return session;
  },
},
pages: {
  signIn: '/auth/signIn',
  newUser: '/auth/signUp',
  verifyRequest: '/auth/mail-confirmation/sign-up',
  signOut: '/auth',
},
```

### 4.5 — Refresh Token Automatico

O refresh token e renovado automaticamente quando o token expira. A renovacao acontece na metade do tempo de vida do token:

```typescript
const tokenMaxAgeInSeconds =
  process.env.NEXT_PUBLIC_ENVIROMENT != 'development'
    ? process.env.NEXT_PUBLIC_SESSION_EXPIRES_IN_SECONDS
      ? parseInt(process.env.NEXT_PUBLIC_SESSION_EXPIRES_IN_SECONDS)
      : 120 * 60  // 2 horas
    : 120 * 60;

const BEFORE_TOKEN_EXPIRES = tokenMaxAgeInSeconds / 2; // renova na metade

async function refreshAccessToken(token, baseURL) {
  const response = await fetch(`${baseURL}/auth/refresh-token`, {
    method: 'POST',
    body: JSON.stringify({ refreshToken: token.refreshToken ?? '' }),
    headers: {
      'Content-type': 'application/json',
      Authorization: `Bearer ${token.accessToken ?? ''}`,
    },
  });
  const { token: tokenResponse, refreshToken } = await response.json();
  return {
    ...token,
    accessToken: tokenResponse,
    refreshToken: refreshToken || token.refreshToken,
    accessTokenExpires: getTokenExpires(tokenResponse, -BEFORE_TOKEN_EXPIRES),
  };
}
```

### 4.6 — Tipos e funcoes auxiliares

```typescript
interface SignInResponse {
  token: string;
  refreshToken: string;
  data: {
    sub: string;
    email: string;
    roles: Array<UserRoleEnum>;
    name: string;
    verified: boolean;
    companyId?: string;
  };
}

interface SessionUser extends User {
  accessToken: string;
  refreshToken: string;
  roles: Array<UserRoleEnum>;
  companyId?: string;
  email?: string;
}

const mapSignInReponseToSessionUser = (response: SignInResponse): SessionUser => {
  const { data, token: accessToken, refreshToken } = response;
  return {
    accessToken,
    refreshToken,
    id: data.sub,
    email: data.email,
    roles: data.roles,
    name: data.name,
    companyId: data.companyId,
  };
};

function getTokenExpires(token: string, threshold = 0): number {
  const { exp } = jwtDecode(token) as { exp: number };
  return +exp * 1000 + threshold;
}
```

---

## 5. QueryClient Setup

### 5.1 — Criacao do QueryClient

```typescript
// src/lib/queryClient.ts
'use client';
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutos — mesmo default do SDK
      retry: 1,
    },
  },
});
```

### 5.2 — Como o SDK usa o QueryClient

O SDK aceita o `QueryClient` via prop `client` do `W3blockUISDKGeneralConfigProvider`:

```typescript
<W3blockUISDKGeneralConfigProvider client={queryClient} ...>
```

Se nenhum `QueryClient` for passado, o SDK cria um interno automaticamente e emite um warning no console. Sempre passe seu proprio `QueryClient` para compartilhar cache entre o SDK e seu codigo.

---

## 6. Estrutura de Diretorio Recomendada

```
src/
├── app/
│   ├── api/
│   │   ├── auth/
│   │   │   └── [...nextauth]/
│   │   │       └── route.ts              ← NextAuth route handler
│   │   └── functions/
│   │       └── onGetNextAuthConfig.ts     ← Config NextAuth (providers, callbacks, refresh)
│   ├── layout.tsx                         ← Root layout com SessionProvider
│   └── [[...page]]/
│       └── page.tsx                       ← Paginas dinamicas
├── configs/
│   └── nextAuthConfig.ts                  ← Enum CredentialProviderName + interfaces
├── providers/
│   └── W3blockUISDKProvider.tsx            ← Provider tree (secao 3.2)
├── lib/
│   └── queryClient.ts                     ← QueryClient compartilhado
└── .env.local                             ← Variaveis de ambiente
```

---

## 7. Checklist de Validacao

| # | Verificacao | Como testar |
|---|-------------|-------------|
| 1 | `.env.local` tem as 6 URLs de API | Verificar que nao ha URLs vazias |
| 2 | `NEXT_PUBLIC_NEXTAUTH_SECRET` esta definido | NextAuth falha silenciosamente sem secret |
| 3 | `NEXTAUTH_URL` aponta para o host correto | Deve ser `http://localhost:3000/` em dev |
| 4 | Route handler existe em `/app/api/auth/[...nextauth]/route.ts` | Testar `GET /api/auth/providers` |
| 5 | `QueryClient` e passado ao `W3blockUISDKGeneralConfigProvider` | Se nao, aparece warning no console |
| 6 | `SessionProvider` do NextAuth wrapa toda a app | Sem ele, `useSession()` retorna undefined |
| 7 | `PixwaySessionContext.Provider` recebe a session | Sem ele, hooks do SDK nao tem token |
| 8 | `companyId` esta correto para o tenant | Todos os hooks usam `companyId` nas rotas |
