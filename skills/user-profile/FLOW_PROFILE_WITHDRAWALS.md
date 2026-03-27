# Flow: Saques (Withdrawals)

Fluxo completo de saques, incluindo gerenciamento de metodos de saque (usuario), solicitacao de saque e acoes administrativas (escrow, conclusao, recusa).

---

## Visao Geral

```
+--------------------------------------------------------------------------+
|                          FLUXO DE SAQUES                                  |
|                                                                           |
|  USUARIO:                                                                 |
|  +-------------------+   +------------------+   +---------------------+  |
|  | 1. Cadastrar      |   | 2. Solicitar     |   | 3. Acompanhar       | |
|  | metodo de saque   |-->| saque            |-->| status do saque     | |
|  | (withdraw account)|   | (KEY API)        |   | (KEY API)           | |
|  +-------------------+   +------------------+   +---------------------+  |
|                                                          |                |
|  ADMIN:                                                  v                |
|  +-------------------+   +------------------+   +---------------------+  |
|  | 4a. Reter fundos  |   | 4b. Concluir     |   | 4c. Recusar         | |
|  | (Escrow)          |-->| saque            |   | saque               | |
|  +-------------------+   +------------------+   +---------------------+  |
+--------------------------------------------------------------------------+
```

---

## Diagrama do Fluxo de Saque

```
Fluxo de Saque:

Usuario                        Admin
  |                              |
  |-- POST /withdraws --------→ |
  |   (status: pending)         |
  |                              |
  |                   PATCH /escrow
  |                   (status: escrowing_resources)
  |                              |
  |                   (status: ready_to_transfer_funds)
  |                              |
  |              ┌───────────────┼───────────────┐
  |              |               |               |
  |       PATCH /conclude  PATCH /refuse         |
  |       (concluded)      (refused)             |
  |              |               |               |
```

---

## Ciclo de Vida de um Saque

```
pending --> escrowing_resources --> ready_to_transfer_funds --> concluded
   |
   +---> refused (admin recusa)
   |
   +---> failed (erro no sistema)
```

### Status

| Status | Descricao | Proximo passo |
|--------|-----------|---------------|
| `pending` | Saque solicitado, aguardando acao admin | Admin pode: Escrow ou Recusar |
| `escrowing_resources` | Recursos sendo retidos (processando) | Aguardar transicao automatica |
| `ready_to_transfer_funds` | Fundos retidos, pronto para transferencia | Admin pode: Concluir |
| `concluded` | Saque concluido com sucesso | Final |
| `failed` | Erro no processamento | Final |
| `refused` | Saque recusado pelo admin | Final |

### Mapeamento de status para exibicao

**Visao admin:**
```typescript
const statusMapping = {
  pending: 'Pendente',
  escrowing_resources: 'Retendo fundos',
  ready_to_transfer_funds: 'Pronto para transferir',
  concluded: 'Concluido',
  failed: 'Falha',
  refused: 'Recusado',
};
```

**Visao usuario:** (simplificado)
```typescript
const statusMapping = {
  pending: 'Pendente',
  escrowing_resources: 'Pendente',
  ready_to_transfer_funds: 'Pendente',
  concluded: 'Concluido',
  failed: 'Falha',
  refused: 'Recusado',
};
```

> **Nota:** Para o usuario, `escrowing_resources` e `ready_to_transfer_funds` sao exibidos como "Pendente" para simplificar a experiencia.

---

## Fluxo Passo a Passo

O ciclo completo de um saque passa por 4 etapas, sendo as 2 primeiras do usuario e as 2 ultimas do administrador.

### Step 1: Usuario solicita saque

O usuario cria uma solicitacao de saque informando a wallet de origem, valor e conta de destino. O status inicial e `pending`.

**Endpoint:** `POST /{companyId}/withdraws` (KEY API)

#### Hook SDK

```typescript
// Step 1: Solicitar saque
import { useRequestWithdraw } from '@w3block/w3block-ui-sdk';

const { mutateAsync: requestWithdraw } = useRequestWithdraw();
const result = await requestWithdraw({
  fromWalletAddress: '0x1234...abcd',
  amount: '100.00',
  withdrawAccountId: 'uuid-da-conta',
  memo: 'Saque mensal',
});
```

