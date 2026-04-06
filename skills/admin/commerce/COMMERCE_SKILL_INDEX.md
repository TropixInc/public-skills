---
id: COMMERCE_SKILL_INDEX
title: "Módulo Commerce - Índice de Skills"
module: offpix
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - index
depends_on:
  - AUTH_SKILL_INDEX
---

# Módulo Commerce - Índice de Skills

O módulo Commerce alimenta o motor de e-commerce da W3Block: produtos (NFTs, ERC20, externos), pedidos multi-moeda, pagamentos multi-provedor, promoções/cupons, divisão de receita, tags, reembolsos e relatórios de vendas.

**Serviço Backend:** Commerce Service
**Swagger:** https://commerce.w3block.io/docs
**URL Base:** `https://commerce.w3block.io`

---

## Documentos

| # | Documento | Tipo | Versão | Status | Descrição |
|---|-----------|------|--------|--------|-----------|
| 1 | [COMMERCE_SKILL_INDEX](./COMMERCE_SKILL_INDEX.md) | Índice | 1.0.0 | Implementado | Este arquivo — ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) | Ref. API | 1.0.0 | Implementado | Todos os endpoints, DTOs, enums, schemas |
| 3 | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) | Fluxo | 1.0.0 | Implementado | Criar, atualizar, publicar, cancelar produtos |
| 4 | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) | Fluxo | 1.0.0 | Implementado | Pré-visualização de pedido, criação, pagamento, entrega |
| 5 | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) | Fluxo | 1.0.0 | Implementado | Cupons, descontos, whitelists, escopo de produto |
| 6 | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) | Fluxo | 1.0.0 | Implementado | Divisão de receita, configuração de gateway de pagamento |

---

## Início Rápido

```
1. Autenticar             → Veja auth/AUTH_SKILL_INDEX.md
2. Criar um produto       → FLOW_COMMERCE_PRODUCT_LIFECYCLE.md
3. Configurar pagamentos  → FLOW_COMMERCE_SPLITS_GATEWAYS.md
4. Aceitar pedidos        → FLOW_COMMERCE_ORDER_PURCHASE.md
5. Gerenciar promoções    → FLOW_COMMERCE_PROMOTIONS.md
6. Revisar vendas         → COMMERCE_API_REFERENCE.md § Orders (Admin)
```

---

## Tabela de Decisão

| Eu quero… | Leia isto |
|-----------|-----------|
| Listar/criar/publicar produtos | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Adicionar variantes ou regras de pedido a um produto | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Pré-visualizar um pedido (calcular preços, taxas, descontos) | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Criar um pedido e processar pagamento | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Cancelar/reembolsar um pedido | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Criar cupons ou descontos automáticos | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Restringir promoções a usuários/produtos específicos | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Configurar gateways Stripe/ASAAS | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
| Configurar regras de distribuição de receita | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
| Gerenciar categorias de produtos (tags) | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) § Tags |
| Exportar relatórios de vendas | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) § Exports |
| Consultar qualquer endpoint | [COMMERCE_API_REFERENCE](./COMMERCE_API_REFERENCE.md) |

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Criação de pedido falha com "insufficient allowance" | Execute a pré-visualização do pedido primeiro — verifique `currencyAllowanceState` para cada moeda de pagamento |
| 2 | Produto preso no status PUBLISHING | A publicação é assíncrona. Faça polling em `GET /admin/companies/{companyId}/products/{productId}` até que o status mude para PUBLISHED |
| 3 | Código de cupom rejeitado | Os códigos são em MAIÚSCULAS e únicos por empresa. Verifique `maxUsages`, `maxUsagesPerUser`, intervalo de datas e escopo de produto/whitelist |
| 4 | Percentuais de split não somam corretamente | O total de % de split para um determinado `type` não deve exceder 100. Cada tipo (PRODUCT_PRICE, CLIENT_SERVICE_FEE, etc.) é independente |
| 5 | Pagamento expira antes da conclusão | A expiração padrão é configurável por gateway. Defina `checkoutExpireTime` na configuração do gateway |

---

## Matriz: Endpoints x Documentos

| Padrão de Endpoint | Ref. API | Produto | Pedido | Promoções | Splits |
|---------------------|----------|---------|--------|-----------|--------|
| `/admin/.../products` | X | X | | | |
| `/admin/.../products/{id}/publish` | X | X | | | |
| `/admin/.../products/{id}/variants` | X | X | | | |
| `/admin/.../products/{id}/order-rules` | X | X | | | |
| `/companies/{id}/orders` (usuário) | X | | X | | |
| `/companies/{id}/orders/preview` | X | | X | | |
| `/admin/.../orders` | X | | X | | |
| `/admin/.../orders/{id}/approve-payment` | X | | X | | |
| `/admin/.../orders/{id}/cancel` | X | | X | | |
| `/admin/.../orders/{id}/refund` | X | | X | | |
| `/admin/.../promotions` | X | | | X | |
| `/admin/.../promotions/{id}/products` | X | | | X | |
| `/admin/.../promotions/{id}/whitelists` | X | | | X | |
| `/admin/.../split-configurations` | X | | | | X |
| `/admin/.../configurations/providers/*` | X | | | | X |
| `/admin/.../tags` | X | | | | |
| `/globals/currencies` | X | | X | | X |
| `/admin/.../exports/*` | X | | | | |
