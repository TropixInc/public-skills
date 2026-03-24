# Checkout Flow — Visão Geral

## Overview

O checkout do W3block é um fluxo de **3 passos** que permite a compra de produtos digitais (NFTs, tokens ERC-20, gift cards, etc.) com 5 métodos de pagamento diferentes. O fluxo é gerenciado por dados transitados via **localStorage** entre as páginas.

```
┌─────────────────┐     localStorage      ┌─────────────────┐     localStorage      ┌─────────────────┐
│   Step 1:       │    (product_cart_     │   Step 2:       │    (order_completed_  │   Step 3:       │
│   Confirmação   │───  info_key)  ─────→│   Pagamento     │───  info_key)  ─────→│   Conclusão     │
│   do Carrinho   │                       │                 │                       │                 │
└─────────────────┘                       └─────────────────┘                       └─────────────────┘
```

**Passos:**

1. **Confirmação (Cart Confirmation)** — Exibe produtos, preços, métodos de pagamento. O usuário seleciona método, aplica cupom, confirma. Dados salvos no localStorage como `OrderPreviewCache`.
2. **Pagamento (Payment)** — Processa o pagamento de acordo com o método escolhido. Cada método tem um subfluxo diferente. Ao concluir, salva `CreateOrderResponse` no localStorage.
3. **Conclusão (Done)** — Mostra o resultado da compra. Faz polling do status até conclusão. Exibe gift cards, QR codes, recibos conforme o tipo de produto.

---

## Pré-requisitos

Para iniciar um checkout, você precisa de:

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | ID da empresa (tenant) | Configuração do ambiente / `useCompanyConfig()` |
| `productId(s)` | ID(s) do(s) produto(s) a comprar | URL da loja / catálogo de produtos |
| `currencyId` | ID da moeda para pagamento | Obtido do produto (`product.prices[].currencyId`) |
| **Autenticação** | Bearer token do pixwayid | Login do usuário via Identity API |
| **Wallet** | Wallet associada ao usuário | Criada automaticamente no primeiro login |

---

## Query Parameters de Entrada

O Step 1 (Confirmação) recebe dados via query string na URL:

| Parâmetro | Tipo | Obrigatório | Status | Descrição |
|-----------|------|-------------|--------|-----------|
| `productIds` | `string` (CSV) | Sim | ✅ Implementado | IDs dos produtos separados por vírgula |
| `currencyId` | `uuid` | Sim | ✅ Implementado | ID da moeda de pagamento |
| `cart` | `"true"` | Não | ❌ Não implementado | Ativa modo carrinho (múltiplos produtos do CartProvider) |
| `coinPayment` | `"true"` | Não | ✅ Implementado | Indica pagamento com crypto/loyalty token |
| `cryptoCurrencyId` | `uuid` | Não | ✅ Implementado | ID da moeda crypto para pagamento misto |
| `destination` | `string` | Não | ❌ Não implementado | Slug do usuário de destino (gift) |
| `batchSize` | `string` | Não | ❌ Não implementado | Tamanho do lote (para compras em quantidade) |
| `quantity` | `string` | Não | ❌ Não implementado | Quantidade específica |
| `sessionId` | `string` | Não | ❌ Não implementado | ID de sessão (para dados de practitioner) |

---

## Chaves de localStorage

O checkout usa localStorage para transitar dados entre os 3 passos:

| Chave | Tipo | Escrita em | Leitura em | Descrição |
|-------|------|-----------|------------|-----------|
| `product_cart_info_key` | `OrderPreviewCache` | Step 1 (Confirmation) | Step 2 (Payment), Step 3 (Completion) | Cache completo do preview com produtos, preços, método de pagamento escolhido |
| `order_completed_info_key` | `CreateOrderResponse` | Step 2 (Payment) via `useCreateOrder()` | Step 3 (Completion), Step 2 (guard) | Resposta da criação do pedido com IDs, status, dados de pagamento |

> **Nota:** Diferente do SDK React original, esta implementação usa apenas 2 chaves de localStorage. As constantes são definidas em `src/types/checkout.ts` como `STORAGE_KEYS.PRODUCT_CART_INFO` e `STORAGE_KEYS.ORDER_COMPLETED_INFO`.

---

