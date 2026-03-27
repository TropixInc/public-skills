# Setup API Patterns

Padroes de comunicacao com a API W3block. Este documento cobre como o SDK faz chamadas HTTP, gerencia autenticacao e trata erros.

---

## 1. W3blockAPI Enum e Mapeamento de URLs

### 1.1 — Enum dos microservicos

```typescript
export enum W3blockAPI {
  ID,       // 0 — Identity: auth, usuarios, KYC
  KEY,      // 1 — Blockchain: NFTs, wallets, tokens
  COMMERCE, // 2 — Commerce: produtos, pedidos, pagamentos
  POLL,     // 3 — Poll: enquetes
  PASS,     // 4 — Pass: token passes, beneficios
}
```

### 1.2 — Como o SDK resolve URLs

O SDK mapeia automaticamente cada valor de `W3blockAPI` para a URL base do microservico correspondente. As URLs sao configuradas via variaveis de ambiente (`NEXT_PUBLIC_PIXWAY_ID_API_URL`, `NEXT_PUBLIC_PIXWAY_KEY_API_URL`, etc.) e injetadas no contexto pelo `W3blockApiProvider`.

Voce nao precisa resolver URLs manualmente — basta passar o enum para `useAxios()`:

```typescript
import { useAxios, W3blockAPI } from '@w3block/w3block-ui-sdk';

// O SDK resolve a URL base automaticamente
const axios = useAxios(W3blockAPI.COMMERCE);
```

### 1.3 — Interface de URLs disponveis

```typescript
interface W3blockAPIContextInterface {
  w3blockKeyAPIUrl: string;
  w3blockIdAPIUrl: string;
  w3blockCommerceAPIUrl: string;
  w3blockPdfAPIUrl: string;
  w3BlockPollApiUrl: string;
  w3BlockPassApiUrl: string;
}
```

> **Nota:** As URLs sao injetadas pelo `W3blockApiProvider` que recebe as props do `W3blockUISDKGeneralConfigProvider`.

---

## 2. useAxios(type) — Hook Principal de HTTP

O `useAxios` e o hook central para todas as chamadas HTTP. Retorna uma instancia Axios configurada (publica ou autenticada) para o microservico especificado.

### 2.1 — Import e assinatura

```typescript
import { useAxios, W3blockAPI } from '@w3block/w3block-ui-sdk';

const axios = useAxios(type: W3blockAPI);
```

### 2.2 — Comportamento

| Condicao | Retorno | Header Authorization |
|----------|---------|---------------------|
| Token existe e valido | Instancia Axios autenticada | `Bearer <token>` (interceptor automatico) |
| Token nao existe | Instancia Axios publica | Nenhum |
| Token existe mas expirado | Faz `signOut` e redireciona para login | N/A |

O hook internamente:
- Detecta se o usuario esta autenticado (via token do contexto de sessao)
- Se autenticado, adiciona um interceptor que insere `Authorization: Bearer <token>` em cada request
- Se o token estiver expirado, faz logout automatico e redireciona para a tela de login

### 2.3 — Uso tipico

```typescript
// Em qualquer hook de dominio:
const axios = useAxios(W3blockAPI.COMMERCE);

// GET
const response = await axios.get(
  PixwayAPIRoutes.PRODUCT_BY_SLUG
    .replace('{companyId}', companyId)
    .replace('{slug}', slug)
);

// POST
const response = await axios.post(
  PixwayAPIRoutes.CREATE_ORDER.replace('{companyId}', companyId),
  bodyData
);

// PATCH
const response = await axios.patch(
  PixwayAPIRoutes.PATCH_PROFILE.replace('{companyId}', companyId),
  { name: 'Novo Nome' }
);
```

---

## 3. Padrao de Substituicao de URL Templates

As rotas em `PixwayAPIRoutes` usam placeholders `{companyId}`, `{tenantId}`, `{userId}`, etc. A substituicao e feita manualmente com `.replace()`:

```typescript
// Padrao:
PixwayAPIRoutes.SOME_ROUTE
  .replace('{companyId}', companyId)
  .replace('{tenantId}', tenantId)
  .replace('{userId}', userId)
```

### Exemplos de rotas comuns

