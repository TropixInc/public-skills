# Flow: Loyalty Payment

Fluxo para pre-visualizar e executar pagamentos utilizando pontos de fidelidade.

---

## Visao Geral

O fluxo de pagamento com loyalty consiste em dois passos:

1. **Preview** — Pre-visualizar o pagamento (calcular pontos, validar dados)
2. **Criar pagamento** — Confirmar e executar a transacao

Opcionalmente, antes do pagamento, o operador pode **buscar o usuario por codigo** temporario.

Todos os endpoints utilizam a **Key API** (`https://api.w3block.io`), exceto a busca de usuario por codigo que usa a **ID API**.

---

## Passo 0 (Opcional): Buscar Usuario por Codigo

### Endpoint

```
GET /users/{companyId}/code/{userId}
```

**API:** ID (`W3blockAPI.ID`) — **nao e Key API**

### Hook: `useGetUserByCode`

**Arquivo:** `src/modules/business/hooks/useGetUserByCode.tsx`

```typescript
const { mutateAsync: getUserByCode } = useGetUserByCode();

const user = await getUserByCode(codigoTemporario);
```

### Comportamento

- Usa `useMutation` — chamada imperativa
- O `{userId}` no path e na verdade o codigo temporario do usuario (nao o UUID)
- Retorna dados do usuario associado ao codigo
- Utilizado em cenarios presenciais: operador de loja escaneia/digita o codigo do cliente

### Quando usar

- Pagamento presencial onde o operador nao sabe o userId
- O cliente apresenta um QR code ou codigo numerico no app
- O operador digita o codigo para identificar o cliente

---

## Passo 1: Preview de Pagamento

### Endpoint

```
PATCH /{companyId}/loyalties/rewards/payment/preview
```

### Hook: `useGetPaymentPreview`

**Arquivo:** `src/modules/business/hooks/useGetPaymentPreview.tsx`

```typescript
const { mutateAsync: getPreview } = useGetPaymentPreview();

const preview = await getPreview({
  loyaltyId: "uuid-do-programa",
  amount: "50.00",
  points: "100",
  userId: "uuid-do-usuario",
  userCode: "codigo-temporario"
});
```

### Body: PaymentPreviewDTO

```typescript
interface PaymentPreviewDTO {
  loyaltyId: string;   // UUID do programa de fidelidade
  amount: string;      // Valor monetario (ex: "50.00")
  points: string;      // Quantidade de pontos (ex: "100")
  userId: string;      // UUID do usuario pagador
  userCode: string;    // Codigo temporario do usuario
}
```

### Comportamento

- Usa `useMutation` com `PATCH`
- Valida os dados e retorna a pre-visualizacao do pagamento
- Permite ao operador/usuario confirmar antes de executar
- Em caso de erro, loga no console e lanca excecao

### Campos importantes

| Campo | Descricao |
|-------|-----------|
| `loyaltyId` | UUID do programa de fidelidade (obtido de `useGetCompanyLoyalties`) |
| `amount` | Valor monetario como string. Usar a taxa de conversao do loyalty para calcular |
| `points` | Pontos a serem debitados. Respeitar `pointPrecision` do loyalty |
| `userId` | UUID do usuario. Pode ser obtido via `useGetUserByCode` |
| `userCode` | Codigo temporario do usuario (obtido via app do cliente) |

---

## Passo 2: Criar Pagamento

### Endpoint

```
PATCH /{companyId}/loyalties/rewards/payment
```

### Hook: `useCreatePayment`

**Arquivo:** `src/modules/business/hooks/useCreatePayment.tsx`

```typescript
const { mutateAsync: createPayment } = useCreatePayment();

const result = await createPayment({
  loyaltyId: "uuid-do-programa",
  amount: "50.00",
  points: "100",
  userId: "uuid-do-usuario",
  userCode: "codigo-temporario",
  description: "Pagamento no estabelecimento XYZ"
});
```

### Body: CreatePaymentDTO

```typescript
interface CreatePaymentDTO {
  loyaltyId: string;    // UUID do programa de fidelidade
  amount: string;       // Valor monetario
  points: string;       // Quantidade de pontos
  userId: string;       // UUID do usuario pagador
  userCode: string;     // Codigo temporario do usuario
  description: string;  // Descricao do pagamento (OBRIGATORIO)
}
```

