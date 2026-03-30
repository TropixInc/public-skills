---
id: FLOW_CHECKOUT_PAYMENT_CRYPTO
title: "Checkout - Pagamento Crypto"
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
  - crypto
depends_on:
  - FLOW_CHECKOUT_OVERVIEW
  - CHECKOUT_API_REFERENCE
---

# Pagamento Crypto (Braza / ERC-20)

## Overview

Pagamento com criptomoedas, com dois subfluxos distintos:

1. **Braza** — Bridge fiat-to-crypto. Usuário fornece CPF + `quoteId` (cotação). O backend pode gerar um PIX como forma de liquidação.
2. **ERC-20 (crypto nativo via wallet interna)** — Pagamento com token ERC-20 direto da wallet interna do usuário. Requer verificação de **allowance** antes da compra. Suporta multi-payment (parte crypto + parte fiat). Entrada livre de valor crypto.

## Prerequisites

- Step 1 (Cart Confirmation) concluído
- `product_cart_info_key` no localStorage
- Para **Braza**: `choosedPayment.paymentProvider` = `braza`, `choosedPayment.paymentMethod` = `crypto`
- Para **ERC-20**: URL params `coinPayment=true` e `cryptoCurrencyId=<uuid>` na URL do Step 1

---

## Query Parameters (Step 1)

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `coinPayment` | `"true"` | Ativa fluxo de pagamento crypto |
| `cryptoCurrencyId` | `uuid` | ID da moeda crypto para pagamento |

---

## Subfluxo 1: Braza (fiat-to-crypto bridge)

### Step 1: Page Load — Exibição do Formulário

- **Screen**: Formulário `BrazaPaymentForm` com campos CPF/CNPJ e resumo da cotação.
- **User Action**: Nenhuma (automático).
- **State Changes**: Como provider é `braza`, renderiza formulário com dados do `providerData`.

**Campos do formulário Braza:**

| Campo | Input | Descrição |
|-------|-------|-----------|
| CPF/CNPJ | `cpf_cnpj` | Documento do comprador (validado via `cpf-cnpj-validator`) |
| Quote ID | `quote_id` | ID da cotação Braza (pré-preenchido do `providerData`, invisível) |
| Transparent Checkout | `transparent_checkout` | Flag booleana (sempre `true`, invisível) |

O `quoteId` vem do `choosedPayment.providerData.quoteId` retornado no Order Preview.

**Resumo da cotação exibido:**

| Campo | Source | Descrição |
|-------|--------|-----------|
| USD Amount | `providerData.usdAmount` | Valor em dólares |
| Exchange Rate | `providerData.usdQuote` | Taxa de câmbio |
| Fees | `providerData.feesAmount` | Taxas da Braza |
| IOF | `providerData.iof` | Imposto IOF |
| Total (BRL) | `providerData.brlAmount` | Valor total em reais |

**Dados do preview:**

```json
{
  "providerData": {
    "brlAmount": "500.00",
    "feesAmount": "5.00",
    "iof": "1.90",
    "quoteId": "uuid-quote",
    "usdAmount": "100.00",
    "usdQuote": "5.00",
    "usdVetQuote": "4.95"
  }
}
```

---

### Step 2: Submit — Criação do Pedido

