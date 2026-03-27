# Flow: Loyalty Balance

Fluxo para consultar o saldo de pontos de fidelidade de um usuario e listar os programas de fidelidade da empresa.

---

## Visao Geral

Este fluxo cobre duas operacoes fundamentais do sistema de loyalty:

1. **Consultar saldo** — Buscar o saldo de pontos de um usuario especifico
2. **Listar loyalties** — Obter todos os programas de fidelidade da empresa (admin/operador)

Ambas utilizam a **Key API** (`https://api.w3block.io`).

---

## 1. Consultar Saldo do Usuario

### Endpoint

```
GET /{companyId}/loyalties/users/balance/{userId}
```

### Hook: `useGetUserBalance`

**Arquivo:** `src/modules/business/hooks/useGetUserBalance.tsx`

```typescript
const { mutateAsync: getUserBalance } = useGetUserBalance();

// Chamar sob demanda:
const balance = await getUserBalance(userId);
```

### Comportamento

- Usa `useMutation` (nao `useQuery`) — a busca de saldo e **imperativa**, nao automatica
- API utilizada: `W3blockAPI.KEY`
- Substitui `{companyId}` e `{userId}` na rota
- Retorna os dados de saldo diretamente (`response.data`)
- Em caso de erro, loga no console e lanca excecao via `handleNetworkException`

### Quando usar

- Ao abrir a tela de saldo/carteira do usuario
- Antes de iniciar um pagamento com pontos (para verificar saldo disponivel)
- Apos concluir um pagamento (para atualizar o saldo exibido)

### Fluxo

```
Usuario abre tela de saldo
        |
        v
Chamar mutateAsync(userId)
        |
        v
GET /{companyId}/loyalties/users/balance/{userId}
        |
        v
Exibir saldo retornado
```

---

## 2. Listar Programas de Fidelidade da Empresa

### Endpoint

```
GET /{companyId}/loyalties/admin?limit=49
```

### Hook: `useGetCompanyLoyalties`

**Arquivo:** `src/modules/business/hooks/useGetCompanyLoyalties.tsx`

```typescript
const { data: loyalties, isLoading } = useGetCompanyLoyalties();
```

### Comportamento

- Usa `useQuery` — fetch automatico quando as condicoes sao atendidas
- API utilizada: `W3blockAPI.KEY`
- Retorna `items` do response como `LoyaltyInterface[]`
- Query key inclui: rota, companyId, profile.id
- Limite fixo de 49 registros via query param

### Condicoes de habilitacao (enabled)

A query so executa quando **todas** as condicoes sao verdadeiras:

```typescript
profile?.id != null &&
companyId != null &&
(profile.roles.includes('superAdmin') ||
  profile.roles.includes('admin') ||
  profile.roles.includes('loyaltyOperator'))
```

### Roles necessarias

| Role | Acesso |
|------|--------|
| `superAdmin` | Sim |
| `admin` | Sim |
| `loyaltyOperator` | Sim |
| Outros | Nao — query desabilitada |

### Resposta: LoyaltyInterface

Cada item retornado tem a seguinte estrutura:

```typescript
interface LoyaltyInterface {
  id: string;                        // UUID do programa
  createdAt: string;
  updatedAt: string;
  companyId: string;
  contractId: string;                // ID do contrato ERC-20 (usado nos relatorios)
  name: string;                      // Nome do programa
  settings: {
    paymentView: {
      enabled: boolean;              // Pagamento com pontos habilitado?
    };
  };
  compensateRule: {
    period: string;                  // Periodo de compensacao
  };
  priority: number;
  tokenIssuanceMethod: string;
  tokenIssuanceAddress: string;
  tokenTransferabilityMethod: string;
  tokenTransferabilityAddress: string;
  pointPrecision: string;            // "integer" ou "decimal"
  paymentViewSettings: {
    pointsEquivalent: {
      currency: string;              // Ex: "BRL"
      pointsValue: number;           // Ex: 100
      currencyValue: number;         // Ex: 1.00
    };
  };
  rules: Rule[];
}
```

### Campos importantes

| Campo | Uso |
|-------|-----|
| `contractId` | Usado como `loyaltyId` nos endpoints de ERC20 history (relatorios) |
| `settings.paymentView.enabled` | Determina se o programa permite pagamento com pontos |
| `pointPrecision` | Define como exibir os pontos (`integer` = sem decimais, `decimal` = com decimais) |
| `paymentViewSettings.pointsEquivalent` | Taxa de conversao pontos <-> moeda |
| `rules` | Regras de acumulo e resgate configuradas para o programa |

### Fluxo

```
Pagina admin carrega
        |
        v
useGetCompanyLoyalties() executa automaticamente
(se profile e roles estao corretos)
        |
        v
GET /{companyId}/loyalties/admin?limit=49
        |
        v
Retorna LoyaltyInterface[]
        |
        v
Exibir lista de programas de fidelidade
        |
        v
contractId do primeiro loyalty e usado
nos relatorios de ERC20 history
```

---

## Fluxo Combinado: Saldo + Loyalties

Em um cenario tipico de painel administrativo:

```
1. useGetCompanyLoyalties()           -> Lista programas de fidelidade
        |
        v
2. Operador seleciona um loyalty
        |
        v
3. getUserBalance(userId)             -> Busca saldo do usuario
        |
        v
4. Exibir: nome do programa, saldo, taxa de conversao
```

---

## Exemplo API-first (sem SDK)

```typescript
/** Consultar saldo do usuario */
const fetchUserBalance = async (
  companyId: string,
  userId: string,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/users/balance/${userId}`,
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

  return response.json();
};

/** Listar programas de fidelidade da empresa */
const fetchCompanyLoyalties = async (
  companyId: string,
  token: string,
  limit: number = 49
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/admin?limit=${limit}`,
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

  return response.json(); // { items: LoyaltyInterface[] }
};
```

---

## Observacoes Tecnicas

1. **useMutation para GET:** O `useGetUserBalance` usa `useMutation` apesar de ser um GET. Isso e intencional — permite buscar saldo sob demanda (quando o operador escaneia o codigo do usuario, por exemplo) em vez de automaticamente ao montar o componente.

2. **Limite de 49:** O endpoint de listagem usa `?limit=49` fixo no hook. Se a empresa tiver mais de 49 programas de fidelidade, os excedentes nao serao retornados.

3. **contractId vs id:** O `contractId` do `LoyaltyInterface` e usado como `{loyaltyId}` nos endpoints de ERC20 history. Nao confundir com o `id` do programa.

4. **Dependencia de perfil:** O `useGetCompanyLoyalties` depende do perfil do usuario estar carregado (via `useProfileWithKYC`) para verificar roles. Se o perfil ainda nao carregou, a query fica desabilitada.
