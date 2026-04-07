---
id: FLOW_CHECKOUT_PAYMENT_STRIPE
title: "Checkout - Pagamento Stripe"
module: checkout
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-24"
authors:
  - rafaelmhp
tags:
  - checkout
  - payment
  - stripe
depends_on:
  - FLOW_CHECKOUT_OVERVIEW
  - CHECKOUT_API_REFERENCE
---

# Pagamento via Stripe

## Overview

O Stripe suporta **dois fluxos** dependendo do `paymentMethod` selecionado:

1. **Stripe + credit_card** → Stripe Elements (PaymentElement) com criação automática do pedido no page load
2. **Stripe + pix** → Criação automática do pedido que retorna `paymentUrl` para iframe

**Particularidade:** Diferente dos outros métodos, o Stripe cria o pedido **antes** do usuário preencher dados de pagamento (credit_card) ou interagir (pix).

## Prerequisites

- Step 1 (Cart Confirmation) concluído
- `product_cart_info_key` no localStorage com `choosedPayment.paymentProvider` = `stripe`
- `choosedPayment.paymentMethod` = `credit_card` ou `pix`

---

## Roteamento por paymentMethod

```
choosedPayment.paymentProvider === "stripe"
├── paymentMethod === "credit_card"
│   → Stripe Elements (clientSecret + publicKey)
│   → Cache de 5 minutos
│   → StripePaymentView component
│
└── paymentMethod === "pix"
    → Auto-create order (paymentUrl na response)
    → IframePaymentView component
    → Sem cache
```

---

## Steps

### Step 1: Page Load — Criação Automática do Pedido

- **Screen**: Spinner de loading enquanto o pedido é criado automaticamente.
- **User Action**: Nenhuma (automático no page load).
- **Frontend Validation**: Verifica se existe `stripe_order_cache` no localStorage com cache válido (< 5 minutos) e produtos correspondentes. Se sim, reutiliza o cache sem chamar a API.

**Com cache válido (skip API call):**

```
stripeOrderCache existe
  && isStripeCacheValid(timestamp) → diffMs < 5 min
  && isStripeCacheMatchingProducts(cache, productCache) → mesmos productIds e amount
→ Usa clientSecret e publicKey do cache
```

