---
id: COMMERCE_API_REFERENCE
title: "Commerce - API Reference"
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

# Commerce API Reference

## Base URLs

| Environment | URL |
|-------------|-----|
| Production | `https://commerce.w3block.io` |
| Swagger | https://commerce.w3block.io/docs |

## Authentication

All authenticated endpoints require:

```
Authorization: Bearer {accessToken}
```

Admin endpoints additionally require the user to have `Admin` or `SuperAdmin` tenant role.

Multi-tenancy: every endpoint is scoped by `companyId` in the URL path.

---

## Enums

### OrderStatus

| Value | Description |
|-------|-------------|
| `pending` | Initial state after creation |
| `confirming_payment` | Awaiting payment confirmation |
| `waiting_delivery` | Payment confirmed, preparing delivery |
| `delivering` | Product tokens being delivered to wallet |
| `concluded` | Order completed successfully |
| `failed` | Order failed (see `failReason`) |
| `cancelling` | Cancellation in progress |
| `cancelled` | Fully cancelled |
| `partially_cancelled` | Some items cancelled |
| `expired` | Expiration time exceeded |

### OrderProductStatus

| Value | Description |
|-------|-------------|
| `pending` | Awaiting processing |
| `delivering` | Being delivered |
| `concluded` | Delivered |
| `failed` | Delivery failed |
| `cancelling` | Cancellation in progress |
| `cancelled` | Cancelled |
| `expired` | Expired |

### PaymentStatus

| Value | Description |
|-------|-------------|
| `pending` | Created, awaiting processing |
| `processing` | Being processed by provider |
| `concluded` | Payment successful |
| `failed` | Transaction failed |
| `cancelling` | Cancellation initiated |
| `cancelled` | Cancelled |
| `expired` | Expired |
| `refunded` | Refunded to user |

### BillingStatus

| Value | Description |
|-------|-------------|
| `pending` | Not yet billed |
| `charged_on_provider` | Charged to payment provider |
| `charged_on_invoice` | Invoiced to company |

### RefundStatus

| Value | Description |
|-------|-------------|
| `pending` | Refund initiated |
| `processing` | Processing |
| `concluded` | Completed |
| `failed` | Failed |
| `waiting_approval` | Awaiting admin approval |

### ProductStatus

| Value | Description |
|-------|-------------|
| `draft` | In development, not visible |
| `publishing` | Being published (async) |
| `updating` | Being updated |
| `published` | Live for sale |
| `cancelled` | No longer available |
| `sold` | Sold out |

### ProductType

| Value | Description |
|-------|-------------|
| `nft` | ERC721/1155 tokens |
| `erc20` | Fungible tokens |
| `external` | Third-party / off-chain products |

### ProductDistributionType

| Value | Description |
|-------|-------------|
| `random` | Random token assigned to buyer |
| `fixed` | Specific token per user (via `fixedMatch`) |
| `sequential` | Tokens assigned in order |

### ProductTokenStatus

| Value | Description |
|-------|-------------|
| `publishing` | Being minted/listed |
| `for_sale` | Available for purchase |
| `sold` | Purchased |
| `locked` | Reserved in an active order |
| `updating` | Being modified |
| `locked_by_booking` | Reserved for a booking |
| `cancelled` | Cancelled |

### PaymentProvider

| Value | Description |
|-------|-------------|
| `pagar_me` | Pagar.me (Brazil) |
| `paypal` | PayPal |
| `transfer` | Manual bank transfer |
| `stripe` | Stripe |
| `asaas` | ASAAS (Brazil) |
| `crypto` | Cryptocurrency |
| `free` | Free (zero-cost orders) |
| `braza` | Braza |

### PaymentMethod

| Value | Description |
|-------|-------------|
| `credit_card` | Credit card |
| `debit_card` | Debit card |
| `pix` | PIX (Brazil instant payment) |
| `crypto` | Cryptocurrency |
| `transfer` | Bank transfer |
| `billet` | Boleto bancário |
| `google_pay` | Google Pay |
| `apple_pay` | Apple Pay |

