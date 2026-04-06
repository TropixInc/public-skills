---
id: FLOW_CONTRACTS_COLLECTION_MANAGEMENT
title: "Contratos - Gerenciamento de Coleções de Tokens"
module: offpix/contracts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - tokens
  - collections
  - editions
  - publish
  - bulk
depends_on:
  - CONTRACTS_API_REFERENCE
  - FLOW_CONTRACTS_NFT_LIFECYCLE
---

# Gerenciamento de Coleções de Tokens

## Visão Geral

Uma coleção de tokens agrupa múltiplas edições de tokens (NFTs individuais) sob metadados compartilhados e um contrato NFT publicado. Este fluxo abrange a criação de coleções, configuração de templates de metadados, gerenciamento de edições, publicação na blockchain e importação/exportação em massa via XLSX. No frontend, as coleções são gerenciadas em **Tokens** (`/dash/tokens/collections/{id}`).

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da empresa | Fluxo de auth / configuração do ambiente |
| Contrato NFT publicado | Contrato com `status: published` | [Ciclo de Vida NFT](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Subcategoria (template) | Template de coleção com campos de metadados | Criar via endpoint de subcategories |

## Máquinas de Estado

**Coleção:** `DRAFT → PUBLISHED`

**Edição:**
```
DRAFT ──→ READY_TO_MINT ──→ MINTING ──→ MINTED
  │                                        │
  ├→ DRAFT_ERROR                          ├→ TRANSFERRING → TRANSFERRED
  ├→ IMPORT_ERROR                         ├→ BURNING → BURNED
  │                                        │
  └→ LOCKED_FOR_BUY → (back to DRAFT)    ├→ TRANSFER_FAILURE
                                           └→ BURN_FAILURE
```

---

## Fluxo: Criar uma Coleção

### Passo 1 (Opcional): Criar um Template de Coleção

Templates definem quais campos de metadados estão disponíveis para os tokens na coleção.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/subcategories` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "name": "Digital Art Template",
  "categoryId": "category-uuid",
  "tokenTemplate": {
    "fields": [
      { "name": "artist", "type": "TEXTFIELD", "required": true },
      { "name": "year", "type": "YEAR", "required": true },
      { "name": "medium", "type": "SELECT", "options": ["Digital", "Mixed Media"] },
      { "name": "dimensions", "type": "DIMENSIONS_2D" },
      { "name": "highResImage", "type": "IMAGE" }
    ]
  }
}
```

**Tipos de campos do template:** `DATE`, `BOOLEAN`, `DIMENSIONS_2D`, `DIMENSIONS_3D`, `NUMERIC`, `RADIOGROUP`, `SELECT`, `IMAGE`, `TEXTAREA`, `TEXTFIELD`, `YEAR`

### Passo 2: Criar Rascunho da Coleção

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection"
}
```

**Requisição Completa:**
```json
{
  "contractId": "published-contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection",
  "description": "Limited edition digital artworks",
  "mainImage": "https://cdn.example.com/collection.png",
  "tokenData": {
    "artist": "John Doe",
    "year": 2026,
    "medium": "Digital"
  },
  "quantity": 100,
  "rangeInitialToMint": "1-50",
  "ownerAddress": "0xOwnerWallet...",
  "similarTokens": true,
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

**Observações:**
- `similarTokens: true` significa que todas as edições compartilham os mesmos metadados (tokenData). Defina como `false` para edições únicas
- `rangeInitialToMint` especifica quais edições serão mintadas na publicação (ex.: "1-50" minta as edições 1 até 50)
- `quantity` define o número total de edições na coleção
- Se `contractId` for omitido, a coleção é independente de contrato (pode ser atribuída depois)

### Passo 3: Sincronizar Edições em Rascunho

Cria os registros individuais de edições de token em rascunho para a coleção.

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| PATCH | `/{companyId}/token-collections/{collectionId}/sync-draft-tokens` | Bearer (Admin) |

**Resposta (201):**
```json
{
  "jobId": "job-uuid"
}
```

Esta é uma **operação assíncrona** processada via fila Bull. O job cria `quantity` edições em rascunho numeradas sequencialmente.

---

## Fluxo: Gerenciar Edições (Tokens Individuais)

### Listar Edições

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-editions` | Bearer (Admin) |

**Parâmetros de Query:**

| Parâmetro | Tipo | Padrão | Descrição |
|-----------|------|--------|-----------|
| `tokenCollectionId` | UUID | — | Filtrar por coleção |
| `status` | TokenEditionStatus[] | — | Filtrar por status |
| `editionNumber` | number | — | Edição específica |
| `walletAddresses` | string[] | — | Filtrar por carteira do proprietário |
| `userId` | UUID | — | Filtrar por usuário |
| `search` | string | — | Buscar no nome/descrição |
| `sortBy` | string | `editionNumber` | Coluna de ordenação |
| `orderBy` | `ASC` \| `DESC` | `ASC` | Direção da ordenação |
| `page`, `limit` | integer | 1, 10 | Paginação |

### Atualizar Metadados da Edição

Para coleções com `similarTokens: false`, cada edição pode ter metadados únicos.

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-editions/update-token-metadata` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "tokenEditionIds": ["edition-uuid-1", "edition-uuid-2"],
  "tokenData": {
    "artist": "Updated Name",
    "year": 2027
  }
}
```

Se a coleção tiver `settings.sendWebhookWhenTokenEditionIsUpdated: true`, um webhook é disparado após a atualização.

### Atribuir RFID

Vincular uma tag RFID física a uma edição de token.

**Passo 1: Verificar disponibilidade do RFID**
```
GET /{companyId}/token-editions/check-rfid?rfid=RFID-VALUE-123
```
Resposta: `{ "used": false }`

**Passo 2: Atribuir RFID**
```
PATCH /{companyId}/token-editions/{editionId}/rfid
```
```json
{ "rfid": "RFID-VALUE-123" }
```

---

## Fluxo: Publicar uma Coleção

### Passo 1: Marcar Edições como Prontas

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-editions/ready-to-mint` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "editionId": ["edition-uuid-1", "edition-uuid-2", "edition-uuid-3"]
}
```

Transição de status: `draft` → `readyToMint`

### Passo 2: Estimar Gas

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{companyId}/token-collections/estimate-gas` | Bearer (Admin) |

