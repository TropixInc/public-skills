---
id: FLOW_COMMERCE_SPLITS_GATEWAYS
title: "Commerce - Divisão de Receita e Gateways de Pagamento"
module: offpix
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - commerce
  - splits
  - gateways
  - payments
  - stripe
  - asaas
depends_on:
  - AUTH_SKILL_INDEX
  - COMMERCE_API_REFERENCE
---

# Commerce — Divisão de Receita e Gateways de Pagamento

## Visão Geral

Este fluxo cobre duas configurações relacionadas: **gateways de pagamento** (conectar Stripe, ASAAS ou outros provedores) e **divisão de receita** (distribuir a receita do pedido entre as partes interessadas). Ambos devem ser configurados antes de aceitar pedidos.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | JWT com papel Admin | Autenticação no serviço de ID |
| Credenciais do provedor | Chaves do Stripe ou ID de carteira ASAAS | Painéis dos provedores |
| UUIDs de usuários | Destinatários da receita devem ser usuários registrados | Módulo de Contatos |

## Entidades e Relacionamentos

```
CompanySplitConfiguration ──→ User (recipient)
                          ──→ CompanyConfiguration (parent)
                          ──→ Product (optional scope)

DeferredSplit ──→ Payment (source)
              ──→ User (recipient)
              ──→ Currency
```

### Tipos de Split

Cada pedido gera múltiplos componentes de taxas. Os splits definem quem recebe qual percentual de cada componente:

```
Total do Pedido
  ├── Product Price ────── dividido entre vendedores / parceiros
  ├── Client Service Fee ─ dividido entre plataforma / parceiros
  ├── Company Service Fee  dividido entre empresa / W3Block
  ├── Gas Fee ──────────── split para cobertura de custo de gas
  └── Resale Fee ───────── split em vendas secundárias
```

---

## Fluxo: Configurar Gateways de Pagamento

### Passo 1: Verificar Moedas Disponíveis

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/globals/currencies` | Público | — |

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "brl-uuid",
      "name": "Brazilian Real",
      "code": "BRL",
      "symbol": "R$",
      "crypto": false,
      "erc20contractAddress": null
    },
    {
      "id": "matic-uuid",
      "name": "MATIC",
      "code": "MATIC",
      "symbol": "MATIC",
      "crypto": true,
      "erc20contractAddress": "0x..."
    }
  ],
  "meta": { "totalItems": 15 }
}
```

---

### Passo 2: Configurar Stripe

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/configurations/providers/stripe` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "secret": "sk_live_...",
  "publicKey": "pk_live_..."
}
```

