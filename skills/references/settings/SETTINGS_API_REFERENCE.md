---
id: SETTINGS_API_REFERENCE
title: "Referência da API de Configurações e Faturamento"
module: offpix/settings
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - settings
  - billing
  - tenant
  - api-reference
depends_on:
  - SETTINGS_SKILL_INDEX
---

# Referência da API de Configurações e Faturamento

Referência completa da API para o módulo de Configurações e Faturamento da W3Block. Cobre 16 endpoints divididos entre gerenciamento de faturamento (12) e configurações do tenant (4), além de todos os DTOs, enums e relacionamentos entre entidades.

## URLs Base

| Ambiente | Serviço | URL |
|----------|---------|-----|
| Produção | Pixway ID | https://pixwayid.w3block.io |
| Swagger | Pixway ID | https://pixwayid.w3block.io/docs/ |

## Autenticação

Todos os endpoints autenticados requerem:
```
Authorization: Bearer {accessToken}
```

Os requisitos de papel são indicados por endpoint. Papéis: **SuperAdmin**, **Admin**, **Operator**, **User**, **Integration** (chave API).

---

## Entidades e Relacionamentos

```
BillingPlanEntity (1) ----> (N) TenantPlanEntity ----> (1) TenantEntity
       |                            |                         |
       |                            |                    (1)--+--(1)
       |                            |                         |
       |                      (N) TenantBillingLogEntity   TenantConfigurationsEntity
       |                            |
       |                      (N) TenantPlanLogEntity
       |
       +-- BillingFeature[] (enabledFeatures)
       +-- BillingPlanLimit (por recurso)

BillingUsageEntity (N) ----> (1) TenantEntity
BillingCurrencyExchangeRateEntity (standalone)
TenantClientEntity (N) ----> (1) TenantEntity
```

### BillingPlanEntity

Definição mestre do plano. Contém preços, limites, funcionalidades e configuração de exibição.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | Chave primária |
| `name` | string | Nome de exibição do plano |
| `basePrice` | number | Preço base mensal em USD |
| `isDefault` | boolean | Se este é o plano padrão para novos tenants |
| `isPublic` | boolean | Se o plano é visível em listagens públicas |
| `limits.nftContracts` | BillingPlanLimit | Limites de criação de contratos NFT |
| `limits.nftCollections` | BillingPlanLimit | Limites de coleções NFT |
| `limits.nftTokens` | BillingPlanLimit | Limites de tokens NFT |
| `limits.tokensPerCollection` | BillingPlanLimit | Limites de tokens por coleção |
| `limits.nftMints` | BillingPlanLimit | Limites de minting NFT |
| `limits.erc20Contracts` | BillingPlanLimit | Limites de contratos ERC20 |
| `limits.erc20Mints` | BillingPlanLimit | Limites de minting ERC20 |
| `pricing.extraNftMintPrice` | number | Custo por mint NFT adicional além do limite do plano |
| `pricing.nftTransactionPrice` | number | Custo por transação NFT |
| `pricing.erc20TransactionPrice` | number | Custo por transação ERC20 |
| `pricing.nftSaleTransactionPrice` | number | Custo por transação de venda NFT |
| `pricing.erc20SaleTransactionPrice` | number | Custo por transação de venda ERC20 |
| `fees.primarySaleFee.minValue` | number | Taxa mínima em vendas primárias |
| `fees.primarySaleFee.percentage` | number | Taxa percentual em vendas primárias |
| `fees.resaleFee` | number | Taxa em revendas |
| `resourceLimits.activeWallets` | number | Máximo de carteiras ativas |
| `resourceLimits.onSaleProducts` | number | Máximo de produtos à venda |
| `resourceLimits.storage` | number | Limite de armazenamento (bytes) |
| `resourceLimits.bandwidth` | number | Limite de largura de banda (bytes) |
| `resourceLimits.maxUploadFileSizes.image` | number | Tamanho máximo de upload de imagem (bytes) |
| `resourceLimits.maxUploadFileSizes.video` | number | Tamanho máximo de upload de vídeo (bytes) |
| `resourceLimits.maxUploadFileSizes.others` | number | Tamanho máximo de upload de outros arquivos (bytes) |
| `userLimits.users` | number | Máximo de usuários |
| `userLimits.maxClients` | number | Máximo de clientes OAuth |
| `userLimits.maxLoyaltyPartners` | number | Máximo de parceiros de fidelidade |
| `features.enabledFeatures` | BillingFeature[] | Lista de funcionalidades habilitadas |
| `display.homeSortOrder` | number | Ordem na página de planos |
| `display.homeContent` | object | Conteúdo exibido na página de planos |
| `trial.testDays` | number | Período de teste em dias |