- **User Action**: Preenche CPF e clica "Finalizar compra".

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
      "expectedPrice": "100.00"
    }
  ],
  "signedGasFee": "===assinatura",
  "currencyId": "uuid-moeda",
  "destinationWalletAddress": "0x...",
  "payments": [
    {
      "currencyId": "uuid-moeda",
      "paymentMethod": "crypto",
      "paymentProvider": "braza",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {
        "cpf_cnpj": "12345678900",
        "transparent_checkout": true,
        "quote_id": "uuid-quote"
      }
    }
  ]
}
```

- **Response Handling**: Dependendo da configuração Braza:
  - Pode retornar dados PIX (`publicData.pix`) → exibe QR code + polling
  - Pode retornar `paymentUrl` → exibe iframe
  - Pode processar direto → redirect para completion

- **Error States específicos do Braza**:

| errorCode | Descrição |
|-----------|-----------|
| `braza-email-already-attached-to-other-cpf-error` | Email do usuário já vinculado a outro CPF na Braza |
| `braza-phone-already-attached-to-other-cpf-error` | Telefone já vinculado a outro CPF na Braza |

---

## Subfluxo 2: ERC-20 (crypto nativo via wallet interna)

### Step 1 (na Confirmation): Input de Valor Crypto + Verificação de Allowance

Quando a URL contém `coinPayment=true&cryptoCurrencyId=<uuid>`:

1. **CryptoPaymentInput** — Campo de entrada livre (controlado pelo pai) para o valor crypto a utilizar. Exibe saldo da wallet via `useWalletBalance(cryptoCurrency.code)`.
2. **Preview sempre single-payment fiat** — O `useOrderPreview` envia apenas 1 payment fiat 100%. O valor crypto é estado local (`cryptoAmount`) e o split é calculado localmente para exibição (sem re-fetch do preview ao alterar o valor crypto).
3. **AllowanceModal** — Quando `currencyAllowanceState === 'required'`, exibe modal ao clicar "Continuar".

**CryptoPaymentInput:**
- Input `type="text"` com `inputMode="decimal"` para entrada numérica
- Mostra split: crypto portion + fiat remaining

**AllowanceModal:**

| State | Significado | Ação |
|-------|-------------|------|
| `required` | Precisa aumentar allowance | Exibe botão "Aprovar gasto" |
| `processing` | Allowance em processamento | Exibe spinner |
| `allowed` | Allowance suficiente | Fecha modal e navega para pagamento |
| `null` | Não aplicável | Prossegue normalmente |

**Fluxo de allowance:**
1. Clica "Aprovar gasto" → `PATCH /companies/{companyId}/currencies/{currencyId}/increase-allowance`
2. Inicia polling do preview a cada 6s (`ALLOWANCE_POLL_INTERVAL_MS`)
3. Quando `currencyAllowanceState = 'allowed'` → fecha modal → salva cache → navega
4. Timeout de 30s (`ALLOWANCE_TIMEOUT_MS`) → exibe erro, permite retry

**Cache salvo no localStorage:**
```json
{
  "...campos padrão do OrderPreviewCache...",
  "isCoinPayment": true,
  "cryptoCurrencyId": "uuid-crypto",
  "cryptoAmount": "50.00"
}
```

---

### Step 2: Construção do Multi-Payment e Pagamento

No Step 2, `buildCreateOrderRequest()` constrói o array `payments` com lógica multi-payment:

**Lógica do `buildCreateOrderRequest()`:**

```
Se NÃO é coinPayment:
  → 1 payment: fiat 100% (fluxo normal)

Se É coinPayment E isFree (totalPrice = 0):
  → 1 payment: crypto 100% percentage

Se É coinPayment E cryptoAmount >= totalPrice:
  → 1 payment: crypto 100% percentage (auto-submit, spinner)
  → NÃO envia pagamento fiat quando crypto cobre o total

Se É coinPayment E 0 < cryptoAmount < totalPrice:
  → 2 payments: fiat all_remaining (sem campo amount) + crypto fixed
  → Exibe formulário do fiat provider (cartão, PIX, etc.)

Se É coinPayment E cryptoAmount = 0:
  → 1 payment: fiat 100% percentage (fallback, sem pagamento crypto)
