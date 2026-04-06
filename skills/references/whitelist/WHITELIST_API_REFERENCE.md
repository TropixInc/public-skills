---
id: WHITELIST_API_REFERENCE
title: "Whitelist - Referência da API"
module: offpix/whitelist
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - whitelist
  - user-groups
  - entries
  - access-control
  - api-reference
---

# Referência da API de Whitelist

Referência completa de endpoints para o módulo de Whitelist da W3Block. Este módulo gerencia duas entidades -- Whitelists (grupos de acesso nomeados) e Whitelist Entries (regras de acesso individuais dentro de um grupo) -- servido pelo backend PixwayID.

> **Nome no frontend:** O frontend Offpix se refere a whitelists como **"Grupos de Usuários"** em toda a interface.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://pixwayid.w3block.io` |
| Staging | *Disponível — use a variante do subdomínio de staging* |
| Swagger | https://pixwayid.w3block.io/docs/ |

## Autenticação

Todos os endpoints requerem:

```
Authorization: Bearer {accessToken}
```

Todos os endpoints de whitelist requerem o papel **Admin** por padrão. Os endpoints `check-user` também aceitam requisições do próprio usuário (um usuário verificando seu próprio acesso).

---

## Enums

### WhitelistEntryType

Define o tipo de regra de acesso para uma entrada da whitelist. O significado do campo `value` muda com base no tipo.

| Valor | Label no Frontend | Descrição | O campo `value` contém |
|-------|-------------------|-----------|------------------------|
| `user_id` | Usuário | Acesso direto do usuário | UUID do usuário |
| `email` | Email | Acesso baseado em email | Endereço de email |
| `wallet_address` | Carteira | Acesso por carteira blockchain | Endereço de carteira (hex) |
| `collection_holder` | Contrato | Titular de coleção NFT externa | Endereço do contrato (hex) |
| `kyc_approved_context` | -- | Acesso para usuários com KYC aprovado em um context específico | UUID do context |
| `key_collection_holder` | Collection NFT (desabilitado na UI) | Titular de coleção da plataforma W3Block KEY | UUID da coleção |
| `key_erc20_holder` | -- | Titular de token ERC-20 da plataforma W3Block KEY | UUID do contrato ERC-20 |

### WhitelistEntriesSortBy

| Valor | Descrição |
|-------|-----------|
| `createdAt` | Ordenar por data de criação (padrão) |
| `updatedAt` | Ordenar por data da última atualização |

---

## Entidades e Relacionamentos

```
Whitelist (1) ---> (N) WhitelistEntry
    |
    +--- name: string (obrigatório)
    +--- tenantId: uuid
    +--- walletGroups: WalletGroupEntity[] (promoções on-chain)

WhitelistEntry:
    +--- type: WhitelistEntryType (obrigatório)
    +--- value: string (obrigatório, convertido para minúsculas ao salvar)
    +--- additionalData: JSON (obrigatório para tipos holder, null caso contrário)
```

**Restrição de unicidade:** A combinação `(whitelistId, type, value)` é única, **exceto** para os tipos `collection_holder` e `key_collection_holder` que permitem valores duplicados (para suportar diferentes configurações de `additionalData` como diferentes intervalos de token ID).

### Schemas de AdditionalData

#### CollectionHolderAdditionalData (type: `collection_holder`)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `chainId` | number | Sim | ID da cadeia blockchain (ex: 80001 para Mumbai, 137 para Polygon) |
| `startTokenId` | number | Não | Início do intervalo de token ID (mín: 1) |
| `endTokenId` | number | Não | Fim do intervalo de token ID (mín: 1) |
| `metadataRequirements` | object[] | Não | Array de pares chave-valor que os metadados do token devem corresponder |

#### KeyCollectionTokenHolderAdditionalData (type: `key_collection_holder`)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `amount` | number | Não | Número mínimo de tokens necessários (mín: 0) |
| `metadataRequirements` | object[] | Não | Array de pares chave-valor que os metadados do token devem corresponder |

#### KeyErc20HolderAdditionalData (type: `key_erc20_holder`)

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `amount` | number | Não | Saldo mínimo de tokens necessário (mín: 0) |

---

## Endpoints

### CRUD de Whitelist

#### POST /whitelists/{tenantId}

Criar uma nova whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| POST | `/whitelists/{tenantId}` | Bearer (Admin) | 201 Created |

**Corpo da Requisição:**

```json
{
  "name": "VIP Members"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição da whitelist |

**Resposta (201):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

---

#### GET /whitelists/{tenantId}

Listar todas as whitelists de um tenant com paginação.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/whitelists/{tenantId}` | Bearer (Admin) | 200 OK |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | number | Não | Número da página (padrão: 1) |
| `limit` | number | Não | Itens por página (padrão: 10) |
| `search` | string | Não | Filtrar por nome (correspondência parcial) |
| `orderBy` | string | Não | `ASC` ou `DESC` |
| `sortBy` | string | Não | Campo para ordenação |

