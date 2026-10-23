---
id: LOYALTY_API_REFERENCE
title: "Loyalty - API Reference"
module: loyalty
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-10-23"
authors:
  - fernandodevpascoal
tags:
  - api
  - loyalty
---

# Loyalty API Reference

Referencia completa de endpoints, schemas, DTOs e enums do dominio de loyalty/recompensas do W3block.

**Base URL:** `https://api.w3block.io` (W3blockAPI.KEY)

---

## Endpoints

### 1. Consultar Saldo do Usuario

```
GET /{companyId}/loyalties/users/balance/{userId}
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetUserBalance` (useMutation — chamado como mutation, nao query)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `userId` | string (UUID) | ID do usuario |

**Resposta:** Array de saldos por programa de fidelidade.

**Observacao:** Apesar de ser um GET, o hook usa `useMutation` em vez de `useQuery`. Isso permite chamar a busca de saldo sob demanda (imperativo) em vez de automaticamente.

---

### 2. Listar Loyalties da Empresa

```
GET /{companyId}/loyalties/admin?limit=49
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetCompanyLoyalties` (useQuery)
**Roles:** `superAdmin`, `admin`, `loyaltyOperator`

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |

**Query params:**

| Param | Tipo | Default | Descricao |
|-------|------|---------|-----------|
| `limit` | number | 49 | Maximo de registros retornados |

**Resposta:**

```typescript
{
  items: LoyaltyInterface[]
}
```

**Condicoes de habilitacao:**
- `profile.id` nao nulo
- `companyId` nao nulo
- Usuario com role `superAdmin`, `admin` ou `loyaltyOperator`

---

### 3. Preview de Pagamento

```
PATCH /{companyId}/loyalties/rewards/payment/preview
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetPaymentPreview` (useMutation)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |

**Body (PaymentPreviewDTO):**

```typescript
{
  loyaltyId: string;   // UUID do programa de fidelidade
  amount: string;      // Valor monetario (ex: "50.00")
  points: string;      // Quantidade de pontos (ex: "100")
  userId: string;      // UUID do usuario pagador
  userCode: string;    // Codigo temporario do usuario
}
```

**Resposta:** Dados de pre-visualizacao do pagamento com calculo de pontos.

---

### 4. Criar Pagamento

```
PATCH /{companyId}/loyalties/rewards/payment
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useCreatePayment` (useMutation)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |

**Body (CreatePaymentDTO):**

```typescript
{
  loyaltyId: string;    // UUID do programa de fidelidade
  amount: string;       // Valor monetario
  points: string;       // Quantidade de pontos
  userId: string;       // UUID do usuario pagador
  userCode: string;     // Codigo temporario do usuario
  description: string;  // Descricao do pagamento
}
```

**Resposta:** Dados da transacao criada.

**Diferenca do Preview:** O `CreatePaymentDTO` inclui o campo `description` que nao existe no `PaymentPreviewDTO`.

---

### 5. Resumo de Staking

```
GET /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useStakingSummary` (usePrivateQuery)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `userId` | string (UUID) | ID do usuario (obtido da sessao) |
| `loyaltyId` | string (UUID) | ID do programa de fidelidade (obtido do loyaltyWallet) |

**Query params (paginacao):**

| Param | Tipo | Default | Descricao |
|-------|------|---------|-----------|
| `page` | number | 1 | Pagina atual |
| `limit` | number | 10 | Itens por pagina |
| `sortBy` | string | `createdAt` | Campo de ordenacao |
| `orderBy` | string | `DESC` | Direcao da ordenacao |

**Resposta:** Lista paginada de registros de staking (SummaryDto).

**Condicoes de habilitacao:**
- `companyId` definido
- `session.id` definido

---

### 6. Resgatar Staking

```
PATCH /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useRedeemStaking` (useMutation)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `userId` | string (UUID) | ID do usuario (obtido da sessao) |
| `loyaltyId` | string (UUID) | ID do programa de fidelidade (obtido do loyaltyWallet) |

**Body:** Nenhum (PATCH sem body).

**Resposta:** Dados do resgate executado.

---

### 7. Recompensas Diferidas por Usuario

```
GET /{companyId}/loyalties/users/deferred/{userId}
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetDeferredByUserId` (useQuery)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `userId` | string (UUID) | ID do usuario |

**Query params (automaticos via filtro de status):**

| Status param | Descricao |
|-------------|-----------|
| `status=deferred` | Recompensa diferida |
| `status=pending` | Pendente de processamento |
| `status=started` | Processamento iniciado |
| `status=pool` | Em pool de agrupamento |
| `status=success` | Processada com sucesso |
| `withdrawBlockedOnly=true` | Apenas agendadas (saque bloqueado) |
| `withdrawBlockedOnly=false` | Apenas recebidas (saque liberado) |

**Filtros pre-definidos por tipo:**

