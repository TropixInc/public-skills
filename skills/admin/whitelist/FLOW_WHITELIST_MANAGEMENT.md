---
id: FLOW_WHITELIST_MANAGEMENT
title: "Whitelist - Gerenciamento (Criar, Entradas, Verificacao de Acesso, On-Chain)"
module: offpix/whitelist
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - whitelist
  - user-groups
  - entries
  - access-control
  - on-chain
depends_on:
  - WHITELIST_API_REFERENCE
---

# Gerenciamento de Whitelist

## Visao Geral

Este fluxo cobre o ciclo de vida completo de whitelists (chamadas de **"Grupos de Usuarios"** no frontend): criacao de uma whitelist, adicao de entradas de varios tipos (e-mail, carteira, ID de usuario, detentor de contrato), listagem e filtragem de entradas, verificacao de acesso de usuarios e opcionalmente promover uma whitelist on-chain. Whitelists sao usadas para controlar o acesso a produtos, promocoes e operacoes blockchain dentro de um tenant.

## Pre-requisitos

| Requisito | Descricao | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticacao / configuracao do ambiente |

---

## Fluxo: Criar uma Whitelist

### Passo 1: Criar a Whitelist

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| POST | `/whitelists/{tenantId}` | Bearer (Admin) |

**Requisicao Minima:**

```json
{
  "name": "VIP Members"
}
```

**Resposta (201):**

```json
{
  "id": "a1b2c3d4-5678-9abc-def0-123456789abc",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z"
}
```

**Observacoes:**
- O frontend redireciona para a pagina de edicao apos a criacao, usando o `id` retornado
- Apenas o campo `name` e aceito; `tenantId` e extraido do caminho da URL

---

## Fluxo: Editar uma Whitelist

### Passo 1: Obter Whitelist Atual

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

**Resposta (200):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z",
  "walletGroups": []
}
```

### Passo 2: Atualizar o Nome

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "name": "Updated VIP Members"
}
```

**Resposta (200):** Retorna o objeto whitelist atualizado.

---

## Fluxo: Adicionar Entradas a uma Whitelist

Apos criar uma whitelist, adicione entradas para definir quem tem acesso. O frontend apresenta quatro opcoes de tipo de entrada (com uma quinta, "Collection NFT", atualmente desabilitada).

### Passo 1: Adicionar uma Entrada

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| POST | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) |

Escolha o payload apropriado com base no tipo de entrada:

**Entrada de e-mail (minima):**

```json
{
  "type": "email",
  "value": "user@example.com"
}
```

**Entrada de endereco de carteira:**

```json
{
  "type": "wallet_address",
  "value": "0xd3304183ec1fa687e380b67419875f97f1db05f5"
}
```

**Entrada de ID de usuario:**

```json
{
  "type": "user_id",
  "value": "user-uuid-here"
}
```

**Entrada de detentor de colecao (completa com additionalData):**

```json
{
  "type": "collection_holder",
  "value": "0xd3304183ec1fa687e380b67419875f97f1db05f5",
  "additionalData": {
    "chainId": 137,
    "startTokenId": 1,
    "endTokenId": 100,
    "metadataRequirements": [
      { "trait_type": "rarity", "value": "legendary" }
    ]
  }
}
```

**Entrada de detentor de colecao KEY:**

```json
{
  "type": "key_collection_holder",
  "value": "collection-uuid",
  "additionalData": {
    "amount": 1,
    "metadataRequirements": [
      { "trait_type": "tier", "value": "gold" }
    ]
  }
}
```

**Entrada de detentor de ERC-20 KEY:**

```json
{
  "type": "key_erc20_holder",
  "value": "erc20-contract-uuid",
  "additionalData": {
    "amount": 100
  }
}
```

**Entrada de contexto aprovado por KYC:**

```json
{
  "type": "kyc_approved_context",
  "value": "context-uuid"
}
```

**Resposta (201):**

```json
{
  "id": "entry-uuid",
  "whitelistId": "a1b2c3d4-...",
  "type": "email",
  "value": "user@example.com",
  "additionalData": null,
  "createdAt": "2026-04-01T12:00:00.000Z",
  "updatedAt": "2026-04-01T12:00:00.000Z"
}
```