```

> **Regra importante:** Quando não há valor a pagar em fiat, NÃO enviar o pagamento fiat na payload. Da mesma forma, quando não há valor crypto, NÃO enviar o pagamento crypto.

**Cenário: pagamento misto (crypto + fiat)**

```json
{
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "amountType": "all_remaining",
      "providerInputs": { "cpf_cnpj": "...", "transparent_checkout": true }
    },
    {
      "currencyId": "uuid-crypto",
      "paymentMethod": "crypto",
      "amountType": "fixed",
      "amount": "50.00"
    }
  ]
}
```

> **Notas sobre a payload:**
> - `all_remaining` NÃO envia campo `amount` (o backend calcula o restante).
> - Pagamento crypto NÃO envia `paymentProvider` nem `providerInputs`.

**Cenário: 100% crypto (auto-submit)**

O pedido é criado automaticamente sem formulário (similar ao Transfer/Free), exibindo spinner.

```json
{
  "payments": [
    {
      "currencyId": "uuid-crypto",
      "paymentMethod": "crypto",
      "amountType": "percentage",
      "amount": "100"
    }
  ]
}
```

**Cenário: 100% fiat (crypto amount = 0)**

Fallback para pagamento fiat padrão, sem incluir pagamento crypto na payload.

```json
{
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": { "cpf_cnpj": "...", "transparent_checkout": true }
    }
  ]
}
```

---

## API Sequence

### Braza
1. `POST /companies/{companyId}/orders/preview` — Preview com cotação Braza
2. `POST /companies/{companyId}/orders` — Criação do pedido com `quote_id`

### ERC-20
1. `POST /companies/{companyId}/orders/preview` — Preview single-payment fiat (100%) com `currencyAllowanceState`
2. `GET /{companyId}/loyalties/users/balance/{userId}` (Key API) — Busca saldo da wallet crypto
3. `PATCH /companies/{companyId}/currencies/{currencyId}/increase-allowance` — Aumentar allowance (se necessário)
4. `POST /companies/{companyId}/orders/preview` — Polling de allowance (6s, até `allowed`)
5. `POST /companies/{companyId}/orders` — Criação do pedido (100% crypto, split crypto+fiat, ou 100% fiat)

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Allowance insuficiente | Modal `AllowanceModal` com botão "Aprovar gasto" |
| Allowance timeout (30s) | Exibe erro no modal, permite retry |
| CPF vinculado a outro email (Braza) | Mensagem de erro específica |
| Cotação Braza expirada / sem `providerData` | Mensagem "Cotação expirada" com instrução para voltar |

---

## Implementação (Next.js)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutConfirmation` | `checkout-confirmation.tsx` | Orquestrador Step 1 — lê URL params, renderiza `CryptoPaymentInput`, gerencia `AllowanceModal` |
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador Step 2 — `buildCreateOrderRequest()` com multi-payment, renderiza `BrazaPaymentForm` |
| `CryptoPaymentInput` | `crypto-payment-input.tsx` | Input livre de valor crypto + resumo do split |
| `AllowanceModal` | `allowance-modal.tsx` | Modal de aprovação de allowance ERC-20 (3 estados: required, processing, error) |
| `BrazaPaymentForm` | `braza-payment-form.tsx` | Formulário CPF/CNPJ + resumo da cotação Braza |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useOrderPreview()` | `useMutation` | Preview sempre single-payment fiat (100%). Parâmetros: `productIds`, `currencyId`, `paymentMethod`, `couponCode`. Valor crypto é estado local. |
| `useWalletBalance()` | `useQuery` | Busca saldo da wallet por `currencyCode` (ex: "FCT") via Key API |
| `useCurrencyAllowance()` | custom hook | Gerencia ciclo completo de allowance (request, polling, timeout) |
| `useCreateOrder()` | `useMutation` | Cria pedido via `POST /orders` |

### API Functions

| Função | Arquivo | Descrição |
|--------|---------|-----------|
| `increaseCurrencyAllowance()` | `commerce.ts` | `PATCH /currencies/{id}/increase-allowance` — solicita aumento de allowance |

### Constantes

| Constante | Valor | Descrição |
|-----------|-------|-----------|
| `ALLOWANCE_POLL_INTERVAL_MS` | `6000` | Intervalo de polling de allowance |
| `ALLOWANCE_TIMEOUT_MS` | `30000` | Timeout para aprovação de allowance |

### Types

| Type | Arquivo | Descrição |
|------|---------|-----------|
| `BrazaProviderData` | `checkout.ts` | Dados da cotação Braza (brlAmount, feesAmount, iof, quoteId, usdAmount, usdQuote, usdVetQuote) |
| `CurrencyAllowanceState` | `checkout.ts` | `'required' \| 'processing' \| 'allowed' \| null` |
| `OrderPreviewCache.isCoinPayment` | `checkout.ts` | Flag booleana para fluxo crypto |
| `OrderPreviewCache.cryptoCurrencyId` | `checkout.ts` | ID da moeda crypto |
| `OrderPreviewCache.cryptoAmount` | `checkout.ts` | Valor crypto informado pelo usuário |

### i18n Keys

| Namespace | Keys |
|-----------|------|
| `checkout.confirmation` | `cryptoAmount`, `cryptoPortion`, `fiatPortion` |
| `checkout.payment.braza` | `usdAmount`, `exchangeRate`, `fees`, `iof`, `quoteExpired` |
| `checkout.payment.allowance` | `title`, `description`, `processing`, `amountNeeded`, `approve`, `timeout` |
