# Checkout Skill Index

Índice master de toda a documentação de checkout do W3block. Use este documento como ponto de entrada para implementar qualquer parte do fluxo de checkout.

---

## Documentos

| # | Documento | Descrição | Status | Quando usar |
|---|-----------|-----------|--------|-------------|
| 1 | [CHECKOUT_API_REFERENCE.md](./CHECKOUT_API_REFERENCE.md) | Endpoints, schemas JSON, enums, erros, constantes | ✅ Referência | Consulta de API em qualquer momento |
| 2 | [FLOW_CHECKOUT_OVERVIEW.md](./FLOW_CHECKOUT_OVERVIEW.md) | Visão geral do fluxo em 3 passos, arquitetura Next.js | ✅ Atualizado | Entender a arquitetura antes de implementar |
| 3 | [FLOW_CHECKOUT_CART_CONFIRMATION.md](./FLOW_CHECKOUT_CART_CONFIRMATION.md) | Step 1: Carrinho, preview, seleção de método, cupom | ✅ Implementado | Implementar tela de confirmação |
| 4 | [FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md](./FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md) | Cartão de crédito via Asaas/Pagar.me | ✅ Implementado | Checkout transparente com cartão |
| 5 | [FLOW_CHECKOUT_PAYMENT_PIX.md](./FLOW_CHECKOUT_PAYMENT_PIX.md) | PIX com QR code (Asaas) + iframe (Pagar.me) | ✅ Implementado | Pagamento PIX com polling |
| 6 | [FLOW_CHECKOUT_PAYMENT_STRIPE.md](./FLOW_CHECKOUT_PAYMENT_STRIPE.md) | Stripe: Elements (credit_card) + iframe (PIX) | ✅ Implementado | Pagamento via Stripe |
| 7 | [FLOW_CHECKOUT_PAYMENT_CRYPTO.md](./FLOW_CHECKOUT_PAYMENT_CRYPTO.md) | Braza (fiat-to-crypto) + ERC-20 com allowance | ✅ Implementado | Crypto nativo ou bridge |
| 8 | [FLOW_CHECKOUT_PAYMENT_TRANSFER.md](./FLOW_CHECKOUT_PAYMENT_TRANSFER.md) | Transferência bancária / pedidos gratuitos | ✅ Implementado | Pagamento manual ou free |
| 9 | [FLOW_CHECKOUT_COMPLETION.md](./FLOW_CHECKOUT_COMPLETION.md) | Step 3: Status, polling, resumo de compra | ✅ Implementado | Tela de conclusão |
| 10 | [FLOW_MY_ORDERS.md](./FLOW_MY_ORDERS.md) | Lista de pedidos do usuário com status e detalhes | ✅ Implementado | Tela "Minhas Compras" |

---

## Guia Rápido

### Para implementar um checkout básico:

```
1. Leia: FLOW_CHECKOUT_OVERVIEW.md         → Entenda o fluxo geral
2. Leia: FLOW_CHECKOUT_CART_CONFIRMATION.md → Implemente Step 1
3. Leia: [seu método de pagamento]          → Implemente Step 2
4. Leia: FLOW_CHECKOUT_COMPLETION.md        → Implemente Step 3
5. Consulte: CHECKOUT_API_REFERENCE.md      → Para detalhes de schemas/enums
```

### Implementação mínima (API-first):

```
1. POST /companies/{companyId}/orders/preview   → Calcular preços
2. POST /companies/{companyId}/orders            → Criar pedido + pagar
3. GET  /companies/{companyId}/orders/{orderId}  → Polling de status
```

### Armadilhas Comuns (leia antes de implementar!)

| # | Problema | Solução |
|---|----------|---------|
| 1 | `installmentPrice.toFixed()` dá TypeError | API pode retornar como string. Usar `Number(inst.installmentPrice).toFixed(2)` |
| 2 | Todos os preços da API são `string` | Sempre usar `parseFloat()` para cálculos numéricos |
| 3 | `providersForSelection` pode estar vazio | Verificar `providers?.length > 0` antes de auto-selecionar |
| 4 | Cache do localStorage pode ser inválido | Sempre envolver `JSON.parse()` em try/catch |

---

## Tabela de Decisão: Método de Pagamento