### Comportamento

- Usa `useMutation` com `PATCH`
- Executa o pagamento definitivamente (nao reversivel via API)
- Em caso de erro, loga no console e lanca excecao
- A diferenca do preview e o campo `description` que e obrigatorio

### Diferenca entre Preview e Create

| Campo | PaymentPreviewDTO | CreatePaymentDTO |
|-------|:-:|:-:|
| `loyaltyId` | **X** | **X** |
| `amount` | **X** | **X** |
| `points` | **X** | **X** |
| `userId` | **X** | **X** |
| `userCode` | **X** | **X** |
| `description` | | **X** |

---

## Fluxo Completo

```
Operador abre tela de pagamento
        |
        v
[Opcional] Digitar codigo do cliente
        |
        v
useGetUserByCode(codigo)
GET /users/{companyId}/code/{codigo}
        |
        v
Identificar usuario (userId, nome, etc.)
        |
        v
Preencher dados do pagamento:
- loyaltyId (programa de fidelidade)
- amount (valor monetario)
- points (pontos equivalentes)
        |
        v
useGetPaymentPreview(PaymentPreviewDTO)
PATCH /{companyId}/loyalties/rewards/payment/preview
        |
        v
Exibir pre-visualizacao ao operador:
- Pontos que serao debitados
- Valor equivalente
- Dados do usuario
        |
        v
Operador confirma?
    |           |
   Sim         Nao -> Cancelar
    |
    v
useCreatePayment(CreatePaymentDTO)
PATCH /{companyId}/loyalties/rewards/payment
        |
        v
Pagamento executado com sucesso
        |
        v
[Opcional] Atualizar saldo via useGetUserBalance
```

---

## Calculo de Pontos e Valores

A taxa de conversao entre pontos e moeda e definida no `LoyaltyInterface`:

```typescript
paymentViewSettings: {
  pointsEquivalent: {
    currency: "BRL",      // Moeda de referencia
    pointsValue: 100,     // 100 pontos
    currencyValue: 1.00   // = R$ 1,00
  }
}
```

**Formula de conversao:**

```
pontos_necessarios = (valor_monetario / currencyValue) * pointsValue
valor_monetario = (pontos / pointsValue) * currencyValue
```

**Exemplo:**
- Taxa: 100 pontos = R$ 1,00
- Compra de R$ 50,00 = 5.000 pontos
- 250 pontos = R$ 2,50

---

## Exemplo API-first (sem SDK)

```typescript
/** Pre-visualizar pagamento com pontos */
const fetchPaymentPreview = async (
  companyId: string,
  body: {
    loyaltyId: string;
    amount: string;
    points: string;
    userId: string;
    userCode: string;
  },
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/rewards/payment/preview`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    }
  );

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message);
  }

  return response.json();
};

/** Criar pagamento com pontos de fidelidade */
const createPayment = async (
  companyId: string,
  body: {
    loyaltyId: string;
    amount: string;
    points: string;
    userId: string;
    userCode: string;
    description: string;
  },
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/loyalties/rewards/payment`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
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

1. **Ambos usam PATCH:** Tanto o preview quanto o create usam metodo PATCH (nao POST). Isso e uma peculiaridade da API.

2. **Valores como string:** `amount` e `points` sao strings, nao numeros. Enviar como string para evitar problemas de precisao.

3. **pointPrecision:** Se o loyalty tem `pointPrecision: "integer"`, nao enviar pontos com casas decimais. Se `"decimal"`, casas decimais sao permitidas.

4. **userCode obrigatorio:** Mesmo no cenario onde ja se tem o `userId`, o `userCode` e um campo obrigatorio nos DTOs. Ele valida a identidade do usuario no momento da transacao.

5. **Preview antes de criar:** Sempre chamar o preview antes de criar o pagamento. O preview valida os dados e retorna o calculo exato. Nao calcular manualmente e enviar direto ao create.

6. **Erro de saldo insuficiente:** Se o usuario nao tiver pontos suficientes, a API retornara erro no preview ou no create. Tratar adequadamente na UI.