### BillingPlanLimit

Estrutura de limite usada para cada recurso medido.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `daily` | number ou null | Limite suave diário |
| `weekly` | number ou null | Limite suave semanal |
| `monthly` | number ou null | Limite suave mensal |
| `lifetime` | number ou null | Limite suave vitalício |
| `hardPerDay` | number ou null | Limite rígido (imposto) diário |
| `hardPerWeek` | number ou null | Limite rígido (imposto) semanal |
| `hardPerMonth` | number ou null | Limite rígido (imposto) mensal |
| `hardLifetime` | number ou null | Limite rígido (imposto) vitalício |

> **Limites suaves vs. rígidos:** Limites suaves são rastreados para faturamento/relatórios. Limites rígidos são impostos e rejeitam operações quando excedidos.

### TenantPlanEntity

Vincula um tenant ao seu plano de faturamento com overrides customizados opcionais.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `tenantId` | UUID | Identificador do tenant (único) |
| `planId` | UUID | Referência ao BillingPlanEntity |
| `customLimits` | Partial&lt;BillingPlanLimits&gt; | Overrides opcionais dos limites do plano |
| `startCycleDate` | Date | Início do ciclo de faturamento atual |
| `endCycleDate` | Date | Fim do ciclo de faturamento atual |
| `primaryCreditCardId` | string | Cartão de crédito padrão para cobranças |
| `coupon` | string | Código de cupom aplicado |

### BillingUsageEntity

Rastreia uso medido por tenant.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `tenantId` | UUID | Identificador do tenant |
| `type` | string | Tipo de uso (ver BillingUsageKnownTypes) |
| `isLifeTime` | boolean | Se é um contador vitalício |
| `value` | number | Contagem de uso atual |
| `metadata` | object | Metadados adicionais de uso |

> **Índice único:** `(tenantId, type, isLifeTime)` — um registro por tenant por tipo de uso por flag vitalício.

### TenantBillingLogEntity

Registro imutável de cada cobrança de ciclo de faturamento.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `tenantId` | UUID | Identificador do tenant |
| `startCycleDate` | Date | Data de início do ciclo |
| `endCycleDate` | Date | Data de fim do ciclo |
| `paymentDate` | Date | Quando o pagamento foi processado |
| `creditCardId` | string | Cartão usado para pagamento |
| `cyclePrice` | number | Preço calculado do ciclo |
| `commerceOrderId` | string | Pedido de commerce associado |
| `paidValue` | number | Valor efetivamente cobrado |
| `paymentTries` | number | Número de tentativas de cobrança |
| `planId` | UUID | Plano no momento da cobrança |
| `customLimits` | object | Limites customizados no momento da cobrança |
| `finalReport` | BillingEventSummary[] | Detalhamento de eventos de uso |

### TenantPlanLogEntity

Trilha de auditoria para mudanças de plano.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `tenantId` | UUID | Identificador do tenant |
| `oldPlanId` | UUID | Plano anterior |
| `newPlanId` | UUID | Novo plano |
| `userId` | UUID | Usuário que fez a mudança |

### BillingCurrencyExchangeRateEntity

Taxas de conversão de moeda para faturamento multi-moeda.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `currencyId` | string | Identificador da moeda |
| `currencyCode` | string | Código ISO da moeda |
| `usdExchangeRate` | number | Taxa de câmbio fixa em USD |
| `dynamicRate` | number | Taxa de câmbio dinâmica (ao vivo) |
| `dynamicCurrencyCode` | string | Código da moeda para taxa dinâmica |

### TenantEntity

