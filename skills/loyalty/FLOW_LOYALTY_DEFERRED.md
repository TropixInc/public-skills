# Flow: Loyalty Deferred

Fluxo para consultar e exportar recompensas diferidas (cashback agendado) do sistema de fidelidade.

---

## Visao Geral

Recompensas diferidas sao pontos de fidelidade que serao creditados no futuro, com base em regras de compensacao (`compensateRule.period`). Este fluxo cobre:

1. **Deferred por usuario** — Consultar recompensas diferidas de um usuario especifico
2. **Deferred geral** — Consultar todas as recompensas diferidas da empresa
3. **Exportar XLSX** — Baixar relatorio de recompensas diferidas em planilha

Todos utilizam a **Key API** (`https://api.w3block.io`).

---

## Sistema de Filtros de Status

Todos os tres endpoints compartilham a mesma logica de filtro por status. Os status sao cumulativos na query string:

### Status base (sempre incluidos)

```
status=deferred&status=pending&status=started&status=pool&status=success
```

### Filtro por tipo

| Tipo | Parametro adicional | Descricao |
|------|---------------------|-----------|
| `scheduled` | `withdrawBlockedOnly=true` | Recompensas agendadas (saque bloqueado) |
| `received` | `withdrawBlockedOnly=false` | Recompensas ja recebidas (saque liberado) |
| (nenhum) | Nenhum | Todas as recompensas, sem filtro de saque |

### Logica no codigo

```typescript
const statusQuery = () => {
  if (status === 'scheduled') {
    return 'status=deferred&status=pending&status=started&status=pool&status=success&withdrawBlockedOnly=true';
  } else if (status === 'received') {
    return 'status=deferred&status=pending&status=started&status=pool&status=success&withdrawBlockedOnly=false';
  }
  return 'status=deferred&status=pending&status=started&status=pool&status=success';
};
```

---

## 1. Deferred por Usuario

### Endpoint

```
GET /{companyId}/loyalties/users/deferred/{userId}
```

### Hook: `useGetDeferredByUserId`

**Arquivo:** `src/modules/business/hooks/useGetDeferredByUserId.ts`

```typescript
const { data, isLoading } = useGetDeferredByUserId(
  userId,           // string — UUID do usuario
  filter,           // object — filtros de paginacao/busca (opcional)
  enabled,          // boolean — habilitar/desabilitar a query
  status            // string — 'scheduled' | 'received' | undefined
);
```

### Parametros

| Param | Tipo | Obrigatorio | Descricao |
|-------|------|:-----------:|-----------|
| `userId` | string | Sim | UUID do usuario |
| `filter` | object | Nao | Filtros de paginacao (page, limit, etc.) |
| `enabled` | boolean | Nao | Habilitar a query (default: false implicitamente) |
| `status` | string | Nao | `'scheduled'`, `'received'`, ou undefined |

### Comportamento

- Usa `useQuery` com fetch automatico
- Limpa o objeto de filtro com `cleanObject` (remove campos nulos/undefined)
- Monta query string com `URLSearchParams` + filtro de status
- `retry: false` — nao retenta em caso de erro
- `refetchOnWindowFocus: false` — nao recarrega ao focar janela
- `refetchOnMount: false` — nao recarrega ao remontar componente

### Condicoes de habilitacao

```typescript
enabled: !!userId && !!companyId && enabled
```

Todas as tres condicoes devem ser verdadeiras.

### Resposta: ErcTokenHistoryInterfaceResponse

```typescript
interface ErcTokenHistoryInterfaceResponse {
  items: Erc20TokenHistory[];     // Lista de transacoes
  summary?: {
    balance?: number;             // Saldo resumido
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

---

## 2. Deferred Geral (Todos os Usuarios)

### Endpoint

```
GET /{companyId}/loyalties/users/deferred
```

### Hook: `useGetDeferred`

**Arquivo:** `src/modules/business/hooks/useGetDeferred.tsx`

```typescript
const { data, isLoading } = useGetDeferred(
  filter,           // object — filtros de paginacao/busca (opcional)
  enabled,          // boolean — habilitar/desabilitar a query
  status            // string — 'scheduled' | 'received' | undefined
);
```

### Parametros

| Param | Tipo | Obrigatorio | Descricao |
|-------|------|:-----------:|-----------|
| `filter` | object | Nao | Filtros de paginacao (page, limit, etc.) |
| `enabled` | boolean | Nao | Habilitar a query |
| `status` | string | Nao | `'scheduled'`, `'received'`, ou undefined |

### Comportamento

- Identico ao "Deferred por Usuario", mas sem `{userId}` no path
- Retorna recompensas diferidas de **todos** os usuarios da empresa
- Tipicamente usado por administradores
- `retry: false` e `refetchOnWindowFocus: false`

### Condicoes de habilitacao

```typescript
enabled: !!companyId && enabled
```

### Resposta

Mesma estrutura `ErcTokenHistoryInterfaceResponse` do endpoint por usuario.

---

## 3. Exportar Diferidos em XLSX

### Endpoint

```
GET /{companyId}/loyalties/users/deferred/xlsx
```

### Hook: `useGetXlsxDeferred`

**Arquivo:** `src/modules/business/hooks/useGetXlsxDeferred.tsx`

```typescript
const { mutateAsync: downloadXlsx } = useGetXlsxDeferred();

const response = await downloadXlsx({
  filter: { page: '1', limit: '100' },
  status: 'scheduled'
});

