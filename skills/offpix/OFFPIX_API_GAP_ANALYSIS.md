---
id: OFFPIX_API_GAP_ANALYSIS
title: "Offpix Frontend vs Backend API - Gap Analysis (Admin Role)"
module: offpix
version: "1.1.0"
type: analysis
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - gap-analysis
  - api
  - frontend
  - backend
  - admin
---

# Offpix Frontend vs Backend API - Gap Analysis

Endpoints que existem nos backends com a **role Admin** mas **não estão implementados** no frontend Offpix.

**Escopo:** Apenas endpoints acessíveis pela role `Admin` (ou `Administrator`). Endpoints exclusivos de SuperAdmin, Integration, User-only, ou públicos foram excluídos.

---

## Commerce Backend

### Bookings (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/admin/companies/:companyId/bookings` | Criar booking |
| GET | `/admin/companies/:companyId/bookings` | Listar bookings |
| GET | `/admin/companies/:companyId/bookings/:bookingId` | Obter booking |
| PATCH | `/admin/companies/:companyId/bookings/:bookingId` | Atualizar booking |
| PATCH | `/admin/companies/:companyId/bookings/:bookingId/publish` | Publicar booking |
| PATCH | `/admin/companies/:companyId/bookings/:bookingId/cancel` | Cancelar booking |
| PATCH | `/admin/companies/:companyId/bookings/:bookingId/resolve` | Resolver booking |
| GET | `/admin/companies/:companyId/bookings/:bookingId/requests` | Listar requests |
| PATCH | `/admin/companies/:companyId/bookings/:bookingId/requests/:id/resolve` | Resolver request |
| PATCH | `/companies/:companyId/bookings/:bookingId/requests/:id/cancel` | Cancelar request |

---

### FAQ (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/faq/contexts` | Listar contextos FAQ |
| POST | `/admin/companies/:companyId/faq/contexts` | Criar contexto FAQ |
| GET | `/admin/companies/:companyId/faq/contexts/:contextId` | Obter contexto |
| PATCH | `/admin/companies/:companyId/faq/contexts/:contextId` | Atualizar contexto |
| GET | `/admin/companies/:companyId/faq/contexts/:contextId/items` | Listar itens do contexto |
| GET | `/admin/companies/:companyId/faq/contexts/:contextId/items/:faqItemId` | Obter item do contexto |
| GET | `/admin/companies/:companyId/faq/items` | Listar todos os itens FAQ |
| POST | `/admin/companies/:companyId/faq/items` | Criar item FAQ |
| GET | `/admin/companies/:companyId/faq/items/:faqItemId` | Obter item FAQ |
| PATCH | `/admin/companies/:companyId/faq/items/:faqItemId` | Atualizar item FAQ |

---

### Rewards (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/rewards` | Listar rewards |
| POST | `/admin/companies/:companyId/rewards` | Criar reward |
| GET | `/admin/companies/:companyId/rewards/:rewardId` | Obter reward |
| PATCH | `/admin/companies/:companyId/rewards/:rewardId` | Atualizar reward |
| DELETE | `/admin/companies/:companyId/rewards/:rewardId` | Deletar reward |
| PATCH | `/admin/companies/:companyId/rewards/:rewardId/activate` | Ativar reward |
| PATCH | `/admin/companies/:companyId/rewards/:rewardId/deactivate` | Desativar reward |

---

### Webhooks Configuration (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/webhooks-configurations` | Listar configs |
| POST | `/admin/companies/:companyId/webhooks-configurations` | Criar config |
| GET | `/admin/companies/:companyId/webhooks-configurations/:id` | Obter config |
| PATCH | `/admin/companies/:companyId/webhooks-configurations/:id` | Atualizar config |
| PATCH | `/admin/companies/:companyId/webhooks-configurations/:id/enable` | Habilitar |
| PATCH | `/admin/companies/:companyId/webhooks-configurations/:id/disable` | Desabilitar |
| GET | `/admin/companies/:companyId/webhooks-configurations/:id/webhooks` | Listar webhooks enviados |
| GET | `/admin/companies/:companyId/webhooks-configurations/:id/webhooks/:webhookId` | Obter webhook enviado |

---

### Refunds (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/refunds` | Listar reembolsos |
| PATCH | `/admin/companies/:companyId/refunds/:refundId/approve` | Aprovar reembolso |

---