**Resposta (200):**

```json
{
  "items": [
    {
      "id": "a1b2c3d4-...",
      "tenantId": "tenant-uuid",
      "name": "VIP Members",
      "createdAt": "2026-04-01T00:00:00.000Z",
      "updatedAt": "2026-04-01T00:00:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 1,
    "itemCount": 1,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

---

#### GET /whitelists/{tenantId}/{id}

Obter uma whitelist pelo ID, incluindo seus wallet groups (promoções on-chain).

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 200 OK |

**Resposta (200):**

```json
{
  "id": "a1b2c3d4-...",
  "tenantId": "tenant-uuid",
  "name": "VIP Members",
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z",
  "walletGroups": []
}
```

---

#### PATCH /whitelists/{tenantId}/{id}

Atualizar o nome de uma whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| PATCH | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 200 OK |

**Corpo da Requisição:**

```json
{
  "name": "Updated Name"
}
```

**Resposta (200):** Mesmo que a resposta do GET por ID com valores atualizados.

---

#### DELETE /whitelists/{tenantId}/{id}

Exclusão suave de uma whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| DELETE | `/whitelists/{tenantId}/{id}` | Bearer (Admin) | 204 No Content |

**Notas:**
- Esta é uma exclusão suave (o registro não é fisicamente removido)
- As entradas pertencentes a esta whitelist não são explicitamente excluídas, mas se tornam inacessíveis

---

### Entradas da Whitelist

#### POST /whitelists/{tenantId}/{id}/entries

Adicionar uma entrada a uma whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| POST | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) | 201 Created |

**Requisição Mínima (tipo email):**

```json
{
  "type": "email",
  "value": "user@example.com"
}
```

**Requisição Completa (collection holder com additionalData):**

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

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `type` | WhitelistEntryType | Sim | Tipo da entrada |
| `value` | string | Sim | O identificador de acesso (email, endereço, UUID, etc.) -- validado por tipo |
| `additionalData` | object | Condicional | Obrigatório para `collection_holder`, `key_collection_holder`, `key_erc20_holder`. Ver Schemas de AdditionalData acima |

**Resposta (201):**

```json
{
  "id": "entry-uuid",
  "whitelistId": "a1b2c3d4-...",
  "type": "email",
  "value": "user@example.com",
  "additionalData": null,
  "createdAt": "2026-04-01T00:00:00.000Z",
  "updatedAt": "2026-04-01T00:00:00.000Z"
}
```

**Notas:**
- O campo `value` é **convertido para minúsculas** ao salvar (emails, endereços de carteira, UUIDs são todos armazenados em minúsculas)
- O campo `value` é validado com base no `type` usando `IsValidWhitelistValueValidator`
- Para o tipo `user_id`, o serviço verifica se o usuário existe e pertence ao mesmo tenant

---

#### GET /whitelists/{tenantId}/{id}/entries

Listar entradas de uma whitelist com paginação e filtros.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/whitelists/{tenantId}/{id}/entries` | Bearer (Admin) | 200 OK |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | number | Não | Número da página (padrão: 1) |
| `limit` | number | Não | Itens por página (padrão: 10) |
| `search` | string | Não | Buscar por valor; também busca nomes/emails de usuários correspondentes se >= 5 caracteres |
| `type` | WhitelistEntryType[] | Não | Filtrar por tipo(s) de entrada |
| `sortBy` | WhitelistEntriesSortBy | Não | Campo de ordenação (padrão: `createdAt`) |
| `orderBy` | string | Não | `ASC` ou `DESC` |
| `showWallets` | boolean | Não | Se `true`, inclui endereços de carteira resolvidos para entradas email/user_id |

**Resposta (200):**

