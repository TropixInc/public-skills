---
id: SETTINGS_SKILL_INDEX
title: "Índice de Skills de Configurações e Faturamento"
module: offpix/settings
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Índice de Skills de Configurações e Faturamento

Documentação do módulo de Configurações e Faturamento da W3Block. Cobre seleção de plano de faturamento, gerenciamento de cartão de crédito, histórico de ciclos de cobrança, rastreamento e simulação de uso, controle de funcionalidades (feature-gating), cancelamento de assinatura, gerenciamento de perfil do tenant e configuração do tenant (provedores de autenticação, KYC, passwordless, serviço de e-mail, exclusão de conta).

**Serviço:** Pixway ID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [SETTINGS_API_REFERENCE](./SETTINGS_API_REFERENCE.md) | 1.0.0 | Todos os 16 endpoints, DTOs, enums, relacionamentos de entidades | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) | 1.0.0 | Seleção de plano, cartão de crédito, ciclos de cobrança, resumo de uso, simulação, cancelamento | Implementado | Gerenciar faturamento e assinaturas |
| 3 | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) | 1.0.0 | Perfil do tenant, configurações (provedores de auth, KYC, passwordless, e-mail), verificações de funcionalidades | Implementado | Configurar as definições do tenant |

---

## Guia Rápido

### Para implementação de faturamento:

```
1. Leia: FLOW_SETTINGS_BILLING_MANAGEMENT.md     -> Seleção de plano, pagamentos, ciclos
2. Consulte: SETTINGS_API_REFERENCE.md             -> Para enums, DTOs, casos extremos
```

### Para configuração do tenant:

```
1. Leia: FLOW_SETTINGS_TENANT_CONFIGURATION.md   -> Perfil, provedores de auth, configuração de KYC
2. Consulte: SETTINGS_API_REFERENCE.md             -> Para schemas de configuração
```

### Configuração mínima de faturamento (3 chamadas):

```
1. GET    /billing/plans                            -> Listar planos disponíveis
2. PATCH  /billing/{tenantId}/credit-card           -> Definir método de pagamento
3. PATCH  /billing/{tenantId}/plan                  -> Selecionar um plano
```

### Configuração mínima do tenant (2 chamadas):

```
1. PUT    /tenant/profile/{tenantId}                -> Atualizar perfil do tenant
2. POST   /tenant/configurations/{tenantId}         -> Definir configuração de auth/KYC/e-mail
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Mudança de plano rejeitada | Certifique-se de que um cartão de crédito válido esteja configurado antes de mudar para um plano pago |
| 2 | Verificação de funcionalidade retorna false inesperadamente | Verifique se o plano do tenant inclui a funcionalidade em `enabledFeatures` |
| 3 | Datas do ciclo de cobrança erradas após mudança de plano | As datas do ciclo são redefinidas quando o plano muda; verifique `startCycleDate`/`endCycleDate` |
| 4 | Limites de uso excedidos mas sem erro | Alguns limites são flexíveis (apenas rastreados); verifique os campos `hard*` para limites aplicados |
| 5 | Atualização de configuração parece não ter efeito | Configurações do tenant usam JSONB; atualizações parciais podem exigir o envio do objeto completo |
| 6 | Não é possível cancelar a assinatura | Apenas o papel Admin pode cancelar; certifique-se da autorização adequada |
| 7 | Incompatibilidade de moeda no faturamento | Verifique `BillingCurrencyExchangeRate` para taxas de conversão corretas em USD |
| 8 | Limites personalizados não sendo aplicados | `customLimits` no `TenantPlan` são sobrescrições parciais; campos não definidos voltam aos padrões do plano |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Listar planos de faturamento disponíveis | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Configurar um cartão de crédito para faturamento | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Mudar o plano do meu tenant | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Visualizar histórico de ciclos de cobrança | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Verificar uso e custos | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Simular o custo de um evento de faturamento | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Cancelar uma assinatura | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Registrar um evento de uso (integração) | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Acionar cobrança/fatura manual | [FLOW_SETTINGS_BILLING_MANAGEMENT](./FLOW_SETTINGS_BILLING_MANAGEMENT.md) |
| Verificar se uma funcionalidade está habilitada | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Atualizar perfil do tenant | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Configurar provedores de auth (Google, Apple) | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Habilitar autenticação passwordless | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Configurar definições de KYC | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Configurar serviço de e-mail | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |
| Obter detalhes do tenant | [FLOW_SETTINGS_TENANT_CONFIGURATION](./FLOW_SETTINGS_TENANT_CONFIGURATION.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Gerenc. Faturamento | Config. Tenant |
|----------|:--------:|:-------------------:|:--------------:|
| GET /billing/plans | X | X | |
| POST /billing/{tenantId}/register-billing-usage | X | X | |
| GET /billing/{tenantId}/is-feature-enabled/{feature} | X | | X |
| GET /billing/{tenantId}/state | X | X | |
| PATCH /billing/{tenantId}/credit-card | X | X | |
| PATCH /billing/{tenantId}/plan | X | X | |
| PATCH /billing/{tenantId}/cancel | X | X | |
| GET /billing/{tenantId}/cycles | X | X | |
| GET /billing/{tenantId}/billing-summary | X | X | |
| GET /billing/{tenantId}/simulate-billing-event-usage | X | X | |
| GET /billing/{tenantId}/billing-usages | X | X | |
| PATCH /billing/{tenantId}/invoice-and-charge | X | X | |
| POST /tenant/configurations/{tenantId} | X | | X |
| GET /tenant/configurations/{tenantId} | X | | X |
| PUT /tenant/profile/{tenantId} | X | | X |
| GET /tenant/{tenantId} | X | | X |
