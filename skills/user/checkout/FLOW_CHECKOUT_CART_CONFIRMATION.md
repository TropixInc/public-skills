---
id: FLOW_CHECKOUT_CART_CONFIRMATION
title: "Checkout - Confirmacao de Carrinho"
module: checkout
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-24"
authors:
  - rafaelmhp
tags:
  - checkout
  - cart
depends_on:
  - FLOW_CHECKOUT_OVERVIEW
  - CHECKOUT_API_REFERENCE
---

# Step 1: Confirmação do Carrinho (Cart Confirmation)

## Overview

Este é o primeiro passo do checkout. O usuário vê os produtos selecionados, preços calculados, métodos de pagamento disponíveis, pode aplicar cupom de desconto e selecionar a moeda de pagamento. Ao clicar em "Continuar", os dados são salvos no localStorage e o usuário é redirecionado para o Step 2 (Pagamento).

## Prerequisites

- Usuário autenticado (Bearer token válido)
- Wallet associada ao usuário
- `productIds` e `currencyId` na query string da URL
- `companyId` configurado no ambiente

---

## Steps

### Step 1: Page Load e Inicialização

- **Screen**: Card de checkout com spinner de loading.
- **User Action**: Nenhuma (automático).
- **Frontend Logic**: Extrai `productIds` (CSV) e `currencyId` da query string via `useSearchParams()`. Se vazios, exibe mensagem "Carrinho vazio". A autenticação é gerenciada pelo middleware do `next-auth`, não em componente.
- **API Call**: Nenhuma (apenas parse de URL).
- **State Changes**: `productIds` e `currencyId` são extraídos. `useUserWallet()` inicia busca da wallet.

---

### Step 2: Parse de URL Parameters

- **Screen**: Card de checkout com loading.
- **User Action**: Nenhuma (automático).
- **Frontend Validation**: Extrai `productIds` (CSV split) e `currencyId` da query string. Se vazios, exibe card com mensagem de carrinho vazio.

**Parâmetros extraídos:**

```
URL: /checkout/confirmation?productIds=uuid1,uuid2&currencyId=uuid-brl

productIds → ["uuid1", "uuid2"]
currencyId → "uuid-brl"
```

- **State Changes**: `productIds` (array) e `currencyId` (string) disponíveis via `useSearchParams()`.

---

### Step 3: Chamada Order Preview

- **Screen**: Spinner de loading enquanto calcula.
- **User Action**: Nenhuma (automático, disparado por `useDebounce` 300ms após `productIds`, `currencyIdState` e `token` estarem disponíveis).

- **API Call**:

```
POST /companies/{companyId}/orders/preview
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body (pagamento simples):**

```json
{
  "orderProducts": [
    {
      "productId": "uuid-produto-1",
      "quantity": "1",
      "selectBestPrice": true
    }
  ],
  "payments": [
    {
      "currencyId": "uuid-brl",
      "amountType": "percentage",
      "amount": "100",
      "paymentMethod": "credit_card"
    }
  ],
  "currencyId": "uuid-brl",
  "acceptIncompleteCart": true,
  "couponCode": ""
}
```

> **Nota:** O preview usa `OrderProductPreview` (com `quantity` como `string` e `selectBestPrice: true`), que é diferente do `OrderProduct` usado no Create Order (com `quantity` como `number` e `selectBestPrice` condicional — `true` apenas para produtos ERC-20, `undefined` para os demais).

> **Nota sobre coin payment:** Mesmo no fluxo ERC-20 (coin payment), o preview é **sempre single-payment fiat** (100%). O valor crypto é gerenciado como estado local no componente (`cryptoAmount`) e o split crypto/fiat é calculado localmente para exibição. O multi-payment (fiat + crypto) é construído apenas no momento da criação do pedido (`buildCreateOrderRequest` no Step 2).

- **Response Handling**:

O response `OrderPreviewResponse` contém:

| Campo processado | O que acontece |
|------------------|----------------|
| `providersForSelection` | Lista de métodos de pagamento disponíveis. Se nenhum método estava selecionado, seleciona o primeiro automaticamente. |
| `productsErrors` | Se existem erros (ex: limite de compra), salva em estado e exibe alerta. |
| `products` | Lista de produtos validados com preços calculados. |
| `payments` | Detalhamento de pagamentos por moeda (totalPrice, cartPrice, fees). |
| `appliedCoupon` | Se um cupom foi aplicado com sucesso. |
| `cashback` | Informações de cashback (para coin payments). |
| `availableInstallments` | Parcelas disponíveis (atualiza o `choosedPayment` se cartão selecionado). |

- **Error States**:

| Erro | Ação |
|------|------|
| `400` — Produto não encontrado | Exibe `requestError` na tela |
| `resale-purchase-batch-size-error` | Exibe alerta inline (não bloqueia preview) |
| `Not Found` | Exibe alerta inline |
| Erro genérico de rede | Exibe mensagem + botão "Voltar para Home" |

- **State Changes**: `orderPreview` atualizado, `choosedPayment` definido, `productErros` se houver.

---

### Step 4: Exibição do Preview

- **Screen**: O usuário vê:
  - **ProductSummary** — card com imagem (ou vídeo thumbnail), nome, descrição, preço unitário
  - **PriceBreakdown** — subtotal (`cartPrice`), taxa de serviço (`clientServiceFee`), gas fee (`gasFee.gasFee`), total (`totalPrice`)
  - Se houver desconto de cupom: preço original riscado (`originalCartPrice`) + preço com desconto
  - **PaymentMethodSelector** — RadioGroup com métodos disponíveis
  - **CouponInput** — campo de texto + botão "Aplicar"

- **User Action**: O usuário revisa os dados e pode modificar método de pagamento ou aplicar cupom.

---

### Step 5: Seleção de Método de Pagamento

- **Screen**: `PaymentMethodSelector` — RadioGroup com label e ícone para cada método (CreditCard icon para `credit_card`, QrCode icon para `pix`).

- **User Action**: Seleciona um método de pagamento.

- **Frontend Logic**: Cada provider é identificado por `${paymentProvider}-${paymentMethod}` como key única do RadioGroup.

- **State Changes**: `choosedPayment` atualizado. Dispara novo preview com debounce de 300ms (`PREVIEW_DEBOUNCE_MS`).

**Re-preview automático:** Ao mudar o método, o preview é refeito para recalcular preços/parcelas.

---

### Step 6: Aplicação de Cupom (Opcional)

- **Screen**: Campo de texto "Código do cupom" + botão "Aplicar".

- **User Action**: Digita o código e clica "Aplicar".

- **Frontend Validation**: Nenhuma validação local — o cupom é validado pelo backend no Order Preview.

- **API Call**: Novo Order Preview com `couponCode` preenchido:

```json
{
  "orderProducts": [...],
  "currencyId": "uuid-brl",
  "couponCode": "DESCONTO10",
  "payments": [...]
}
```

- **Response Handling**:
  - Se `appliedCoupon` retorna com valor: exibe "Cupom 'DESCONTO10' aplicado com sucesso!" em verde. Os preços são recalculados (preço original riscado, novo preço exibido).
  - Se `appliedCoupon` retorna `null`: exibe "Cupom inválido" em vermelho.

- **State Changes**: `couponCodeInput` atualizado. Preços recalculados via novo preview.

**UTM como cupom automático:** Se existe `utmParams.utm_campaign` não expirado, é usado automaticamente como código de cupom na primeira chamada de preview.

---

### Step 7: Seleção de Moeda (Modo Carrinho) — NÃO IMPLEMENTADO

> **Nota:** Esta funcionalidade existe no SDK original mas **não está implementada** na versão Next.js atual. O `currencyId` é fixo, vindo da query string.

---

### Step 8: Click "Continuar" — Construção do OrderPreviewCache

- **Screen**: Botão "Continuar" habilitado quando: preview carregado, método de pagamento selecionado, não está em loading.

- **User Action**: Clica no botão "Continuar".

- **Frontend Validation**:
  - `preview.data` deve existir (não nulo)
  - `choosedPayment` deve estar selecionado
  - Preview não pode estar carregando

- **State Changes**: A função `handleContinue()` executa:

1. **Constrói `OrderPreviewCache`** — mapeia produtos com `expectedPrice`, `quantity: 1`, `variantIds: []`. Inclui `signedGasFee`, `choosedPayment`, `destinationWalletAddress`.

2. **Salva no localStorage** (chave: `STORAGE_KEYS.PRODUCT_CART_INFO`)

3. **Navegação**: `router.push('/checkout/payment')`

> **Nota:** Coin payment (ERC-20) e crypto allowance estão implementados. Não há suporte a custom `proccedAction` nesta implementação.

---

## API Sequence

1. `POST /companies/{companyId}/orders/preview` — Preview inicial (automático no page load)
2. `POST /companies/{companyId}/orders/preview` — Re-preview ao mudar método de pagamento
3. `POST /companies/{companyId}/orders/preview` — Re-preview ao aplicar cupom
4. `POST /companies/{companyId}/orders/preview` — Re-preview ao mudar moeda
5. `POST /companies/{companyId}/orders/preview` — Re-preview periódico (30s se coin payment)

> **Nota**: Todas as chamadas de preview usam o mesmo endpoint. O debounce de 300ms evita chamadas excessivas.

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Preview retorna erro genérico | Exibe mensagem de erro + botão "Voltar para Home" |
| Preview retorna `productsErrors` | Exibe alerta por produto com o limite atingido |
| Preview retorna `appliedCoupon: null` com couponCode | Exibe "Cupom inválido" em vermelho |
| Carrinho sem moedas comuns | Exibe lista de produtos incompatíveis + botão "Esvaziar carrinho" |
| Usuário é `commerce.orderReceiver` | Bloqueia toda a tela com "Usuário não pode comprar" |
| Token expirado durante preview | Redireciona para login |
| Coin payment sem saldo suficiente | Desabilita botão "Continuar" + exibe alerta de saldo insuficiente |

---

## Implementação Real (Next.js)

### Componente Principal: `CheckoutConfirmation`

Arquivo: `src/components/checkout/checkout-confirmation.tsx`

O componente é um `'use client'` component que gerencia todo o fluxo de confirmação.

### Componentes Utilizados

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutConfirmation` | `checkout-confirmation.tsx` | Orquestrador — state, API calls, cache, navegação, fluxo crypto |
| `ProductSummary` | `product-summary.tsx` | Card de produto (imagem/vídeo, nome, descrição, preço) |
| `PriceBreakdown` | `price-breakdown.tsx` | Subtotal, fees, total, desconto de cupom (strikethrough) |
| `PaymentMethodSelector` | `payment-method-selector.tsx` | RadioGroup com providers + ícones (CreditCard, QrCode) |
| `CouponInput` | `coupon-input.tsx` | Input de cupom + botão aplicar + feedback verde |
| `CryptoPaymentInput` | `crypto-payment-input.tsx` | Input controlado de valor crypto + saldo da wallet (coin payment) |
| `AllowanceModal` | `allowance-modal.tsx` | Modal de aprovação de allowance ERC-20 (coin payment) |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useOrderPreview()` | `useMutation` | Chama `POST /orders/preview` (sempre single-payment fiat 100%). Parâmetros: `productIds`, `currencyId`, `paymentMethod`, `couponCode` |
| `useUserWallet()` | `useQuery` | Busca wallet address do usuário via `GET /users/profile` |
| `useWalletBalance()` | `useQuery` | Busca saldo da wallet por `currencyCode` via Key API (usado no `CryptoPaymentInput`) |
| `useCurrencyAllowance()` | custom hook | Gerencia ciclo de allowance ERC-20: request, polling (6s), timeout (30s) |

### Fluxo Simplificado

```
1. Mount → extrair productIds, currencyId, coinPayment, cryptoCurrencyId da URL (useSearchParams)
2. Limpar caches anteriores (product_cart_info_key, order_completed_info_key, stripe_order_cache)
3. Chamar preview API automaticamente (sempre single-payment fiat 100%)
4. Auto-selecionar primeiro método de pagamento disponível
5. Usuário seleciona método → debounce 300ms → re-preview
6. Usuário aplica cupom → re-preview com couponCode
7. (Coin payment) Usuário insere valor crypto → estado local, sem re-preview
8. (Coin payment + allowance required) Clicar "Continuar" → AllowanceModal → PATCH increase-allowance → polling
9. Clicar "Continuar" → construir OrderPreviewCache (com campos crypto se aplicável) → salvar localStorage → navegar para /checkout/payment
```

### Guard de Carrinho Vazio

Se `productIds` ou `currencyId` estão vazios, renderiza card com mensagem "Carrinho vazio".

### Construção do Cache (handleContinue)

O cache `OrderPreviewCache` é construído assim:

```typescript
const orderProducts = data.products.map((p) => {
  const isErc20Product = p.type === 'erc20';
  const priceInCurrency = p.prices?.find((pr) => pr.currencyId === currencyId);

  let expectedPrice: string;
  if (isErc20Product) {
    // ERC-20: usa totalPrice com selectBestPrice: true
    expectedPrice = data.totalPrice ?? '0';
  } else if (priceInCurrency?.amount) {
    // Preço do produto na moeda selecionada
    expectedPrice = priceInCurrency.amount;
  } else if (data.products.length === 1) {
    // Produto único: cartPrice = preço do produto (sem fees)
    expectedPrice = data.cartPrice ?? '0';
  } else {
    expectedPrice = '0';
  }

  return {
    productId: p.id,
    expectedPrice,
    quantity: 1,
    variantIds: [],
    selectBestPrice: isErc20Product ? true : undefined,
  };
});