#### Campos do request

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `fromWalletAddress` | `string` | Sim | Endereco da wallet de origem |
| `amount` | `string` | Sim | Valor do saque (como string) |
| `withdrawAccountId` | `string` | Sim | ID da conta de destino |
| `memo` | `string` | Sim | Observacao/descricao |
| `erc20ContractAddress` | `string` | Nao | Endereco do contrato ERC-20 |
| `erc20ChainId` | `string` | Nao | Chain ID do token |
| `erc20ContractId` | `string` | Nao | ID interno do contrato |
| `userId` | `string` | Nao | ID do usuario (se diferente do logado) |

> **Importante:** Este endpoint usa a **KEY API** (`W3blockAPI.KEY`), nao a ID API.

---

### Step 2: Usuario consulta status

O usuario pode acompanhar o status do saque a qualquer momento.

**Endpoint:** `GET /{companyId}/withdraws/{id}` (KEY API)

#### Hook SDK

```typescript
// Step 2: Consultar status
import { useGetSpecificWithdraw } from '@w3block/w3block-ui-sdk';

const { data: withdraw } = useGetSpecificWithdraw(companyId, withdrawId);
// withdraw.status: 'pending' | 'escrowing_resources' | 'ready_to_transfer_funds' | 'concluded' | 'refused'
```

#### Dados disponiveis na resposta

```typescript
// Dados disponiveis:
// withdraw?.data?.status    -> Status do saque
// withdraw?.data?.amount    -> Valor (string)
// withdraw?.data?.createdAt -> Data de criacao
// withdraw?.data?.reason    -> Motivo (se recusado)
// withdraw?.data?.receiptAsset?.directLink -> Link do comprovante (se concluido)
```

---

### Step 3: Admin retem recursos (Escrow)

O administrador retem os recursos do saque, congelando os fundos antes da transferencia efetiva. Muda status de `pending` para `escrowing_resources`, e depois automaticamente para `ready_to_transfer_funds`.

**Endpoint:** `PATCH /{companyId}/withdraws/admin/{id}/escrow` (KEY API)

#### Hook SDK

```typescript
// Step 3: Escrow (Admin)
import { useEscrowWithdraw } from '@w3block/w3block-ui-sdk';

const { mutateAsync: escrowWithdraw } = useEscrowWithdraw();
await escrowWithdraw({ companyId, withdrawId });
```

> **Nota:** Nenhum body e necessario. O status transita automaticamente de `escrowing_resources` para `ready_to_transfer_funds`.

---

### Step 4a: Admin conclui saque

O administrador conclui o saque, anexando um comprovante de transferencia. Requer status `ready_to_transfer_funds`.

**Endpoint:** `PATCH /{companyId}/withdraws/admin/{id}/conclude` (KEY API)

#### Hook SDK

```typescript
// Step 4a: Concluir (Admin)
import { useConcludeWithdraw } from '@w3block/w3block-ui-sdk';

const { mutateAsync: concludeWithdraw } = useConcludeWithdraw();
await concludeWithdraw({ companyId, withdrawId, receiptAssetId: 'uuid-comprovante' });
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `receiptAssetId` | `string` | Sim | ID do asset com o comprovante de transferencia |

> **Importante:** O `receiptAssetId` e o ID de um asset previamente feito upload via `useUploadAssets`. No componente admin, o upload e feito via `InputWithdrawCommerce`.

---

### Step 4b: Admin recusa saque

O administrador recusa o saque, informando o motivo. Disponivel quando status e `pending`.

**Endpoint:** `PATCH /{companyId}/withdraws/admin/{id}/refuse` (KEY API)

#### Hook SDK

```typescript
// Step 4b: Recusar (Admin)
import { useRefuseWithdraw } from '@w3block/w3block-ui-sdk';

const { mutateAsync: refuseWithdraw } = useRefuseWithdraw();
await refuseWithdraw({ companyId, withdrawId, reason: 'Dados inconsistentes' });
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `reason` | `string` | Sim | Motivo da recusa (nao pode ser vazio) |

---

## Exemplo API-first (sem SDK)

Para integracao direta via `fetch`, sem depender do SDK. Todos os endpoints usam a KEY API (`https://api.w3block.io`).