| Tipo (status param) | Filtro aplicado |
|---------------------|-----------------|
| `scheduled` | Todos os status + `withdrawBlockedOnly=true` |
| `received` | Todos os status + `withdrawBlockedOnly=false` |
| (nenhum) | Todos os status sem filtro de withdraw |

**Resposta:** `ErcTokenHistoryInterfaceResponse`

**Condicoes de habilitacao:**
- `userId` definido
- `companyId` definido
- `enabled` param = true

---

### 8. Recompensas Diferidas (Geral)

```
GET /{companyId}/loyalties/users/deferred
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetDeferred` (useQuery)
**Roles:** Qualquer usuario autenticado (tipicamente admin)

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |

**Query params:** Mesmos filtros de status do endpoint "Deferred by User" (ver acima).

**Resposta:** `ErcTokenHistoryInterfaceResponse`

**Condicoes de habilitacao:**
- `companyId` definido
- `enabled` param = true

---

### 9. Exportar Diferidos em XLSX

```
GET /{companyId}/loyalties/users/deferred/xlsx
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetXlsxDeferred` (useMutation)
**Roles:** Qualquer usuario autenticado (tipicamente admin)

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |

**Query params:** Mesmos filtros de status do endpoint "Deferred".

**Resposta:** Blob (arquivo XLSX). O hook configura `responseType: 'blob'`.

**Observacao:** Usa `useMutation` para permitir download sob demanda. O parametro passado ao mutate e:
```typescript
{
  filter?: Record<string, string>;  // Filtros de paginacao/busca
  status?: 'scheduled' | 'received';  // Tipo de filtro de status
}
```

---

### 10. Historico ERC20 por Usuario

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history/{userId}
```

**API:** KEY
**Autenticacao:** Bearer token
**Enum:** `GET_ERC_TOKENS_BY_LOYALTY_ID`
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `loyaltyId` | string (UUID) | contractId do programa de fidelidade |
| `userId` | string (UUID) | ID do usuario |

**Resposta:** `ErcTokenHistoryInterfaceResponse`

---

### 11. Historico ERC20 Admin (Todos)

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetAllReportsAdmin` (useQuery)
**Roles:** `admin`, `superAdmin`

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `loyaltyId` | string (UUID) | contractId do primeiro loyalty da lista |

**Query params:** Filtros dinamicos via `cleanObject(filter)`.

**Resposta:** `ErcTokenHistoryInterfaceResponse`

**Condicoes de habilitacao:**
- `loyalties.length > 0`
- `loyalties[0].contractId` definido
- Role `admin` ou `superAdmin`
- `companyId` definido

**Importante:** O `loyaltyId` no path e na verdade o `contractId` do primeiro loyalty da empresa, obtido via `useLoyaltiesInfo().loyalties[0].contractId`.

---

### 12. Historico ERC20 por Operador

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history/operator/{operatorId}
```

**API:** KEY
**Autenticacao:** Bearer token
**Hook:** `useGetAllReportsByOperatorId` (useQuery)
**Roles:** `loyaltyOperator`

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `loyaltyId` | string (UUID) | contractId do primeiro loyalty da lista |
| `operatorId` | string (UUID) | ID do operador (profile.id do usuario logado) |

**Query params:** Filtros dinamicos via `cleanObject(filter)`.

**Resposta:** `ErcTokenHistoryInterfaceResponse`

**Condicoes de habilitacao:**
- `loyalties.length > 0`
- `loyalties[0].contractId` definido
- Role `loyaltyOperator`
- `companyId` definido
- `profile.id` definido

---

### Endpoint Auxiliar: Buscar Usuario por Codigo

```
GET /users/{companyId}/code/{userId}
```

**API:** ID (W3blockAPI.ID) — **Nao e Key API**
**Autenticacao:** Bearer token
**Hook:** `useGetUserByCode` (useMutation)
**Roles:** Qualquer usuario autenticado

**Path params:**

| Param | Tipo | Descricao |
|-------|------|-----------|
| `companyId` | string (UUID) | ID da empresa/tenant |
| `userId` | string | Codigo temporario do usuario |

**Resposta:** Dados do usuario associado ao codigo.

**Uso:** Antes de executar um pagamento, o operador pode buscar o usuario pelo codigo temporario (exibido no app do cliente).

---

## Schemas / Interfaces TypeScript

### LoyaltyInterface

```typescript
interface LoyaltyInterface {
  id: string;
  createdAt: string;
  updatedAt: string;
  companyId: string;
  contractId: string;                    // ID do contrato ERC-20
  settings: {
    paymentView: {
      enabled: boolean;                  // Se pagamento com pontos esta habilitado
    };
  };
  name: string;                          // Nome do programa de fidelidade
  compensateRule: {
    period: string;                      // Periodo de compensacao (ex: cashback diferido)
  };
  priority: number;
  tokenIssuanceMethod: string;           // Metodo de emissao de tokens
  tokenIssuanceAddress: string;          // Endereco de emissao
  tokenTransferabilityMethod: string;    // Metodo de transferencia
  tokenTransferabilityAddress: string;   // Endereco de transferencia
  pointPrecision: string;               // "integer" ou "decimal"
  paymentViewSettings: {
    pointsEquivalent: {
      currency: string;                  // Moeda de referencia (ex: "BRL")
      pointsValue: number;              // Quantidade de pontos
      currencyValue: number;            // Valor monetario equivalente
    };
  };
  rules: Rule[];                         // Regras de acumulo/resgate
}