### Order Documents (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/admin/companies/:companyId/orders/:orderId/order-documents` | Criar documento |
| GET | `/admin/companies/:companyId/orders/:orderId/order-documents` | Listar documentos |
| DELETE | `/admin/companies/:companyId/orders/:orderId/order-documents/:id` | Deletar documento |

---

### Order Products (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/order-products` | Listar order products |
| GET | `/admin/companies/:companyId/order-products/xls` | Exportar order products XLS |

---

### Product Variants (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/products/:productId/variants` | Listar variantes |
| POST | `/admin/companies/:companyId/products/:productId/variants` | Criar variante |
| GET | `/admin/companies/:companyId/products/:productId/variants/:variantId` | Obter variante |
| PATCH | `/admin/companies/:companyId/products/:productId/variants/:variantId` | Atualizar variante |
| DELETE | `/admin/companies/:companyId/products/:productId/variants/:variantId` | Deletar variante |
| GET | `/admin/companies/:companyId/products/:productId/variants/:variantId/values` | Listar valores |
| POST | `/admin/companies/:companyId/products/:productId/variants/:variantId/values` | Criar valor |
| GET | `/admin/companies/:companyId/products/:productId/variants/:variantId/values/:id` | Obter valor |
| DELETE | `/admin/companies/:companyId/products/:productId/variants/:variantId/values/:id` | Deletar valor |

---

### Payment Providers (parcialmente implementado)

Configurados no frontend: Stripe (básico), ASAAS. **Faltam:**

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/admin/companies/:companyId/configurations/providers/pagar-me` | Configurar Pagar.me |
| PATCH | `/admin/companies/:companyId/configurations/providers/paypal` | Configurar PayPal |
| PATCH | `/admin/companies/:companyId/configurations/providers/transfer` | Configurar Transfer |
| PATCH | `/admin/companies/:companyId/configurations/providers/stripe/connect` | Stripe OAuth Connect |
| PATCH | `/admin/companies/:companyId/configurations/providers/stripe/finish-connection` | Finalizar Stripe OAuth |
| PATCH | `/admin/companies/:companyId/configurations/providers/stripe/refresh-connection` | Refresh Stripe OAuth |
| DELETE | `/admin/companies/:companyId/configurations/providers/:provider` | Deletar provedor |

---

### Orders — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/orders/xls` | Exportar orders XLS |
| PATCH | `/admin/companies/:companyId/orders/:orderId/force-deliver` | Forçar entrega |

---

### Split Configurations — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/admin/companies/:companyId/split-configurations` | Criar split config |
| DELETE | `/admin/companies/:companyId/split-configurations/:id` | Deletar split config |
| GET | `/admin/companies/:companyId/split-configurations/:id/correlated-configurations` | Configs correlacionadas |

---

### Promotions — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/admin/companies/:companyId/promotions` | Criar promoção |
| PATCH | `/admin/companies/:companyId/promotions/:promotionId` | Atualizar promoção |
| DELETE | `/admin/companies/:companyId/promotions/:promotionId` | Deletar promoção |
| POST | `/admin/companies/:companyId/promotions/:promotionId/products` | Adicionar produto |
| PATCH | `/admin/companies/:companyId/promotions/:promotionId/products` | Override produtos |
| GET | `/admin/companies/:companyId/promotions/:promotionId/products/:productId` | Obter produto da promoção |
| DELETE | `/admin/companies/:companyId/promotions/:promotionId/products/:productId` | Remover produto |
| POST | `/admin/companies/:companyId/promotions/:promotionId/whitelists` | Adicionar whitelist |
| PATCH | `/admin/companies/:companyId/promotions/:promotionId/whitelists` | Override whitelists |
| GET | `/admin/companies/:companyId/promotions/:promotionId/whitelists/:whitelistId` | Obter whitelist |
| DELETE | `/admin/companies/:companyId/promotions/:promotionId/whitelists/:whitelistId` | Remover whitelist |

---

### Products — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/products/get-by-slug/:slug` | Obter produto por slug |
| GET | `/admin/companies/:companyId/products/:productId/order-rules/:orderRuleId` | Obter order rule |
| PATCH | `/admin/companies/:companyId/products/:productId/order-rules/:orderRuleId` | Atualizar order rule |
| DELETE | `/admin/companies/:companyId/products/:productId/order-rules/:orderRuleId` | Deletar order rule |

---

