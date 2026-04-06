---
id: CONFIGURATIONS_SKILL_INDEX
title: "Índice de Skills de Configurações"
module: offpix/configurations
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
---

# Índice de Skills de Configurações

Documentação para o módulo de Configurações do W3Block. Cobre gerenciamento dinâmico de formulários (contexts, tenant contexts e campos/inputs de formulário) usados para KYC, cadastro, pesquisas e coleta de dados personalizados.

**Serviço:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos entre entidades | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) | 1.0.0 | Criar, atualizar, listar e configurar formulários (contexts) | Implementado | Construir funcionalidades de gerenciamento de formulários |
| 3 | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) | 1.0.0 | Adicionar, atualizar e recuperar campos de formulário (19 tipos de input) | Implementado | Adicionar ou gerenciar campos de formulário |
| 4 | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) | 1.0.0 | Ativar/desativar cadastro de usuários, configurar formulário de cadastro | Implementado | Habilitar/desabilitar registro de usuários |

---

## Guia Rápido

### Para uma implementação básica de formulário:

```
1. Leia: FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md   → Criar um formulário (context)
2. Leia: FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md    → Adicionar campos ao formulário
3. Leia: FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md   → Habilitar cadastro (se aplicável)
4. Consulte: CONFIGURATIONS_API_REFERENCE.md        → Para enums, DTOs, casos extremos
```

### Implementação mínima via API (3 chamadas):

```
1. POST /tenants/{id}/admin/tenant-context    → Criar formulário + configuração do tenant
2. POST /tenants/{id}/tenant-input            → Adicionar campos (repetir por campo)
3. GET  /tenants/{id}/tenant-input/slug/{s}   → Renderizar formulário para usuários (público)
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Validação do slug falha | Deve corresponder a `^[a-z0-9]+(?:-[a-z0-9]+)*$` (minúsculo, sem espaços/underscores) |
| 2 | Endpoint PATCH errado | O PATCH admin atualiza context + config; o PATCH comum atualiza apenas `active` e `data` |
| 3 | Adicionar campo obrigatório dispara revisão de usuários | Planeje a estrutura do formulário antes de criar inputs para evitar notificações em massa |
| 4 | Colisão de `attributeName` | Sempre defina um `attributeName` personalizado ao usar múltiplos campos do mesmo tipo |
| 5 | `type` é imutável após a criação | Exclua e recrie o input se precisar de um tipo diferente |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Criar um novo formulário KYC/pesquisa | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Adicionar campos a um formulário | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Habilitar/desabilitar cadastro de usuários | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) |
| Renderizar um formulário para usuários finais | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) (endpoint público por slug) |
| Listar tipos de formulário e tipos de campo disponíveis | [CONFIGURATIONS_API_REFERENCE](./CONFIGURATIONS_API_REFERENCE.md) (seção de enums) |
| Configurar fluxo de aprovação KYC | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) (schema de data) |
| Construir formulários multi-etapas | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) (campo step) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Ciclo de Vida do Context | Gerenc. de Inputs | Ativação de Cadastro |
|----------|:-------:|:-----------------:|:----------:|:-----------------:|
| POST /contexts | X | | | |
| GET /contexts | X | | | |
| PATCH /contexts/:id | X | | | |
| DELETE /contexts/:id | X | | | |
| POST /tenants/:id/tenant-context | X | X | | |
| GET /tenants/:id/tenant-context | X | X | | X |
| GET /tenants/:id/tenant-context/:id | X | X | | |
| GET /tenants/:id/tenant-context/slug/:slug | X | X | | |
| PATCH /tenants/:id/tenant-context/:id | X | X | | X |
| GET /tenants/:id/tenant-context/:id/approvers | X | X | | |
| POST /tenants/:id/admin/tenant-context | X | X | | X |
| PATCH /tenants/:id/admin/tenant-context/:id | X | X | | |
| GET /tenants/:id/admin/tenant-context/:id/inputs | X | | X | |
| POST /tenants/:id/tenant-input | X | | X | |
| PATCH /tenants/:id/tenant-input/:id | X | | X | |
| GET /tenants/:id/tenant-input/slug/:slug | X | | X | |
| GET /tenants/:id/tenant-input | X | | X | |
| GET /tenants/:id/tenant-input/:id | X | | X | |
| GET /tenant-contexts/activate-signup/:id | | | | X |
| GET /tenant-contexts/deactivate-signup/:id | | | | X |
| GET /data-types/:id | X | | | X |