interface Rule {
  id: string;
  createdAt: string;
  updatedAt: string;
  companyId: string;
  loyaltyId: string;
  whitelistId: string;
  value: string;
  available: string;
  type: string;
  name: string;
  description: string;
  priority: number;
}
```

**Arquivo fonte:** `src/modules/shared/interfaces/ILoyalty.ts`

---

### CreatePaymentDTO

```typescript
interface CreatePaymentDTO {
  loyaltyId: string;    // UUID do programa de fidelidade
  amount: string;       // Valor monetario (string numerica)
  points: string;       // Quantidade de pontos (string numerica)
  userId: string;       // UUID do usuario pagador
  userCode: string;     // Codigo temporario do usuario
  description: string;  // Descricao do pagamento
}
```

**Arquivo fonte:** `src/modules/business/hooks/useCreatePayment.tsx`

---

### PaymentPreviewDTO

```typescript
interface PaymentPreviewDTO {
  loyaltyId: string;    // UUID do programa de fidelidade
  amount: string;       // Valor monetario
  points: string;       // Quantidade de pontos
  userId: string;       // UUID do usuario pagador
  userCode: string;     // Codigo temporario do usuario
}
```

**Arquivo fonte:** `src/modules/business/hooks/useGetPaymentPreview.tsx`

---

### SummaryDto (Staking)

```typescript
interface SummaryDto {
  actionId: string;
  amount: string;
  companyId: string;
  createdAt: string;
  executeAt: string;             // Data de execucao
  expiresAt: string;             // Data de expiracao
  fromAddress: string;
  id: string;
  isExpired: boolean;            // Se ja expirou
  isPoolGrouping: boolean;       // Se faz parte de agrupamento em pool
  isWithdrawal: boolean;         // Se e uma retirada
  loyaltyId: string;
  parentMinterId: string;
  parentTransferId: string;
  poolAddress: string;
  rollbackActionId: string;
  sendEmail: boolean;
  stakingRuleId: string;         // ID da regra de staking aplicada
  status: string;
  toAddress: string;
  updatedAt: string;
  userId: string;
}
```

**Arquivo fonte:** `src/modules/shared/hooks/useStaking.ts`

---

### ErcTokenHistoryInterfaceResponse

```typescript
interface ErcTokenHistoryInterfaceResponse {
  items: Erc20TokenHistory[];
  summary?: {
    balance?: number;
  };
  meta: {
    totalItems: number;
    itemCount: number;
    itemsPerPage: number;
    totalPages: number;
    currentPage: number;
  };
}

interface Erc20TokenHistory {
  id: string;
  createdAt: string;
  updatedAt: string;
  contractId: string;
  companyId: string;
  type: Erc20ActionType;          // 'transfer' | 'burn' | 'minter'
  status: Erc20ActionStatus;      // 'created' | 'started' | 'success' | 'failed' | 'wait_event' | 'external'
  contractAddress: string;
  chainId: number;
  sender: string;
  txHash: string;
  txId: string;
  metadata: {
    confirmedAt: string;
    transaction: Transaction;
    commerce?: Commerce;
    description?: string;
  };
  request: {
    to: string;
    amount: string;
    from?: string;
    metadata?: any;
  };
  executeAt: string;
  loyaltiesTransactions: LoyaltiesTransaction[];
  to?: FromToInterface;
  from?: FromToInterface;
}

interface FromToInterface {
  id?: string;
  phone?: string;
  type?: string;
  user_name?: string;
  wallet?: string;
  cpf?: string;
  email?: string;
}