### Projects — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/projects/:projectId` | Obter projeto |
| PATCH | `/admin/companies/:companyId/projects/:projectId` | Atualizar projeto |
| DELETE | `/admin/companies/:companyId/projects/:projectId` | Deletar projeto |
| GET | `/admin/companies/:companyId/projects/check/host-availability` | Verificar disponibilidade de host |
| POST | `/admin/companies/:companyId/projects/:projectId/hosts` | Vincular host |
| DELETE | `/admin/companies/:companyId/projects/:projectId/hosts/:hostId` | Desvincular host |
| POST | `/admin/companies/:companyId/projects/:projectId/pages` | Criar/upsert page |
| PATCH | `/admin/companies/:companyId/projects/:projectId/pages/:pageId` | Atualizar page |
| DELETE | `/admin/companies/:companyId/projects/:projectId/pages/:pageId` | Deletar page |

---

### Exports — endpoint faltante

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/admin/companies/:companyId/exports` | Listar exports |

---

### Component Modules

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/globals/component-modules` | Listar component modules |

---

## Registry Backend (KEY)

### Vouchers (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/:companyId/vouchers` | Criar voucher |
| GET | `/:companyId/vouchers` | Listar vouchers |
| DELETE | `/:companyId/vouchers/:id` | Desabilitar voucher |

---

### Withdrawals — admin (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/:companyId/withdraws/admin` | Listar todos os saques |
| GET | `/:companyId/withdraws/admin/:id` | Obter saque |
| PATCH | `/:companyId/withdraws/admin/:id/escrow` | Escrow saque |
| PATCH | `/:companyId/withdraws/admin/:id/refuse` | Recusar saque |
| PATCH | `/:companyId/withdraws/admin/:id/conclude` | Concluir saque |

---

### Contracts — role management

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/contracts/has-role` | Verificar role no contrato |
| PATCH | `/:companyId/contracts/grant-role` | Conceder role no contrato |

---

### ERC20 Contracts — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/erc20-contracts/has-role` | Verificar role (ERC20) |
| PATCH | `/:companyId/erc20-contracts/grant-role` | Conceder role (ERC20) |
| PATCH | `/:companyId/erc20-contracts/:id/transfer-config` | Config de transferência ERC20 |

---

### ERC20 Tokens — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/erc20-tokens/:id/burn` | Burn ERC20 |
| GET | `/:companyId/erc20-tokens/:id/history` | Histórico completo ERC20 |
| GET | `/:companyId/erc20-tokens/:id/history/operator/:operatorId` | Histórico por operador |
| GET | `/:companyId/erc20-tokens/:id/history/:userId` | Histórico por usuário |

---

### Token Editions — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/token-editions/retry-bulk-by-collection` | Retry bulk por collection |
| GET | `/:companyId/token-editions/xls` | Exportar editions XLS |
| PATCH | `/:companyId/token-editions/mint-on-demand` | Mint on demand |
| PATCH | `/:companyId/token-editions/locked-for-buy` | Travar para compra |
| PATCH | `/:companyId/token-editions/unlocked-for-buy` | Destravar |
| PATCH | `/:companyId/token-editions/notify-externally-minted` | Notificar mint externo |
| PATCH | `/:companyId/token-editions/update-token-metadata` | Atualizar metadata |
| PATCH | `/:companyId/token-editions/:id` | Editar edition |
| PATCH | `/:companyId/token-editions/:id/ready-to-mint` | Ready to mint (single) |
| PATCH | `/:companyId/token-editions/:id/transfer-token/email` | Transferir por email |

---

### Token Collections — endpoint faltante

| Method | Path | Purpose |
|--------|------|---------|
| DELETE | `/:companyId/token-collections/:id/burn` | Burn collection publicada |

---

### Tokens

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/tokens/check-collection-token-holder` | Verificar token holder |

---

### Loyalties Admin — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/:companyId/loyalties/admin/:loyaltyId` | Obter loyalty |
| PATCH | `/:companyId/loyalties/admin/:loyaltyId` | Atualizar loyalty |
| POST | `/:companyId/loyalties/admin/:loyaltyId/rules` | Criar regra |
| PATCH | `/:companyId/loyalties/admin/:loyaltyId/rules/:ruleId` | Atualizar regra |
| GET | `/:companyId/loyalties/admin/:loyaltyId/rules/:ruleId` | Obter regra |
| DELETE | `/:companyId/loyalties/admin/:loyaltyId/rules/:ruleId` | Deletar regra |

---

