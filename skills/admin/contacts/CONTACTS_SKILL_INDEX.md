---
id: CONTACTS_SKILL_INDEX
title: "Índice de Skills de Contatos"
module: offpix/contacts
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Índice de Skills de Contatos

Documentação para o módulo de Contatos do W3Block. Cobre gerenciamento de usuários (convite, edição, roles), fluxos de submissão e aprovação de documentos KYC, elegibilidade para royalties, contatos externos e configuração de provedor de pagamento.

**Serviços:** PixwayID (usuários, KYC), Registry (royalties, contatos externos), Commerce (provedores de pagamento)
**Swagger:** https://pixwayid.w3block.io/docs/ | https://api.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [CONTACTS_API_REFERENCE](./CONTACTS_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos entre entidades | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) | 1.0.0 | Listar, convidar, editar contatos, roles, royalties, contatos externos | Implementado | Gerenciar usuários/contatos |
| 3 | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) | 1.0.0 | Buscar campos do formulário, enviar documentos, multi-etapas, reenvio | Implementado | Construir interface de submissão KYC |
| 4 | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) | 1.0.0 | Listar, revisar, aprovar/rejeitar/solicitar revisão de submissões KYC | Implementado | Construir interface de revisão/admin KYC |

---

## Guia Rápido

### Para uma implementação básica de contatos:

```
1. Leia: FLOW_CONTACTS_USER_MANAGEMENT.md     → Listar/convidar/editar usuários
2. Leia: FLOW_CONTACTS_KYC_SUBMISSION.md      → Construir fluxo de submissão de documentos
3. Leia: FLOW_CONTACTS_KYC_APPROVAL.md        → Construir fluxo de revisão admin
4. Consulte: CONTACTS_API_REFERENCE.md        → Para enums, DTOs, casos extremos
```

### Pré-requisitos de outros módulos:

```
1. Módulo Auth       → Sign-in (obter JWT)
2. Configurações     → Configurar contexts e campos de formulário KYC
3. Contatos          → Gerenciar usuários e fluxos KYC
```

### Implementação mínima via API:

```
1. POST /users/invite                                     → Criar usuário
2. GET  /tenants/{id}/tenant-input/slug/{slug}            → Obter campos do formulário
3. POST /{id}/users/documents/{userId}/context/{ctxId}    → Enviar documentos KYC
4. GET  /customer-infos/{id}/search                       → Listar submissões
5. PATCH /{id}/users/contexts/{userId}/{ctxId}/approve    → Aprovar KYC
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | CPF deve ser único por tenant | Cada CPF só pode ser usado uma vez por tenant |
| 2 | Rejeição é terminal | Use "solicitar revisão" em vez de "rejeitar" se o reenvio for desejado |
| 3 | Multiface selfie requer CPF + data de nascimento | Envie os documentos de CPF e data de nascimento ANTES da selfie |
| 4 | Elegibilidade para royalties é uma chamada de API separada | Convidar um usuário não cria registros de royalty — requer chamada ao backend Registry |
| 5 | Uploads de arquivo precisam do serviço de assets primeiro | Faça upload dos arquivos para obter o `assetId`, depois referencie na submissão do documento |
| 6 | Filtragem de whitelist de KycApprover | Se `approverWhitelistIds` estiver definido, apenas aprovadores na whitelist podem revisar |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Listar todos os clientes/equipe/parceiros | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Convidar um novo usuário | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Editar perfil/roles de usuário | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Enviar documentos KYC | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Construir um formulário KYC multi-etapas | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Lidar com reenvio de documentos | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Listar revisões KYC pendentes | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Aprovar/rejeitar submissões KYC | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Solicitar reenvio de documentos | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Gerenciar elegibilidade para royalties | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Adicionar contatos externos de carteira | [FLOW_CONTACTS_USER_MANAGEMENT](./FLOW_CONTACTS_USER_MANAGEMENT.md) |
| Configurar formulários KYC primeiro | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Gerenc. Usuários | Submissão KYC | Aprovação KYC |
|----------|:-------:|:---------:|:----------:|:-----------:|
| GET /{id}/users | X | X | | |
| GET /{id}/users/{userId} | X | X | | |
| POST /users/invite | X | X | | |
| PATCH /{id}/users/{userId} | X | X | | |
| GET /users/documents/find-user-by-any | X | X | | |
| GET /users/wallets/by-address/{addr} | X | X | | |
| GET /tenant-input/{id}/slug/{slug} | | | X | |
| POST /users/documents/{userId}/context/{ctxId} | X | | X | |
| GET /users/documents/{userId} | X | | X | X |
| GET /users/documents/{userId}/context/{ctxId} | X | | X | X |
| GET /users/contexts/find | X | | | X |
| GET /users/contexts/{userId} | X | | | X |
| GET /users/contexts/{userId}/{ucId} | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/approve | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/reject | X | | | X |
| PATCH /users/contexts/{userId}/{ctxId}/require-review | X | | | X |
| GET /customer-infos/{id}/search | X | | | X |
| POST /{id}/contracts/royalty-eligible/create | X | X | | |
| PATCH /{id}/contracts/royalty-eligible | X | X | | |
| GET /{id}/external-contacts | X | X | | |
| POST /{id}/external-contacts/import | X | X | | |
| PATCH /users/{id}/{userId}/referrer | X | X | | |
