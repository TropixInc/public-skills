# Pagamento com Cartão de Crédito (Pagar.me / Asaas)

## Overview

Pagamento via checkout transparente com cartão de crédito, processado pelo **Asaas** ou **Pagar.me**. O usuário preenche os dados do cartão diretamente no formulário (sem redirect), podendo usar cartões salvos e selecionar parcelamento.

## Prerequisites

- Step 1 (Cart Confirmation) concluído
- `product_cart_info_key` no localStorage com `choosedPayment.paymentProvider` = `asaas` ou `pagar_me` (SDK enum: `PaymentMethod.ASAAS`)
- `choosedPayment.paymentMethod` = `credit_card`
- `choosedPayment.inputs` contendo os campos requeridos pelo provider

---

## Steps

### Step 1: Page Load — Leitura do Cache

- **Screen**: Spinner de loading.
- **User Action**: Nenhuma (automático).
- **API Call**: Nenhuma (leitura do localStorage).
- **Response Handling**: Lê `OrderPreviewCache` do `product_cart_info_key`. Verifica `choosedPayment.paymentProvider`.
- **State Changes**: Se provider é `asaas` e `totalPrice !== 0`: `loading = false`, renderiza formulário. Define `installment` como primeira opção de `availableInstallments`.

> **Nota:** Diferente do SDK original, a implementação Next.js NÃO faz preview periódico na página de pagamento. Os dados vêm do cache do Step 1.

---

### Step 2: Exibição do Formulário

- **Screen**: O formulário `CheckoutPaymentComponent` exibe os campos baseados no array `inputs` do `choosedPayment`:

**Campos do formulário (todos sempre exibidos):**

| Campo | Input | Validação |
|-------|-------|-----------|
| CPF/CNPJ | `cpf_cnpj` | Formato + dígitos verificadores (cpf-cnpj-validator) |
| Número do cartão | `credit_card_number` | Algoritmo de Luhn (card-validator). **Enviado com espaços** |
| Validade | `credit_card_expiry` | `MM/YY`, não expirado (card-validator) |
| CVV | `credit_card_ccv` | 3-4 dígitos (regex) |
| Nome no cartão | `credit_card_holder_name` | Min 3 caracteres |
| Telefone | `credit_card_holder_phone` | Min 1 caractere (obrigatório) |
| CEP | `credit_card_holder_postal_code` | Min 1 caractere (obrigatório) |
| Parcelas | `installments` | Seleção da lista |

> **Nota:** Todos os campos são sempre exibidos independente do provider (Asaas ou Pagar.me). O formulário não é dinâmico baseado no array `inputs` — todos os campos acima são sempre renderizados. Os campos `credit_card_holder_cpf_cnpj`, `credit_card_holder_phone` e `credit_card_holder_postal_code` são enviados para qualquer provider.

**Parcelamento:** Se `availableInstallments` existe, exibe seletor de parcelas com valor final e juros por parcela.

- **User Action**: Preenche os campos do formulário.

---

### Step 3: Validação Frontend

- **Frontend Validation**: A validação é feita automaticamente pelo Zod schema + react-hook-form:

```
cpfCnpj           → cpf.isValid(digits) || cnpj.isValid(digits) via cpf-cnpj-validator
cardNumber         → cardValidator.number(val.replace(/\s/g, '')).isValid via card-validator
expiry             → cardValidator.expirationDate(val).isValid via card-validator
cvv                → /^\d{3,4}$/ (regex)
holderName         → min 3 caracteres
phone              → min 1 caractere (obrigatório)
postalCode         → min 1 caractere (obrigatório)
installmentCount   → min 1 caractere (string, convertido para number no submit)
```

Erros são exibidos inline abaixo de cada campo automaticamente pelo react-hook-form.

---

### Step 4: Verificação de Pedido Duplicado

Se o backend retorna `errorCode: 'similar-order-not-accepted'`, exibe modal `SimilarOrderModal` perguntando se deseja prosseguir. Se o usuário confirma, re-envia com `acceptSimilarOrderInShortPeriod: true`.

---

### Step 5: Criação do Pedido