### Solicitar saque

```typescript
const requestWithdraw = async (companyId: string, body: RequestWithdrawDto, token: string) => {
  const response = await fetch(`https://api.w3block.io/${companyId}/withdraws`, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(body),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }
  return response.json();
};
```

### Consultar status

```typescript
const getWithdraw = async (companyId: string, id: string, token: string) => {
  const response = await fetch(`https://api.w3block.io/${companyId}/withdraws/${id}`, {
    headers: { 'Authorization': `Bearer ${token}` },
  });
  if (!response.ok) throw new Error('Erro ao consultar saque');
  return response.json();
};
```

### Admin: Escrow

```typescript
const escrowWithdraw = async (companyId: string, id: string, token: string) => {
  const response = await fetch(`https://api.w3block.io/${companyId}/withdraws/admin/${id}/escrow`, {
    method: 'PATCH',
    headers: { 'Authorization': `Bearer ${token}` },
  });
  if (!response.ok) throw new Error('Erro ao reter recursos');
  return response.json();
};
```

### Admin: Concluir

```typescript
const concludeWithdraw = async (companyId: string, id: string, receiptAssetId: string, token: string) => {
  const response = await fetch(`https://api.w3block.io/${companyId}/withdraws/admin/${id}/conclude`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ receiptAssetId }),
  });
  if (!response.ok) throw new Error('Erro ao concluir saque');
  return response.json();
};
```

### Admin: Recusar

```typescript
const refuseWithdraw = async (companyId: string, id: string, reason: string, token: string) => {
  const response = await fetch(`https://api.w3block.io/${companyId}/withdraws/admin/${id}/refuse`, {
    method: 'PATCH',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ reason }),
  });
  if (!response.ok) throw new Error('Erro ao recusar saque');
  return response.json();
};
```

---

## Acoes do Usuario (Metodos de Saque)

Antes de solicitar um saque, o usuario precisa ter ao menos uma conta de saque cadastrada. Esses endpoints usam a **ID API** (`https://id.w3block.io`), nao a KEY API.

### Listar Metodos de Saque

Lista as contas de saque cadastradas pelo usuario.

#### Hook: useGetWithdrawsMethods

```typescript
const { data: methods, isLoading } = useGetWithdrawsMethods(userId, type);
```

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `userId` | `string` | ID do usuario |
| `type` | `string` | Tipo do metodo de saque |

#### API

```
GET /users/{tenantId}/withdraw-accounts/{userId}
Authorization: Bearer <token>
API: ID
```

---

### Cadastrar Metodo de Saque

Cadastra uma nova conta de saque (ex: conta bancaria, PIX, etc.).

#### Hook: useCreateWithdrawMethod

```typescript
const createMethod = useCreateWithdrawMethod();

createMethod.mutate(payload, {
  // payload: AddWithdrawAccountDto do @w3block/sdk-id
  onSuccess: () => {
    // Metodo criado com sucesso
  },
});
```

#### SDK

```typescript
sdk.api.users.addUserWithdrawAccount(tenantId, userId, payload)
// tenantId e userId sao obtidos automaticamente pelo hook
```

> **Nota:** O `AddWithdrawAccountDto` e definido pelo pacote `@w3block/sdk-id`. Consulte o SDK para os campos especificos.

---

### Remover Metodo de Saque

Remove uma conta de saque existente.

#### Hook: useDeleMethodPayment

```typescript
const deleteMethod = useDeleMethodPayment();

deleteMethod.mutate(accountId, {
  onSuccess: () => {
    // Metodo removido com sucesso
  },
});
```

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `accountId` | `string` | ID da conta de saque a remover |

#### SDK

```typescript
sdk.api.users.deleteUserWithdrawAccount(tenantId, userId, accountId)
```

---

## Acoes Administrativas

### Consultar Saque (Admin)

Visao admin inclui dados do usuario (nome, email).

#### Hook: useGetSpecificWithdrawAdmin

```typescript
const { data, isFetching, refetch } = useGetSpecificWithdrawAdmin(id);

// Dados adicionais na visao admin:
// data?.data?.user?.name  -> Nome do usuario
// data?.data?.user?.email -> Email do usuario
```

#### API

