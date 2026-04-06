---
id: FLOW_COMMERCE_ORDER_PURCHASE
title: "Commerce - Fluxo de Pedido e Compra"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - orders
  - payments
  - refunds
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
  - FLOW_COMMERCE_PRODUCT_LIFECYCLE
---

# Commerce — Fluxo de Pedido e Compra

## Visão Geral

Este fluxo cobre o ciclo de vida completo da compra: pré-visualização de um pedido (cálculo de preços, taxas e descontos), criação do pedido, processamento do pagamento e tratamento de cancelamentos/reembolsos. A W3Block suporta pedidos multi-moeda com múltiplos métodos de pagamento em uma única transação.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | Token de acesso JWT | Autenticação do serviço de identidade |
| Produto publicado | Produto com status `published` | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Endereço de carteira | Carteira de destino para entrega de NFT | Carteira conectada do usuário |
| Gateway configurado | Pelo menos um provedor de pagamento configurado | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |

## Entidades e Relacionamentos

```
Order (1) ──→ (N) OrderProduct   ← itens no pedido
Order (1) ──→ (N) Payment        ← suporte a multi-pagamento
Order (1) ──→ (N) OrderDocument  ← arquivos anexados (notas fiscais, etc.)
Order (1) ──→ (N) Refund         ← solicitações de reembolso
OrderProduct ──→ ProductToken    ← token específico sendo comprado
Payment ──→ Currency             ← moeda do pagamento
```

### Ciclo de Vida do Status do Pedido

```
PENDING ──→ CONFIRMING_PAYMENT ──→ WAITING_DELIVERY ──→ DELIVERING ──→ CONCLUDED
                                                                          │
                                                                        FAILED
                                        │
                          CANCELLING ──→ CANCELLED / PARTIALLY_CANCELLED
                                        │
                                      EXPIRED
```

---

## Fluxo: Comprar um Produto

### Etapa 1: Pré-visualizar o Pedido

Sempre faça a pré-visualização antes de criar. Isso calcula preços, taxas, provedores de pagamento disponíveis e valida o carrinho.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/companies/{companyId}/orders/preview` | Público | application/json |

**Requisição Mínima:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "quantity": 1,
      "expectedPrices": [
        { "currencyId": "currency-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "currency-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining"
    }
  ]
}
```

**Requisição Completa:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "selectBestPrice": true,
      "quantity": 1,
      "variantIds": ["variant-value-uuid"],
      "expectedPrices": [
        { "currencyId": "brl-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "percentage",
      "amount": "50"
    },
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "asaas",
      "paymentMethod": "pix",
      "amountType": "all_remaining"
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "couponCode": "SUMMER20",
  "signedGasFees": [
    { "currencyId": "matic-uuid", "signedGasFee": "1000000000000000" }
  ]
}
```

**Resposta (200):**
```json
{
  "products": [
    {
      "id": "product-uuid",
      "name": "My NFT",
      "prices": [...],
      "promotions": [...]
    }
  ],
  "productsErrors": [],
  "appliedCoupon": "SUMMER20",
  "payments": [
    {
      "currencyId": "brl-uuid",
      "cartPrice": "50000000000000000000",
      "clientServiceFee": "2500000000000000000",
      "gasFee": "0",
      "totalPrice": "52500000000000000000",
      "originalCartPrice": "60000000000000000000",
      "originalClientServiceFee": "3000000000000000000",
      "originalTotalPrice": "63000000000000000000",
      "currencyAllowanceState": "AVAILABLE",
      "providersForSelection": [
        { "paymentProvider": "stripe", "paymentMethod": "credit_card", "available": true },
        { "paymentProvider": "asaas", "paymentMethod": "pix", "available": true },
        { "paymentProvider": "crypto", "paymentMethod": "crypto", "available": false }
      ]
    }
  ],
  "cashback": {
    "currencyId": "loyalty-uuid",
    "amount": "50000000000000000000",
    "cashbackAmount": "500000000000000000"
  }
}
```

**Notas:**
- `productsErrors` lista problemas como "fora de estoque", "whitelist obrigatória", "limite de compra atingido"
- Valores de `currencyAllowanceState`: `AVAILABLE`, `INSUFFICIENT_ALLOWANCE` — verifique isso antes de criar o pedido
- `providersForSelection` indica quais métodos de pagamento estão disponíveis para cada moeda
- `originalCartPrice` vs `cartPrice` mostra o valor do desconto
- A pré-visualização **não é persistida** — é apenas um cálculo

---

### Etapa 2: Criar o Pedido

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/companies/{companyId}/orders` | Bearer | application/json |

