---
id: FLOW_COMMERCE_PROMOTIONS
title: "Commerce - Promoções e Cupons"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - promotions
  - coupons
  - discounts
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Promoções e Cupons

## Visão Geral

O sistema de promoções suporta dois tipos: **cupons** (baseados em código, o usuário insere um código) e **descontos** (automáticos, aplicados sem código). Ambos podem ser direcionados a produtos específicos, usuários, intervalos de datas e métodos de pagamento. As promoções rastreiam o uso por usuário e globalmente, e podem ser combinadas ou restritas.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | JWT com papel Admin | Autenticação no serviço de ID |
| Produtos publicados | Produtos para aplicar promoções | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |

## Entidades e Relacionamentos

```
Promotion (1) ──→ (N) PromotionProduct     ← quais produtos se qualificam
Promotion (1) ──→ (N) PromotionWhitelist   ← quais usuários se qualificam
Promotion (1) ──→ (N) PromotionUsageLog    ← trilha de auditoria
Promotion (N) ──→ (1) User (owner)         ← criador (para indicações)
```

### Tipos de Promoção

| Tipo | Gatilho | Exemplo |
|------|---------|---------|
| `coupon` | Usuário insere um código no checkout | Código "SUMMER20" para 20% de desconto |
| `discount` | Aplicado automaticamente se as condições forem atendidas | 10% de desconto para pagamentos via PIX |

---

## Fluxo: Criar um Cupom

### Passo 1: Criar a Promoção

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions` | Bearer (Admin) | application/json |

**Requisição Mínima (cupom simples):**
```json
{
  "publicDescription": "20% off summer sale",
  "type": "coupon",
  "amountType": "percentage",
  "amount": "20",
  "code": "SUMMER20",
  "amountByCart": true,
  "applyToAllProducts": true,
  "applyToAllUsers": true,
  "isCombinable": false
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "description": "Internal: Summer 2026 campaign - Instagram influencer codes",
  "publicDescription": "20% off your purchase!",
  "type": "coupon",
  "amountType": "percentage",
  "amount": "20",
  "amountByCart": true,
  "code": "SUMMER20",
  "applyToAllProducts": false,
  "applyToAllUsers": false,
  "isCombinable": false,
  "startAt": "2026-04-01T00:00:00.000Z",
  "endAt": "2026-06-30T23:59:59.000Z",
  "maxUsages": 1000,
  "maxUsagesPerUser": 3,
  "requirements": {
    "minCartPrice": "10000000000000000000",
    "maxCartItems": 5,
    "paymentMethods": ["credit_card", "pix"],
    "currencyIds": ["brl-currency-uuid"],
    "notApplicableProductIds": ["excluded-product-uuid"]
  }
}
```

**Resposta (201):**
```json
{
  "id": "promotion-uuid",
  "companyId": "company-uuid",
  "type": "coupon",
  "code": "SUMMER20",
  "amountType": "percentage",
  "amount": "20",
  "usages": 0,
  "maxUsages": 1000,
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `publicDescription` | string | Sim | Descrição visível ao cliente |
| `type` | enum | Sim | `coupon` ou `discount` |
| `amountType` | enum | Sim | `fixed` ou `percentage` |
| `amount` | string | Sim | Valor do desconto (big number para fixed, 0-100 para percentage) |
| `amountByCart` | boolean | Sim | `true`: aplicar uma vez por pedido. `false`: aplicar por item |
| `applyToAllProducts` | boolean | Sim | Escopo: todos os produtos ou específicos |
| `applyToAllUsers` | boolean | Sim | Escopo: todos os usuários ou apenas da whitelist |
| `isCombinable` | boolean | Sim | Pode acumular com outras promoções |
| `code` | string | Sim (cupom) | Código em MAIÚSCULAS, único por empresa |
| `description` | string | Não | Notas internas do administrador |
| `startAt` | ISO 8601 | Não | Data de início da promoção |
| `endAt` | ISO 8601 | Não | Data de término da promoção |
| `maxUsages` | number | Não | Limite global de uso |
| `maxUsagesPerUser` | number | Não | Limite de uso por usuário |
| `requirements` | object | Não | Condições adicionais (veja abaixo) |

**Objeto Requirements:**

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `minCartPrice` | string | Total mínimo do carrinho (big number) |
| `maxCartPrice` | string | Total máximo do carrinho |
| `minCartItems` | number | Mínimo de itens no carrinho |
| `maxCartItems` | number | Máximo de itens no carrinho |
| `minSameCartItems` | number | Quantidade mínima do mesmo produto |
| `maxSameCartItems` | number | Quantidade máxima do mesmo produto |
| `paymentMethods` | string[] | Restringir a estes métodos de pagamento |
| `currencyIds` | uuid[] | Restringir a estas moedas |
| `destinationWalletAddresses` | string[] | Restringir a estas carteiras de destino |
| `notApplicableProductIds` | uuid[] | Excluir estes produtos |

**Observações:**
- Os códigos de cupom são automaticamente convertidos para maiúsculas
- Os códigos devem ser únicos por empresa — duplicatas retornam 409
- O campo `amount` usa big numbers para o tipo `fixed`, mas números regulares (0-100) para `percentage`
- `amountByCart: true` significa "20% de desconto no total do carrinho". `false` significa "20% de desconto em cada item qualificado"

---

### Passo 2: Definir Escopo para Produtos Específicos (se applyToAllProducts = false)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/products` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "productId": "product-uuid",
  "maxUsages": 500,
  "maxUsagesPerUser": 2
}
```

**Observações:**
- Cada produto em uma promoção pode ter seus próprios limites de uso
- Use o método do SDK `setAndOverridePromotionProducts` para substituir todos os produtos de uma vez

**Para substituir todos os produtos de uma vez:**
```json
{
  "products": [
    { "productId": "product-1-uuid", "maxUsages": 100 },
    { "productId": "product-2-uuid", "maxUsages": 200 }
  ]
}
```

---

### Passo 3: Restringir a Usuários Específicos (se applyToAllUsers = false)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/promotions/{promotionId}/whitelists` | Bearer (Admin) | application/json |