- **User Action**: Clica "Finalizar compra" (ou confirma pedido similar).

- **API Call**:

```
POST /companies/{companyId}/orders
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "orderProducts": [
    {
      "productId": "uuid-produto",
      "expectedPrice": "105.50",
      "quantity": 1,
      "variantIds": []
    }
  ],
  "currencyId": "uuid-brl",
  "paymentMethod": "credit_card",
  "providerInputs": {
    "cpf_cnpj": "123.456.789-00",
    "transparent_checkout": true,
    "credit_card_number": "4242 4242 4242 4242",
    "credit_card_expiry": "12/28",
    "credit_card_ccv": "123",
    "credit_card_holder_name": "JOAO SILVA",
    "credit_card_holder_cpf_cnpj": "123.456.789-00",
    "credit_card_holder_phone": "(11) 91234-5678",
    "credit_card_holder_postal_code": "12345-678",
    "installments": 1
  },
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": "DESCONTO10",
  "passShareCodeData": {},
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {
        "cpf_cnpj": "123.456.789-00",
        "transparent_checkout": true,
        "credit_card_number": "4242 4242 4242 4242",
        "credit_card_expiry": "12/28",
        "credit_card_ccv": "123",
        "credit_card_holder_name": "JOAO SILVA",
        "credit_card_holder_cpf_cnpj": "123.456.789-00",
        "credit_card_holder_phone": "(11) 91234-5678",
        "credit_card_holder_postal_code": "12345-678",
        "installments": 1
      }
    }
  ]
}
```

> **Nota:** `acceptSimilarOrderInShortPeriod` **nao e enviado** na primeira tentativa. Apenas enviado como `true` no retry apos erro `similar-order-not-accepted`.
>
> **Nota:** `couponCode` envia `null` (nao `undefined`) quando vazio. `successUrl` e construida a partir do hostname do tenant via `useTenantBaseUrl()` (nao `window.location.origin`).
>
> **Nota:** `credit_card_number` e enviado **com espacos** (ex: `"4242 4242 4242 4242"`). O campo nao e limpo antes de enviar.

- **Response Handling** (201 Created):
  - `status`: geralmente `confirming_payment` (processamento em andamento)
  - Salva `CreateOrderResponse` no `order_completed_info_key`
  - Envia evento de tracking `purchase`
  - Redireciona para Step 3 (`/checkout/completed`)

- **Error States**:

| Código | Mensagem | Ação |
|--------|----------|------|
| `similar-order-not-accepted` | Pedido similar detectado | Exibe modal de confirmação |
| `400` Cartão inválido | "Informe o endereço do titular do cartão" | Exibe mensagem traduzida |
| `422` Pagamento falhou | Mensagem do provider | Exibe erro no formulário, permite retry |
| `braza-email-already-attached-to-other-cpf-error` | Email vinculado a outro CPF | Exibe mensagem específica Braza |

- **State Changes**: `orderResponse` salvo no localStorage, navega para completion.

---

## API Sequence

1. `POST /companies/{companyId}/orders` — Criação do pedido com pagamento (único endpoint chamado nesta página)

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Validação de campo falha | Mensagem inline abaixo do campo |
| Pedido duplicado | Modal: "Pedido similar detectado. Continuar?" |
| Cartão recusado | Mensagem de erro do provider no formulário |
| Erro de rede | Mensagem genérica + botão "Voltar para Home" |
| Token expirado | Redireciona para login |

---

