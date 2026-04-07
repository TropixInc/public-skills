---
id: FLOW_COMMERCE_PRODUCT_LIFECYCLE
title: "Commerce - Ciclo de Vida do Produto"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - products
  - variants
  - order-rules
  - tags
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Ciclo de Vida do Produto

## Visão Geral

Produtos na W3Block representam itens vendáveis — NFTs (ERC721/1155), tokens fungíveis (ERC20) ou bens externos. Um produto passa por um ciclo de vida definido: **Draft → Publishing → Published → (Sold | Cancelled)**. Este fluxo cobre a criação de produtos, adição de variantes e regras de pedido, publicação e gerenciamento de tags.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | Token de acesso JWT com papel Admin | Autenticação no serviço de ID |
| Smart contract | Contrato implantado (para produtos NFT/ERC20) | Módulo de Tokens |
| Currencies | Pelo menos uma moeda configurada | `GET /globals/currencies` |

## Entidades e Relacionamentos

```
Product (1) ──→ (N) ProductToken       ← unidades vendáveis individuais
Product (1) ──→ (N) ProductVariant     ← dimensões de variante (Tamanho, Cor)
  ProductVariant (1) ──→ (N) ProductVariantValue  ← opções (P, M, G)
Product (1) ──→ (N) ProductOrderRule   ← restrições de compra
Product (M) ←──→ (N) Tag              ← categorização
```

### Ciclo de Vida do Status do Produto

```
DRAFT ──→ PUBLISHING ──→ PUBLISHED ──→ SOLD (todos os tokens vendidos)
  ↑          │                │
  │          ↓                ↓
  │       (falha)         CANCELLED
  │          │
  └──────────┘ (retry)

PUBLISHED ──→ UPDATING ──→ PUBLISHED
```

---

## Fluxo: Criar um Produto

### Passo 1: Fazer Upload das Imagens do Produto

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/assets` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "type": "image",
  "target": "PRODUCT"
}
```

**Resposta (200):**
```json
{
  "id": "asset-uuid",
  "uploadParams": {
    "url": "https://api.cloudinary.com/v1_1/...",
    "fields": {
      "api_key": "...",
      "signature": "...",
      "timestamp": "..."
    }
  }
}
```

**Observações:**
- Faça upload do arquivo real para a URL do Cloudinary retornada em `uploadParams`
- Salve o `assetId` e as URLs `original` / `thumb` resultantes para o produto

---

### Passo 2: Criar o Produto (Draft)

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/products` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "My NFT Collection",
  "contractAddress": "0x1234...abcd",
  "chainId": 137,
  "prices": [
    { "currencyId": "currency-uuid", "amount": "1000000000000000000" }
  ],
  "distributionType": "random"
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "name": "My NFT Collection",
  "description": "A limited edition digital collectible",
  "contractAddress": "0x1234...abcd",
  "chainId": 137,
  "prices": [
    { "currencyId": "currency-uuid-brl", "amount": "50000000000000000000" },
    { "currencyId": "currency-uuid-matic", "amount": "1000000000000000000" }
  ],
  "images": [
    { "assetId": "asset-uuid", "original": "https://...", "thumb": "https://..." }
  ],
  "distributionType": "random",
  "pricingType": "product",
  "type": "nft",
  "slug": "my-nft-collection",
  "startSaleAt": "2026-04-01T00:00:00.000Z",
  "endSaleAt": "2026-06-30T23:59:59.000Z",
  "canResale": true,
  "onDemandMintEnabled": false,
  "htmlContent": "<p>Rich description with HTML</p>",
  "tags": ["tag-uuid-1", "tag-uuid-2"],
  "draftData": {
    "keyCollectionId": "collection-uuid",
    "range": "1-100",
    "quantity": "100"
  }
}
```