**Query:** `?contractId={uuid}&initialQuantityToMint={count}&ownerAddress={addr}`

**Resposta:** Estimativa de gas com níveis fast/proposed/safe.

### Passo 3: Publicar

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{companyId}/token-collections/publish/{collectionId}` | Bearer (Admin) | application/json |

O corpo da requisição pode incluir metadados atualizados da coleção (mesmo DTO de atualização).

**Resposta (201):**
```json
{
  "tokenCollection": {
    "id": "collection-uuid",
    "status": "published",
    "quantityMinted": 50,
    "..."
  },
  "validationErrors": []
}
```

**O que acontece internamente:**
1. Todas as edições `readyToMint` transitam para `minting`
2. Jobs de minting na blockchain são criados
3. Conforme cada mint é confirmado: edição → `minted`, recebe `tokenId`, `mintedHash`, `mintedAt`
4. Status da coleção → `published`

**Se a validação falhar:** Retorna o array `validationErrors` com erros a nível de campo usando `PublishValidationsEnum`.

---

## Fluxo: Importação/Exportação em Massa via XLSX

### Exportar Template

```
GET /{companyId}/token-collections/{collectionId}/export/xlsx
```

Retorna um arquivo XLSX binário com os dados atuais das edições e colunas do template.

### Importar Edições

```
POST /{companyId}/token-collections/{collectionId}/import/xlsx
Content-Type: multipart/form-data
```

Faça upload de um arquivo XLSX preenchido (máx. 10MB). Retorna `{ "jobId": "..." }` para processamento assíncrono.

**Observações:**
- O XLSX deve seguir a estrutura do template obtido na exportação
- Linhas inválidas criam edições com status `importError`
- Erros de validação são armazenados no JSONB `errorFields` da edição

---

## Fluxo: Excluir / Queimar Coleção

### Excluir Coleção em Rascunho

```
DELETE /{companyId}/token-collections/{collectionId}/draft
```

Funciona apenas em coleções `draft`. Faz soft-delete da coleção e de todas as edições em rascunho.

### Queimar Coleção Publicada

```
DELETE /{companyId}/token-collections/{collectionId}/burn
```

Faz burn de todos os tokens mintados na coleção. Esta é uma operação on-chain.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | Erros de PublishValidation | Edições possuem dados inválidos | Corrija os erros listados na resposta `validationErrors` |
| 400 | CONTRACT_HAS_NO_ADDRESS | Contrato ainda não publicado | Publique o contrato primeiro |
| 400 | CONTRACT_NO_ENABLE_MINTER | Contrato sem a feature `admin:minter` | Crie um novo contrato com a feature minter |
| 400 | RFID_HAS_ALREADY_BEEN_USED | RFID atribuído a outra edição | Use um RFID diferente |
| 400 | JOB_RUNNING | Outra operação assíncrona em andamento | Aguarde o job atual finalizar |
| 404 | NotFoundException | Coleção ou edição não encontrada | Verifique os IDs |

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Edições não criadas após a coleção | Chame `sync-draft-tokens` para gerar os registros de edição |
| 2 | Publicação falha com erros de validação | Cada edição deve passar pela validação do template. Verifique `errorFields` nas edições com falha |
| 3 | Conflitos de RFID | Verifique a disponibilidade do RFID antes de atribuir. RFIDs são globalmente únicos (não por coleção) |
| 4 | Importação XLSX cria edições com `importError` | Baixe o template primeiro, preencha corretamente e reimporte |
| 5 | Não é possível excluir coleção publicada | Use burn ao invés de delete para coleções publicadas |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Contrato NFT | Crie o contrato antes das coleções | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Operações de Token | Mint, transferência, burn de tokens individuais | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Referência da API | Detalhes completos dos endpoints | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) |