Registro principal do tenant.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | UUID | Chave primária |
| `name` | string | Nome de exibição do tenant |
| `document` | string | Documento empresarial (CNPJ, EIN, etc.) |
| `countryCode` | string | Código de país ISO |
| `wallets` | object[] | Carteiras blockchain associadas |
| `clientId` | string | Identificador do cliente OAuth |
| `info` | object | Informações adicionais do tenant |
| `roles` | string[] | Papéis em nível de tenant |
| `operatorAddress` | string | Endereço de operador blockchain |
| `setupDone` | boolean | Se a configuração inicial foi concluída |

### TenantConfigurationsEntity

Configurações de funcionalidades e integração específicas do tenant.

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `tenantId` | UUID | Identificador do tenant (único) |
| `kyc` | JSONB | Configuração do provedor KYC |
| `signUp` | JSONB | Configuração do fluxo de cadastro |
| `passwordless` | object | `{ enabled: boolean }` |
| `googleOAuth` | object | Configuração do provedor Google OAuth |
| `appleOAuth` | object | Configuração do provedor Apple OAuth |
| `oneSignal` | object | Configuração de push notification OneSignal |
| `twilioWhatsApp` | object | Configuração de mensagens Twilio WhatsApp |
| `clearSale` | object | Configuração de detecção de fraude ClearSale |
| `emailService` | object | Configuração do provedor de serviço de email |
| `automaticAccountExclusion` | boolean | Excluir automaticamente contas inativas |

---

## Enums

### BillingFeature (23 valores)

Funcionalidades que podem ser habilitadas ou desabilitadas por plano.

| Valor | Descrição |
|-------|-----------|
| `MINT_ERC_721` | Mintar tokens ERC-721 (NFT) |
| `MINT_ERC_20` | Mintar tokens ERC-20 (fungíveis) |
| `MINT_WITH_ROYALTY` | Mintar com configuração de royalty |
| `MINT_WITH_VARIABLE_ROYALTIES` | Mintar com porcentagens de royalty variáveis |
| `MINT_PERMISSIONED_ERC_20` | Mintar tokens ERC-20 permissionados |
| `MINT_CUSTODIAL_ERC_20` | Mintar tokens ERC-20 custodiais |
| `BATCH_MINT` | Operações de minting em lote |
| `CUSTOM_NFT_TEMPLATES` | Templates de exibição NFT customizados |
| `TRANSFER_NFT` | Transferir tokens NFT entre carteiras |
| `WHITE_LABEL` | Marca white-label |
| `EXTERNAL_ECOMMERCE_INTEGRATION` | Integração com plataforma de e-commerce externa |
| `SMART_CONTRACT_PORTABILITY` | Exportar/importar smart contracts |
| `CUSTODIAL_WALLET` | Gerenciamento de carteira custodial |
| `CUSTODIAL_WALLET_PORTABILITY` | Exportar chaves de carteira custodial |
| `BASIC_TOKEN_PASS` | Token pass básico (controle de acesso) |
| `FULL_TOKEN_PASS` | Token pass completo (benefícios avançados) |
| `LOYALTY` | Funcionalidades de programa de fidelidade |
| `KYC` | Verificação Know Your Customer |
| `COMMERCE` | Funcionalidades de commerce/vitrine |
| `CONSTRUCTOR` | Construtor/builder visual |
| `CUSTOM_DOMAIN` | Configuração de domínio customizado |

### BillingUsageKnownTypes (11 valores)

Tipos de eventos de uso medidos.

| Valor | Descrição |
|-------|-----------|
| `COMMERCE_PRODUCT_PURCHASE` | Compra de produto no commerce |
| `COMMERCE_PRODUCT_PUBLISHED` | Produto publicado na vitrine |
| `KEY_NFT_MINTED` | Token NFT mintado |
| `KEY_ERC20_MINTED` | Token ERC20 mintado |
| `KEY_NFT_COLLECTION_CREATED` | Coleção NFT criada |
| `KEY_NFT_CONTRACT_CREATED` | Contrato NFT implantado |
| `KEY_ERC20_CONTRACT_CREATED` | Contrato ERC20 implantado |
| `NFT_SALE_TRANSACTION` | Venda NFT concluída |
| `ERC20_SALE_TRANSACTION` | Venda ERC20 concluída |
| `NFT_TRANSACTION` | Transação NFT genérica (transferência, etc.) |
| `ERC20_TRANSACTION` | Transação ERC20 genérica (transferência, etc.) |