### Loyalties Users — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/:companyId/loyalties/users/balance/:userId/:loyaltyId` | Balance por loyalty |
| GET | `/:companyId/loyalties/users/deferred` | Saldo diferido |
| GET | `/:companyId/loyalties/users/deferred/xlsx` | Exportar diferido XLS |
| GET | `/:companyId/loyalties/users/deferred/:userId` | Diferido por user |
| GET | `/:companyId/loyalties/users/:userId/staking/:loyaltyId` | Info de staking |
| PATCH | `/:companyId/loyalties/users/:userId/staking/:loyaltyId/redeem` | Resgatar staking |

---

### Loyalties Rewards — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/loyalties/rewards/cashback/preview` | Preview cashback |
| PATCH | `/:companyId/loyalties/rewards/payment/preview` | Preview payment |
| PATCH | `/:companyId/loyalties/rewards/payment` | Pagar com pontos |

---

### External Contacts — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| PATCH | `/:companyId/external-contacts/:id` | Atualizar contato externo |

---

### Exports

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/:companyId/exports` | Listar exports |
| GET | `/:companyId/exports/:exportId` | Obter export |

---

### Jobs

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/:companyId/jobs` | Listar todos os jobs |

---

## PixwayID Backend (ID)

### Users — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/users/find-user-by-email` | Buscar user por email |
| GET | `/users/:tenantId/report/:email` | Relatório do user |
| GET | `/users/:tenantId/xls` | Exportar users XLS |
| PATCH | `/users/:tenantId/:id/token` | Atualizar token do user |
| PATCH | `/users/:tenantId/:userId/exclude-account` | Excluir conta do user |
| POST | `/users/:tenantId/:userId/toggle-operator-role` | Toggle role operador |
| GET | `/users/:tenantId/code/:userCode` | Buscar user por código |
| POST | `/users/:tenantId/code/:userId` | Gerar código temporário |

---

### Tenant Integration (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/tenant-integration/:tenantId` | Listar integrações |
| POST | `/tenant-integration/:tenantId` | Criar integração |
| PATCH | `/tenant-integration/:tenantId/:id/accept` | Aceitar integração |
| PATCH | `/tenant-integration/:tenantId/:id/reject` | Rejeitar integração |

---

### Tenant Hosts — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/tenant-hosts/:tenantId` | Criar host |
| GET | `/tenant-hosts/:tenantId/main-host` | Obter host principal |
| GET | `/tenant-hosts/:tenantId/:id` | Obter host |
| PATCH | `/tenant-hosts/:tenantId/:id` | Atualizar host |

---

### Whitelist — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/whitelists/:tenantId/check-user` | Check user multi-whitelist |
| POST | `/whitelists/:tenantId` | Criar whitelist |
| DELETE | `/whitelists/:tenantId/:id` | Deletar whitelist |
| PATCH | `/whitelists/:tenantId/:id/promote-on-chain` | Promover on-chain |
| GET | `/whitelists/:tenantId/:id/check-user` | Check user single whitelist |
| DELETE | `/whitelists/:tenantId/:id/entries/:entryId` | Deletar entry |

---

### Billing — endpoint faltante

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/billing/:tenantId/billing-usages` | Listar billing usages |

---

### Tenant Context — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/tenant-context/:tenantId` | Criar tenant context |
| GET | `/tenant-context/:tenantId/:tenantContextId/approvers` | Listar aprovadores |

---

### Exports

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/exports/:tenantId` | Listar exports |
| GET | `/exports/:tenantId/:exportId` | Obter export |

---

### Users Documents — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/users/:tenantId/documents/find-user-by-any` | Buscar user por CPF/ID |
| GET | `/users/:tenantId/documents/:userId/context/:contextId` | Docs por contexto |

---

### Notifications — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/notifications/:tenantId` | Criar/atualizar notificação |

---

## Pass Backend

### Token Pass Benefit Addresses (módulo inteiro — 0% implementado)

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/tenants/:tenantId/token-pass-benefit-addresses` | Criar endereço |
| GET | `/tenants/:tenantId/token-pass-benefit-addresses` | Listar endereços |
| GET | `/tenants/:tenantId/token-pass-benefit-addresses/:id` | Obter endereço |
| PATCH | `/tenants/:tenantId/token-pass-benefit-addresses/:id` | Atualizar endereço |
| DELETE | `/tenants/:tenantId/token-pass-benefit-addresses/:id` | Deletar endereço |

---

### Token Pass Benefits — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/tenants/:tenantId/token-pass-benefits/usages` | Listar usages |
| GET | `/tenants/:tenantId/token-pass-benefits/usages/xls` | Exportar usages XLS |
| DELETE | `/tenants/:tenantId/token-pass-benefits/:id` | Deletar benefit |
| POST | `/tenants/:tenantId/token-pass-benefits/:id/register-use` | Registrar uso |
| POST | `/tenants/:tenantId/token-pass-benefits/:id/register-use-by-user` | Registrar uso por user |
| POST | `/tenants/:tenantId/token-pass-benefits/register-use-by-qrcode` | Registrar uso por QR |
| GET | `/tenants/:tenantId/token-pass-benefits/:id/verify` | Verificar benefit |