const cache: OrderPreviewCache = {
  payments: data.payments,
  products: data.products,
  orderProducts,
  currencyId,
  signedGasFee: data.gasFee?.signature ?? '',
  totalPrice: data.totalPrice,
  clientServiceFee: data.clientServiceFee,
  gasFee: data.gasFee ?? { signature: '', gasFee: '0' },
  cartPrice: data.cartPrice,
  choosedPayment,
  couponCode: appliedCoupon ?? null,
  originalCartPrice: data.originalCartPrice,
  originalClientServiceFee: data.originalClientServiceFee,
  originalTotalPrice: data.originalTotalPrice,
  destinationWalletAddress: walletAddress ?? '',
  // Campos crypto (apenas para coin payment)
  ...(isCryptoFlow && {
    isCoinPayment: true,
    cryptoCurrencyId,
    cryptoAmount,
  }),
};
```

> **Notas sobre o cache:**
> - `expectedPrice`: Para ERC-20 usa `totalPrice` (backend calcula com `selectBestPrice`). Para outros, usa preço do produto na moeda selecionada, fallback para `cartPrice` (produto único) ou `'0'`.
> - `selectBestPrice`: Apenas `true` para produtos ERC-20, `undefined` para os demais.
> - `signedGasFee` (string) — `signedGasFees` (array) foi removido.
> - `couponCode` usa `null` (não string vazia `''`) quando nenhum cupom está aplicado.
> - Campos crypto (`isCoinPayment`, `cryptoCurrencyId`, `cryptoAmount`) são condicionalmente incluídos.
