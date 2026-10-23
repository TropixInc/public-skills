---
id: FLOW_LOYALTY_STAKING
title: "Loyalty - Staking"
module: loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-10-23"
authors:
  - fernandodevpascoal
tags:
  - loyalty
  - staking
depends_on:
  - LOYALTY_API_REFERENCE
---

# Flow: Loyalty Staking

Fluxo para consultar o resumo de staking e resgatar pontos travados de um programa de fidelidade.

---

## Visao Geral

O sistema de staking permite que pontos de fidelidade fiquem "travados" por um periodo, rendendo beneficios. Este fluxo cobre:

1. **Resumo de staking** — Consultar todos os registros de staking do usuario
2. **Resgate de staking** — Resgatar pontos que ja atingiram o prazo

Ambos utilizam a **Key API** (`https://api.w3block.io`).

---

## 1. Resumo de Staking

### Endpoint

```
GET /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}
```

### Hook: `useStakingSummary`

**Arquivo:** `src/modules/shared/hooks/useStaking.ts`

```typescript
const { data: stakingSummary, isLoading } = useStakingSummary({
  page: 1,
  limit: 10,
  sortBy: 'createdAt',
  orderBy: 'DESC'
});
```

### Comportamento

- Usa `usePrivateQuery` — query automatica com autenticacao
- O `userId` e obtido automaticamente da sessao (`usePixwaySession`)
- O `loyaltyId` e obtido automaticamente do `loyaltyWallet` (`useUserWallet`)
- Usa o primeiro loyalty wallet disponivel: `loyaltyWallet[0].loyaltyId`
- Parametros de paginacao sao opcionais (tem defaults)

### Parametros de Paginacao (QueryParams)

| Param | Tipo | Default | Descricao |
|-------|------|---------|-----------|
| `page` | number | 1 | Pagina atual |
| `limit` | number | 10 | Itens por pagina |
| `sortBy` | string | `createdAt` | Campo de ordenacao |
| `orderBy` | string | `DESC` | Direcao: `ASC` ou `DESC` |

### Condicoes de habilitacao

```typescript
enabled: !!companyId && !!session?.id
```

Adicionalmente, a query so e montada corretamente se `loyaltyWallet` tem pelo menos um item.

### Resposta: SummaryDto

Cada registro de staking retornado:

```typescript
interface SummaryDto {
  id: string;                    // UUID do registro de staking
  actionId: string;              // ID da acao associada
  amount: string;                // Quantidade de pontos travados
  companyId: string;
  createdAt: string;             // Data de criacao
  executeAt: string;             // Data de execucao programada
  expiresAt: string;             // Data de expiracao
  fromAddress: string;           // Endereco de origem
  toAddress: string;             // Endereco de destino
  isExpired: boolean;            // Se o staking ja expirou
  isPoolGrouping: boolean;       // Se faz parte de agrupamento em pool
  isWithdrawal: boolean;         // Se e uma operacao de retirada
  loyaltyId: string;             // ID do programa de fidelidade
  parentMinterId: string;        // ID do mint pai (se aplicavel)
  parentTransferId: string;      // ID da transferencia pai (se aplicavel)
  poolAddress: string;           // Endereco do pool
  rollbackActionId: string;      // ID da acao de rollback
  sendEmail: boolean;            // Se envia email de notificacao
  stakingRuleId: string;         // ID da regra de staking aplicada
  status: string;                // Status atual do staking
  updatedAt: string;
  userId: string;
}
```

### Campos importantes para a UI

| Campo | Uso na interface |
|-------|-----------------|
| `amount` | Quantidade de pontos travados (exibir ao usuario) |
| `executeAt` | Quando o staking sera executado/liberado |
| `expiresAt` | Quando o staking expira (se nao resgatado) |
| `isExpired` | Indicar visualmente se ja expirou |
| `status` | Status atual: mostrar badge/indicador |
| `stakingRuleId` | Identificar qual regra de staking foi aplicada |

---

## 2. Resgatar Staking

### Endpoint

```
PATCH /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem
```

### Hook: `useRedeemStaking`

**Arquivo:** `src/modules/shared/hooks/useStaking.ts`

```typescript
const { mutateAsync: redeemStaking } = useRedeemStaking();

await redeemStaking();
```

### Comportamento

