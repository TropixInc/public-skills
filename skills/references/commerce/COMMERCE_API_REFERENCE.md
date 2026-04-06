---
id: COMMERCE_API_REFERENCE
title: "Commerce - Referência da API"
module: offpix
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - api-reference
depends_on:
  - AUTH_API_REFERENCE
---

# Referência da API de Commerce

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://commerce.w3block.io` |
| Swagger | https://commerce.w3block.io/docs |

## Autenticação

Todos os endpoints autenticados requerem:

```
Authorization: Bearer {accessToken}
```

Endpoints de admin adicionalmente requerem que o usuário tenha o papel `Admin` ou `SuperAdmin` no tenant.

Multi-tenancy: cada endpoint tem escopo por `companyId` no caminho da URL.

---

## Enums

### OrderStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Estado inicial após criação |
| `confirming_payment` | Aguardando confirmação de pagamento |
| `waiting_delivery` | Pagamento confirmado, preparando entrega |
| `delivering` | Tokens do produto sendo entregues na carteira |
| `concluded` | Pedido concluído com sucesso |
| `failed` | Pedido falhou (ver `failReason`) |
| `cancelling` | Cancelamento em andamento |
| `cancelled` | Totalmente cancelado |
| `partially_cancelled` | Alguns itens cancelados |
| `expired` | Tempo de expiração excedido |

### OrderProductStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Aguardando processamento |
| `delivering` | Sendo entregue |
| `concluded` | Entregue |
| `failed` | Entrega falhou |
| `cancelling` | Cancelamento em andamento |
| `cancelled` | Cancelado |
| `expired` | Expirado |

### PaymentStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Criado, aguardando processamento |
| `processing` | Sendo processado pelo provedor |
| `concluded` | Pagamento bem-sucedido |
| `failed` | Transação falhou |
| `cancelling` | Cancelamento iniciado |
| `cancelled` | Cancelado |
| `expired` | Expirado |
| `refunded` | Reembolsado ao usuário |

### BillingStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Ainda não faturado |
| `charged_on_provider` | Cobrado no provedor de pagamento |
| `charged_on_invoice` | Faturado para a empresa |

### RefundStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Reembolso iniciado |
| `processing` | Processando |
| `concluded` | Concluído |
| `failed` | Falhou |
| `waiting_approval` | Aguardando aprovação do admin |

### ProductStatus

| Valor | Descrição |
|-------|-----------|
| `draft` | Em desenvolvimento, não visível |
| `publishing` | Sendo publicado (assíncrono) |
| `updating` | Sendo atualizado |
| `published` | Ativo para venda |
| `cancelled` | Não mais disponível |
| `sold` | Esgotado |

### ProductType

| Valor | Descrição |
|-------|-----------|
| `nft` | Tokens ERC721/1155 |
| `erc20` | Tokens fungíveis |
| `external` | Produtos de terceiros / off-chain |

### ProductDistributionType

| Valor | Descrição |
|-------|-----------|
| `random` | Token aleatório atribuído ao comprador |
| `fixed` | Token específico por usuário (via `fixedMatch`) |
| `sequential` | Tokens atribuídos em ordem |

### ProductTokenStatus

| Valor | Descrição |
|-------|-----------|
| `publishing` | Sendo mintado/listado |
| `for_sale` | Disponível para compra |
| `sold` | Comprado |
| `locked` | Reservado em um pedido ativo |
| `updating` | Sendo modificado |
| `locked_by_booking` | Reservado para uma reserva |
| `cancelled` | Cancelado |

### PaymentProvider

| Valor | Descrição |
|-------|-----------|
| `pagar_me` | Pagar.me (Brasil) |
| `paypal` | PayPal |
| `transfer` | Transferência bancária manual |
| `stripe` | Stripe |
| `asaas` | ASAAS (Brasil) |
| `crypto` | Criptomoeda |
| `free` | Gratuito (pedidos sem custo) |
| `braza` | Braza |

### PaymentMethod

| Valor | Descrição |
|-------|-----------|
| `credit_card` | Cartão de crédito |
| `debit_card` | Cartão de débito |
| `pix` | PIX (pagamento instantâneo Brasil) |
| `crypto` | Criptomoeda |
| `transfer` | Transferência bancária |
| `billet` | Boleto bancário |
| `google_pay` | Google Pay |
| `apple_pay` | Apple Pay |

### PromotionType

| Valor | Descrição |
|-------|-----------|
| `coupon` | Baseado em código, usuário deve inserir o código |
| `discount` | Automático, aplicado sem código |

### PromotionAmountType

| Valor | Descrição |
|-------|-----------|
| `fixed` | Valor fixo em moeda |
| `percentage` | Porcentagem do carrinho/item |

### PromotionWhitelistType

| Valor | Descrição |
|-------|-----------|
| `w3block_id_whitelist` | Whitelist de ID de usuário W3Block |
| `email` | Whitelist baseada em email |