**Requisição Completa:**
```json
{
  "secret": "sk_live_...",
  "publicKey": "pk_live_...",
  "minPaymentPrice": "500",
  "checkoutExpireTime": 1800
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `secret` | string | Sim | Chave secreta do Stripe |
| `publicKey` | string | Sim | Chave publicável do Stripe |
| `minPaymentPrice` | string | Não | Valor mínimo de pagamento (menor unidade) |
| `checkoutExpireTime` | number | Não | Segundos até o pagamento expirar (padrão: 1800 = 30 min) |

---

### Passo 3: Configurar ASAAS

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/configurations/providers/asaas` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "apiKey": "aact_..."
}
```

**Requisição Completa:**
```json
{
  "apiKey": "aact_...",
  "checkoutExpireTime": 3600,
  "minPaymentPrice": "1000",
  "maxInstallments": 12,
  "minInstallmentPrice": "5000",
  "canSaveCreditCard": true,
  "percentageSplit": "5"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `apiKey` | string | Sim | Chave de API do ASAAS |
| `checkoutExpireTime` | number | Não | Segundos até o pagamento expirar |
| `minPaymentPrice` | string | Não | Valor mínimo de pagamento |
| `maxInstallments` | number | Não | Máximo de parcelas no cartão de crédito |
| `minInstallmentPrice` | string | Não | Valor mínimo por parcela |
| `canSaveCreditCard` | boolean | Não | Permitir que usuários salvem cartões |
| `percentageSplit` | string | Não | Percentual de split do ASAAS |

---

### Passo 4: Selecionar Combinações de Provedor-Moeda-Método

Defina quais métodos de pagamento estão disponíveis para cada moeda.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/configurations/providers-selections` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "currencyId": "brl-uuid",
  "paymentProvider": "stripe",
  "paymentMethod": "credit_card"
}
```

**Observações:**
- Chame este endpoint uma vez por combinação que deseja habilitar
- Exemplo de configuração para BRL:
  - Stripe + credit_card
  - Stripe + debit_card
  - ASAAS + pix
  - ASAAS + billet
- A resposta da pré-visualização do pedido mostrará `providersForSelection` com base nessas configurações
- Combinações inválidas retornam um erro "Invalid payment provider"

---

### Passo 5: Obter Configuração Atual

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/admin/companies/{companyId}/configurations` | Bearer (Admin) | — |

Retorna a configuração completa de commerce da empresa, incluindo todos os provedores e seleções configurados.

---

## Fluxo: Configurar Provedor de Pagamento do Usuário

Para plataformas onde usuários individuais (vendedores) precisam de suas próprias contas de pagamento.

### Verificar Provedores Configurados do Usuário

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/companies/{companyId}/users/{userId}/providers/check-configured-providers` | Bearer | — |

### Configurar Provedor para o Usuário

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/companies/{companyId}/users/{userId}/providers/{provider}` | Bearer | application/json |

**Requisição (Stripe):**
```json
{
  "accountId": "acct_..."
}
```

**Requisição (ASAAS):**
```json
{
  "walletId": "..."
}
```

**Observações:**
- Contas de provedor de pagamento do usuário são necessárias para a divisão de receita — o sistema precisa saber para onde enviar a parte de cada usuário
- Stripe usa IDs de conta Connect (`acct_...`)
- ASAAS usa IDs de carteira

---

## Fluxo: Configurar Divisão de Receita

### Passo 1: Criar uma Configuração de Split

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/admin/companies/{companyId}/split-configurations` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80,
  "description": "Seller receives 80% of product price on Stripe credit card sales",
  "paymentProvider": "stripe",
  "paymentMethod": "credit_card",
  "productId": "specific-product-uuid",
  "contractAddress": "0x1234...abcd",
  "chainId": 137
}
```

**Resposta (201):**
```json
{
  "id": "split-config-uuid",
  "companyId": "company-uuid",
  "type": "product_price",
  "userId": "seller-user-uuid",
  "percentage": 80,
  "createdAt": "2026-03-31T12:00:00.000Z"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `type` | enum | Sim | Tipo de split: `product_price`, `client_service_fee`, `company_service_fee`, `gas_fee`, `resale_fee` |
| `userId` | uuid | Sim | Usuário destinatário da receita |
| `percentage` | number | Sim | 0-100, percentual do tipo de taxa |
| `description` | string | Não | Descrição administrativa |
| `paymentProvider` | enum | Não | Aplicar apenas a este provedor |
| `paymentMethod` | enum | Não | Aplicar apenas a este método |
| `productId` | uuid | Não | Aplicar apenas a este produto |
| `contractAddress` | string | Não | Aplicar apenas a este contrato |
| `chainId` | integer | Não | Aplicar apenas a esta blockchain |

**Observações:**
- Splits sem filtros (sem `paymentProvider`, `productId`, etc.) se aplicam a **todos** os pedidos
- Filtros restringem o escopo: um split com `paymentProvider: "stripe"` se aplica apenas a pagamentos via Stripe
- Múltiplos splits podem existir para o mesmo tipo — seus percentuais devem somar no máximo 100%
- Se o total de % de split for < 100%, o percentual restante vai para o tesouro da empresa por padrão

---

### Passo 2: Listar Configurações de Split

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| GET | `/admin/companies/{companyId}/split-configurations` | Bearer (Admin) | — |

---

### Passo 3: Atualizar um Split

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/admin/companies/{companyId}/split-configurations/{configId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "percentage": 85,
  "description": "Updated seller share to 85%"
}
```

---

### Passo 4: Excluir um Split

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| DELETE | `/admin/companies/{companyId}/split-configurations/{configId}` | Bearer (Admin) | — |

---

## Como os Splits Funcionam no Momento do Pedido

1. Quando um pedido é concluído, o sistema calcula cada componente de taxa:
   - `currencyAmount` → preço do produto
   - `clientServiceFee` → taxa do comprador
   - `companyServiceFee` → taxa da empresa
   - `gasFee` → gas da blockchain
   - `resaleFee` → taxa de venda secundária

2. Para cada componente, as configurações de split correspondentes são encontradas (filtradas por provedor, método, produto, cadeia)

3. Cada destinatário recebe seu percentual como um `DeferredSplit`

4. Splits diferidos são executados de forma assíncrona:
   ```
   PENDING → PENDING_USER_ACCOUNT → PROCESSING_SPLIT → UNDER_USER_ACCOUNT → PROCESSING_WITHDRAW → WITHDRAWN
   ```

5. Se o destinatário não tiver uma conta de provedor de pagamento, o split aguarda em `PENDING_USER_ACCOUNT`

---

## Exemplo: Configuração Completa de Gateway + Split

```
Cenário: Marketplace com Stripe (cartão de crédito) e ASAAS (PIX) para BRL

1. Configurar Stripe:
   PATCH /admin/.../configurations/providers/stripe
   { "secret": "sk_live_...", "publicKey": "pk_live_..." }

2. Configurar ASAAS:
   PATCH /admin/.../configurations/providers/asaas
   { "apiKey": "aact_..." }

3. Habilitar métodos de pagamento:
   POST /admin/.../configurations/providers-selections
   { "currencyId": "brl-uuid", "paymentProvider": "stripe", "paymentMethod": "credit_card" }

   POST /admin/.../configurations/providers-selections
   { "currencyId": "brl-uuid", "paymentProvider": "asaas", "paymentMethod": "pix" }

4. Configurar splits:
   POST /admin/.../split-configurations
   { "type": "product_price", "userId": "seller-uuid", "percentage": 80 }

   POST /admin/.../split-configurations
   { "type": "product_price", "userId": "platform-uuid", "percentage": 15 }

   // 5% restantes ficam no tesouro da empresa

   POST /admin/.../split-configurations
   { "type": "client_service_fee", "userId": "platform-uuid", "percentage": 100 }
```

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Provedor de pagamento inválido | Provedor não reconhecido ou não suportado | Verifique os provedores válidos: `stripe`, `asaas`, `pagar_me`, `paypal`, `transfer`, `crypto`, `free`, `braza` |
| 400 | Credenciais inválidas | Chaves do Stripe ou chave de API do ASAAS inválidas | Verifique as chaves no painel do provedor |
| 400 | Percentual excede 100 | Total de splits para um tipo excede 100% | Reduza os percentuais; total por tipo deve ser no máximo 100% |
| 400 | Usuário não encontrado | userId do destinatário do split não existe | Crie o usuário em contatos primeiro |
| 404 | Configuração não encontrada | Empresa não possui configuração de commerce | Certifique-se de que o commerce está habilitado: `GET /admin/companies/{companyId}/is-enabled` |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Método de pagamento não aparece na pré-visualização do pedido | Você deve criar uma provider-selection para cada combinação de moeda + provedor + método |
| 2 | Splits não são executados | O usuário destinatário precisa de uma conta de provedor de pagamento configurada (Stripe Connect ou carteira ASAAS) |
| 3 | Split preso em `PENDING_USER_ACCOUNT` | O destinatário não configurou seu provedor de pagamento. Use os endpoints de configuração de provedor do usuário |
| 4 | Provedor errado recebe o split | Splits sem filtro de `paymentProvider` se aplicam a todos os provedores. Adicione um filtro de provedor para restringir |
| 5 | Splits para um produto específico não funcionam | Passe `productId` na configuração de split para definir o escopo. Sem ele, o split se aplica a todos os produtos |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticação | Token admin necessário | [AUTH_SKILL_INDEX](../auth/AUTH_SKILL_INDEX.md) |
| Ciclo de Vida do Produto | Produtos devem existir para splits com escopo de produto | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) |
| Pedido e Compra | Splits são executados após o pagamento do pedido | [FLOW_COMMERCE_ORDER_PURCHASE](./FLOW_COMMERCE_ORDER_PURCHASE.md) |
