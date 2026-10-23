---
id: FLOW_LOYALTY_ADMIN_REPORTS
title: "Loyalty - Relatórios administrativos"
module: loyalty
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-10-23"
authors:
  - fernandodevpascoal
tags:
  - loyalty
  - reports
  - admin
depends_on:
  - LOYALTY_API_REFERENCE
---

# Flow: Loyalty Admin Reports

Fluxo para consultar relatorios administrativos de historico de transacoes ERC-20 no sistema de fidelidade.

---

## Visao Geral

Os relatorios administrativos permitem visualizar o historico de transacoes de tokens ERC-20 associados aos programas de fidelidade. Existem dois niveis de acesso:

1. **Relatorio Admin** — Todas as transacoes de todos os operadores (role: `admin` ou `superAdmin`)
2. **Relatorio por Operador** — Apenas as transacoes do operador logado (role: `loyaltyOperator`)

Ambos utilizam a **Key API** (`https://api.w3block.io`).

---

## Pre-requisito: Carregar Loyalties

Ambos os relatorios dependem do `contractId` do primeiro programa de fidelidade da empresa. Este dado e obtido via `useLoyaltiesInfo()`:

```typescript
const { loyalties } = useLoyaltiesInfo();
const loyaltyContractId = loyalties[0].contractId;
```

**Importante:** O `{loyaltyId}` nos endpoints de ERC-20 history e na verdade o `contractId` do `LoyaltyInterface`, **nao** o `id`.

Se `loyalties` estiver vazio ou `loyalties[0].contractId` for undefined, os relatorios ficam desabilitados.

---

## 1. Relatorio Admin (Todas as Transacoes)

### Endpoint

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history
```

### Hook: `useGetAllReportsAdmin`

**Arquivo:** `src/modules/business/hooks/useGetAllReportsAdmin.tsx`

```typescript
const { data: reports, isLoading } = useGetAllReportsAdmin(filter);
```

### Parametros

| Param | Tipo | Descricao |
|-------|------|-----------|
| `filter` | object | Filtros de paginacao/busca (page, limit, search, etc.) |

### Comportamento

- Usa `useQuery` com fetch automatico
- Limpa o filtro com `cleanObject` antes de montar query string
- Substitui `{companyId}` e `{loyaltyId}` (com `loyalties[0].contractId`)
- Retorna `ErcTokenHistoryInterfaceResponse`

### Condicoes de habilitacao

```typescript
enabled:
  loyalties.length > 0 &&
  loyalties[0].contractId != undefined &&
  (profile?.roles.includes('admin') ||
    profile?.roles.includes('superAdmin')) &&
  !!companyId
```

**Todas** as condicoes devem ser verdadeiras:
- Pelo menos um programa de fidelidade carregado
- `contractId` do primeiro loyalty definido
- Usuario com role `admin` ou `superAdmin`
- `companyId` definido

### Roles necessarias

| Role | Acesso |
|------|--------|
| `admin` | Sim |
| `superAdmin` | Sim |
| `loyaltyOperator` | **Nao** — usar relatorio por operador |
| Outros | Nao |

---

## 2. Relatorio por Operador

### Endpoint

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history/operator/{operatorId}
```

### Hook: `useGetAllReportsByOperatorId`

**Arquivo:** `src/modules/business/hooks/useGetAllReportsByOperatorId.tsx`

```typescript
const { data: reports, isLoading } = useGetAllReportsByOperatorId(filter);
```

### Parametros

| Param | Tipo | Descricao |
|-------|------|-----------|
| `filter` | object | Filtros de paginacao/busca |

### Comportamento

- Usa `useQuery` com fetch automatico
- Substitui `{companyId}`, `{loyaltyId}` (contractId), e `{operatorId}` (profile.id)
- O `operatorId` e automaticamente o `profile.id` do usuario logado
- Retorna apenas as transacoes realizadas por este operador

### Condicoes de habilitacao

```typescript
enabled:
  loyalties.length > 0 &&
  loyalties[0].contractId != undefined &&
  profile?.roles.includes('loyaltyOperator') &&
  !!companyId &&
  !!profile?.id
```