```json
{
  "items": [
    {
      "id": "entry-uuid",
      "whitelistId": "a1b2c3d4-...",
      "type": "email",
      "value": "user@example.com",
      "additionalData": null,
      "createdAt": "2026-04-01T00:00:00.000Z",
      "updatedAt": "2026-04-01T00:00:00.000Z",
      "wallets": []
    }
  ],
  "meta": {
    "totalItems": 1,
    "itemCount": 1,
    "itemsPerPage": 10,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

**Notas:**
- Quando `search` é fornecido com >= 5 caracteres, o serviço também busca usuários por nome/email e inclui correspondências do tipo `user_id` para os usuários encontrados
- O campo `wallets` só é preenchido quando `showWallets=true`

---

#### DELETE /whitelists/{tenantId}/{id}/entries/{entryId}

Remover uma entrada de uma whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| DELETE | `/whitelists/{tenantId}/{id}/entries/{entryId}` | Bearer (Admin) | 204 No Content |

**Notas:**
- Esta é uma exclusão suave
- Se a whitelist foi promovida on-chain (possui wallet groups), os endereços de carteira resolvidos da entrada também são removidos dos wallet groups associados

---

### Verificação de Acesso

#### GET /whitelists/{tenantId}/{id}/check-user

Verificar se um usuário específico tem acesso através de uma única whitelist.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/whitelists/{tenantId}/{id}/check-user` | Bearer (Admin / Próprio) | 200 OK |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `userId` | uuid | Sim | ID do usuário a verificar |
| `disableCache` | boolean | Não | Ignorar o cache de 120 segundos (padrão: false) |

**Resposta (200):**

```json
{
  "whitelistId": "a1b2c3d4-...",
  "userId": "user-uuid",
  "hasAccess": true
}
```

**Notas:**
- Os resultados são cacheados por 120 segundos por padrão
- Um usuário pode verificar seu próprio acesso (via decorator `AcceptSelfUserRequest`)
- A verificação avalia todos os tipos de entrada: correspondências diretas (user_id, email, wallet_address, kyc_approved_context) e correspondências baseadas em titularidade (collection_holder, key_collection_holder, key_erc20_holder)
- Lança `WhitelistCheckException` se nenhum detalhe de verificação for retornado

---

#### GET /whitelists/{tenantId}/check-user

Verificar se um usuário tem acesso através de múltiplas whitelists de uma vez.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/whitelists/{tenantId}/check-user` | Bearer (Admin / Próprio) | 200 OK |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `userId` | uuid | Sim | ID do usuário a verificar |
| `whitelistsIds` | uuid[] | Sim | Array de IDs de whitelist para verificar |
| `disableCache` | boolean | Não | Ignorar o cache de 120 segundos (padrão: false) |

**Resposta (200):**

```json
{
  "hasAccess": true,
  "details": [
    {
      "whitelistId": "whitelist-uuid-1",
      "userId": "user-uuid",
      "hasAccess": true
    },
    {
      "whitelistId": "whitelist-uuid-2",
      "userId": "user-uuid",
      "hasAccess": false
    }
  ]
}
```

**Notas:**
- `hasAccess` no nível superior é `true` se o usuário tem acesso a **qualquer** uma das whitelists listadas
- `details` fornece resultados de acesso por whitelist
- Mesmo comportamento de cache da verificação de whitelist única (TTL de 120 segundos)

---

### Promoção On-Chain

#### PATCH /whitelists/{tenantId}/{id}/promote-on-chain

Promover uma whitelist para um wallet group on-chain, habilitando controle de acesso em nível de blockchain.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| PATCH | `/whitelists/{tenantId}/{id}/promote-on-chain` | Bearer (Admin) | 200 OK |

**Corpo da Requisição:**

```json
{
  "chainId": 137
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `chainId` | number | Sim | ID da cadeia blockchain de destino |

**Resposta (200):**

Retorna um objeto `WalletGroupResponseDto` representando o wallet group on-chain criado.

**Notas:**
- O tenant deve ter um `operatorAddress` configurado
- Uma whitelist só pode ser promovida uma vez por `chainId`; tentar promover novamente para a mesma cadeia lança `WhitelistAlreadyOnChainException`
- Após a criação, o wallet group é enfileirado para sincronização e publicação na blockchain
- Excluir entradas de uma whitelist promovida também remove as carteiras correspondentes do wallet group on-chain

---

## Referência de Erros

| Exceção | Status HTTP | Causa |
|---------|-------------|-------|
| `WhitelistNotFoundException` | 404 | ID da whitelist não encontrado para o tenant fornecido |
| `WhitelistDeleteException` | 400 | Operação de exclusão suave afetou 0 linhas |
| `WhitelistEntrySaveException` | 400 | Entrada duplicada (mesmo whitelistId + type + value) ou erro desconhecido ao salvar |
| `WhitelistEntryDeleteException` | 400 | Exclusão suave da entrada afetou 0 linhas |
| `WhitelistEntryNotFoundException` | 400 | Entrada não encontrada na whitelist/tenant especificado |
| `WhitelistEntryUserNotFoundException` | 400 | ID do usuário não existe ou pertence a um tenant diferente |
| `WhitelistAlreadyOnChainException` | 400 | Whitelist já possui um wallet group para o chainId especificado |
| `WhitelistOnChainException` | 400 | Falha ao criar wallet group durante promoção on-chain |
| `WhitelistCheckException` | 400 | Erro durante verificação de acesso do usuário (nenhum detalhe retornado) |