interface LoyaltiesTransaction {
  id: string;
  createdAt: string;
  updatedAt: string;
  companyId: string;
  loyaltyId: string;
  userId: string;
  fromAddress: string;
  toAddress: string;
  amount: string;
  metadata?: Metadata2;
  executeAt: string;
  status: string;
  actionId: string;
  parentMinterId?: null;
  parentTransferId?: null;
  isWithdrawal: boolean;
  poolAddress?: null;
  sendEmail: boolean;
}
```

**Arquivo fonte:** `src/modules/dashboard/interface/ercTokenHistoryInterface.ts`

---

## Enums

### Erc20ActionType

```typescript
enum Erc20ActionType {
  TRANSFER = 'transfer',
  BURN = 'burn',
  MINTER = 'minter',
}
```

### Erc20ActionStatus

```typescript
enum Erc20ActionStatus {
  CREATED = 'created',
  STARTED = 'started',
  SUCCESS = 'success',
  FAILED = 'failed',
  WAIT_EVENT = 'wait_event',
  EXTERNAL = 'external',
}
```

**Arquivo fonte:** `src/modules/shared/interfaces/Statement.ts`

---

## Rotas no PixwayAPIRoutes (Enum)

```typescript
// Loyalty
GET_LOYALTY_USER_BALANCE     = '/{companyId}/loyalties/users/balance/{userId}'
GET_COMPANY_LOYALTIES        = '/{companyId}/loyalties/admin'
GET_LOYALTY_PREVIEW          = '/{companyId}/loyalties/rewards/payment/preview'
CREATE_LOYALTY_PAYMENT       = '/{companyId}/loyalties/rewards/payment'
STAKING_SUMMARY              = '/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}'
REDEEM_STAKING               = '/{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem'
GET_DEFERRED_BY_USER_ID      = '/{companyId}/loyalties/users/deferred/{userId}'
GET_DEFERRED                 = '/{companyId}/loyalties/users/deferred'
GET_DEFERRED_XLSX            = '/{companyId}/loyalties/users/deferred/xlsx'

// ERC20 History (Relatorios)
GET_ERC_TOKENS_BY_LOYALTY_ID                    = '/{companyId}/erc20-tokens/{loyaltyId}/history/{userId}'
GET_ERC_TOKENS_BY_LOYALTY_ID_ADMIN              = '/{companyId}/erc20-tokens/{loyaltyId}/history'
GET_ERC_TOKENS_BY_LOYALTY_ID_BY_OPERATOR_ID     = '/{companyId}/erc20-tokens/{loyaltyId}/history/operator/{operatorId}'

// Auxiliar (ID API)
GET_TEMPORARY_CODE           = '/users/{companyId}/code/{userId}'
```

**Arquivo fonte:** `src/modules/shared/enums/PixwayAPIRoutes.ts`

---

## Controle de Acesso por Role

| Role | Descricao | Endpoints permitidos |
|------|-----------|---------------------|
| `superAdmin` | Administrador geral da plataforma | Todos os endpoints de loyalty |
| `admin` | Administrador da empresa/tenant | Listar loyalties, relatorios admin, todos endpoints de usuario |
| `loyaltyOperator` | Operador de fidelidade (caixa/atendente) | Listar loyalties, relatorios por operador, pagamentos |
| (autenticado) | Qualquer usuario logado | Saldo, pagamento, staking, deferred por usuario |

---

## Formato Padrao de Erros

Todas as APIs W3block retornam erros no seguinte formato:

```json
{
  "statusCode": 400,
  "message": "mensagem-de-erro",
  "error": "Bad Request"
}
```

Quando `message` e um array:

```json
{
  "statusCode": 400,
  "message": ["campo-x must be a string", "campo-y should not be empty"],
  "error": "Bad Request"
}
```

### Exemplos de Respostas de Erro

| Status | Exemplo `message` | Causa |
|--------|-------------------|-------|
| `400` | `"insufficient-balance"` | Saldo de loyalty insuficiente para pagamento |
| `400` | `"invalid-loyalty-id"` | loyaltyId ou contractId invalido |
| `401` | `"Unauthorized"` | Token ausente ou expirado |
| `403` | `"Forbidden"` | Sem permissao (role insuficiente) |
| `404` | `"Not Found"` | Recurso inexistente |

---

## Armadilhas Comuns

| # | Problema | Solucao |
|---|----------|---------|
| 1 | `amount` e `points` sao strings, nao numeros | Sempre usar `parseFloat()` ou `Number()` para calculos |
| 2 | `useGetUserBalance` usa `useMutation` em vez de `useQuery` | Chamar via `.mutateAsync(userId)` em vez de esperar auto-fetch |
| 3 | `loyaltyId` nos endpoints de ERC20 e na verdade `contractId` | Usar `loyalties[0].contractId`, nao `loyalties[0].id` |
| 4 | `useGetXlsxDeferred` retorna Blob | Configurar `responseType: 'blob'` e tratar como download de arquivo |
| 5 | Filtros de status sao multiplos na mesma query | Repetir `status=` para cada valor: `status=deferred&status=pending&...` |
| 6 | `pointPrecision` pode ser `integer` ou `decimal` | Validar antes de exibir — inteiros nao devem mostrar casas decimais |
| 7 | Endpoint admin requer `loyalties[0].contractId` carregado | Garantir que `useGetCompanyLoyalties` ja retornou antes de chamar relatorios |
