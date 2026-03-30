---
id: FLOW_CHECKOUT_COMPLETION
title: "Checkout - Conclusao"
module: checkout
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-24"
authors:
  - rafaelmhp
tags:
  - checkout
  - completion
depends_on:
  - FLOW_CHECKOUT_OVERVIEW
  - CHECKOUT_API_REFERENCE
---

# Step 3: Tela de Conclusão (Completion)

## Overview

A tela de conclusão exibe o resultado da compra após o pagamento. Mostra o status atual do pedido, faz polling automático para status transicionais, e exibe resumo do produto + preços do cache do Step 1.

> **Nota:** Diferente do SDK original que tem 3 modos de renderização (Gift Card, Coin Payment, Normal), a implementação Next.js atual tem **um modo único** com status dinâmico. Gift cards e coin payment não estão implementados.

## Prerequisites

- Step 2 (Payment) concluído
- `order_completed_info_key` no localStorage com `CreateOrderResponse`
- `product_cart_info_key` no localStorage com `OrderPreviewCache` (dados do Step 1)

---

## Steps

### Step 1: Page Load — Leitura dos Caches

- **Screen**: Carregando (spinner ou resumo parcial dos dados em cache).
- **User Action**: Nenhuma (automático).

**Dados lidos do localStorage:**

| Chave | Dados usados |
|-------|-------------|
| `order_completed_info_key` | `id`, `status`, `failReason`, `deliverDelayMessage` |
| `product_cart_info_key` | `products`, `payments` (preview cache do Step 1) |

- **State Changes**: Se status é transitional, inicia polling. Caso contrário, exibe resultado final.

---

### Step 2: Polling de Status (condicional)

Ativado quando o status atual é **transitional** (`PENDING`, `CONFIRMING_PAYMENT`, `WAITING_DELIVERY`, `DELIVERING`).

- **API Call** (a cada 3 segundos via `useOrderStatusPolling`):

```
GET /companies/{companyId}/orders/{orderId}?fetchNewestStatus=true
Authorization: Bearer <token>
```

- **Response Handling**:

| Status retornado | Ação |
|-----------------|------|
| `pending`, `confirming_payment`, `waiting_delivery`, `delivering` | Continua polling. Exibe spinner. |
| `concluded` | Para polling. Exibe check verde + "Compra realizada com sucesso!" |
| `failed` | Para polling. Exibe X vermelho + `failReason` |
| `cancelled`, `cancelling`, `partially_cancelled` | Para polling. Exibe X vermelho + "Pedido cancelado" |
| `expired` | Para polling. Exibe X vermelho + "Pedido expirado" |

- **State Changes**: `currentOrder` usa polled data se disponível, senão usa dados iniciais do localStorage.

---

### Step 3: Renderização

- **Screen**: Card único com:

**Status Icon:**
| Estado | Ícone |
|--------|-------|
| Transitional (pending, confirming, waiting, delivering) | `Loader2` animado (spinner) |
| Sucesso (concluded) ou Entregando (waiting_delivery, delivering) | `CheckCircle` verde |
| Falha / Cancelado / Expirado | `XCircle` vermelho (destructive) |

**Mensagem:**
| Estado | Mensagem |
|--------|----------|
| Transitional | "Processando sua compra..." + aviso "Você pode sair desta página. O processo continuará normalmente." |
| Concluded | "Compra realizada com sucesso!" |
| Waiting/Delivering | "Pagamento confirmado!" |
| Failed | "A compra falhou" |
| Cancelled | "Pedido cancelado" |
| Expired | "Pedido expirado" |

**Componentes exibidos:**
1. `OrderStatusBadge` — badge colorido com label traduzido do status
2. `ProductSummary` — para cada produto do preview cache
3. `PriceBreakdown` — subtotal, fees, total (com desconto de cupom se aplicável)
4. Se `isFailed` e `failReason` contém "successful start delivery": mensagem de falha na entrega + aviso de estorno
5. Se `isFailed` e `failReason` outro: mensagem de erro genérica em vermelho

**Botão de Ação:**
| Condição | Label | Ação |
|----------|-------|------|
| Sucesso / Entregando | "Ir para Home" | Limpa localStorage + `router.push('/')` |
| Falha / Cancelado / Expirado | "Tentar novamente" | Limpa localStorage + redireciona para Step 1 com mesmos `productIds` e `currencyId` |

> **Nota:** O botão "Tentar novamente" reconstrói a URL de confirmação: `/checkout/confirmation?productIds=id1,id2&currencyId=uuid`

---

## API Sequence

1. `GET /companies/{companyId}/orders/{orderId}?fetchNewestStatus=true` — Polling a cada 3s (para status transicionais)

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Pedido falhou (`status: failed`) | Exibe `failReason` + botão "Tentar novamente" → volta ao Step 1 |
| Falha na entrega (`failReason` contém "successful start delivery") | Exibe mensagem especial: "Não foi possível entregar o produto" + "O pagamento será estornado. Entre em contato com o administrador." |
| Pedido cancelado/expirado | Exibe mensagem + botão "Tentar novamente" → volta ao Step 1 |
| Status transitional | Spinner + polling automático a cada 3s + aviso de que o usuário pode sair da página sem perder o progresso |
| Sem `ORDER_COMPLETED_INFO` no cache | Redireciona para `/` (home) |

### Erro "None of order products could have a successful start delivery"

Este erro indica que o pagamento foi processado com sucesso, mas a entrega do produto falhou. As causas mais comuns são:

- **Falta de saldo na carteira da empresa** — a empresa não possui tokens/saldo suficiente para transferir ao comprador.
- **Falta de estoque do produto** — o produto não possui unidades disponíveis para entrega.

Quando este erro é detectado no `failReason`, a tela exibe ao usuário:
1. "Não foi possível concluir seu pedido."
2. "O pagamento será estornado. Em caso de dúvidas, entre em contato com o administrador."

> **Importante:** Não exibir o motivo técnico ao usuário final. As causas (saldo/estoque) são informação interna para o administrador do tenant.

---

## Implementação Real (Next.js)

### Componente Principal: `CheckoutCompletion`

Arquivo: `src/components/checkout/checkout-completion.tsx`

### Componentes Utilizados

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutCompletion` | `checkout-completion.tsx` | Orquestrador — carrega caches, gerencia polling, renderiza status |
| `ProductSummary` | `product-summary.tsx` | Card de produto (compartilhado com Step 1) |
| `PriceBreakdown` | `price-breakdown.tsx` | Breakdown de preços (compartilhado com Step 1) |
| `OrderStatusBadge` | `order-status-badge.tsx` | Badge colorido por status (verde/amarelo/vermelho/azul) |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useOrderStatusPolling()` | `useQuery` | Polling a cada 3s. Ativado apenas para status transicionais |

### Constantes de Status

```typescript
const TRANSITIONAL_STATUSES = [PENDING, CONFIRMING_PAYMENT, WAITING_DELIVERY, DELIVERING];
const FAILURE_STATUSES = [FAILED];
const CANCELLED_STATUSES = [CANCELLED, CANCELLING, PARTIALLY_CANCELLED];
```

### Guard no Page Load

Se `ORDER_COMPLETED_INFO` não existe no localStorage → redireciona para `/` (home).
Se `PRODUCT_CART_INFO` existe → carrega para exibir produtos e preços (opcional, apenas para display).

### Cleanup do localStorage

A função `cleanup()` remove ambas as chaves (`PRODUCT_CART_INFO` e `ORDER_COMPLETED_INFO`) antes de navegar para qualquer outra página.