```
GET /{companyId}/withdraws/admin/{id}
Authorization: Bearer <token>
API: KEY
```

---

### Fluxo Admin: Decisoes por Status

```
Status atual            | Acoes disponiveis
------------------------|-----------------------------------
pending                 | [Reter Fundos] [Recusar]
escrowing_resources     | (aguardar - mensagem de espera)
ready_to_transfer_funds | [Concluir Saque]
concluded               | (nenhuma - exibe comprovante)
failed                  | (nenhuma)
refused                 | (nenhuma - exibe motivo)
```

---

## Componentes de Referencia

| Componente | Arquivo | Descricao |
|------------|---------|-----------|
| `WithdrawAdminActions` | `auth/components/WithdrawAdminActions.tsx` | Painel admin para gerenciar saque individual |
| `WithdrawInternal` | `auth/components/WithdrawInternal.tsx` | Visao do usuario de um saque especifico |
| `InputWithdrawCommerce` | `auth/components/InputWithdrawCommerce.tsx` | Input de upload de comprovante |

---

## Resumo de Hooks

| Hook | Arquivo | Tipo | API | Role |
|------|---------|------|-----|------|
| `useGetWithdrawsMethods` | `auth/hooks/useGetWithdrawsMethods.ts` | `usePaginatedQuery` | ID | Usuario |
| `useCreateWithdrawMethod` | `auth/hooks/useCreateWithdrawMethod.ts` | `useMutation` | ID (SDK) | Usuario |
| `useDeleMethodPayment` | `auth/hooks/useDeleMethodPayment.ts` | `useMutation` | ID (SDK) | Usuario |
| `useRequestWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` | KEY | Usuario |
| `useGetSpecificWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `usePrivateQuery` | KEY | Usuario |
| `useGetSpecificWithdrawAdmin` | `auth/hooks/useRequestWithdraw.ts` | `usePrivateQuery` | KEY | Admin |
| `useEscrowWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` | KEY | Admin |
| `useConcludeWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` | KEY | Admin |
| `useRefuseWithdraw` | `auth/hooks/useRequestWithdraw.ts` | `useMutation` | KEY | Admin |

---

## Armadilhas e Dicas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | `amount` vem como string da API | Sempre usar `parseFloat(amount).toFixed(2)` para exibicao |
| 2 | Withdraw usa KEY API, metodos de saque usam ID API | Verificar `W3blockAPI.KEY` vs `W3blockAPI.ID` no hook correto |
| 3 | `useDeleMethodPayment` tem nome incomum | Nome do hook e "Dele" (nao "Delete") — usar exatamente como esta |
| 4 | Conclusao exige `receiptAssetId` | Fazer upload do comprovante antes de chamar conclude |
| 5 | Recusa exige `reason` nao vazio | Validar que o campo reason nao esta vazio antes de enviar |
| 6 | Status `escrowing_resources` e transitorio | Admin deve aguardar transicao para `ready_to_transfer_funds` |
| 7 | Varias exports no mesmo arquivo | `useRequestWithdraw.ts` exporta 6 hooks diferentes — importar pelo nome correto |
| 8 | Escrow e obrigatorio antes de conclude | O fluxo admin e: request -> escrow -> conclude. Nao pode pular escrow |
| 9 | Refuse so funciona no status `pending` | Apos escrow, nao e mais possivel recusar — somente concluir |
| 10 | `companyId` vem do `useCompanyConfig()` | Os hooks de withdraw obtem `companyId` automaticamente via context |
| 11 | Upload de comprovante antes de conclude | Use `useUploadAssets` para obter o `receiptAssetId` antes de chamar conclude |
| 12 | Polling de status apos escrow | Status `escrowing_resources` pode levar segundos — implemente polling ou use `refetch()` |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Verificar saldo antes de sacar | Consultar saldo do token | [FLOW_TOKENS_BALANCE](../tokens/FLOW_TOKENS_BALANCE.md) |
| Conferir wallet de origem | Listar NFTs e tokens na wallet | [FLOW_TOKENS_NFT_WALLET](../tokens/FLOW_TOKENS_NFT_WALLET.md) |
| Autenticar usuario para saque | Obter access_token via login | [FLOW_SIGNUP](../auth/FLOW_SIGNUP.md) |