| Enum | Template | Microservico |
|------|----------|--------------|
| `GET_PROFILE` | `users/profile` | ID |
| `SIGN_IN` | `auth/signin` | ID |
| `REFRESH_TOKEN` | `auth/refresh-token` | ID |
| `PRODUCT_BY_SLUG` | `/companies/{companyId}/products/get-by-slug/{slug}` | COMMERCE |
| `CREATE_ORDER` | `/companies/{companyId}/orders` | COMMERCE |
| `ORDER_PREVIEW` | `/companies/{companyId}/orders/preview` | COMMERCE |
| `TOKEN_PASS` | `/token-passes/tenants/{tenantId}` | PASS |
| `PASS_BENEFIT` | `/token-pass-benefits/tenants/{tenantId}` | PASS |
| `GET_POLL_BY_SLUG` | `/polls/{companyId}/slug-details/{slug}` | POLL |
| `TOKEN_COLLECTIONS` | `{companyId}/token-collections` | KEY |
| `GET_WALLETS` | `/users/{companyId}/wallets/{userId}` | ID |

---

## 4. usePrivateQuery — Queries Autenticadas

Wrapper de `useQuery` do TanStack React Query que **so executa quando o usuario esta autenticado** (token existe).

### 4.1 — Import e assinatura

```typescript
import { usePrivateQuery } from '@w3block/w3block-ui-sdk';

const result = usePrivateQuery(
  queryKey: QueryKey,
  queryFn: QueryFunction,
  options?: QueryConfig   // mesmas opcoes do useQuery, sem queryKey/queryFn
);
```

### 4.2 — Comportamento

- Se `token` nao existe -> query **nao executa** (`enabled: false`)
- Se `options.enabled` for passado -> faz AND logico com `Boolean(token)`
- Retorno identico ao `useQuery` do TanStack

### 4.3 — Exemplo de uso

```typescript
import { usePrivateQuery, useAxios, W3blockAPI, PixwayAPIRoutes, useCompanyConfig, handleNetworkException } from '@w3block/w3block-ui-sdk';

export const useMeuRecurso = (id: string) => {
  const axios = useAxios(W3blockAPI.COMMERCE);
  const { companyId } = useCompanyConfig();

  return usePrivateQuery(
    ['meu-recurso', companyId, id],
    async () => {
      try {
        const url = PixwayAPIRoutes.MINHA_ROTA
          .replace('{companyId}', companyId)
          .replace('{id}', id);
        const { data } = await axios.get(url);
        return data;
      } catch (err) {
        throw handleNetworkException(err);
      }
    },
    {
      enabled: !!id,
      retry: 1,
      refetchOnWindowFocus: false,
      refetchOnMount: false,
    }
  );
};
```

---

## 5. usePublicQuery — Queries Publicas

Wrapper fino de `useQuery` para endpoints publicos (sem autenticacao).

### 5.1 — Import e assinatura

```typescript
import { usePublicQuery } from '@w3block/w3block-ui-sdk';

const result = usePublicQuery(
  queryKey: QueryKey,
  queryFn: QueryFunction,
  options?: QueryConfig   // mesmas opcoes do useQuery, sem queryKey/queryFn
);
```

### 5.2 — Diferenca vs usePrivateQuery

| Aspecto | `usePrivateQuery` | `usePublicQuery` |
|---------|-------------------|------------------|
| Requer token | Sim (auto-desabilita sem token) | Nao |
| `enabled` default | `Boolean(token)` | `true` (padrao do React Query) |
| Uso tipico | Perfil, pedidos, wallets | Produtos, tenant info, FAQ |

### 5.3 — Exemplo de uso

```typescript
import { usePublicQuery, useAxios, W3blockAPI, PixwayAPIRoutes, useCompanyConfig } from '@w3block/w3block-ui-sdk';

export const useProdutoPorSlug = (slug: string) => {
  const axios = useAxios(W3blockAPI.COMMERCE);
  const { companyId } = useCompanyConfig();

  return usePublicQuery(
    ['produto', companyId, slug],
    async () => {
      const url = PixwayAPIRoutes.PRODUCT_BY_SLUG
        .replace('{companyId}', companyId)
        .replace('{slug}', slug);
      const { data } = await axios.get(url);
      return data;
    },
    {
      enabled: !!slug,
    }
  );
};
```

---

## 6. Padrao useMutation

O SDK usa `useMutation` do TanStack React Query para operacoes de escrita (POST, PATCH, DELETE).

### 6.1 — Padrao com useAxios