### CompanySplitType

| Valor | Descrição |
|-------|-----------|
| `product_price` | Parte do preço do produto |
| `client_service_fee` | Parte da taxa do cliente |
| `company_service_fee` | Parte da taxa da empresa |
| `gas_fee` | Parte das taxas de gas da blockchain |
| `resale_fee` | Parte da taxa de revenda |

### DeferredSplitStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Criado, aguardando execução |
| `pending_user_account` | Aguardando configuração da conta do destinatário |
| `processing_split` | Executando split |
| `under_user_account` | Fundos na conta do usuário |
| `processing_withdraw` | Processando saque |
| `withdrawn` | Concluído |
| `user_account_creation_failed` | Criação da conta falhou |
| `split_failed` | Execução do split falhou |
| `withdraw_failed` | Saque falhou |

---

## Endpoints

Todos os caminhos são prefixados com a URL base do serviço. `{companyId}` = UUID do tenant.

### Produtos (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/products` | Admin | Criar produto |
| GET | `/admin/companies/{companyId}/products` | Admin | Listar produtos (paginado) |
| GET | `/admin/companies/{companyId}/products/{productId}` | Admin | Obter detalhes do produto |
| GET | `/admin/companies/{companyId}/products/get-by-slug/{slug}` | Admin | Obter produto por slug |
| PATCH | `/admin/companies/{companyId}/products/{productId}` | Admin | Atualizar produto |
| PATCH | `/admin/companies/{companyId}/products/{productId}/publish` | Admin | Publicar produto |
| PATCH | `/admin/companies/{companyId}/products/{productId}/cancel` | Admin | Cancelar produto |

### Produtos — Público

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/companies/{companyId}/products` | Público | Listar produtos publicados (vitrine) |

### Variantes de Produto (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/products/{productId}/variants` | Admin | Criar variante |
| PATCH | `/admin/companies/{companyId}/products/{productId}/variants/{variantId}` | Admin | Atualizar variante |
| DELETE | `/admin/companies/{companyId}/products/{productId}/variants/{variantId}` | Admin | Excluir variante |

### Regras de Pedido de Produto (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/products/{productId}/order-rules` | Admin | Criar regra de pedido |
| GET | `/admin/companies/{companyId}/products/{productId}/order-rules` | Admin | Listar regras de pedido |
| PATCH | `/admin/companies/{companyId}/products/{productId}/order-rules/{ruleId}` | Admin | Atualizar regra |
| DELETE | `/admin/companies/{companyId}/products/{productId}/order-rules/{ruleId}` | Admin | Excluir regra |

### Pedidos (Usuário)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/companies/{companyId}/orders` | Bearer | Criar pedido |
| POST | `/companies/{companyId}/orders/preview` | Público | Calcular prévia do pedido |
| GET | `/companies/{companyId}/orders` | Bearer | Listar pedidos do usuário (paginado) |
| GET | `/companies/{companyId}/orders/{orderId}` | Proprietário | Obter detalhes do pedido |
| GET | `/companies/{companyId}/orders/{orderId}/payments` | Proprietário | Obter pagamentos do pedido |
| POST | `/companies/{companyId}/orders/{orderId}/pay` | Proprietário | Processar pagamento do pedido |
| POST | `/companies/{companyId}/orders/{orderId}/coupon` | Proprietário | Resgatar cupom no pedido |
| GET | `/companies/{companyId}/orders/{orderId}/coupon` | Proprietário | Obter informações do cupom do pedido |
| GET | `/companies/{companyId}/orders/get-by-deliver-id/{deliverId}` | Público | Consultar pedido por ID de entrega |

### Pedidos (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/admin/companies/{companyId}/orders` | Admin | Listar todos os pedidos (filtrado) |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/status` | Admin | Atualizar status do pedido |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/approve-payment` | Admin | Aprovar pagamento pendente |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/cancel` | Admin | Cancelar pedido |
| POST | `/admin/companies/{companyId}/orders/{orderId}/refund` | Admin | Criar reembolso |

### Pedidos — Uso e Exportações

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/companies/{companyId}/orders/usage-report` | Bearer | Obter relatório de uso |
| POST | `/admin/companies/{companyId}/exports/generate/orders` | Admin | Solicitar exportação de vendas |
| GET | `/admin/companies/{companyId}/exports/{exportId}` | Admin | Obter status da exportação |

### Promoções (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/promotions` | Admin | Criar promoção |
| GET | `/admin/companies/{companyId}/promotions` | Admin | Listar promoções (paginado) |
| GET | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Obter promoção |
| PATCH | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Atualizar promoção |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Excluir promoção (soft) |

### Produtos de Promoção (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Admin | Listar produtos da promoção |
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Admin | Adicionar produto à promoção |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}/products/{productId}` | Admin | Remover produto |

### Whitelists de Promoção (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Admin | Listar entradas da whitelist |
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Admin | Adicionar entrada à whitelist |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists/{whitelistId}` | Admin | Remover entrada |