**Requisição Mínima:**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "quantity": 1,
      "expectedPrices": [
        { "currencyId": "currency-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "currency-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining",
      "providerInputs": {
        "credit_card_id": "card-uuid"
      }
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "signedGasFees": [
    { "currencyId": "currency-uuid" }
  ]
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "orderProducts": [
    {
      "productTokenId": "token-uuid",
      "selectBestPrice": true,
      "quantity": 1,
      "variantIds": ["variant-value-uuid"],
      "expectedPrices": [
        { "currencyId": "brl-uuid", "expectedPrice": "50000000000000000000" }
      ]
    }
  ],
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card",
      "amountType": "all_remaining",
      "providerInputs": {
        "credit_card_id": "card-uuid",
        "save_credit_card": true,
        "installments": 3
      }
    }
  ],
  "destinationWalletAddress": "0xabc...123",
  "destinationUserId": "user-uuid",
  "addressId": "address-uuid",
  "couponCode": "SUMMER20",
  "successUrl": "https://myapp.com/order/success",
  "utmParams": {
    "utm_source": "instagram",
    "utm_campaign": "summer_sale"
  },
  "signedGasFees": [
    { "currencyId": "matic-uuid", "signedGasFee": "1000000000000000" }
  ],
  "acceptSimilarOrderInShortPeriod": false,
  "passShareCodeData": {}
}
```

**Resposta (201):**
```json
{
  "id": "order-uuid",
  "companyId": "company-uuid",
  "userId": "user-uuid",
  "status": "pending",
  "deliverId": "ABC123",
  "destinationWalletAddress": "0xabc...123",
  "currencyAmount": [
    { "currencyId": "brl-uuid", "amount": "50000000000000000000" }
  ],
  "clientServiceFee": [
    { "currencyId": "brl-uuid", "amount": "2500000000000000000" }
  ],
  "products": [...],
  "payments": [...],
  "expiresIn": "2026-03-31T12:30:00.000Z",
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `orderProducts` | array | Sim | 1-100 itens para comprar |
| `orderProducts[].productTokenId` | uuid | Sim | Token específico para comprar |
| `orderProducts[].quantity` | number | Sim | Quantidade |
| `orderProducts[].expectedPrices` | array | Sim | Preço esperado por moeda (previne alterações de preço) |
| `orderProducts[].selectBestPrice` | boolean | Não | Selecionar automaticamente o menor preço |
| `orderProducts[].variantIds` | string[] | Não | Valores de variante selecionados |
| `orderProducts[].resellerId` | uuid | Não | Usuário revendedor (venda secundária) |
| `payments` | array | Sim | Pelo menos 1 método de pagamento |
| `payments[].currencyId` | uuid | Sim | Moeda do pagamento |
| `payments[].paymentProvider` | enum | Sim | Provedor a utilizar |
| `payments[].paymentMethod` | enum | Sim | Método a utilizar |
| `payments[].amountType` | string | Sim | `percentage`, `fixed` ou `all_remaining` |
| `payments[].amount` | string | Condicional | Obrigatório se amountType for `percentage` ou `fixed` |
| `payments[].providerInputs` | object | Não | Dados específicos do provedor (ID do cartão, parcelas, CPF) |
| `destinationWalletAddress` | string | Sim (NFT) | Endereço Ethereum para receber os tokens |
| `destinationUserId` | uuid | Não | Enviar para a carteira de outro usuário |
| `addressId` | uuid | Não | Endereço de entrega |
| `couponCode` | string | Não | Código de cupom para aplicar |
| `successUrl` | string | Não | URL de redirecionamento após pagamento |
| `signedGasFees` | array | Sim | Confirmação de taxa de gas por moeda |
| `utmParams` | object | Não | Rastreamento de campanha |
| `acceptSimilarOrderInShortPeriod` | boolean | Não | Permitir pedidos duplicados (padrão: false) |

**Notas:**
- `expectedPrices` funciona como trava de preço — o pedido falha se o preço mudou desde a pré-visualização
- `amountType: "all_remaining"` deve ser usado no último pagamento para cobrir o restante
- O pedido inicia com status `pending` e tem um tempo de expiração
- `deliverId` é gerado automaticamente — usado para rastrear a entrega externamente
- `acceptSimilarOrderInShortPeriod: false` previne compras duplicadas acidentais

---

### Etapa 3: Processar Pagamento (se não processado automaticamente)

Alguns fluxos de pagamento (ex: PIX) requerem uma segunda etapa após a criação do pedido.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/companies/{companyId}/orders/{orderId}/pay` | Bearer (Proprietário) | application/json |

**Requisição:**
```json
{
  "payments": [
    {
      "currencyId": "brl-uuid",
      "paymentProvider": "asaas",
      "paymentMethod": "pix",
      "amountType": "all_remaining"
    }
  ],
  "successUrl": "https://myapp.com/order/success"
}
```

**Resposta (200):**
```json
{
  "id": "order-uuid",
  "status": "confirming_payment",
  "payments": [
    {
      "id": "payment-uuid",
      "status": "processing",
      "publicData": {
        "qrCode": "00020126...",
        "qrCodeUrl": "https://..."
      }
    }
  ]
}
```

**Notas:**
- Para PIX: a resposta inclui `publicData.qrCode` e `qrCodeUrl` para o comprador
- Para cartão de crédito: o pagamento geralmente é processado automaticamente durante a criação do pedido
- Para transferência manual: o admin deve aprovar via endpoint `approve-payment`
- Transições de status do pagamento: `pending → processing → concluded` (ou `failed`)

---

### Etapa 4: Verificar Status do Pedido

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/companies/{companyId}/orders/{orderId}` | Bearer (Proprietário) | — |

**Resposta (200):**
```json
{
  "id": "order-uuid",
  "status": "concluded",
  "deliverId": "ABC123",
  "deliverDate": "2026-03-31T12:15:00.000Z",
  "products": [
    {
      "orderId": "order-uuid",
      "productTokenId": "token-uuid",
      "status": "concluded",
      "deliveredAt": "2026-03-31T12:15:00.000Z",
      "currencyAmount": [...],
      "productToken": {
        "id": "token-uuid",
        "metadata": { "name": "NFT #1", "image": "https://..." }
      }
    }
  ],
  "payments": [
    {
      "id": "payment-uuid",
      "status": "concluded",
      "amount": "50000000000000000000",
      "paymentProvider": "stripe",
      "paymentMethod": "credit_card"
    }
  ]
}
```

---

## Fluxo: Gerenciamento de Pedidos pelo Admin

### Aprovar um Pagamento Pendente

Para pedidos de transferência manual que requerem aprovação do admin.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/approve-payment` | Bearer (Admin) | — |

**Notas:**
- Transiciona o pedido de `confirming_payment` para `waiting_delivery`
- Aplica-se apenas a pedidos com provedor de pagamento `transfer`

### Cancelar um Pedido (Admin)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/cancel` | Bearer (Admin) | — |

**Notas:**
- Aciona o cancelamento de todos os pagamentos pendentes
- Os tokens do produto são liberados de volta para o status `for_sale`
- Se alguns itens já foram entregues: o status se torna `partially_cancelled`
- Os logs de uso de promoção são revertidos (`reverted: true`)

### Criar um Reembolso

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/orders/{orderId}/refund` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "currencyId": "currency-uuid",
  "amount": "50000000000000000000"
}
```

**Resposta (201):**
```json
{
  "id": "refund-uuid",
  "status": "pending",
  "amount": "50000000000000000000",
  "orderId": "order-uuid"
}
```

**Ciclo de vida do reembolso:**
```
PENDING ──→ WAITING_APPROVAL ──→ PROCESSING ──→ CONCLUDED
                                             ↓
                                          FAILED
```

Aprovar com: `POST /admin/companies/{companyId}/refunds/{refundId}/approve`

---

## Fluxo: Listar e Filtrar Pedidos (Admin)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/admin/companies/{companyId}/orders` | Bearer (Admin) | — |

**Parâmetros de Query:**

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `status` | string[] | Filtrar por status(es) do pedido |
| `startDate` | ISO 8601 | Pedidos criados após esta data |
| `endDate` | ISO 8601 | Pedidos criados antes desta data |
| `paymentMethod` | string | Filtrar por método de pagamento |
| `userId` | uuid | Filtrar por comprador |
| `productId` | uuid | Filtrar por produto |
| `utmCampaign` | string | Filtrar por campanha UTM |
| `page` | number | Número da página |
| `limit` | number | Itens por página |
| `sortBy` | string | Campo de ordenação |
| `orderBy` | string | `ASC` ou `DESC` |

**A resposta inclui `meta.ordersSummary`:**
```json
{
  "items": [...],
  "meta": {
    "totalItems": 500,
    "currentPage": 1,
    "ordersSummary": [
      {
        "currencyId": "brl-uuid",
        "total": "1500000000000000000000",
        "totalClientServiceFee": "75000000000000000000",
        "totalCompanyServiceFee": "30000000000000000000",
        "totalCurrencyAmount": "1395000000000000000000",
        "totalGasFee": "0",
        "totalOrders": 120
      }
    ]
  }
}
```

---

## Fluxo: Exportar Relatório de Vendas

### Etapa 1: Solicitar Exportação

| Método | Caminho | Auth |
|--------|---------|------|
| POST | `/admin/companies/{companyId}/exports/generate/orders` | Admin |

### Etapa 2: Verificar Status

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/admin/companies/{companyId}/exports/{exportId}` | Admin |

Faça polling a cada 2 segundos até que o status indique conclusão. A resposta inclui uma URL de download.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Produto indisponível | Token não está no status `for_sale` | Verificar disponibilidade do produto/token |
| 400 | Limite de compra excedido | Usuário excedeu o limite da regra de pedido | Verificar `purchaseLimit` nas regras de pedido do produto |
| 400 | Restrição de whitelist | Usuário não está na whitelist obrigatória | Adicionar usuário à whitelist ou remover restrição |
| 400 | Incompatibilidade de preço | `expectedPrice` não corresponde ao preço atual | Executar pré-visualização novamente e usar os preços atualizados |
| 400 | Carteira inválida | Endereço de carteira de destino inválido | Verificar formato do endereço Ethereum |
| 400 | Saldo insuficiente | Moeda de pagamento não disponível | Verificar `currencyAllowanceState` na pré-visualização |
| 409 | Pedido similar existe | Pedido duplicado em curto período | Definir `acceptSimilarOrderInShortPeriod: true` ou aguardar |
| 422 | Verificação de conta necessária | Email/telefone não verificado | Concluir verificação de conta primeiro |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Pedido criado mas sem processamento de pagamento | Cartão de crédito: passar `providerInputs.credit_card_id`. PIX: chamar endpoint `/pay` após criação |
| 2 | Erro "Incompatibilidade de preço" na criação do pedido | Sempre execute a pré-visualização primeiro, depois use os preços exatos da resposta da pré-visualização em `expectedPrices` |
| 3 | Pedido expira antes do pagamento | Verificar campo `expiresIn`. Configurar `checkoutExpireTime` nas configurações do gateway para janelas mais longas |
| 4 | Reembolso travado em `pending` | Reembolsos requerem aprovação do admin. Chamar o endpoint de aprovação |
| 5 | Total do multi-pagamento não confere | Usar `amountType: "all_remaining"` no último pagamento para cobrir automaticamente o restante |
| 6 | Rastreamento de entrega não encontrado | `deliverId` é gerado automaticamente em maiúsculas. Usar `GET /companies/{companyId}/orders/get-by-deliver-id/{deliverId}` |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticação | Bearer token obrigatório | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Ciclo de Vida do Produto | Produtos devem estar publicados primeiro | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Promoções | Cupons aplicados durante o pedido | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Splits e Gateways | Configuração de pagamento deve existir | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