**Referencia de Campos:**

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `type` | WhitelistEntryType | Sim | Um de: `email`, `wallet_address`, `user_id`, `collection_holder`, `kyc_approved_context`, `key_collection_holder`, `key_erc20_holder` |
| `value` | string | Sim | O identificador -- validado por tipo. Convertido para minusculas ao salvar |
| `additionalData` | object | Condicional | Obrigatorio para `collection_holder` (deve incluir `chainId`), opcional para `key_collection_holder` e `key_erc20_holder` |

**Observacoes:**
- Para o tipo `user_id`, o backend verifica se o usuario existe e pertence ao mesmo tenant antes de salvar
- Entradas duplicadas (mesma whitelist + type + value) sao rejeitadas com um erro de "duplicate key", exceto para `collection_holder` e `key_collection_holder` que permitem duplicatas (para diferentes configuracoes de additionalData)
- O frontend invalida o cache da lista de entradas em caso de sucesso

---

## Fluxo: Listar e Filtrar Entradas

### Passo 1: Buscar Entradas com Filtros

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) |

**Query minima:**

```
GET /whitelists/{tenantId}/{id}/entries?page=1&limit=10
```

**Query completa (com filtros):**

```
GET /whitelists/{tenantId}/{id}/entries?page=1&limit=10&search=user@example&type=email&type=wallet_address&sortBy=createdAt&orderBy=DESC&showWallets=true
```

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `search` | string | Busca pelo valor da entrada. Para buscas >= 5 caracteres, tambem pesquisa usuarios por nome/e-mail e inclui entradas `user_id` correspondentes |
| `type` | string[] | Filtrar por um ou mais tipos de entrada. Passe multiplos parametros `type` para filtrar por varios tipos |
| `sortBy` | string | `createdAt` (padrao) ou `updatedAt` |
| `showWallets` | boolean | Quando `true`, resolve enderecos de carteira para entradas `email` e `user_id` |

**Observacoes:**
- O frontend usa um componente `MultipleSelect` para filtrar por tipo e um campo de busca textual
- O comportamento de busca e aprimorado: para consultas de 5+ caracteres, tambem pesquisa a tabela de usuarios e corresponde entradas `user_id` para usuarios encontrados

---

## Fluxo: Excluir uma Entrada

### Passo 1: Remover a Entrada

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| DELETE | `/whitelists/{tenantId}/{id}/entries/{entryId}` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta:** 204 No Content

**Observacoes:**
- Esta e uma exclusao logica (soft delete)
- Se a whitelist foi promovida on-chain (possui wallet groups), os enderecos de carteira resolvidos da entrada sao removidos dos wallet groups associados como parte da mesma transacao
- O frontend exibe um modal de confirmacao antes de excluir

---

## Fluxo: Excluir uma Whitelist

### Passo 1: Excluir a Whitelist

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| DELETE | `/whitelists/{tenantId}/{id}` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta:** 204 No Content

**Observacoes:**
- Esta e uma exclusao logica (soft delete) realizada dentro de uma transacao de banco de dados
- O frontend invalida o cache da lista de whitelists em caso de sucesso

---

## Fluxo: Verificar Acesso do Usuario

### Verificacao em Whitelist Unica

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/whitelists/{tenantId}/{id}/check-user?userId={userId}` | Bearer (Admin / Self) |

**Resposta (200):**

```json
{
  "whitelistId": "a1b2c3d4-...",
  "userId": "user-uuid",
  "hasAccess": true
}
```

### Verificacao em Multiplas Whitelists

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/whitelists/{tenantId}/check-user?userId={userId}&whitelistsIds={id1}&whitelistsIds={id2}` | Bearer (Admin / Self) |

**Resposta (200):**

```json
{
  "hasAccess": true,
  "details": [
    { "whitelistId": "id1", "userId": "user-uuid", "hasAccess": true },
    { "whitelistId": "id2", "userId": "user-uuid", "hasAccess": false }
  ]
}
```

**Como o acesso e avaliado:**