```typescript
import { useMutation } from '@tanstack/react-query';
import { useAxios, useCompanyConfig, W3blockAPI, PixwayAPIRoutes } from '@w3block/w3block-ui-sdk';

export const usePatchProfile = () => {
  const axios = useAxios(W3blockAPI.ID);
  const { companyId } = useCompanyConfig();

  const _patchProfile = (name: string) => {
    return axios.patch(
      PixwayAPIRoutes.PATCH_PROFILE.replace('{companyId}', companyId),
      { name }
    );
  };

  const patchProfile = useMutation(_patchProfile);
  return patchProfile;
};
```

### 6.2 — Padrao geral

```typescript
export const useMinhaOperacao = () => {
  const axios = useAxios(W3blockAPI.COMMERCE); // ou ID, KEY, POLL, PASS
  const { companyId } = useCompanyConfig();

  const executar = (dados: MeuTipo) => {
    return axios.post(
      PixwayAPIRoutes.MINHA_ROTA.replace('{companyId}', companyId),
      dados
    );
  };

  return useMutation(executar);
};

// No componente:
const { mutate, mutateAsync, isLoading, error } = useMinhaOperacao();
mutate(dados);
// ou
const result = await mutateAsync(dados);
```

---

## 7. useCompanyConfig() — Configuracao do Tenant

Retorna dados do tenant atual do contexto.

### 7.1 — Import e uso

```typescript
import { useCompanyConfig } from '@w3block/w3block-ui-sdk';

const { companyId, appBaseUrl, name } = useCompanyConfig();
```

### 7.2 — Interface retornada

```typescript
interface IW3blockUISDKGereralConfigContext {
  companyId: string;         // UUID do tenant
  logoUrl: string | undefined;
  appBaseUrl: string;        // ex: "https://meusite.com"
  connectProxyPass: string;  // proxy pass (default: "/")
  name?: string;             // nome do tenant
}
```

### 7.3 — Uso tipico

```typescript
const { companyId, appBaseUrl, name } = useCompanyConfig();

// Usar companyId em rotas:
const url = PixwayAPIRoutes.CREATE_ORDER.replace('{companyId}', companyId);
```

---

## 8. usePixwaySession() — Sessao NextAuth

Retorna a sessao NextAuth do contexto `PixwaySessionContext`.

### 8.1 — Import e uso

```typescript
import { usePixwaySession } from '@w3block/w3block-ui-sdk';

const session = usePixwaySession();
```

### 8.2 — Tipo retornado

Retorna `SessionContextValue` do NextAuth:

```typescript
{
  data: Session | null;    // session.user, session.accessToken, etc.
  status: 'loading' | 'authenticated' | 'unauthenticated';
  update: (data?) => Promise<Session | null>;
}
```

### 8.3 — Uso tipico

```typescript
const session = usePixwaySession();

if (session.status === 'authenticated') {
  console.log(session.data?.accessToken);
  console.log(session.data?.user?.email);
}
```

### 8.4 — useToken() e useSessionUser() (derivados)

Hooks derivados de `usePixwaySession` que facilitam o acesso a dados especificos da sessao:

**useToken** — retorna apenas o access token da sessao:

```typescript
import { useToken } from '@w3block/w3block-ui-sdk';

const token = useToken(); // string (access token) ou ''
```

**useSessionUser** — retorna o usuario da sessao com tipagem completa:

```typescript
import { useSessionUser } from '@w3block/w3block-ui-sdk';

const user = useSessionUser(); // SessionUser | null
```

Tipo retornado por `useSessionUser`:

```typescript
interface SessionUser extends User {
  accessToken: string;
  refreshToken: string;
  roles: Array<UserRoleEnum>;
  companyId?: string;
  email?: string;
}
```

---

## 9. useProfile() — Perfil do Usuario

Hook que busca o perfil do usuario logado via Identity API.

### 9.1 — Import e uso

```typescript
import { useProfile } from '@w3block/w3block-ui-sdk';

const { data, isLoading, error } = useProfile();
```

O hook usa `usePrivateQuery` internamente, portanto so executa quando o usuario esta autenticado. Comportamentos automaticos:
- Se o perfil nao for encontrado ou ocorrer erro, redireciona para a tela de login
- Se o usuario nao estiver verificado (e passwordless nao estiver habilitado), redireciona para a tela de verificacao
- `retry: 1`, `refetchOnWindowFocus: false`, `refetchOnMount: false`