### Tags (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/tags` | Admin | Criar tag |
| GET | `/admin/companies/{companyId}/tags` | Admin | Listar tags (paginado) |
| GET | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Obter tag |
| PATCH | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Atualizar tag |
| DELETE | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Excluir tag |

### Configurações de Split (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/split-configurations` | Admin | Criar configuração de split |
| GET | `/admin/companies/{companyId}/split-configurations` | Admin | Listar configurações de split |
| GET | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Obter configuração de split |
| PATCH | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Atualizar configuração de split |
| DELETE | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Excluir configuração de split |

### Configuração de Gateway de Pagamento (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/admin/companies/{companyId}/configurations` | Admin | Obter configuração de commerce da empresa |
| GET | `/admin/companies/{companyId}/is-enabled` | Admin | Verificar se o commerce está habilitado |
| PATCH | `/admin/companies/{companyId}/configurations/providers/stripe` | Admin | Configurar Stripe |
| PATCH | `/admin/companies/{companyId}/configurations/providers/asaas` | Admin | Configurar ASAAS |
| POST | `/admin/companies/{companyId}/configurations/providers-selections` | Admin | Definir combinações de provedor-moeda-método |

### Provedores de Pagamento (Usuário)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/companies/{companyId}/users/{userId}/providers/check-configured-providers` | Bearer | Verificar provedores configurados |
| POST | `/companies/{companyId}/users/{userId}/providers/{provider}` | Bearer | Configurar provedor para usuário |
| GET | `/companies/{companyId}/users/{userId}/providers` | Bearer | Listar provedores do usuário |

### Reembolsos (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/admin/companies/{companyId}/refunds` | Admin | Listar reembolsos (paginado) |
| GET | `/admin/companies/{companyId}/refunds/{refundId}` | Admin | Obter detalhes do reembolso |
| POST | `/admin/companies/{companyId}/refunds/{refundId}/approve` | Admin | Aprovar reembolso |
| PATCH | `/admin/companies/{companyId}/refunds/{refundId}` | Admin | Atualizar reembolso |

### Assets (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/assets` | Admin | Obter autorização de upload |

Destinos de asset: `PRODUCT`, `ORDER_DOCUMENT`, `REFUND`, `STOREFRONT_PAGE`, `STOREFRONT_THEME`

### Moedas (Global)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| GET | `/globals/currencies` | Público | Listar todas as moedas disponíveis |

### Projetos (Admin)

| Método | Caminho | Auth | Descrição |
|--------|---------|------|-----------|
| POST | `/admin/companies/{companyId}/projects` | Admin | Criar projeto de commerce |

---

## DTOs Principais

### ProductPrice

```json
{
  "currencyId": "uuid",
  "amount": "string (big number)"
}
```

### OrderProductDto (dentro de CreateOrderDto)

```json
{
  "productTokenId": "uuid",
  "selectBestPrice": false,
  "resellerId": "uuid | null",
  "variantIds": ["uuid"],
  "quantity": 1,
  "expectedPrices": [
    { "currencyId": "uuid", "expectedPrice": "1000000000000000000" }
  ]
}
```

### OrderMultiPaymentSelectionDto

```json
{
  "currencyId": "uuid",
  "paymentProvider": "stripe",
  "paymentMethod": "credit_card",
  "amountType": "percentage | all_remaining | fixed",
  "amount": "string | null",
  "providerInputs": {
    "ssn": "string",
    "installments": 1,
    "credit_card_id": "uuid",
    "save_credit_card": false
  }
}
```

### UtmParams

```json
{
  "utm_source": "string",
  "utm_medium": "string",
  "utm_campaign": "string",
  "utm_term": "string",
  "utm_content": "string"
}
```

### PromotionRequirements

```json
{
  "minCartPrice": "string",
  "maxCartPrice": "string",
  "minCartItems": 1,
  "maxCartItems": 10,
  "minSameCartItems": 1,
  "maxSameCartItems": 5,
  "paymentMethods": ["credit_card", "pix"],
  "currencyIds": ["uuid"],
  "destinationWalletAddresses": ["0x..."],
  "notApplicableProductIds": ["uuid"]
}
```

---

## Paginação

Todos os endpoints de listagem aceitam:

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `page` | number | 1 | Número da página |
| `limit` | number | 10-20 | Itens por página |
| `sortBy` | string | `createdAt` | Campo de ordenação |
| `orderBy` | string | `DESC` | `ASC` ou `DESC` |
| `search` | string | — | Busca por texto completo |

A resposta encapsula os itens em `OffpixPaginatedResponse<T>`:

```json
{
  "items": [...],
  "meta": {
    "totalItems": 100,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 10,
    "currentPage": 1
  }
}
```