- Usa `useMutation` com `PATCH`
- **Nao recebe body** — os parametros sao todos no path
- O `userId` e obtido da sessao (`usePixwaySession`)
- O `loyaltyId` e obtido do `loyaltyWallet` (`useUserWallet`)
- Resgata todos os pontos elegiveis de uma vez
- Em caso de erro, loga no console e lanca excecao

### Fluxo de Resgate

```
Usuario visualiza resumo de staking
        |
        v
Verifica se ha pontos elegiveis para resgate
(isExpired == false && status adequado)
        |
        v
Usuario clica "Resgatar"
        |
        v
useRedeemStaking().mutateAsync()
PATCH /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem
        |
        v
Pontos resgatados com sucesso
        |
        v
Invalidar cache / refetch do staking summary
        |
        v
Atualizar saldo do usuario (useGetUserBalance)
```

---

## Fluxo Completo: Consultar e Resgatar

```
Tela de Staking carrega
        |
        v
useStakingSummary() executa automaticamente
GET /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}?page=1&limit=10&sortBy=createdAt&orderBy=DESC
        |
        v
Exibir lista de registros de staking:
- Quantidade travada (amount)
- Data de execucao (executeAt)
- Data de expiracao (expiresAt)
- Status (expirado ou ativo)
        |
        v
Usuario tem pontos para resgatar?
    |           |
   Sim         Nao -> Exibir mensagem
    |
    v
Botao "Resgatar" habilitado
        |
        v
useRedeemStaking().mutateAsync()
PATCH /.../staking/{loyaltyId}/redeem
        |
        v
Sucesso -> Refetch summary + balance
```

---

## Dependencias de Dados

O fluxo de staking depende de dois dados que sao obtidos automaticamente:

### 1. Session (userId)

```typescript
const { data: session } = usePixwaySession();
// session.id = userId
```

O usuario deve estar autenticado. Se `session.id` nao existir, as queries ficam desabilitadas.

### 2. Loyalty Wallet (loyaltyId)

```typescript
const { loyaltyWallet } = useUserWallet();
// loyaltyWallet[0].loyaltyId = ID do programa de fidelidade
```

O usuario deve ter pelo menos um loyalty wallet. Se `loyaltyWallet` estiver vazio, o `loyaltyId` sera string vazia e a API pode retornar erro.

---

## Exemplo API-first (sem SDK)

```typescript
/** Consultar resumo de staking do usuario */
const fetchStakingSummary = async (
  companyId: string,
  userId: string,
  loyaltyId: string,
  token: string,
  params: { page?: number; limit?: number; sortBy?: string; orderBy?: string } = {}
) => {
  const query = new URLSearchParams({
    page: String(params.page ?? 1),
    limit: String(params.limit ?? 10),
    sortBy: params.sortBy ?? 'createdAt',
    orderBy: params.orderBy ?? 'DESC',
  }).toString();

  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/users/${userId}/staking/${loyaltyId}?${query}`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }

  return response.json(); // Lista paginada de SummaryDto
};

/** Resgatar pontos de staking */
const redeemStaking = async (
  companyId: string,
  userId: string,
  loyaltyId: string,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/users/${userId}/staking/${loyaltyId}/redeem`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }

  return response.json();
};
```

---

## Observacoes Tecnicas

1. **Primeiro loyalty wallet:** Tanto o summary quanto o redeem usam `loyaltyWallet[0].loyaltyId`. Se o usuario tiver multiplos programas de fidelidade, apenas o primeiro sera considerado. Para staking de outros programas, seria necessario passar o loyaltyId manualmente.

2. **Query string manual:** O hook monta a query string manualmente com `URLSearchParams`, concatenando com `?` ao path. Todos os parametros de paginacao sao incluidos.

3. **refetchOnWindowFocus: false:** O staking summary nao e recarregado ao focar a janela. Isso evita chamadas desnecessarias. Para atualizar, usar `refetch()` manualmente apos o resgate.

4. **refetchOnMount: false:** O summary tambem nao e recarregado ao remontar o componente. O cache do React Query e utilizado.

5. **Redeem sem body:** O endpoint de resgate nao recebe dados no body. Ele resgata todos os pontos elegiveis do usuario para aquele programa de fidelidade.

6. **amount e string:** O campo `amount` do `SummaryDto` e string. Converter com `parseFloat()` ou `Number()` antes de exibir ou calcular.