---

## Endpoints

### Endpoints de Faturamento (12)

#### 1. Listar Planos de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/plans` | None (Público) | Qualquer |

Retorna uma lista paginada de planos de faturamento disponíveis.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | number | Não | Número da página (padrão: 1) |
| `limit` | number | Não | Itens por página (padrão: 10) |
| `isPublic` | boolean | Não | Filtrar por visibilidade pública |

**Resposta (200):** Lista paginada de planos com limites, preços, taxas, limites de recursos, limites de usuário, funcionalidades e configuração de exibição.

---

#### 2. Registrar Uso de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| POST | `/billing/{tenantId}/register-billing-usage` | Bearer token | SuperAdmin, Integration |

Registra um evento de uso medido no faturamento do tenant.

**Requisição Mínima:**
```json
{
  "type": "KEY_NFT_MINTED"
}
```

**Requisição Completa:**
```json
{
  "type": "KEY_NFT_MINTED",
  "value": 5,
  "metadata": {
    "contractId": "contract-uuid",
    "collectionId": "collection-uuid"
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `type` | BillingUsageKnownTypes | Sim | Tipo de evento de uso |
| `value` | number | Não | Número de unidades (padrão: 1) |
| `metadata` | object | Não | Metadados contextuais |

**Resposta (201):** Corpo vazio em caso de sucesso.

---

#### 3. Verificar Funcionalidade Habilitada

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/is-feature-enabled/{feature}` | Bearer token | Admin, User, Operator |

Verifica se uma funcionalidade de faturamento específica está habilitada para o plano atual do tenant.

**Resposta (200):**
```json
{ "enabled": true }
```

---

#### 4. Obter Estado de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/state` | Bearer token | Admin |

Retorna o estado completo de faturamento de um tenant, incluindo plano atual, limites, uso e informações do ciclo.

---

#### 5. Definir Cartão de Crédito

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| PATCH | `/billing/{tenantId}/credit-card` | Bearer token | Admin |

Define ou atualiza o cartão de crédito principal para faturamento.

**Requisição:**
```json
{ "creditCardId": "card_abc123" }
```

---

#### 6. Mudar Plano

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| PATCH | `/billing/{tenantId}/plan` | Bearer token | Admin |

Muda o plano de faturamento do tenant. Redefine as datas do ciclo de faturamento.

**Requisição:**
```json
{
  "planId": "new-plan-uuid",
  "coupon": "PROMO2026"
}
```

**Notas:**
- Cria um registro de auditoria TenantPlanLogEntity (oldPlanId, newPlanId, userId)
- Redefine `startCycleDate` e `endCycleDate` para o novo ciclo
- Um cartão de crédito válido deve estar definido para planos pagos

---

#### 7. Cancelar Assinatura

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| PATCH | `/billing/{tenantId}/cancel` | Bearer token | Admin |

Cancela a assinatura atual do tenant, revertendo para o plano padrão.

**Notas:**
- O tenant reverte para o plano padrão (gratuito)
- Cria uma entrada no log de auditoria
- Funcionalidades ativas do plano cancelado são desabilitadas imediatamente

---

#### 8. Obter Ciclos de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/cycles` | Bearer token | Admin |

Retorna o histórico de ciclos de faturamento com detalhes de cobrança.

---

#### 9. Obter Resumo de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/billing-summary` | Bearer token | Admin |

Retorna um resumo de uso e custos para o ciclo de faturamento atual.

---

#### 10. Simular Uso de Evento de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/simulate-billing-event-usage` | Bearer token | Admin |

Simula o custo de um evento de faturamento sem realmente registrá-lo. Útil para mostrar aos usuários os custos projetados antes de realizarem uma ação.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `type` | BillingUsageKnownTypes | Sim | Tipo de evento de uso a simular |
| `value` | number | Não | Número de unidades a simular (padrão: 1) |

---