---

### Token Passes — endpoints faltantes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/tenants/:tenantId/token-passes/users/:userId` | Passes do user |
| GET | `/tenants/:tenantId/token-passes/:id/token-editions/:editionNumber/benefits` | Benefits por edition |
| DELETE | `/tenants/:tenantId/token-passes/:id` | Deletar pass |

---

### Share Codes

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/tenants/:tenantId/token-pass-share-codes` | Criar share code |

---

### Exports

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/tenants/:tenantId/exports` | Listar exports |
| GET | `/tenants/:tenantId/exports/:exportId` | Obter export |

---

## Resumo por Módulo

| Backend | Módulo | Endpoints Faltantes | Implementação |
|---------|--------|-------------------|---------------|
| Commerce | Bookings | 10 | 0% — módulo inteiro |
| Commerce | FAQ | 10 | 0% — módulo inteiro |
| Commerce | Rewards | 7 | 0% — módulo inteiro |
| Commerce | Webhooks Config | 8 | 0% — módulo inteiro |
| Commerce | Order Documents | 3 | 0% — módulo inteiro |
| Commerce | Order Products | 2 | 0% — módulo inteiro |
| Commerce | Refunds | 2 | 0% — módulo inteiro |
| Commerce | Product Variants | 9 | 0% — módulo inteiro |
| Commerce | Payment Providers | 7 | Parcial (faltam Pagar.me, PayPal, Transfer, Stripe OAuth) |
| Commerce | Promotions | 11 | Parcial (CRUD + produto/whitelist detail) |
| Commerce | Projects | 9 | Parcial (faltam update/delete/hosts/pages edit) |
| Commerce | Orders | 2 | Parcial (faltam XLS export, force deliver) |
| Commerce | Split Configs | 3 | Parcial (faltam create, delete, correlated) |
| Commerce | Products | 4 | Parcial (faltam slug, order rule detail) |
| Commerce | Component Modules | 1 | 0% |
| Commerce | Exports | 1 | 0% |
| Registry | Vouchers | 3 | 0% — módulo inteiro |
| Registry | Withdrawals (admin) | 5 | 0% — módulo inteiro |
| Registry | Token Editions | 10 | Parcial (ops avançadas) |
| Registry | Loyalties Admin | 6 | Parcial (CRUD regras) |
| Registry | Loyalties Users | 6 | Parcial (staking, deferred) |
| Registry | Loyalties Rewards | 3 | 0% |
| Registry | ERC20 Tokens | 4 | Parcial (burn, history) |
| Registry | ERC20 Contracts | 3 | Parcial (roles, transfer config) |
| Registry | Contracts | 2 | Parcial (role management) |
| Registry | Token Collections | 1 | Parcial (burn) |
| Registry | External Contacts | 1 | Parcial (update) |
| Registry | Tokens | 1 | 0% |
| Registry | Exports | 2 | 0% |
| Registry | Jobs | 1 | Parcial |
| PixwayID | Users | 8 | Parcial (advanced ops) |
| PixwayID | Tenant Integration | 4 | 0% — módulo inteiro |
| PixwayID | Tenant Hosts | 4 | Parcial (write ops) |
| PixwayID | Whitelist | 6 | Parcial (delete, on-chain, check) |
| PixwayID | Tenant Context | 2 | Parcial (create, approvers) |
| PixwayID | Users Documents | 2 | Parcial |
| PixwayID | Billing | 1 | Parcial (usages) |
| PixwayID | Exports | 2 | 0% |
| PixwayID | Notifications | 1 | Parcial |
| Pass | Benefit Addresses | 5 | 0% — módulo inteiro |
| Pass | Benefits | 7 | Parcial (usages, register, verify) |
| Pass | Passes | 3 | Parcial (user passes, delete) |
| Pass | Share Codes | 1 | 0% |
| Pass | Exports | 2 | 0% |
| **Total** | | **~162** | |