### PromotionType

| Value | Description |
|-------|-------------|
| `coupon` | Code-based, user must enter code |
| `discount` | Automatic, applied without code |

### PromotionAmountType

| Value | Description |
|-------|-------------|
| `fixed` | Fixed currency amount |
| `percentage` | Percentage of cart/item |

### PromotionWhitelistType

| Value | Description |
|-------|-------------|
| `w3block_id_whitelist` | W3Block user ID whitelist |
| `email` | Email-based whitelist |

### CompanySplitType

| Value | Description |
|-------|-------------|
| `product_price` | Share of the product price |
| `client_service_fee` | Share of the client fee |
| `company_service_fee` | Share of the company fee |
| `gas_fee` | Share of blockchain gas fees |
| `resale_fee` | Share of the resale fee |

### DeferredSplitStatus

| Value | Description |
|-------|-------------|
| `pending` | Created, awaiting execution |
| `pending_user_account` | Waiting for recipient account setup |
| `processing_split` | Executing split |
| `under_user_account` | Funds in user account |
| `processing_withdraw` | Processing withdrawal |
| `withdrawn` | Completed |
| `user_account_creation_failed` | Account creation failed |
| `split_failed` | Split execution failed |
| `withdraw_failed` | Withdrawal failed |

---

## Endpoints

All paths are prefixed with the service base URL. `{companyId}` = tenant UUID.

### Products (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products` | Admin | Create product |
| GET | `/admin/companies/{companyId}/products` | Admin | List products (paginated) |
| GET | `/admin/companies/{companyId}/products/{productId}` | Admin | Get product details |
| GET | `/admin/companies/{companyId}/products/get-by-slug/{slug}` | Admin | Get product by slug |
| PATCH | `/admin/companies/{companyId}/products/{productId}` | Admin | Update product |
| PATCH | `/admin/companies/{companyId}/products/{productId}/publish` | Admin | Publish product |
| PATCH | `/admin/companies/{companyId}/products/{productId}/cancel` | Admin | Cancel product |

### Products — Public

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/companies/{companyId}/products` | Public | List published products (storefront) |

### Product Variants (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/variants` | Admin | Create variant |
| PATCH | `/admin/companies/{companyId}/products/{productId}/variants/{variantId}` | Admin | Update variant |
| DELETE | `/admin/companies/{companyId}/products/{productId}/variants/{variantId}` | Admin | Delete variant |

### Product Order Rules (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/order-rules` | Admin | Create order rule |
| GET | `/admin/companies/{companyId}/products/{productId}/order-rules` | Admin | List order rules |
| PATCH | `/admin/companies/{companyId}/products/{productId}/order-rules/{ruleId}` | Admin | Update rule |
| DELETE | `/admin/companies/{companyId}/products/{productId}/order-rules/{ruleId}` | Admin | Delete rule |

### Orders (User)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/companies/{companyId}/orders` | Bearer | Create order |
| POST | `/companies/{companyId}/orders/preview` | Public | Compute order preview |
| GET | `/companies/{companyId}/orders` | Bearer | List user's orders (paginated) |
| GET | `/companies/{companyId}/orders/{orderId}` | Owner | Get order details |
| GET | `/companies/{companyId}/orders/{orderId}/payments` | Owner | Get order payments |
| POST | `/companies/{companyId}/orders/{orderId}/pay` | Owner | Process payment on order |
| POST | `/companies/{companyId}/orders/{orderId}/coupon` | Owner | Redeem coupon on order |
| GET | `/companies/{companyId}/orders/{orderId}/coupon` | Owner | Get coupon info for order |
| GET | `/companies/{companyId}/orders/get-by-deliver-id/{deliverId}` | Public | Look up order by delivery ID |