#### 11. Listar Usos de Faturamento

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/billing/{tenantId}/billing-usages` | Bearer token | Admin |

Retorna todos os registros de uso do tenant.

---

#### 12. Faturar e Cobrar

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| PATCH | `/billing/{tenantId}/invoice-and-charge` | Bearer token | SuperAdmin |

Aciona manualmente faturamento e cobrança para o ciclo de faturamento atual do tenant. Tipicamente usado por administradores da plataforma para correções manuais de faturamento.

**Notas:**
- Cria um TenantBillingLogEntity com `finalReport`
- Cobra no cartão de crédito principal do tenant
- Incrementa `paymentTries` em caso de falha
- Apenas SuperAdmin -- não disponível para admins do tenant

---

### Endpoints de Configurações do Tenant (4)

#### 13. Criar/Atualizar Configuração do Tenant

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| POST | `/tenant/configurations/{tenantId}` | Bearer token | Admin |

Cria ou atualiza a configuração do tenant. Usa comportamento upsert (cria se não existir, atualiza se existir).

**Requisição Mínima:**
```json
{
  "passwordless": { "enabled": true }
}
```

**Requisição Completa:**
```json
{
  "kyc": { "provider": "clear_sale", "enabled": true, "requiredForPurchase": false },
  "signUp": { "requireEmailVerification": true, "allowedDomains": ["example.com"] },
  "passwordless": { "enabled": true },
  "googleOAuth": { "clientId": "google-client-id", "clientSecret": "google-client-secret", "enabled": true },
  "appleOAuth": { "clientId": "apple-client-id", "teamId": "apple-team-id", "keyId": "apple-key-id", "enabled": true },
  "oneSignal": { "appId": "onesignal-app-id", "apiKey": "onesignal-api-key" },
  "twilioWhatsApp": { "accountSid": "twilio-sid", "authToken": "twilio-token", "fromNumber": "+15551234567" },
  "clearSale": { "enabled": true, "apiKey": "clearsale-api-key" },
  "emailService": { "provider": "sendgrid", "apiKey": "sendgrid-api-key", "fromEmail": "noreply@example.com" },
  "automaticAccountExclusion": false
}
```

**Resposta (201):** TenantConfigurationsEntity criado/atualizado.

---

#### 14. Obter Configuração do Tenant

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/tenant/configurations/{tenantId}` | Bearer token | Admin |

Obtém as configurações atuais do tenant.

---

#### 15. Atualizar Perfil do Tenant

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| PUT | `/tenant/profile/{tenantId}` | Bearer token | Admin |

Atualiza as informações de perfil do tenant (nome, documento, país).

**Requisição:**
```json
{
  "name": "My Company LLC",
  "document": "12345678000190",
  "countryCode": "BR",
  "info": { "website": "https://example.com", "industry": "retail" }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do tenant |
| `document` | string | Não | Número de registro empresarial (CNPJ, EIN, etc.) |
| `countryCode` | string | Não | Código de país ISO 3166-1 alpha-2 |
| `info` | object | Não | Metadados adicionais do tenant |

---

#### 16. Obter Detalhes do Tenant

| Método | Caminho | Auth | Papel |
|--------|---------|------|-------|
| GET | `/tenant/{tenantId}` | Bearer token | Admin |

Retorna detalhes completos do tenant incluindo carteiras, papéis e status de configuração.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | VALIDATION_ERROR | Campos obrigatórios ausentes ou inválidos no corpo da requisição | Verificar campos obrigatórios e tipos |
| 401 | UNAUTHORIZED | Bearer token ausente ou expirado | Re-autenticar |
| 403 | FORBIDDEN | Papel insuficiente para o endpoint | Verificar se o papel do usuário corresponde ao requisito do endpoint |
| 404 | NOT_FOUND | Tenant ou plano não encontrado | Verificar se tenantId/planId existem |
| 409 | CONFLICT | Configuração ou atribuição de plano duplicada | Verificar estado existente antes de criar |
| 422 | UNPROCESSABLE_ENTITY | Violação de regra de negócio (ex: sem cartão de crédito para plano pago) | Ler mensagem de erro para regra específica |

---

## Documentos Relacionados

| Documento | Relacionamento |
|----------|---------------|
| [SETTINGS_SKILL_INDEX](./SETTINGS_SKILL_INDEX.md) | Ponto de entrada do módulo |
| [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) | Guia do fluxo de faturamento |
| [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) | Guia do fluxo de configuração |
| [AUTH_API_REFERENCE](../auth/AUTH_API_REFERENCE.md) | Autenticação (obrigatória para todos os endpoints autenticados) |