// response.data e um Blob (arquivo XLSX)
```

### Comportamento

- Usa `useMutation` — download sob demanda (imperativo)
- Configura `responseType: 'blob'` no axios
- Mesma logica de filtros de status dos outros endpoints deferred
- Retorna um Blob que deve ser tratado como arquivo para download

### Parametros do mutate

```typescript
{
  filter?: Record<string, string>;        // Filtros de paginacao/busca
  status?: 'scheduled' | 'received';      // Tipo de filtro de status
}
```

### Tratamento do download

```typescript
// Exemplo de como tratar o Blob para download:
const response = await downloadXlsx({ filter, status });
const blob = new Blob([response.data]);
const url = window.URL.createObjectURL(blob);
const link = document.createElement('a');
link.href = url;
link.download = 'deferred-rewards.xlsx';
link.click();
window.URL.revokeObjectURL(url);
```

---

## Fluxo Completo: Tela de Recompensas Diferidas

```
Tela de Recompensas Diferidas carrega
        |
        v
Selecionar aba: "Agendadas" ou "Recebidas"
        |
        +--- Agendadas (status = 'scheduled')
        |        |
        |        v
        |   useGetDeferred(filter, true, 'scheduled')
        |   GET /{companyId}/loyalties/users/deferred?...&withdrawBlockedOnly=true
        |        |
        |        v
        |   Exibir lista de recompensas com saque bloqueado
        |
        +--- Recebidas (status = 'received')
                 |
                 v
            useGetDeferred(filter, true, 'received')
            GET /{companyId}/loyalties/users/deferred?...&withdrawBlockedOnly=false
                 |
                 v
            Exibir lista de recompensas com saque liberado

        |
        v
Botao "Exportar XLSX"
        |
        v
useGetXlsxDeferred().mutateAsync({ filter, status })
GET /{companyId}/loyalties/users/deferred/xlsx?...
        |
        v
Download do arquivo XLSX
```

### Fluxo para usuario especifico

```
Tela de detalhes do usuario
        |
        v
useGetDeferredByUserId(userId, filter, true, 'scheduled')
GET /{companyId}/loyalties/users/deferred/{userId}?...
        |
        v
Exibir recompensas diferidas deste usuario
```

---

## Estrutura de um Registro Diferido (Erc20TokenHistory)

Cada item na lista tem a seguinte estrutura relevante:

| Campo | Descricao |
|-------|-----------|
| `id` | UUID da transacao |
| `type` | Tipo da acao: `transfer`, `burn`, `minter` |
| `status` | Status: `created`, `started`, `success`, `failed`, `wait_event`, `external` |
| `request.amount` | Quantidade de pontos/tokens |
| `request.to` | Endereco de destino |
| `executeAt` | Data programada para execucao |
| `metadata.description` | Descricao da transacao |
| `metadata.commerce` | Dados do comercio (se aplicavel) |
| `loyaltiesTransactions` | Array de transacoes de loyalty associadas |
| `to` | Dados do destinatario (nome, email, wallet, etc.) |
| `from` | Dados do remetente |

### Campos da LoyaltiesTransaction

| Campo | Descricao |
|-------|-----------|
| `amount` | Quantidade de pontos |
| `executeAt` | Data de execucao |
| `status` | Status da transacao de loyalty |
| `isWithdrawal` | Se e uma retirada |
| `metadata.pointsCashback` | Pontos de cashback |
| `metadata.description` | Descricao |
| `metadata.operatorName` | Nome do operador que executou |
| `metadata.loyaltiesDeferredId` | ID do registro diferido |

---

## Exemplo API-first (sem SDK)

```typescript
/** Montar query string de status para endpoints deferred */
const buildDeferredStatusQuery = (status?: 'scheduled' | 'received'): string => {
  const base = 'status=deferred&status=pending&status=started&status=pool&status=success';
  if (status === 'scheduled') return `${base}&withdrawBlockedOnly=true`;
  if (status === 'received') return `${base}&withdrawBlockedOnly=false`;
  return base;
};

/** Consultar recompensas diferidas de um usuario */
const fetchDeferredByUser = async (
  companyId: string,
  userId: string,
  token: string,
  status?: 'scheduled' | 'received',
  filter?: Record<string, string>
) => {
  const statusQuery = buildDeferredStatusQuery(status);
  const filterQuery = filter ? `&${new URLSearchParams(filter).toString()}` : '';

  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/users/deferred/${userId}?${statusQuery}${filterQuery}`,
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

/** Consultar todas as recompensas diferidas da empresa */
const fetchDeferredAll = async (
  companyId: string,
  token: string,
  status?: 'scheduled' | 'received',
  filter?: Record<string, string>
) => {
  const statusQuery = buildDeferredStatusQuery(status);
  const filterQuery = filter ? `&${new URLSearchParams(filter).toString()}` : '';

  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/users/deferred?${statusQuery}${filterQuery}`,
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

1. **cleanObject:** Os filtros sao limpos com `cleanObject` antes de montar a query string. Isso remove campos com valor `null`, `undefined` ou string vazia.

2. **Status multiplos na query:** Os status sao enviados como parametros repetidos: `status=deferred&status=pending&...`. Isso e necessario porque a API espera multiplos valores para o mesmo parametro.

3. **XLSX como Blob:** O endpoint de XLSX retorna binario. O hook configura `responseType: 'blob'` no axios. Nao tentar parsear como JSON.

4. **Paginacao via filtro:** A paginacao nao e gerenciada pelo hook. O componente deve passar os parametros de paginacao no objeto `filter` (ex: `{ page: '1', limit: '10' }`).

5. **Query key inclui status:** A query key do React Query inclui o status, garantindo que trocar entre "agendadas" e "recebidas" faca um novo fetch (e nao use cache incorreto).

6. **withdrawBlockedOnly:** Este parametro e a chave para diferenciar recompensas agendadas de recebidas. `true` = saque bloqueado (agendadas), `false` = saque liberado (recebidas).