### 9.2 — Variante sem redirect

```typescript
import { useProfileWithouRedirect } from '@w3block/w3block-ui-sdk';

// Mesma query, mas sem redirecionamento automatico em caso de erro
const { data, isLoading, error } = useProfileWithouRedirect();
```

### 9.3 — usePatchProfile (mutation)

```typescript
import { usePatchProfile } from '@w3block/w3block-ui-sdk';

const patchProfile = usePatchProfile();

// Atualizar nome do usuario:
patchProfile.mutate('Novo Nome');
```

---

## 10. Error Handling — handleNetworkException

Funcao utilitaria que normaliza erros (Axios e nao-Axios) em um formato padrao.

### 10.1 — Import e assinatura

```typescript
import { handleNetworkException } from '@w3block/w3block-ui-sdk';

function handleNetworkException(
  e: any,
  rules?: Array<Rule>,
  info?: any
): NetworkException;
```

### 10.2 — Tipos

```typescript
type Rule = {
  criteria: 'message' | 'messageLike' | 'statusCode';
  values: Array<string | number>;
  message: string;
  errorCode?: string;
};

type NetworkException = {
  errorCode: string;   // "unknown" ou codigo customizado via rules
  message: string;     // Mensagem de erro da API ou customizada
  statusCode: number;  // HTTP status code ou -1 se nao-Axios
  info?: any;          // Dados extras opcionais
};
```

### 10.3 — Comportamento

- **Erros Axios:** extrai `message`, `statusCode` e `data` da resposta HTTP. Aplica as `rules` para customizar `errorCode` e `message`.
- **Erros nao-Axios:** retorna `errorCode`, `message` e `statusCode` do erro original, ou valores default (`"unknown"`, mensagem generica, `-1`).

### 10.4 — Uso com regras customizadas

```typescript
try {
  const response = await axios.post('/algum-endpoint', data);
  return response.data;
} catch (err) {
  throw handleNetworkException(err, [
    {
      criteria: 'message',
      values: ['Invalid token'],
      message: 'Token invalido ou expirado',
      errorCode: 'INVALID_TOKEN',
    },
    {
      criteria: 'statusCode',
      values: [403],
      message: 'Voce nao tem permissao para esta acao',
      errorCode: 'FORBIDDEN',
    },
    {
      criteria: 'messageLike',
      values: ['already exists'],
      message: 'Este registro ja existe',
      errorCode: 'DUPLICATE',
    },
  ]);
}
```

### 10.5 — Criterios disponveis

| Criterio | Descricao | Comparacao |
|----------|-----------|------------|
| `message` | Match exato na mensagem de erro | `===` |
| `messageLike` | Match parcial (case-insensitive) na mensagem | `indexOf` |
| `statusCode` | Match no HTTP status code | `===` |

---

## 11. JWT Validation Flow

### 11.1 — Validacao no Axios interceptor

Toda requisicao autenticada passa pelo interceptor que valida o JWT:

```
Request -> interceptor -> validateJwtToken(token) -> OK? -> Adiciona header Authorization
                                                   |
                                                 EXPIRADO -> throw Error('Token expired')
```

### 11.2 — Validacao no useAxios

```
useAxios(type) chamado -> token existe?
  +- SIM -> validateJwtToken(token)
  |    +- VALIDO -> retorna instancia Axios autenticada
  |    +- EXPIRADO -> signOut() + redirect para /auth/signIn
  +- NAO -> retorna instancia Axios publica (sem auth)
```

### 11.3 — Validacao no NextAuth (server-side)

```
jwt callback chamado -> e login inicial?
  +- SIM -> salva accessToken, refreshToken, accessTokenExpires
  +- NAO -> Date.now() < accessTokenExpires?
       +- SIM -> retorna token sem alterar
       +- NAO -> refreshAccessToken(token, baseURL)
            +- SUCESSO -> atualiza tokens
            +- ERRO -> retorna { error: 'RefreshAccessTokenError' }
```

### 11.4 — Logica de validacao do JWT

A validacao do JWT compara o campo `exp` do token (em segundos, Unix timestamp) com `Date.now()` (em milissegundos). O token e considerado valido se `exp * 1000 > Date.now()`.

> **Nota:** O `exp` do JWT esta em **segundos** (Unix timestamp). A funcao multiplica por 1000 para comparar com `Date.now()` (milissegundos).