**Por email:**
```json
{
  "type": "email",
  "value": "user@example.com",
  "maxUsages": 5,
  "maxUsagesPerUser": 1
}
```

**Por whitelist W3Block:**
```json
{
  "type": "w3block_id_whitelist",
  "value": "whitelist-uuid",
  "maxUsages": 100
}
```

**Observações:**
- O tipo `email` restringe a um endereço de email específico
- O tipo `w3block_id_whitelist` restringe a todos os usuários em uma whitelist W3Block
- Cada entrada de whitelist pode ter seus próprios limites de uso

---

## Fluxo: Criar um Desconto Automático

Mesmo endpoint dos cupons, mas com `type: "discount"` e sem `code`.

**Requisição:**
```json
{
  "publicDescription": "10% off for PIX payments",
  "type": "discount",
  "amountType": "percentage",
  "amount": "10",
  "amountByCart": true,
  "applyToAllProducts": true,
  "applyToAllUsers": true,
  "isCombinable": true,
  "requirements": {
    "paymentMethods": ["pix"]
  }
}
```

**Observações:**
- Descontos automáticos são avaliados durante a pré-visualização do pedido sem interação do usuário
- Se `isCombinable: true`, este desconto pode acumular com cupons
- A resposta da pré-visualização mostra o desconto aplicado em `originalCartPrice` vs `cartPrice`

---

## Fluxo: Atualizar uma Promoção

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/promotions/{promotionId}` | Bearer (Admin) | application/json |

**Requisição (atualização parcial):**
```json
{
  "maxUsages": 2000,
  "endAt": "2026-09-30T23:59:59.000Z"
}
```

---

## Fluxo: Excluir uma Promoção

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| DELETE | `/admin/companies/{companyId}/promotions/{promotionId}` | Bearer (Admin) | — |

**Observações:**
- Esta é uma **exclusão lógica** (`deletedAt` é definido)
- Pedidos existentes com esta promoção não são afetados
- O código do cupom fica disponível para reutilização após a exclusão

---

## Fluxo: Aplicar um Cupom a um Pedido

Cupons são aplicados durante a criação do pedido ou via um endpoint dedicado em um pedido existente.

### Durante a Criação do Pedido

Passe `couponCode` na requisição de criação do pedido:
```json
{
  "orderProducts": [...],
  "payments": [...],
  "couponCode": "SUMMER20"
}
```

### Em um Pedido Existente

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/companies/{companyId}/orders/{orderId}/coupon` | Bearer (Owner) | application/json |

**Requisição:**
```json
{
  "code": "SUMMER20"
}
```

---

## Como as Promoções São Avaliadas

1. Durante a pré-visualização / criação do pedido, o sistema verifica todas as promoções ativas
2. Para o tipo `discount`: aplicado automaticamente se todas as condições forem atendidas
3. Para o tipo `coupon`: aplicado apenas se o usuário fornecer o código correto
4. Verificações de validação em ordem:
   - A promoção está ativa? (datas, exclusão lógica)
   - O limite global de uso foi atingido?
   - O limite por usuário foi atingido?
   - O usuário corresponde às restrições de whitelist/email?
   - O carrinho atende aos requisitos? (preço mín/máx, itens, método de pagamento)
   - O produto corresponde? (applyToAllProducts ou na lista PromotionProduct)
5. Se `isCombinable: false`, apenas uma promoção não combinável se aplica por pedido
6. O uso é registrado em `PromotionUsageLog` na criação do pedido
7. Se o pedido for cancelado, o uso é revertido (`reverted: true`)

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Código de cupom inválido | Código não encontrado ou expirado | Verifique o código, confira o intervalo de datas |
| 400 | Limite de uso excedido | Limite global ou por usuário atingido | Aumente os limites ou crie uma nova promoção |
| 400 | Requisitos não atendidos | Carrinho não atende aos requisitos | Verifique minCartPrice, paymentMethods, etc. |
| 400 | Produto não elegível | Produto não está no escopo da promoção | Adicione o produto à promoção ou defina applyToAllProducts |
| 400 | Usuário não elegível | Usuário não está na whitelist | Adicione o email do usuário ou entrada na whitelist |
| 409 | Código já existe | Código de cupom duplicado para esta empresa | Use um código diferente |
| 409 | Conflito de não combinável | Múltiplas promoções não combináveis | Defina `isCombinable: true` ou remova a promoção conflitante |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Código de cupom rejeitado como "not found" | Os códigos são em **MAIÚSCULAS**. Envie `"SUMMER20"` e não `"summer20"` |
| 2 | Desconto não aparece na pré-visualização | Verifique: o tipo do desconto é `discount` (não `coupon`)? As datas estão ativas? O carrinho atende aos requisitos? |
| 3 | Promoção aplicada aos produtos errados | Se `applyToAllProducts: false`, você deve adicionar produtos via o endpoint de promotion-products |
| 4 | Contagem de uso não decrementa no cancelamento | O cancelamento define `reverted: true` nos logs de uso, mas não decrementa o contador. O sistema verifica `reverted` ao avaliar limites |
| 5 | Desconto percentual maior que o esperado | `amountByCart: false` aplica o percentual a **cada item**, o que pode exceder as expectativas em carrinhos com múltiplos itens |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticação | Token admin necessário | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Ciclo de Vida do Produto | Produtos devem existir antes do escopo | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Pedido e Compra | Promoções aplicadas durante a criação do pedido | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