**Resposta (201):**
```json
{
  "id": "product-uuid",
  "companyId": "company-uuid",
  "name": "My NFT Collection",
  "status": "draft",
  "prices": [...],
  "distributionType": "random",
  "createdAt": "2026-03-31T12:00:00.000Z",
  "updatedAt": "2026-03-31T12:00:00.000Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do produto |
| `contractAddress` | string | Sim (NFT/ERC20) | Endereço do smart contract |
| `chainId` | integer | Sim (NFT/ERC20) | ID da cadeia blockchain (137=Polygon, 1=Ethereum) |
| `prices` | ProductPrice[] | Sim | Precificação multi-moeda. Valor em wei/menor unidade |
| `distributionType` | enum | Sim | `random`, `fixed` ou `sequential` |
| `description` | string | Não | Descrição do produto |
| `images` | ProductImage[] | Não | Imagens do produto (do upload de assets) |
| `pricingType` | string | Não | `product` (mesmo preço para todos os tokens) ou `token` (preço por token) |
| `type` | enum | Não | `nft`, `erc20`, `external`. Padrão: `nft` |
| `slug` | string | Não | Slug da URL, único por empresa |
| `startSaleAt` | ISO 8601 | Não | Data de início da venda |
| `endSaleAt` | ISO 8601 | Não | Data de término da venda |
| `canResale` | boolean | Não | Habilitar vendas secundárias |
| `onDemandMintEnabled` | boolean | Não | Mintar tokens na compra ao invés de pré-mintar |
| `htmlContent` | string | Não | Conteúdo HTML rico para a página do produto |
| `tags` | string[] | Não | UUIDs de tags para categorização |
| `draftData` | object | Não | Referência de coleção, intervalo de tokens, quantidade |
| `disableSelfPurchase` | boolean | Não | Impedir que o criador do produto compre |
| `requirements` | object | Não | Requisitos de compra (KYC, etc.) |

**Observações:**
- O produto é criado com status `draft` — ainda não é visível para compradores
- Os valores são **big numbers** (wei para crypto, menor unidade da moeda para fiat)
- `draftData.keyCollectionId` vincula a uma coleção de tokens no serviço de registro
- Múltiplos preços permitem vender em diferentes moedas simultaneamente

---

### Passo 3: Adicionar Variantes (Opcional)

Variantes adicionam dimensões como Tamanho ou Cor. Cada variante tem valores com modificadores de preço opcionais.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/variants` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "name": "Size",
  "keyLabel": "size",
  "values": [
    { "name": "Small", "keyValue": "small", "extraAmount": "0" },
    { "name": "Medium", "keyValue": "medium", "extraAmount": "5000000000000000000" },
    { "name": "Large", "keyValue": "large", "extraAmount": "10000000000000000000" }
  ]
}
```

**Observações:**
- `extraAmount` é adicionado ao preço base do produto para aquele valor de variante
- Variantes são excluídas logicamente (`deletedAt`) quando removidas
- Compradores selecionam valores de variante durante a criação do pedido via `variantIds`

---

### Passo 4: Adicionar Regras de Pedido (Opcional)

Regras de pedido controlam quem pode comprar, quando e quantas unidades.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/products/{productId}/order-rules` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "purchaseLimit": 5
}
```

**Requisição Completa:**
```json
{
  "whitelistId": "whitelist-uuid",
  "startAt": "2026-04-01T00:00:00.000Z",
  "endAt": "2026-04-15T23:59:59.000Z",
  "purchaseLimit": 2
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `purchaseLimit` | number | Não | Máximo de compras por usuário (padrão: 5) |
| `whitelistId` | string | Não | Restringir a usuários nesta whitelist |
| `startAt` | ISO 8601 | Não | Início de vigência da regra |
| `endAt` | ISO 8601 | Não | Fim de vigência da regra |

**Observações:**
- Múltiplas regras podem ser adicionadas a um produto; todas devem ser satisfeitas
- As regras são avaliadas durante a criação do pedido
- Se `whitelistId` estiver definido, apenas usuários nessa whitelist podem comprar

---

### Passo 5: Publicar o Produto

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/products/{productId}/publish` | Bearer (Admin) | application/json |

**Requisição:**
Mesmo formato do DTO de criação — passe quaisquer atualizações finais junto com a ação de publicação.

```json
{
  "name": "My NFT Collection",
  "prices": [
    { "currencyId": "currency-uuid", "amount": "50000000000000000000" }
  ]
}
```

**Resposta (200):**
```json
{
  "id": "product-uuid",
  "status": "publishing",
  "..."
}
```

**Observações:**
- O status muda para `publishing` imediatamente, depois para `published` de forma assíncrona
- Durante a publicação, os tokens do produto são criados (mintados ou importados com base em `draftData`)
- Faça polling em `GET /admin/companies/{companyId}/products/{productId}` para verificar quando o status se torna `published`
- Se a publicação falhar, o status volta para `draft` — verifique os detalhes do produto para informações de erro

---

## Fluxo: Gerenciar Tags

Tags fornecem categorização hierárquica. Crie tags primeiro, depois atribua-as aos produtos.

### Criar uma Tag

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/tags` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "Art"
}
```

**Requisição Completa:**
```json
{
  "name": "Digital Art",
  "parentId": "parent-tag-uuid",
  "hide": false
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome da tag |
| `parentId` | uuid | Não | Tag pai para hierarquia |
| `hide` | boolean | Não | Ocultar da interface pública |

### Atribuir Tags ao Produto

Passe `tags: ["tag-uuid-1", "tag-uuid-2"]` na requisição de criação ou atualização do produto.

---

## Fluxo: Cancelar um Produto

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/products/{productId}/cancel` | Bearer (Admin) | — |

**Observações:**
- Apenas produtos `published` podem ser cancelados
- O cancelamento impede novos pedidos, mas não afeta os existentes
- Os tokens do produto associados são definidos com status `cancelled`

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Erro de validação | Campos obrigatórios ausentes ou formato inválido | Verifique os tipos dos campos; valores devem ser strings de big number |
| 400 | Endereço de contrato inválido | Contrato não encontrado na cadeia | Verifique se o contrato está implantado e o endereço está correto |
| 409 | Conflito de slug | Slug já usado por outro produto nesta empresa | Escolha um slug diferente |
| 409 | Produto não está em draft | Tentando publicar um produto que não está em draft | Verifique o status atual; apenas produtos `draft` podem ser publicados |
| 404 | Produto não encontrado | productId inválido ou companyId incorreto | Verifique os IDs |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Produto fica em `publishing` indefinidamente | A publicação é assíncrona. Se estiver travado, o job do backend pode ter falhado. Verifique os detalhes do produto para informações de erro e tente novamente |
| 2 | Valores de preço parecem errados | Os valores são em **wei** (18 decimais para crypto) ou menor unidade fiat. `1 MATIC = 1000000000000000000` |
| 3 | Tags não aparecem no produto | Tags são M:M — passe UUIDs de tags no array `tags` durante criação/atualização. Elas não são herdadas automaticamente de tags pai |
| 4 | Valores extras de variante não se aplicam | Valores extras são adicionados ao preço base. Se o preço base for 0, apenas o valor extra é cobrado |
| 5 | Regras de pedido não são aplicadas | As regras são verificadas no momento da criação do pedido, não na visualização do produto. Certifique-se de que as regras têm intervalos de datas corretos |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticação | Necessária antes de qualquer operação admin | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Pedido e Compra | Compradores adquirem produtos publicados | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
| Promoções | Aplicar descontos/cupons a produtos | [FLOW_COMMERCE_PROMOTIONS](./FLOW_COMMERCE_PROMOTIONS.md) |
| Splits e Gateways | Configurar pagamento e receita para produtos | [FLOW_COMMERCE_SPLITS_GATEWAYS](./FLOW_COMMERCE_SPLITS_GATEWAYS.md) |
