# Pagamento via Transferência Bancária

## Overview

O fluxo mais simples de pagamento. O pedido é criado automaticamente no page load com `providerInputs: {}`. Não requer interação do usuário — exibe spinner enquanto processa, e ao concluir redireciona para a tela de conclusão com status "Pagamento em análise".

Este mesmo fluxo é usado para **pedidos gratuitos** (`totalPrice === 0`), processados pelo provider `free`.

## Prerequisites

- Step 1 (Cart Confirmation) concluído
- `product_cart_info_key` no localStorage com `choosedPayment.paymentMethod` = `transfer`
- Ou `totalPrice === 0` (pedido gratuito)

---

## Steps

### Step 1: Page Load — Processamento Automático

- **Screen**: Spinner com texto "Finalizando pedido..." (`FreeOrderView`).
- **User Action**: Nenhuma — o pagamento é processado automaticamente.

**Dois gatilhos possíveis:**

1. **Transfer**: No `useEffect` do page load, ao detectar `method === PaymentMethodEnum.TRANSFER`, chama `handleSubmit({})` imediatamente.

2. **Free order**: Após debounce de 4 segundos (`FREE_ORDER_DEBOUNCE_MS`), se `parseFloat(totalPrice) === 0`, chama `handleSubmit({})`.

---

### Step 2: Criação do Pedido

- **API Call**:

```
POST /companies/{companyId}/orders
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body (Transfer):**

```json
{
  "orderProducts": [
    {
      "productId": "uuid-produto",
      "expectedPrice": "500.00",
      "quantity": 1,
      "variantIds": []
    }
  ],
  "currencyId": "uuid-brl",
  "paymentMethod": "transfer",
  "providerInputs": {},
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": null,
  "passShareCodeData": {},
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "transfer",
      "paymentProvider": "transfer",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {}
    }
  ]
}
```

**Request Body (Free order):**

```json
{
  "orderProducts": [
    {
      "productId": "uuid-produto-gratis",
      "expectedPrice": "0",
      "quantity": 1,
      "variantIds": []
    }
  ],
  "currencyId": "uuid-brl",
  "paymentMethod": "transfer",
  "providerInputs": {},
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": null,
  "passShareCodeData": {},
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "transfer",
      "paymentProvider": "transfer",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {}
    }
  ]
}
```

> **Nota:** Campos top-level `currencyId`, `paymentMethod`, `providerInputs`, `passShareCodeData` são obrigatórios. `successUrl` usa hostname do tenant via `useTenantBaseUrl()`. `couponCode` envia `null` quando vazio. `signedGasFee`/`signedGasFees` não são enviados.

- **Response Handling** (201 Created):

```json
{
  "id": "uuid-order",
  "status": "confirming_payment",
  "paymentProvider": "transfer",
  "paymentMethod": "transfer",
  "totalAmount": "500.00"
}
```

O frontend:
1. `handleOrderSuccess` detecta que é transfer ou free
2. Redireciona para `/checkout/completed` via `router.push()`

- **Error States**:

| Erro | Ação |
|------|------|
| Erro genérico | Exibe `ErrorMessage` com mensagem + "Contate o suporte" |
| Pedido duplicado | Modal de confirmação |
| Token expirado | Redireciona para login |

---

### Step 3: Tela de Conclusão

Na tela de conclusão (Step 3 do checkout), a transferência bancária mostra:
- **"Pagamento em análise"** — em vez de "Processando na blockchain"
- O admin da empresa precisa confirmar manualmente a transferência
- Enquanto não confirmado, o status permanece `confirming_payment`

---

## API Sequence

1. `POST /companies/{companyId}/orders` — Criação do pedido (automático no page load)

> **Nota:** Não há preview periódico na página de pagamento.

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Erro na criação do pedido | Exibe mensagem de erro + "Contate o suporte" |
| Pedido duplicado | Modal de confirmação (re-envia com `acceptSimilarOrderInShortPeriod: true`) |
| Rede indisponível | Mensagem de erro genérica |

---

## Implementação Real (Next.js)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador — detecta transfer/free e auto-processa |
| `FreeOrderView` | `free-order-view.tsx` | UI: spinner + "Finalizando pedido", mensagem de erro |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useCreateOrder()` | `useMutation` | Cria pedido via `POST /orders` |
| `useTenantBaseUrl()` | custom hook | Resolve `successUrl` para `https://{tenant}/wallet` |

### Constantes

| Constante | Valor | Descrição |
|-----------|-------|-----------|
| `FREE_ORDER_DEBOUNCE_MS` | `4000` | Debounce antes de auto-processar pedido gratuito |
