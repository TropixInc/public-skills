---
id: FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
title: "Tokenizacao - Ciclo de Vida da Colecao (Rascunho ate Publicada)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - collections
  - draft
  - publish
  - lifecycle
depends_on:
  - TOKENIZATION_API_REFERENCE
---

# Ciclo de Vida da Colecao de Tokens

## Visao Geral

Este fluxo cobre o ciclo de vida completo de uma colecao de tokens, desde a criacao ate a publicacao na blockchain. Uma colecao de tokens comeca no status DRAFT, onde as edicoes sao geradas e configuradas, e transiciona para o status PUBLISHED quando implantada on-chain. O fluxo inclui criacao de rascunho, sincronizacao de rascunhos de edicoes, configuracao de metadados, estimativa de custos de gas e publicacao na blockchain.

## Pre-requisitos

| Requisito | Descricao | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da Empresa | Fluxo de autenticacao / configuracao do ambiente |
| Contrato implantado | Contrato no status PUBLISHED com ADMIN_MINTER | [Ciclo de Vida de Contratos NFT](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Template de subcategoria | Define o schema de metadados para edicoes de tokens | Configuracao da plataforma |

---

## Entidades e Relacionamentos

```
Contract (PUBLISHED, ADMIN_MINTER)
    |
    +---> TokenCollection (DRAFT -> PUBLISHED)
              |
              +---> TokenEdition #1 (DRAFT -> READY_TO_MINT -> MINTING -> MINTED)
              +---> TokenEdition #2 (DRAFT -> READY_TO_MINT -> MINTING -> MINTED)
              +---> TokenEdition #3 (DRAFT)
              +---> ...
              +---> TokenEdition #N (DRAFT)

Subcategory Template
    |
    +---> Define campos e regras de validacao do tokenData
    +---> Snapshot salvo como publishedTokenTemplate na publicacao
```

**Relacionamentos-chave:**
- Um Contract pode ter multiplas TokenCollections
- Uma TokenCollection possui exatamente `quantity` TokenEditions
- Cada TokenEdition possui um `editionNumber` sequencial (1 ate quantity)
- Se `similarTokens` for true, todas as edicoes compartilham o tokenData do nivel da colecao
- Se `similarTokens` for false, cada edicao possui tokenData individual

---

## Fluxo: Criar Rascunho da Colecao

### Passo 1: Criar a Colecao

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) |

**Requisicao Minima:**

```json
{
  "name": "My NFT Collection"
}
```

**Requisicao Completa:**

```json
{
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "quantity": 100,
  "initialQuantityToMint": 50,
  "similarTokens": true,
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "rangeInitialToMint": "1-50",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  }
}
```

**Resposta (201):**

```json
{
  "id": "collection-uuid",
  "status": "draft",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  },
  "quantity": 100,
  "initialQuantityToMint": 50,
  "initialQuantity": 100,
  "quantityMinted": 0,
  "rfids": [],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "pass": false,
  "settings": null,
  "rangeInitialToMint": "1-50",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:00:00.000Z"
}
```

**Observacoes:**
- `contractId` e opcional na criacao mas **obrigatorio** antes da publicacao
- `subcategoryId` e obrigatorio para validacao de metadados no momento da publicacao
- `quantity` tem valor padrao 0 se nao fornecido; atualize antes de sincronizar as edicoes
- `similarTokens` tem valor padrao true -- todas as edicoes compartilharao o mesmo tokenData do nivel da colecao
- Salve o `id` retornado para todas as operacoes subsequentes

---

## Fluxo: Sincronizar Rascunhos de Edicoes

Apos criar uma colecao e definir a `quantity`, gere os rascunhos individuais das edicoes.

### Passo 2: Disparar Sincronizacao de Edicoes

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-collections/{id}/sync-draft-tokens` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Passo 3: Consultar Conclusao do Job

A sincronizacao e uma operacao assincrona. Consulte o endpoint de status do job ate a conclusao.

**Endpoint de Consulta:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Resposta (em andamento):**

```json
{
  "id": "job-uuid",
  "status": "processing",
  "progress": 45,
  "createdAt": "2026-04-02T12:05:00.000Z"
}
```

**Resposta (concluido):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "created": 100,
    "updated": 0,
    "deleted": 0
  },
  "createdAt": "2026-04-02T12:05:00.000Z",
  "completedAt": "2026-04-02T12:06:00.000Z"
}
```

**Observacoes:**
- Cria `quantity` edicoes com `editionNumber` sequencial de 1 ate N
- Todas as edicoes iniciam no status DRAFT
- Se edicoes ja existem (de uma sincronizacao anterior), o job re-sincroniza: adiciona edicoes faltantes, remove excedentes
- Se `quantity` foi alterada apos uma sincronizacao anterior, re-sincronizar ajusta as edicoes de acordo
- **Voce deve aguardar este job concluir antes de tentar publicar**

---

## Fluxo: Configurar Colecao (Opcional)

Antes de publicar, voce pode atualizar a configuracao da colecao.

### Passo 4: Atualizar Colecao

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PUT | `/{companyId}/token-collections/{id}` | Bearer (Admin) |