### Orders (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/orders` | Admin | List all orders (filtered) |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/status` | Admin | Update order status |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/approve-payment` | Admin | Approve pending payment |
| PATCH | `/admin/companies/{companyId}/orders/{orderId}/cancel` | Admin | Cancel order |
| POST | `/admin/companies/{companyId}/orders/{orderId}/refund` | Admin | Create refund |

### Orders — Usage & Exports

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/companies/{companyId}/orders/usage-report` | Bearer | Get usage report |
| POST | `/admin/companies/{companyId}/exports/generate/orders` | Admin | Request sales export |
| GET | `/admin/companies/{companyId}/exports/{exportId}` | Admin | Get export status |

### Promotions (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions` | Admin | Create promotion |
| GET | `/admin/companies/{companyId}/promotions` | Admin | List promotions (paginated) |
| GET | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Get promotion |
| PATCH | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Update promotion |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}` | Admin | Delete promotion (soft) |

### Promotion Products (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Admin | List promotion products |
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Admin | Add product to promotion |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}/products/{productId}` | Admin | Remove product |

### Promotion Whitelists (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Admin | List whitelist entries |
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Admin | Add whitelist entry |
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists/{whitelistId}` | Admin | Remove entry |

### Tags (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/tags` | Admin | Create tag |
| GET | `/admin/companies/{companyId}/tags` | Admin | List tags (paginated) |
| GET | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Get tag |
| PATCH | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Update tag |
| DELETE | `/admin/companies/{companyId}/tags/{tagId}` | Admin | Delete tag |

### Split Configurations (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/split-configurations` | Admin | Create split config |
| GET | `/admin/companies/{companyId}/split-configurations` | Admin | List split configs |
| GET | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Get split config |
| PATCH | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Update split config |
| DELETE | `/admin/companies/{companyId}/split-configurations/{configId}` | Admin | Delete split config |

### Payment Gateway Configuration (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/configurations` | Admin | Get company commerce config |
| GET | `/admin/companies/{companyId}/is-enabled` | Admin | Check if commerce is enabled |
| PATCH | `/admin/companies/{companyId}/configurations/providers/stripe` | Admin | Configure Stripe |
| PATCH | `/admin/companies/{companyId}/configurations/providers/asaas` | Admin | Configure ASAAS |
| POST | `/admin/companies/{companyId}/configurations/providers-selections` | Admin | Set provider-currency-method combos |

### Payment Providers (User)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/companies/{companyId}/users/{userId}/providers/check-configured-providers` | Bearer | Check configured providers |
| POST | `/companies/{companyId}/users/{userId}/providers/{provider}` | Bearer | Configure provider for user |
| GET | `/companies/{companyId}/users/{userId}/providers` | Bearer | List user providers |

### Refunds (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/companies/{companyId}/refunds` | Admin | List refunds (paginated) |
| GET | `/admin/companies/{companyId}/refunds/{refundId}` | Admin | Get refund details |
| POST | `/admin/companies/{companyId}/refunds/{refundId}/approve` | Admin | Approve refund |
| PATCH | `/admin/companies/{companyId}/refunds/{refundId}` | Admin | Update refund |

### Assets (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/assets` | Admin | Get upload authorization |

Asset targets: `PRODUCT`, `ORDER_DOCUMENT`, `REFUND`, `STOREFRONT_PAGE`, `STOREFRONT_THEME`

### Currencies (Global)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/globals/currencies` | Public | List all available currencies |

### Projects (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/admin/companies/{companyId}/projects` | Admin | Create commerce project |

---

## Key DTOs

### ProductPrice

```json
{
  "currencyId": "uuid",
  "amount": "string (big number)"
}
```

### OrderProductDto (inside CreateOrderDto)

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

## Pagination

All list endpoints accept:

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | number | 1 | Page number |
| `limit` | number | 10-20 | Items per page |
| `sortBy` | string | `createdAt` | Sort field |
| `orderBy` | string | `DESC` | `ASC` or `DESC` |
| `search` | string | — | Full-text search |

Response wraps items in `OffpixPaginatedResponse<T>`:

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
