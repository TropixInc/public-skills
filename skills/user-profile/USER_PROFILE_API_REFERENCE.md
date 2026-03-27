# User Profile API Reference (KEY API Only)

Referencia dos endpoints de **withdrawals** (saques) que utilizam a KEY API (`https://api.w3block.io`).

> **Nota:** Endpoints de perfil, KYC, documentos, wallets, whitelist, referral e notificacoes usam a Identity API (`https://id.w3block.io`) e nao estao cobertos aqui.

---

## Base URL

| Servico | Producao | Enum |
|---------|----------|------|
| Key API | `https://api.w3block.io` | `W3blockAPI.KEY` |

---

## Autenticacao

Todos os endpoints exigem autenticacao:

```
Authorization: Bearer <access_token>
```

O `access_token` e obtido via login no Identity API. Endpoints marcados como **Admin** exigem role de administrador.

---

## Endpoints

### 1. Request Withdraw

Solicita um saque de fundos.

| Campo | Valor |
|-------|-------|
| **Metodo** | `POST` |
| **Path** | `/{companyId}/withdraws` |
| **API** | KEY |
| **Auth** | Bearer token |

#### Request Body

```json
{
  "fromWalletAddress": "0x1234...abcd",
  "amount": "100.00",
  "withdrawAccountId": "uuid-da-conta-saque",
  "memo": "Saque mensal",
  "erc20ContractAddress": "0xabcd...",
  "erc20ChainId": "137",
  "erc20ContractId": "uuid-contrato",
  "userId": "uuid-usuario"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `fromWalletAddress` | `string` | Sim | Endereco da wallet de origem |
| `amount` | `string` | Sim | Valor do saque |
| `withdrawAccountId` | `string` | Sim | ID da conta de saque destino |
| `memo` | `string` | Sim | Observacao do saque |
| `erc20ContractAddress` | `string` | Nao | Endereco do contrato ERC-20 |
| `erc20ChainId` | `string` | Nao | Chain ID do token |
| `erc20ContractId` | `string` | Nao | ID do contrato |
| `userId` | `string` | Nao | ID do usuario (se diferente do logado) |

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useRequestWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` |

#### Erros Comuns

| Status | Erro | Causa |
|--------|------|-------|
| 400 | `insufficient-balance` | Saldo insuficiente na wallet |
| 400 | `invalid-withdraw-account` | Conta de saque invalida ou inexistente |
| 403 | `forbidden` | Sem permissao para sacar desta wallet |

---

### 2. Get Specific Withdraw

Retorna detalhes de um saque especifico (visao usuario).

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `/{companyId}/withdraws/{id}` |
| **API** | KEY |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `string (uuid)` | ID do tenant |
| `id` | `string (uuid)` | ID do saque |

#### Response

```json
{
  "id": "uuid-withdraw",
  "status": "pending",
  "amount": "100.00",
  "memo": "Saque mensal",
  "fromWalletAddress": "0x1234...abcd",
  "withdrawAccountId": "uuid-conta",
  "createdAt": "2026-03-27T10:00:00Z",
  "updatedAt": "2026-03-27T10:00:00Z"
}
```

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useGetSpecificWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `usePrivateQuery` |

---

### 3. Get Specific Withdraw (Admin)

Retorna detalhes de um saque especifico (visao admin, inclui dados do usuario).

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `/{companyId}/withdraws/admin/{id}` |
| **API** | KEY |
| **Auth** | Bearer token (Admin) |

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useGetSpecificWithdrawAdmin` | `auth/hooks/useRequestWithdraw.ts` | `usePrivateQuery` |

---

### 4. Conclude Withdraw (Admin)

Conclui um saque, anexando comprovante de transferencia.

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `/{companyId}/withdraws/admin/{id}/conclude` |
| **API** | KEY |
| **Auth** | Bearer token (Admin) |

#### Request Body

```json
{
  "receiptAssetId": "uuid-do-asset-comprovante"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `receiptAssetId` | `string` | Sim | ID do asset com o comprovante de transferencia |

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useConcludeWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` |

---

### 5. Escrow Withdraw (Admin)

Retem os recursos do saque (congela fundos antes da transferencia).

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `/{companyId}/withdraws/admin/{id}/escrow` |
| **API** | KEY |
| **Auth** | Bearer token (Admin) |

#### Request Body

Nenhum body necessario.

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useEscrowWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` |

---

### 6. Refuse Withdraw (Admin)

Recusa um saque, informando o motivo.

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `/{companyId}/withdraws/admin/{id}/refuse` |
| **API** | KEY |
| **Auth** | Bearer token (Admin) |

#### Request Body

```json
{
  "reason": "Dados inconsistentes"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `reason` | `string` | Sim | Motivo da recusa |

#### Hook

| Hook | Arquivo | Tipo |
|------|---------|------|
| `useRefuseWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` |

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

### Exemplos de Respostas de Erro

| Status | Exemplo `message` | Causa |
|--------|-------------------|-------|
| `400` | `"insufficient-balance"` | Saldo insuficiente na wallet para o valor do saque |
| `400` | `"invalid-withdraw-account"` | Conta de saque invalida ou nao pertence ao usuario |
| `400` | `"reason is required"` | Tentou recusar saque sem informar motivo |
| `401` | `"Unauthorized"` | Token ausente ou expirado |
| `403` | `"Forbidden"` | Sem permissao — acoes admin exigem role de administrador |
| `404` | `"Not Found"` | Saque nao encontrado (id invalido) |

---

## Enums

### Withdraw Status

```typescript
type WithdrawStatus =
  | 'pending'               // Saque solicitado, aguardando acao admin
  | 'escrowing_resources'   // Recursos sendo retidos
  | 'ready_to_transfer_funds' // Fundos retidos, pronto para transferencia
  | 'concluded'             // Saque concluido com sucesso
  | 'failed'                // Saque falhou
  | 'refused';              // Saque recusado pelo admin
```

### PixwayAPIRoutes (rotas de withdraw)

```typescript
enum PixwayAPIRoutes {
  REQUEST_WITHDRAW = '/{companyId}/withdraws',
  GET_SPECIFIC_WITHDRAW = '/{companyId}/withdraws/{id}',
  GET_SPECIFIC_WITHDRAW_ADMIN = '/{companyId}/withdraws/admin/{id}',
  CONCLUDE_WITHDRAW = '/{companyId}/withdraws/admin/{id}/conclude',
  ESCROW_WITHDRAW = '/{companyId}/withdraws/admin/{id}/escrow',
  REFUSE_WITHDRAW = '/{companyId}/withdraws/admin/{id}/refuse',
}
```

---

## TypeScript Interfaces

```typescript
interface RequestWithdrawDto {
  fromWalletAddress: string;
  amount: string;
  withdrawAccountId: string;
  memo: string;
  erc20ContractAddress?: string;
  erc20ChainId?: string;
  erc20ContractId?: string;
  userId?: string;
}

interface WithdrawResponse {
  id: string;
  status: WithdrawStatus;
  amount: string;
  memo: string;
  fromWalletAddress: string;
  withdrawAccountId: string;
  receiptAssetId?: string;
  reason?: string;
  createdAt: string;
  updatedAt: string;
}

interface ConcludeWithdrawDto {
  receiptAssetId: string;
}

interface RefuseWithdrawDto {
  reason: string;
}
```