| Se o `paymentProvider` é... | E o `paymentMethod` é... | Use este documento | Status |
|----------------------------|--------------------------|-------------------|--------|
| `asaas` | `credit_card` | [Credit Card](./FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md) | ✅ Implementado |
| `asaas` | `pix` | [PIX](./FLOW_CHECKOUT_PAYMENT_PIX.md) | ✅ Implementado |
| `pagar_me` | `credit_card` | [Credit Card](./FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md) | ✅ Implementado (todos os campos) |
| `pagar_me` | `pix` | [PIX](./FLOW_CHECKOUT_PAYMENT_PIX.md) (via iframe) | ✅ Implementado |
| `stripe` | `credit_card` | [Stripe](./FLOW_CHECKOUT_PAYMENT_STRIPE.md) (Fluxo A: Elements) | ✅ Implementado |
| `stripe` | `pix` | [Stripe](./FLOW_CHECKOUT_PAYMENT_STRIPE.md) (Fluxo B: iframe) | ✅ Implementado |
| `braza` | `crypto` | [Crypto](./FLOW_CHECKOUT_PAYMENT_CRYPTO.md) | ✅ Implementado |
| `crypto` | `crypto` | [Crypto](./FLOW_CHECKOUT_PAYMENT_CRYPTO.md) | ✅ Implementado |
| `transfer` | `transfer` | [Transfer](./FLOW_CHECKOUT_PAYMENT_TRANSFER.md) | ✅ Implementado |
| `free` | (qualquer) | [Transfer](./FLOW_CHECKOUT_PAYMENT_TRANSFER.md) | ✅ Implementado |

---

## Matriz: Endpoints x Documentos

| Endpoint | Método | Cart Confirmation | Credit Card | PIX | Stripe | Crypto | Transfer | Completion |
|----------|--------|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `POST .../orders/preview` | Preview | **X** | | | | **X** | | |
| `POST .../orders` | Create Order | | **X** | **X** | **X** | **X** | **X** | |
| `GET .../orders/{id}` | Get Status | | | **X** | | | | **X** |
| `PATCH .../orders/{id}/pay` | Pay Order | | | | **X** | | | |

---

## Fluxo Visual

```
┌────────────────────────────────────────────────────────────────────────┐
│                         CHECKOUT W3BLOCK                               │
│                                                                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │
│  │   Step 1:    │    │   Step 2:    │    │   Step 3:    │              │
│  │ Confirmação  │───→│  Pagamento   │───→│  Conclusão   │              │
│  └──────────────┘    └──────────────┘    └──────────────┘              │
│        │                    │                    │                      │
│        ▼                    ▼                    ▼                      │
│  ┌──────────┐    ┌──────────────────┐    ┌───────────────┐             │
│  │ Preview  │    │ ┌──────────────┐ │    │ ┌───────────┐ │             │
│  │ API      │    │ │ Credit Card  │ │    │ │ Gift Card │ │             │
│  │ Cupom    │    │ │ PIX          │ │    │ │ Coin Pay  │ │             │
│  │ Método   │    │ │ Stripe       │ │    │ │ Normal    │ │             │
│  │ Moeda    │    │ │ Crypto       │ │    │ └───────────┘ │             │
│  └──────────┘    │ │ Transfer     │ │    └───────────────┘             │
│                  │ └──────────────┘ │                                   │
│                  └──────────────────┘                                   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Glossário Rápido

| Termo | Descrição |
|-------|-----------|
| `companyId` | UUID do tenant (empresa) na plataforma W3block |
| `currencyId` | UUID da moeda de pagamento |
| `productId` | UUID do produto digital |
| `OrderPreviewCache` | Dados transitados via localStorage do Step 1 → Step 2 |
| `CreateOrderResponse` | Resposta da criação de pedido, transitada do Step 2 → Step 3 |
| `providerInputs` | Dados específicos do provider de pagamento (cartão, CPF, etc.) |
| `paymentProvider` | Serviço que processa o pagamento (asaas, stripe, pagar_me...) |
| `paymentMethod` | Tipo de pagamento (credit_card, pix, crypto, transfer) |
| `deliverId` | ID de entrega — usado para tracking pós-compra |
| `passShareCode` | Código de gift card para compartilhamento |
| `allowance` | Permissão ERC-20 para gastar tokens em nome do usuário |
| `STORAGE_KEYS` | Constantes TypeScript para chaves de localStorage (`PRODUCT_CART_INFO`, `ORDER_COMPLETED_INFO`) |
| `useOrderPreview` | Hook (useMutation) que chama a API de preview |
| `useCreateOrder` | Hook (useMutation) que cria pedido e auto-salva no localStorage |
| `useOrderStatusPolling` | Hook (useQuery) que faz polling de status a cada 3s |
| `CheckoutPayment` | Componente orquestrador do Step 2 — roteia entre CreditCard, PIX, Stripe, Transfer/Free |