## Implementação Real (Next.js)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador — carrega cache, roteia para CreditCard, PIX, Stripe, Transfer/Free. Gerencia criação do pedido, dialog de pedido similar |
| `CreditCardForm` | `credit-card-form.tsx` | Formulário com react-hook-form + zod. Campos: CPF/CNPJ, número do cartão, validade, CVV, nome do titular, telefone, CEP, parcelas |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useCreateOrder()` | `useMutation` | Cria pedido via `POST /orders`. Auto-salva response no localStorage (`ORDER_COMPLETED_INFO`) |
| `useTenantBaseUrl()` | custom hook | Resolve base URL do tenant para `successUrl`. Prioridade: 1) tenant main host, 2) `NEXT_PUBLIC_TENANT_HOSTNAME`, 3) `window.location.origin` |

### Validação (Zod Schema)

```typescript
const schema = z.object({
  cpfCnpj: z.string().refine((val) => cpf.isValid(digits) || cnpj.isValid(digits)),
  cardNumber: z.string().refine((val) => cardValidator.number(val.replace(/\s/g, '')).isValid),
  expiry: z.string().refine((val) => cardValidator.expirationDate(val).isValid),
  cvv: z.string().regex(/^\d{3,4}$/),
  holderName: z.string().min(3),
  phone: z.string().min(1),
  postalCode: z.string().min(1),
  installmentCount: z.string().min(1),
});
```

### providerInputs Gerados

O formulário transforma os campos em:

```typescript
const providerInputs = {
  cpf_cnpj: data.cpfCnpj,
  credit_card_number: data.cardNumber,           // com espaços (não limpo!)
  credit_card_expiry: data.expiry,
  credit_card_ccv: data.cvv,
  credit_card_holder_name: data.holderName,
  credit_card_holder_cpf_cnpj: data.cpfCnpj,     // mesmo valor do cpf_cnpj
  credit_card_holder_phone: data.phone,           // campo novo
  credit_card_holder_postal_code: data.postalCode, // campo novo
  installments: parseInt(data.installmentCount, 10),
  transparent_checkout: true,
};
```

> **Nota:** O `credit_card_number` **não é limpo** — os espaços da formatação do input são mantidos (ex: `"4242 4242 4242 4242"`). O `credit_card_holder_cpf_cnpj` recebe o mesmo valor do `cpf_cnpj` (o CPF do comprador).

### buildCreateOrderRequest()

O `CheckoutPayment` constrói o request assim:

```typescript
const request: CreateOrderRequest = {
  orderProducts: cache.orderProducts,              // já com quantity: 1, variantIds: []
  currencyId: cache.currencyId,                    // top-level
  paymentMethod: cache.choosedPayment.paymentMethod, // top-level
  providerInputs,                                  // top-level
  destinationWalletAddress: cache.destinationWalletAddress,
  successUrl,                                      // via useTenantBaseUrl() + '/wallet'
  couponCode: cache.couponCode || null,            // null, não undefined
  passShareCodeData: {},                           // sempre objeto vazio
  payments: [{
    currencyId: cache.currencyId,
    paymentMethod: cache.choosedPayment.paymentMethod,
    paymentProvider: cache.choosedPayment.paymentProvider,
    providerInputs,
    amountType: AmountType.PERCENTAGE,
    amount: '100',
  }],
};
// acceptSimilarOrderInShortPeriod apenas adicionado como true no retry
if (acceptSimilar) {
  request.acceptSimilarOrderInShortPeriod = true;
}
```

> **Nota sobre successUrl:** A URL é construída via `useTenantBaseUrl()` hook, que resolve para `https://{tenant-main-host}/wallet`. O hook tenta: 1) hostname principal do tenant (do contexto), 2) `NEXT_PUBLIC_TENANT_HOSTNAME` env var, 3) `window.location.origin` como último recurso. **Não usa** `window.location.origin` diretamente.

### ⚠️ Gotcha: installmentPrice

O campo `installmentPrice` nas parcelas pode vir como `string` da API. Ao exibir no seletor de parcelas, **sempre** usar `Number(inst.installmentPrice).toFixed(2)` em vez de `inst.installmentPrice.toFixed(2)`.

### Guards no Page Load

O `CheckoutPayment` faz dois guards no `useEffect`:
1. Se `ORDER_COMPLETED_INFO` existe no localStorage → redireciona para `/checkout/completed` (pedido já criado)
2. Se `PRODUCT_CART_INFO` não existe → redireciona para `/checkout/confirmation` (precisa fazer Step 1 primeiro)

### Fluxo de Erro

O tratamento de erro verifica se a mensagem contém `'similar-order-not-accepted'`:
- Se sim: abre `Dialog` perguntando se quer prosseguir → re-envia com `acceptSimilarOrderInShortPeriod: true`
- Se não: exibe mensagem de erro no topo do card