**Todas** as condicoes devem ser verdadeiras:
- Pelo menos um programa de fidelidade carregado
- `contractId` do primeiro loyalty definido
- Usuario com role `loyaltyOperator`
- `companyId` definido
- `profile.id` definido

### Roles necessarias

| Role | Acesso |
|------|--------|
| `loyaltyOperator` | Sim (apenas suas proprias transacoes) |
| `admin` | **Nao** — usar relatorio admin |
| `superAdmin` | **Nao** — usar relatorio admin |
| Outros | Nao |

---

## Decisao Automatica: Qual Relatorio Usar

O sistema seleciona o relatorio automaticamente com base na role do usuario:

```
Usuario autenticado
        |
        v
Verificar roles do perfil
        |
        +--- admin ou superAdmin
        |        |
        |        v
        |   useGetAllReportsAdmin(filter)
        |   GET /{companyId}/erc20-tokens/{loyaltyId}/history
        |
        +--- loyaltyOperator
                 |
                 v
            useGetAllReportsByOperatorId(filter)
            GET /{companyId}/erc20-tokens/{loyaltyId}/history/operator/{operatorId}
```

---

## Resposta: ErcTokenHistoryInterfaceResponse

Ambos os endpoints retornam a mesma estrutura:

```typescript
{
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
```

### Estrutura de cada transacao (Erc20TokenHistory)

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `id` | string | UUID da transacao |
| `createdAt` | string | Data de criacao |
| `type` | `Erc20ActionType` | `transfer`, `burn`, `minter` |
| `status` | `Erc20ActionStatus` | `created`, `started`, `success`, `failed`, `wait_event`, `external` |
| `contractAddress` | string | Endereco do contrato |
| `chainId` | number | ID da rede blockchain |
| `sender` | string | Endereco do remetente |
| `txHash` | string | Hash da transacao na blockchain |
| `request.amount` | string | Quantidade de tokens |
| `request.to` | string | Endereco de destino |
| `request.from` | string | Endereco de origem |
| `executeAt` | string | Data de execucao |
| `to` | `FromToInterface` | Dados do destinatario |
| `from` | `FromToInterface` | Dados do remetente |
| `loyaltiesTransactions` | `LoyaltiesTransaction[]` | Transacoes de loyalty associadas |
| `metadata.description` | string | Descricao |
| `metadata.commerce` | object | Dados do comercio |

### FromToInterface

```typescript
interface FromToInterface {
  id?: string;
  phone?: string;
  type?: string;
  user_name?: string;
  wallet?: string;
  cpf?: string;
  email?: string;
}
```

### Metadados da Transacao de Loyalty

Cada `LoyaltiesTransaction` pode conter metadados ricos:

| Campo | Descricao |
|-------|-----------|
| `metadata.operatorName` | Nome do operador que executou a transacao |
| `metadata.operatorUserId` | UUID do operador |
| `metadata.buyerName` | Nome do comprador |
| `metadata.buyerEmail` | Email do comprador |
| `metadata.userName` | Nome do usuario |
| `metadata.userEmail` | Email do usuario |
| `metadata.pointsCashback` | Pontos de cashback aplicados |
| `metadata.description` | Descricao da transacao |
| `metadata.loyaltiesDeferredId` | ID do registro diferido (se aplicavel) |
| `metadata.commerce.orderId` | ID do pedido associado |
| `metadata.commerce.deliverId` | ID de entrega |
| `metadata.poolMetadata` | Dados do pool (saldo, locked, etc.) |

---

## Fluxo Completo: Tela de Relatorios

```
Tela de Relatorios carrega
        |
        v
useLoyaltiesInfo() -> Carregar loyalties
        |
        v
loyalties[0].contractId disponivel?
    |           |
   Nao         Sim
    |           |
    v           v
  Aguardar   Verificar role do usuario
              |
              +--- admin/superAdmin
              |        |
              |        v
              |   useGetAllReportsAdmin(filter)
              |   GET /{companyId}/erc20-tokens/{contractId}/history?...
              |        |
              |        v
              |   Exibir todas as transacoes
              |   (com dados de todos os operadores)
              |
              +--- loyaltyOperator
                       |
                       v
                  useGetAllReportsByOperatorId(filter)
                  GET /{companyId}/erc20-tokens/{contractId}/history/operator/{profileId}?...
                       |
                       v
                  Exibir apenas transacoes deste operador
        |
        v
Aplicar filtros (paginacao, busca, data)
        |
        v
Atualizar query string dos filtros
        |
        v
React Query refetch automatico (query key muda)
```