## Árvore de Decisão: Qual documento de pagamento usar?

Baseado no `paymentProvider` e `paymentMethod` retornados no `providersForSelection` do Order Preview:

```
providersForSelection[].paymentProvider
├── "asaas"
│   ├── paymentMethod: "credit_card"  → FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md
│   └── paymentMethod: "pix"          → FLOW_CHECKOUT_PAYMENT_PIX.md
│
├── "pagar_me"
│   ├── paymentMethod: "credit_card"  → FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md
│   └── paymentMethod: "pix"          → FLOW_CHECKOUT_PAYMENT_PIX.md (via iframe)
│
├── "stripe"
│   ├── paymentMethod: "credit_card"  → FLOW_CHECKOUT_PAYMENT_STRIPE.md (Fluxo A: Elements)
│   └── paymentMethod: "pix"          → FLOW_CHECKOUT_PAYMENT_STRIPE.md (Fluxo B: iframe)
│
├── "braza"
│   └── paymentMethod: "crypto"       → FLOW_CHECKOUT_PAYMENT_CRYPTO.md
│
├── "crypto"
│   └── paymentMethod: "crypto"       → FLOW_CHECKOUT_PAYMENT_CRYPTO.md
│
├── "transfer"
│   └── paymentMethod: "transfer"     → FLOW_CHECKOUT_PAYMENT_TRANSFER.md
│
└── "free"
    └── (qualquer)                    → FLOW_CHECKOUT_PAYMENT_TRANSFER.md (mesmo fluxo)
```

---

## Fluxo de Estado do Pedido

```
pending → confirming_payment → waiting_delivery → delivering → concluded
                                                              ↘ failed
                                                   ↘ cancelled
                                        ↘ expired
```

O frontend faz polling a cada 3 segundos (PIX) ou após ação do usuário para acompanhar a transição de status.

---

## Mapa de Documentos

| Documento | Descrição | Quando usar |
|-----------|-----------|-------------|
| [CHECKOUT_API_REFERENCE.md](./CHECKOUT_API_REFERENCE.md) | Referência de endpoints, schemas, enums | Consulta de API em qualquer momento |
| [FLOW_CHECKOUT_CART_CONFIRMATION.md](./FLOW_CHECKOUT_CART_CONFIRMATION.md) | Step 1: Carrinho e confirmação | Implementar tela de seleção e preview |
| [FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md](./FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md) | Pagamento com cartão (Pagar.me/Asaas) | Checkout transparente com cartão |
| [FLOW_CHECKOUT_PAYMENT_PIX.md](./FLOW_CHECKOUT_PAYMENT_PIX.md) | Pagamento via PIX | QR code + polling de status |
| [FLOW_CHECKOUT_PAYMENT_STRIPE.md](./FLOW_CHECKOUT_PAYMENT_STRIPE.md) | Pagamento via Stripe | Cartão internacional com Stripe Elements |
| [FLOW_CHECKOUT_PAYMENT_CRYPTO.md](./FLOW_CHECKOUT_PAYMENT_CRYPTO.md) | Pagamento crypto (Braza/ERC-20) | Crypto nativo ou bridge fiat-crypto |
| [FLOW_CHECKOUT_PAYMENT_TRANSFER.md](./FLOW_CHECKOUT_PAYMENT_TRANSFER.md) | Transferência bancária | Pagamento manual/free |
| [FLOW_CHECKOUT_COMPLETION.md](./FLOW_CHECKOUT_COMPLETION.md) | Step 3: Tela de conclusão | Resultado, gift cards, recibo |
| [CHECKOUT_SKILL_INDEX.md](./CHECKOUT_SKILL_INDEX.md) | Índice master | Ponto de entrada para qualquer implementação |

---

## Arquitetura da Implementação (Next.js App Router)

A implementação segue a arquitetura **Next.js App Router** com as seguintes tecnologias:

| Tecnologia | Uso |
|-----------|-----|
| Next.js (App Router) | Roteamento e SSR |
| React Query (`@tanstack/react-query`) | Gerenciamento de estado do servidor (mutations + queries) |
| `next-auth` | Autenticação (session + accessToken) |
| `next-intl` | Internacionalização |
| `zod` + `react-hook-form` | Validação de formulários |
| `axios` | Chamadas HTTP |
| shadcn/ui | Componentes UI |
| `cpf-cnpj-validator` | Validação de CPF/CNPJ |
| `card-validator` | Validação de cartão de crédito |
| `date-fns` | Manipulação de datas |
| `sonner` | Toasts / notificações |
| `lucide-react` | Ícones |
| `@stripe/stripe-js` | Stripe SDK (`loadStripe()`) |
| `@stripe/react-stripe-js` | Stripe React components (`Elements`, `PaymentElement`) |

### Rotas

| Rota | Componente | Step |
|------|-----------|------|
| `/checkout/confirmation` | `CheckoutConfirmation` | Step 1: Preview + seleção de pagamento |
| `/checkout/payment` | `CheckoutPayment` | Step 2: Formulário de pagamento |
| `/checkout/completed` | `CheckoutCompletion` | Step 3: Resultado da compra |

### Layout

O layout (`/checkout/layout.tsx`) envolve todas as páginas com:
- `CheckoutErrorBoundary` — captura erros de render e exibe fallback com link para home
- Layout centralizado com `max-w-lg`

### Providers Necessários

- `SessionProvider` (next-auth) — fornece sessão e token
- `QueryClientProvider` (react-query) — gerencia cache e mutations
- `NextIntlClientProvider` — fornece traduções

### Hooks Principais

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useOrderPreview()` | `useMutation` | Chama preview API. Recebe `productIds`, `currencyId`, `paymentMethod`, `couponCode`. Preview é **sempre single-payment fiat** (100%). O split crypto/fiat é calculado localmente e aplicado apenas na criação do pedido. |
| `useCreateOrder()` | `useMutation` | Cria pedido. Auto-salva response no localStorage |
| `useOrderStatusPolling()` | `useQuery` | Polling de status a cada 3s. Para em status finais |
| `useCountdown()` | custom hook | Timer para expiração do PIX. Retorna `formattedTime`, `isActive` |
| `useUserWallet()` | `useQuery` | Busca wallet address do usuário via `/users/profile` |
| `useCompanyConfig()` | custom hook | Retorna `companyId` do ambiente |
| `useTenantBaseUrl()` | custom hook | Resolve base URL do tenant para `successUrl`. Prioridade: 1) tenant main host, 2) `NEXT_PUBLIC_TENANT_HOSTNAME`, 3) `window.location.origin` |
| `useCurrencyAllowance()` | custom hook | Gerencia allowance ERC-20: request, polling (6s), timeout (30s) |

### Componentes Compartilhados

| Componente | Usado em | Props principais |
|-----------|----------|-----------------|
| `ProductSummary` | Step 1, Step 3 | `product: ProductPreview`, `currencySymbol` |
| `PriceBreakdown` | Step 1, Step 3 | Preços (cart, fees, total) + coupon (original prices) |
| `OrderStatusBadge` | Step 3 | `status: OrderStatus` — badge colorido por status |
| `PaymentMethodSelector` | Step 1 | RadioGroup com providers disponíveis |
| `CouponInput` | Step 1 | Input + botão aplicar + feedback visual |
| `CryptoPaymentInput` | Step 1 | Input livre de valor crypto + resumo do split |
| `AllowanceModal` | Step 1 | Modal de aprovação de allowance ERC-20 |

### Componentes de Pagamento

| Componente | Arquivo | Método de Pagamento |
|-----------|---------|---------------------|
| `CreditCardForm` | `credit-card-form.tsx` | Cartão (Asaas/Pagar.me) — campos: CPF, cartão, telefone, CEP, parcelas |
| `PixPaymentView` | `pix-payment-view.tsx` | PIX (Asaas) — formulário CPF + QR code + polling |
| `StripePaymentView` | `stripe-payment-view.tsx` | Stripe — Stripe Elements |
| `IframePaymentView` | `iframe-payment-view.tsx` | Pagar.me PIX / Stripe PIX — iframe com detecção de redirect |
| `BrazaPaymentForm` | `braza-payment-form.tsx` | Braza — formulário CPF/CNPJ + resumo de cotação |
| `FreeOrderView` | `free-order-view.tsx` | Transfer / Free / Crypto-only — spinner |