**Sem cache (ou cache inválido):**

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
  "providerInputs": {},
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": null,
  "passShareCodeData": {},
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "credit_card",
      "paymentProvider": "stripe",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {}
    }
  ]
}
```

> **Nota:** `providerInputs` é `{}` (vazio) — o Stripe Elements lida com os dados do cartão diretamente. Campos top-level `currencyId`, `paymentMethod`, `providerInputs`, `passShareCodeData` são obrigatórios. `successUrl` usa hostname do tenant via `useTenantBaseUrl()`. `couponCode` envia `null` quando vazio.

- **Response Handling** (201 Created):

```json
{
  "id": "uuid-order",
  "status": "pending",
  "paymentProvider": "stripe",
  "paymentInfo": {
    "clientSecret": "pi_xxx_secret_yyy",
    "publicKey": "pk_live_abc123"
  }
}
```

O frontend extrai:
- `clientSecret` = `paymentInfo.clientSecret`
- `publicKey` = `paymentInfo.publicKey`

Salva no `stripe_order_cache`:

```json
{
  "productIds": ["uuid-produto"],
  "amount": "105.50",
  "stripe": {
    "clientSecret": "pi_xxx_secret_yyy",
    "publicKey": "pk_live_abc123"
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

> **Nota:** O cache usa `productIds` (array de strings) em vez de `productData` (objetos). A validação do cache compara os productIds ordenados e verifica se o timestamp tem menos de 5 minutos.

- **State Changes**: `stripeClientSecret` e `stripePublicKey` setados. `loading = false`. `orderResponse` salvo no localStorage.

---

### Step 2: Exibição do Stripe Elements

- **Screen**: Formulário do Stripe Elements renderizado dentro de `<Elements stripe={stripePromise} options={{ clientSecret }}>`:
  - Campo de cartão (número, validade, CVV) — renderizado pelo `<PaymentElement />`
  - Botão "Pagar"
  - Mensagem de erro (se houver)

O Stripe Elements é inicializado com:
```tsx
const stripePromise = loadStripe(publicKey);
<Elements stripe={stripePromise} options={{ clientSecret }}>
  <CheckoutStripeForm />
</Elements>
```

- **User Action**: Preenche os dados do cartão no formulário do Stripe.

---

### Step 3: Confirmação do Pagamento

- **User Action**: Clica "Pagar".

- **Frontend Validation**: O Stripe Elements valida os campos internamente.

- **API Call** (via Stripe SDK, não diretamente):

```typescript
const result = await stripe.confirmPayment({
  elements,
  confirmParams: {
    return_url: `${window.location.origin}/checkout/completed`,
  },
});
```

> **Nota:** O `return_url` do Stripe usa `window.location.origin` (não o tenant base URL), pois o redirect precisa voltar para a aplicação de checkout, não para o site do tenant.

O `stripe.confirmPayment()` faz a chamada para a API do Stripe internamente.

- **Response Handling**:
  - **Sucesso**: Stripe redireciona automaticamente para `return_url` (`/checkout/completed`). O redirect inclui query params do Stripe (`payment_intent`, `payment_intent_client_secret`, `redirect_status`).
  - **Erro**: `result.error` contém `message`, `code`, `decline_code`. Exibe mensagem de erro no formulário.

- **Error States**:

| Erro Stripe | Descrição | Ação |
|-------------|-----------|------|
| `card_declined` | Cartão recusado | Exibe mensagem do Stripe |
| `expired_card` | Cartão expirado | Exibe mensagem |
| `incorrect_cvc` | CVC incorreto | Exibe mensagem |
| `processing_error` | Erro de processamento | Exibe mensagem, permite retry |
| Erro genérico | "Erro não identificado" | Exibe mensagem |

- **State Changes**: Após redirect, a página de completion carrega. O `order_completed_info_key` já contém o `orderResponse` salvo no Step 1.

---

## API Sequence

1. `POST /companies/{companyId}/orders` — Criação do pedido (automático no page load, ou usando cache válido)
2. Stripe SDK `confirmPayment()` — Confirmação do pagamento (via Stripe API)
3. Redirect para `/checkout/completed`

> **Nota:** Não há preview periódico na página de pagamento. Os dados vêm do cache do Step 1.

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Cache Stripe expirado (> 5 min) | Deleta cache, cria novo pedido |
| Cache com produtos diferentes | Deleta cache, cria novo pedido |
| Cartão recusado | Exibe erro do Stripe, permite preencher novamente |
| Erro na criação do pedido | Exibe mensagem genérica + "Voltar para Home" |

---

## Backend Notes

O Stripe funciona diferente dos outros providers:
1. O backend cria um **PaymentIntent** no Stripe ao criar o pedido
2. Retorna `clientSecret` + `publicKey` para o frontend
3. O frontend usa Stripe Elements para coletar dados do cartão
4. `stripe.confirmPayment()` envia os dados direto para o Stripe (PCI-compliant)
5. O Stripe notifica o backend via webhook quando o pagamento é confirmado
6. O backend atualiza o status do pedido

---

## Implementação Real (Next.js)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador — cria pedido no load se Stripe, gerencia cache |
| `StripePaymentView` | `stripe-payment-view.tsx` | Wrapper: inicializa Stripe Elements com `clientSecret` e `publicKey` |
| `StripeForm` | `stripe-payment-view.tsx` (interno) | Formulário real com `PaymentElement` + botão submit |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useCreateOrder()` | `useMutation` | Cria pedido via `POST /orders` |
| `useTenantBaseUrl()` | custom hook | Resolve `successUrl` para `https://{tenant}/wallet` |

### Dependências

```
@stripe/stripe-js     — loadStripe()
@stripe/react-stripe-js — Elements, PaymentElement, useStripe, useElements
```

---

## Fluxo B: Stripe PIX

### Overview

Quando o provider é `stripe` e o método é `pix`, o frontend **não** usa Stripe Elements. O pedido é criado automaticamente no page load e a API retorna uma `paymentUrl` que é exibida em um iframe. O fluxo é similar ao Pagar.me PIX.

### Step 1: Page Load — Criação Automática do Pedido

- **Screen**: Spinner de loading.
- **User Action**: Nenhuma (automático).
- **State Changes**: Lê `OrderPreviewCache` do localStorage. Detecta `paymentProvider === 'stripe'` e `paymentMethod === 'pix'`. O pedido é criado automaticamente via `handleSubmit({})`.

### Step 2: Criação do Pedido

**Request Body:**

```json
{
  "orderProducts": [{ "productId": "uuid", "expectedPrice": "105.50", "quantity": 1, "variantIds": [] }],
  "currencyId": "uuid-moeda",
  "paymentMethod": "pix",
  "providerInputs": {},
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": null,
  "passShareCodeData": {},
  "payments": [{
    "currencyId": "uuid-moeda",
    "paymentMethod": "pix",
    "paymentProvider": "stripe",
    "amountType": "percentage",
    "amount": "100",
    "providerInputs": {}
  }]
}
```

**Response Handling:**

A API retorna `paymentUrl` em `payments[].publicData.paymentUrl` ou `paymentInfo.paymentUrl`. O frontend seta `iframeUrl` e renderiza `IframePaymentView`.

### Step 3: Iframe de Pagamento

- **Screen**: `IframePaymentView` com iframe apontando para URL do Stripe PIX
- **User Action**: Completa o pagamento PIX no iframe
- **Detecção de conclusão**: Quando o iframe redireciona para o mesmo hostname, navega para `/checkout/completed`

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador — cria pedido automaticamente, extrai paymentUrl |
| `IframePaymentView` | `iframe-payment-view.tsx` | Iframe com detecção de redirect |
