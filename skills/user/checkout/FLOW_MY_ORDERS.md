---
id: FLOW_MY_ORDERS
title: "Checkout - Meus Pedidos"
module: checkout
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-24"
authors:
  - rafaelmhp
tags:
  - checkout
  - orders
depends_on:
  - CHECKOUT_API_REFERENCE
---

# Tela: Minhas Compras (My Orders)

## Overview

Tela que lista todos os pedidos do usuário com status, produtos e detalhes de preço. Cada pedido é um card expansível que mostra os produtos comprados e breakdown de preços.

## Rota

```
/my-orders
```

Acessível a partir da Home via botão "Minhas Compras". Requer autenticação.

---

## API

### Listar Pedidos

```
GET /companies/{companyId}/orders
Authorization: Bearer <token>
Query Parameters:
  - page: number (default: 1)
  - limit: number (default: 10)
  - sortBy: string (default: 'createdAt')
  - orderBy: 'ASC' | 'DESC' (default: 'DESC')
```

### Response

```typescript
{
  items: OrderListItem[],
  meta: {
    totalItems: number,
    itemCount: number,
    itemsPerPage: number,
    totalPages: number,
    currentPage: number
  }
}
```

### OrderListItem

```typescript
{
  id: string;
  createdAt: string;
  status: OrderStatus;
  paymentProvider: PaymentProvider;
  paymentMethod: PaymentMethodEnum;
  currencyAmount: string;
  totalAmount: string;
  gasFee: string;
  clientServiceFee: string;
  expiresIn: string | null;
  failReason: string | null;
  deliverId: string | null;
  products: OrderProductInfo[];
  currency: CurrencyResponse;
}
```

### OrderProductInfo

```typescript
{
  productToken: {
    id: string;
    product: {
      id: string;
      name: string;
      images?: { thumb: string }[];
    };
  };
  currencyAmount: string | { amount: string; currencyId: string }[];
}
```

---

## Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `MyOrdersPage` | `src/app/my-orders/page.tsx` | Página com header e lista |
| `OrderList` | `src/components/orders/order-list.tsx` | Gerencia paginação e renderiza cards |
| `OrderCard` | `src/components/orders/order-card.tsx` | Card expansível de um pedido |
| `OrderStatusBadge` | `src/components/checkout/order-status-badge.tsx` | Badge colorido por status (reutilizado do checkout) |

### Hook

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetOrders(page, limit)` | `useQuery` | Busca lista paginada de pedidos |

---

## UI

### Estado Fechado (Card)

```
┌──────────────────────────────────────────┐
│ [Badge: Status]                    R$ XX │
│ DD/MM/YY HH:mm                       [v] │
└──────────────────────────────────────────┘
```

### Estado Expandido (Card)

```
┌──────────────────────────────────────────┐
│ [Badge: Status]                    R$ XX │
│ DD/MM/YY HH:mm                       [^] │
│──────────────────────────────────────────│
│ ID do pedido          abc-123-def-456    │
│                                          │
│ [img] Nome do Produto          R$ XX.XX  │
│ [img] Nome do Produto          R$ XX.XX  │
│                                          │
│ Subtotal                       R$ XX.XX  │
│ Taxa de serviço                R$ X.XX   │
│ Taxa de rede                   R$ X.XX   │
│──────────────────────────────────────────│
│ Total                          R$ XX.XX  │
│                                          │
│ (failReason se houver)                   │
└──────────────────────────────────────────┘
```

### Paginação

Exibida quando há mais de 1 página. Botões anterior/próximo com indicador de página atual.

---

## Implementação

### Estrutura de Arquivos

```
src/
  app/my-orders/page.tsx           # Rota da página
  components/orders/
    order-list.tsx                  # Lista com paginação
    order-card.tsx                  # Card expansível
  hooks/use-get-orders.ts          # Hook de busca
  types/checkout.ts                # Tipos (OrderListItem, PaginatedResponse)
  lib/api/commerce.ts              # Função getOrders()
```

### Paginação

- Default: 10 itens por página
- Ordenação: `createdAt DESC` (mais recentes primeiro)
- Controlada via estado local `page`
- API retorna `meta.totalPages` para controle dos botões

### Informações Exibidas

- **Status**: via `OrderStatusBadge` (mesmas cores do checkout completion)
- **Data**: formatada em `DD/MM/YY HH:mm`
- **Total**: destaque no header do card
- **Produtos**: imagem thumbnail + nome + preço individual
- **Breakdown**: subtotal, taxa de serviço (se > 0), taxa de rede (se > 0), total
- **Erro**: `failReason` em vermelho se o pedido falhou