A verificacao avalia multiplos tipos de entrada nesta ordem:

1. **Correspondencias diretas:** Busca entradas onde o ID, e-mail ou enderecos de carteira do usuario correspondem a entradas `user_id`, `email` ou `wallet_address`. Tambem verifica entradas `kyc_approved_context` contra os contextos KYC aprovados do usuario.
2. **Correspondencias baseadas em detencao:** Para entradas `collection_holder`, `key_collection_holder` e `key_erc20_holder`, chama o servico KEY da W3Block para verificar se as carteiras do usuario detem os tokens necessarios, correspondendo endereco do contrato/ID da colecao, chain ID, quantidade minima e requisitos de metadados.

**Observacoes:**
- Resultados sao cacheados por **120 segundos**. Passe `disableCache=true` para ignorar o cache
- Um usuario pode verificar seu proprio acesso (o decorator `AcceptSelfUserRequest` permite auto-requisicoes no parametro de query `userId`)
- O `hasAccess` de nivel superior e `true` se **qualquer** whitelist conceder acesso

---

## Fluxo: Promover Whitelist On-Chain

### Passo 1: Promover para Wallet Group

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/whitelists/{tenantId}/{id}/promote-on-chain` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "chainId": 137
}
```

**Observacoes:**
- O tenant deve ter um `operatorAddress` configurado
- Cria um `WalletGroup` vinculado a whitelist para a chain especificada
- Uma whitelist so pode ser promovida uma vez por chain ID
- Apos a criacao, o wallet group e enfileirado para sincronizacao e publicacao na blockchain
- Exclusoes subsequentes de entradas da whitelist tambem atualizarao o wallet group on-chain

---

## Tratamento de Erros

| Status | Erro | Causa | Resolucao |
|--------|------|-------|-----------|
| 400 | WhitelistEntrySaveException | Entrada duplicada (mesma whitelist + type + value) | Verifique se a entrada ja existe antes de adicionar |
| 400 | WhitelistEntryUserNotFoundException | ID de usuario nao encontrado ou pertence a outro tenant | Verifique se o usuario existe no mesmo tenant |
| 400 | WhitelistAlreadyOnChainException | Whitelist ja promovida para este chainId | Use o wallet group existente ou escolha uma chain diferente |
| 400 | WhitelistOnChainException | Criacao do wallet group falhou | Verifique a configuracao do endereco do operador do tenant |
| 400 | WhitelistCheckException | Verificacao de acesso nao retornou detalhes | Verifique se os IDs da whitelist e do usuario sao validos |
| 400 | WhitelistEntryNotFoundException | Entrada nao encontrada na whitelist/tenant | Verifique o ID da entrada e a propriedade da whitelist |
| 404 | WhitelistNotFoundException | Whitelist nao encontrada para o tenant | Verifique o ID da whitelist e o tenant |

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Incompatibilidade de maiusculas/minusculas no valor da entrada | Todos os valores sao convertidos para minusculas ao salvar. Sempre compare em minusculas |
| 2 | Adicionando entrada user_id para o tenant errado | O servico valida que o usuario pertence ao mesmo tenant. Use o tenantId correto |
| 3 | additionalData ausente para collection_holder | O campo `chainId` em `additionalData` e obrigatorio para entradas do tipo `collection_holder` |
| 4 | Cache retorna verificacao de acesso desatualizada | Resultados da verificacao de acesso sao cacheados por 120 segundos. Passe `disableCache=true` para resultados em tempo real |
| 5 | Promocao on-chain falha silenciosamente | Garanta que o tenant tenha um `operatorAddress` configurado antes de promover |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Autenticacao Sign-In | Obrigatorio para Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Promocoes de Comercio | Whitelists podem ser vinculadas a promocoes | [FLOW_COMMERCE_PROMOTIONS](../commerce/FLOW_COMMERCE_PROMOTIONS.md) |
| Contatos / KYC | Entradas `kyc_approved_context` dependem de aprovacao KYC | [FLOW_CONTACTS_KYC_APPROVAL](../contacts/FLOW_CONTACTS_KYC_APPROVAL.md) |