**Corpo da Requisicao (parcial -- inclua apenas os campos a atualizar):**

```json
{
  "name": "Updated Collection Name",
  "description": "Updated description with more detail",
  "mainImage": "https://example.com/new-cover.png",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "quantity": 150,
  "initialQuantityToMint": 75,
  "rangeInitialToMint": "1-75",
  "tokenData": {
    "name": "Digital Asset v2",
    "description": "An updated unique piece",
    "image": "https://example.com/asset-v2.png",
    "attributes": [
      { "trait_type": "collection", "value": "Genesis" }
    ]
  },
  "ownerAddress": "0xnewowner1234567890abcdef1234567890abcdef",
  "similarTokens": true,
  "rfids": ["RFID-001", "RFID-002", "RFID-003"],
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

**Resposta (200):** Retorna a TokenCollectionEntity atualizada.

**Observacoes:**
- Permitido apenas quando o status e `draft`
- Se `quantity` for alterada, voce deve re-sincronizar as edicoes (passo 2) antes de publicar
- `contractId` deve referenciar um contrato que esteja PUBLISHED com ADMIN_MINTER habilitado
- `rangeInitialToMint` deve estar em formato de intervalo valido: `"1-50"` (intervalo continuo) ou `"1,5,10"` (edicoes especificas)
- `tokenData` deve estar em conformidade com o schema do template de subcategoria
- A lista de `rfids` deve conter valores unicos que nao estejam em uso por outras edicoes ativas

---

## Fluxo: Publicar Colecao

Quando a colecao estiver totalmente configurada e as edicoes sincronizadas, publique-a na blockchain.

### Passo 5: Publicar na Blockchain

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-collections/publish/{id}` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta (200):**

```json
{
  "id": "collection-uuid",
  "status": "published",
  "companyId": "company-uuid",
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "My NFT Collection",
  "description": "A collection of 100 unique digital assets",
  "mainImage": "https://example.com/collection-cover.png",
  "tokenData": {
    "name": "Digital Asset",
    "description": "A unique piece",
    "image": "https://example.com/asset.png"
  },
  "publishedTokenTemplate": {
    "fields": [
      { "name": "name", "type": "string", "required": true },
      { "name": "description", "type": "string", "required": false },
      { "name": "image", "type": "string", "required": true }
    ]
  },
  "quantity": 100,
  "initialQuantityToMint": 50,
  "initialQuantity": 100,
  "quantityMinted": 0,
  "rfids": [],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "pass": false,
  "rangeInitialToMint": "1-50",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "updatedAt": "2026-04-02T12:30:00.000Z"
}
```

**Verificacoes de Validacao (todas devem passar):**

| Verificacao | Validacao | Codigo de Erro |
|-------------|-----------|----------------|
| Contrato existe e esta publicado | Status do contrato deve ser `published` | `CONTRACT_HAS_NO_ADDRESS` |
| Contrato possui ADMIN_MINTER | Contrato deve ter funcionalidade de minter habilitada | `CONTRACT_NO_ENABLE_MINTER` |
| Endereco do proprietario definido | `ownerAddress` da colecao ou endereco padrao do proprietario da empresa | `COMPANY_HAS_NO_DEFAULT_OWNER_ADDRESS` |
| Edicoes sincronizadas | Todas as edicoes devem existir (contagem corresponde a quantity) | `EDITIONS_NOT_SYNC` |
| Todas as edicoes no status DRAFT | Nenhuma edicao com DRAFT_ERROR ou IMPORT_ERROR | `EDITIONS_WITH_ERRORS` |
| Sem jobs em execucao | Nenhum job assincrono em andamento para esta colecao | `JOB_RUNNING` |
| RFIDs unicos | Sem RFIDs duplicados na lista | `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` |
| RFIDs disponiveis | Nenhum RFID ja em uso por outras colecoes | `RFID_HAS_ALREADY_BEEN_USED` |
| TokenData valido | Se `similarTokens`, tokenData deve corresponder ao template de subcategoria | Varios codigos de validacao de campo |
| Intervalo valido | `rangeInitialToMint` deve estar em formato valido | `INVALID_RANGE` |

**O que acontece na publicacao:**

1. O template de subcategoria e salvo como snapshot em `publishedTokenTemplate`
2. Se `similarTokens` for true, o `tokenData` do nivel da colecao e copiado para todas as edicoes
3. Edicoes no intervalo `rangeInitialToMint` tem seu status definido como `readyToMint`
4. O status da colecao muda de `draft` para `published` (isto e **irreversivel**)
5. Edicoes fora do intervalo `rangeInitialToMint` permanecem no status `draft`

---

## Fluxo: Estimar Gas Antes de Publicar

Estime o custo de gas antes de se comprometer com uma operacao de publicacao.