---

## Endpoint Auxiliar: Historico por Usuario

Alem dos relatorios admin, existe um endpoint para consultar o historico de um usuario especifico:

```
GET /{companyId}/erc20-tokens/{loyaltyId}/history/{userId}
```

**Enum:** `GET_ERC_TOKENS_BY_LOYALTY_ID`

Este endpoint pode ser usado para exibir o extrato detalhado de um usuario individual (ex: ao clicar em um usuario na lista admin).

---

## Filtros Dinamicos

Os filtros sao passados como objeto e transformados em query string:

```typescript
const filter = {
  page: '1',
  limit: '20',
  search: 'joao',
  startDate: '2024-01-01',
  endDate: '2024-12-31'
};

// cleanObject remove campos vazios/nulos
const cleaned = cleanObject(filter);
const queryString = new URLSearchParams(cleaned).toString();
// Resultado: "page=1&limit=20&search=joao&startDate=2024-01-01&endDate=2024-12-31"
```

---

## Exemplo API-first (sem SDK)

```typescript
/** Consultar relatorio admin — todas as transacoes (role: admin/superAdmin) */
const fetchAdminReports = async (
  companyId: string,
  contractId: string,
  token: string,
  filter?: Record<string, string>
) => {
  const query = filter ? `?${new URLSearchParams(filter).toString()}` : '';

  const response = await fetch(
    `https://api.w3block.io/${companyId}/erc20-tokens/${contractId}/history${query}`,
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

  return response.json(); // ErcTokenHistoryInterfaceResponse
};

/** Consultar relatorio por operador (role: loyaltyOperator) */
const fetchOperatorReports = async (
  companyId: string,
  contractId: string,
  operatorId: string,
  token: string,
  filter?: Record<string, string>
) => {
  const query = filter ? `?${new URLSearchParams(filter).toString()}` : '';

  const response = await fetch(
    `https://api.w3block.io/${companyId}/erc20-tokens/${contractId}/history/operator/${operatorId}${query}`,
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

  return response.json(); // ErcTokenHistoryInterfaceResponse
};
```

---

## Observacoes Tecnicas

1. **contractId, nao id:** O `{loyaltyId}` no path dos endpoints ERC-20 e o `contractId` do `LoyaltyInterface`, nao o `id`. Esse e o erro mais comum ao implementar relatorios.

2. **Primeiro loyalty apenas:** Os hooks usam `loyalties[0].contractId`. Se a empresa tiver multiplos programas de fidelidade, apenas o primeiro e considerado nos relatorios. Para relatorios de outros programas, seria necessario modificar os hooks.

3. **Separacao por role:** A decisao entre relatorio admin e por operador e feita pelas condicoes de `enabled` das queries. Apenas uma delas sera ativada por vez, baseado nas roles do usuario logado.

4. **operatorId automatico:** No relatorio por operador, o `{operatorId}` e automaticamente preenchido com `profile.id`. O operador so consegue ver suas proprias transacoes.

5. **cleanObject:** A funcao `cleanObject` importada de `shared/utils/validators` remove campos com valores falsy do objeto de filtro. Isso evita enviar `page=undefined` na query string.

6. **Query key inclui filtros:** A query key do React Query inclui a query string dos filtros, garantindo que mudancas nos filtros (paginacao, busca) disparem um novo fetch automaticamente.

7. **useLoyaltiesInfo:** Este hook (usado internamente) provavelmente encapsula `useGetCompanyLoyalties` e fornece o array `loyalties` de forma mais conveniente. A dependencia entre carregar loyalties e carregar relatorios e gerenciada pelo `enabled` das queries.