---

## 12. TypeScript Interfaces Principais

### 12.1 — JwtInterface

```typescript
export interface JwtInterface {
  sub: string;        // User ID (UUID)
  email: string;
  name: string;
  roles: Array<UserRoleEnum>;  // ex: ['user'], ['admin', 'user']
  verified: boolean;
  iat: number;        // Issued at (Unix timestamp em segundos)
  exp: number;        // Expiration (Unix timestamp em segundos)
}
```

### 12.2 — W3blockAPIContextInterface

```typescript
export interface W3blockAPIContextInterface {
  w3blockKeyAPIUrl: string;
  w3blockIdAPIUrl: string;
  w3blockCommerceAPIUrl: string;
  w3blockPdfAPIUrl: string;
  w3BlockPollApiUrl: string;
  w3BlockPassApiUrl: string;
}
```

### 12.3 — IW3blockUISDKGereralConfigContext

```typescript
export interface IW3blockUISDKGereralConfigContext {
  companyId: string;
  logoUrl: string | undefined;
  appBaseUrl: string;
  connectProxyPass: string;
  name?: string;
}
```

### 12.4 — SessionUser

```typescript
export interface SessionUser extends User {
  accessToken: string;
  refreshToken: string;
  roles: Array<UserRoleEnum>;
  companyId?: string;
  email?: string;
}
```

### 12.5 — SignInResponse (resposta da API de login)

```typescript
export interface SignInResponse {
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
```

### 12.6 — NetworkException (erro normalizado)

```typescript
type NetworkException = {
  errorCode: string;
  message: string;
  statusCode: number;
  info?: any;
};
```

### 12.7 — PixwaySessionContextInterface

```typescript
export interface PixwaySessionContextInterface {
  token?: string;
  companyId?: string;
  user?: {
    name?: string;
  };
}
```

> **Nota:** Na pratica, o `PixwaySessionContext` e tipado como `SessionContextValue` do NextAuth (`{ data, status, update }`), nao como `PixwaySessionContextInterface`. A interface acima e uma tipagem legacy.

---

## 13. Resumo: Como Criar um Hook de Dominio

Receita padrao para criar um hook que consome a API W3block:

### Query autenticada (GET)

```typescript
import { usePrivateQuery } from '@w3block/w3block-ui-sdk';
import { useAxios } from '@w3block/w3block-ui-sdk';
import { W3blockAPI } from '@w3block/w3block-ui-sdk';
import { PixwayAPIRoutes } from '@w3block/w3block-ui-sdk';
import { useCompanyConfig } from '@w3block/w3block-ui-sdk';
import { handleNetworkException } from '@w3block/w3block-ui-sdk';

export const useMeuRecurso = (id: string) => {
  const axios = useAxios(W3blockAPI.COMMERCE);
  const { companyId } = useCompanyConfig();

  return usePrivateQuery(
    ['meu-recurso', companyId, id],
    async () => {
      try {
        const url = PixwayAPIRoutes.MINHA_ROTA
          .replace('{companyId}', companyId)
          .replace('{id}', id);
        const { data } = await axios.get(url);
        return data;
      } catch (err) {
        throw handleNetworkException(err);
      }
    },
    {
      enabled: !!id,
      retry: 1,
    }
  );
};
```

### Query publica (GET sem auth)

```typescript
export const useMeuRecursoPublico = (slug: string) => {
  const axios = useAxios(W3blockAPI.COMMERCE);
  const { companyId } = useCompanyConfig();

  return usePublicQuery(
    ['meu-recurso-publico', companyId, slug],
    async () => {
      const url = PixwayAPIRoutes.MINHA_ROTA_PUBLICA
        .replace('{companyId}', companyId)
        .replace('{slug}', slug);
      const { data } = await axios.get(url);
      return data;
    },
    {
      enabled: !!slug,
    }
  );
};
```

### Mutation (POST/PATCH/DELETE)

```typescript
export const useCriarRecurso = () => {
  const axios = useAxios(W3blockAPI.COMMERCE);
  const { companyId } = useCompanyConfig();

  return useMutation(async (dados: MeuTipo) => {
    try {
      const url = PixwayAPIRoutes.MINHA_ROTA
        .replace('{companyId}', companyId);
      const { data } = await axios.post(url, dados);
      return data;
    } catch (err) {
      throw handleNetworkException(err);
    }
  });
};
```