### Estimar Gas

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-collections/estimate-gas` | Bearer (Admin) |

**Parametros de Query:**

| Parametro | Tipo | Obrigatorio | Descricao |
|-----------|------|-------------|-----------|
| `contractId` | uuid | Sim | ID do contrato para a colecao |
| `initialQuantityToMint` | number | Sim | Numero de tokens a mintar na publicacao |

**Resposta (200):**

```json
{
  "slow": {
    "gasPrice": "30000000000",
    "estimatedCost": "0.005",
    "estimatedTime": "120s"
  },
  "standard": {
    "gasPrice": "50000000000",
    "estimatedCost": "0.008",
    "estimatedTime": "60s"
  },
  "fast": {
    "gasPrice": "80000000000",
    "estimatedCost": "0.012",
    "estimatedTime": "30s"
  }
}
```

**Observacoes:**
- A estimativa de gas utiliza as condicoes atuais da rede e pode flutuar
- Os tres niveis (slow/standard/fast) representam diferentes estrategias de preco de gas
- Os custos sao no token nativo da chain (ex.: MATIC para Polygon)

---

## Fluxo: Excluir Rascunho da Colecao

Remova uma colecao que nao foi publicada.

### Excluir Rascunho

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| DELETE | `/{companyId}/token-collections/{id}/draft` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta:** 204 No Content

**Observacoes:**
- Permitido apenas quando o status e `draft`
- Exclui a colecao e todos os rascunhos de edicoes associados
- Esta operacao e irreversivel
- Nao pode ser chamado em colecoes publicadas (use burn em vez disso)

---

## Fluxo: Queimar Colecao Publicada

Remova uma colecao publicada e queime todos os seus tokens on-chain.

### Queimar Colecao

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| DELETE | `/{companyId}/token-collections/{id}/burn` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta (200):**

```json
{
  "id": "collection-uuid",
  "status": "published",
  "burnInitiated": true
}
```

**Observacoes:**
- Permitido apenas quando o status e `published`
- Inicia transacoes de queima para todas as edicoes mintadas on-chain
- Esta e uma operacao assincrona -- edicoes transicionam de BURNING para BURNED
- A colecao em si permanece no status PUBLISHED mas as edicoes se tornam BURNED

---

## Fluxo: Aumentar Edicoes Apos Publicacao

Adicione mais edicoes a uma colecao publicada.

### Aumentar Edicoes

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-collections/{id}/increase-editions` | Bearer (Integration) |

**Corpo da Requisicao:**

```json
{
  "quantity": 50
}
```

**Resposta (200):** Retorna a TokenCollectionEntity atualizada com a quantidade aumentada.

**Observacoes:**
- Permitido apenas quando o status e `published`
- Requer role Integration (nao Admin padrao)
- Novas edicoes sao criadas a partir do proximo numero sequencial de edicao
- Novas edicoes iniciam no status DRAFT
- A `quantity` da colecao e aumentada pela quantidade especificada

---

## Tratamento de Erros

| Status | Erro | Causa | Resolucao |
|--------|------|-------|-----------|
| 400 | TokenCollectionPublishValidationException | Uma ou mais validacoes de publicacao falharam | Verifique o array `validations` na resposta de erro para problemas especificos |
| 400 | TokenCollectionNotDraftException | Tentativa de editar/excluir uma colecao publicada | Use operacoes de estado publicado (burn, increase-editions) em vez disso |
| 400 | JobRunningException | Job assincrono ainda em andamento | Aguarde o job atual concluir antes de tentar novamente |
| 400 | InvalidRangeException | Formato de rangeInitialToMint invalido | Use o formato como "1-50" ou "1,5,10" |
| 400 | ContractNotPublishedException | Contrato nao esta publicado | Publique o contrato primeiro pelo modulo de Contratos |
| 400 | ContractNoMinterException | Contrato nao possui ADMIN_MINTER | Habilite ADMIN_MINTER no contrato pelo modulo de Contratos |
| 404 | TokenCollectionNotFoundException | Colecao nao encontrada | Verifique o ID da colecao e o companyId |

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Publicacao falha com EDITIONS_NOT_SYNC | Chame `sync-draft-tokens` e aguarde o job concluir antes de publicar |
| 2 | Publicacao falha com EDITIONS_WITH_ERRORS | Liste as edicoes filtradas por status de erro, corrija os problemas (via edicao ou re-importacao), depois tente publicar novamente |
| 3 | Alterar quantity apos sincronizacao | Se voce atualizar `quantity` apos sincronizar, deve re-sincronizar as edicoes antes de publicar |
| 4 | Contrato nao pronto | Garanta que o contrato esteja PUBLISHED com ADMIN_MINTER antes de tentar publicar a colecao |
| 5 | Endereco do proprietario ausente | Defina `ownerAddress` na colecao ou configure um endereco padrao do proprietario na empresa |
| 6 | Conflitos de RFID na publicacao | Verifique todos os RFIDs quanto a unicidade com `check-rfid` antes de atribui-los a colecao |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida de Contratos NFT | Obrigatorio: contrato deve ser implantado e publicado primeiro | [FLOW_CONTRACTS_NFT_LIFECYCLE](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Operacoes de Edicao | Proximo: gerenciar edicoes apos a colecao ser publicada | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Importacao em Massa | Alternativa: configurar edicoes via Excel antes de publicar | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| Autenticacao Sign-In | Obrigatorio para Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
