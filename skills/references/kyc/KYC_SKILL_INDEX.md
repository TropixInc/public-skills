---
id: KYC_SKILL_INDEX
title: "Índice de Skills KYC"
module: offpix/kyc
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# Índice de Skills KYC

Documentação para o módulo KYC (Know Your Customer) da W3Block. Cobre fluxos de verificação de documentos incluindo submissão, aprovação, rejeição, solicitações de revisão, validação de tipos de entrada e configuração de KYC em nível de tenant.

**Serviço:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

> **Nota:** Fluxos básicos de KYC também são referenciados no [módulo de Contatos](../contacts/CONTACTS_SKILL_INDEX.md). Este módulo dedicado fornece cobertura mais aprofundada de regras de validação, tipos de entrada, fluxos de aprovação e opções de configuração.

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|----------|--------|-----------|--------|-------------|
| 1 | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos entre entidades | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) | 1.0.0 | Buscar formulário, fazer upload de arquivos, submeter documentos, multi-etapas, reenvio | Implementado | Construir UI de submissão KYC |
| 3 | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) | 1.0.0 | Listar pendentes, revisar, aprovar/rejeitar/solicitar revisão, trilha de auditoria | Implementado | Construir UI de revisão KYC do admin |
| 4 | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) | 1.0.0 | Configuração de tenant context, aprovação automática, whitelists de aprovadores, notificações | Implementado | Configurar KYC para um tenant |

---

## Guia Rápido

### Para uma implementação completa de KYC:

```
1. Leia: FLOW_KYC_CONFIGURATION.md      → Configurar tenant contexts, inputs, regras de aprovação
2. Leia: FLOW_KYC_SUBMISSION.md         → Construir o fluxo de submissão de documentos para o usuário
3. Leia: FLOW_KYC_APPROVAL.md           → Construir o fluxo de revisão e aprovação do admin
4. Consulte: KYC_API_REFERENCE.md       → Para enums, DTOs, regras de validação, casos extremos
```

### Pré-requisitos de outros módulos:

```
1. Módulo Auth          → Login (obter JWT)
2. Configurações        → Configurar contexts e campos de formulário KYC (tenant-input, tenant-context)
3. Contatos (opcional)  → Convite / gerenciamento de usuários
```

### Implementação mínima via API:

```
1. GET  /tenant-input/{tenantId}/slug/{slug}                  → Obter definição do formulário
2. POST /cloudinary/get-signature                              → Upload de arquivos (se necessário)
3. POST /{tenantId}/users/documents/{userId}/context/{ctxId}   → Submeter documentos KYC
4. GET  /{tenantId}/users/contexts/find?status[]=created       → Listar submissões pendentes
5. PATCH /{tenantId}/users/contexts/{userId}/{ctxId}/approve   → Aprovar submissão
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | CPF deve ser único por tenant | Cada valor de CPF pode ser usado por apenas um usuário por tenant. Submissão de CPF duplicado retorna `CPfDuplicatedException` |
| 2 | Verificação de email obrigatória | Usuários devem ter email verificado antes de submeter documentos KYC (a menos que passwordless esteja habilitado) |
| 3 | Selfie multiface requer CPF + data de nascimento | O serviço de validação biométrica requer que documentos de CPF e data de nascimento sejam submetidos ANTES da selfie |
| 4 | Limite de `maxSubmissions` | Cada context pode definir um número máximo de submissões. Uma vez atingido, `MaxSubmissionReachedException` é lançado |
| 5 | DENIED é um estado terminal | A rejeição define o context como `DENIED` permanentemente. Use "solicitar revisão" em vez disso se o reenvio for desejado |
| 6 | Upload de arquivos precisa do serviço de assets primeiro | Faça upload de arquivos para o Cloudinary via serviço de assets para obter um `assetId`, depois referencie-o na submissão de documento |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Entender todos os endpoints e DTOs de KYC | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
| Submeter documentos KYC para um usuário | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Construir um formulário KYC multi-etapas | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Lidar com reenvio de documentos após revisão | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Listar submissões KYC pendentes | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Aprovar, rejeitar ou solicitar revisão | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Entender a trilha de auditoria | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Configurar aprovação automática ou aprovador específico | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configurar whitelists de aprovadores | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configurar notificações de email KYC | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Entender todos os 19 tipos de entrada | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Validar um número de CPF | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
| Bloquear pedidos de commerce até KYC aprovado | [FLOW_KYC_CONFIGURATION](./FLOW_KYC_CONFIGURATION.md) |
| Configurar campos de formulário KYC (inputs) | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Criar um context KYC | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref API | Submissão | Aprovação | Configuração |
|----------|:-------:|:---------:|:---------:|:------------:|
| GET /tenant-input/{id}/slug/{slug} | | X | | X |
| POST /cloudinary/get-signature | | X | | |
| POST /{id}/users/documents/{userId}/context/{ctxId} | X | X | | |
| GET /{id}/users/documents/{userId} | X | X | X | |
| GET /{id}/users/documents/{userId}/context/{ctxId} | X | X | X | |
| GET /documents/find-user-by-any | X | | X | |
| GET /{id}/users/contexts/find | X | | X | |
| GET /{id}/users/contexts/{userId} | X | | X | |
| GET /{id}/users/contexts/{userId}/{ucId} | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/approve | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/reject | X | | X | |
| PATCH /{id}/users/contexts/{userId}/{ctxId}/require-review | X | | X | |
| GET /tenant-context/{id} | X | | | X |
