---
id: TOKENIZATION_API_REFERENCE
title: "Tokenização - Referência da API"
module: offpix/tokenization
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - token-collections
  - token-editions
  - nft
  - api-reference
---

# Referência da API de Tokenização

Referência completa de endpoints para o módulo de Tokenização da W3Block. Este módulo gerencia duas entidades primárias -- Token Collections (grupos de tokens com configuração compartilhada) e Token Editions (instâncias individuais de tokens dentro de uma coleção) -- servido pelo backend KEY.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://api.w3block.io` |
| Swagger | https://api.w3block.io/docs/ |

## Autenticação

Todos os endpoints requerem:

```
Authorization: Bearer {accessToken}
```

A maioria dos endpoints requer o papel **Admin**. Alguns endpoints também aceitam papéis **Integration** ou **User** conforme indicado por endpoint.

---

## Enums

### TokenCollectionStatus

Define o estado do ciclo de vida de uma coleção de tokens.

| Valor | Descrição |
|-------|-----------|
| `draft` | Coleção está sendo configurada, ainda não on-chain |
| `published` | Coleção foi publicada na blockchain |

### TokenEditionStatusEnum

Define o estado do ciclo de vida de uma edição individual de token.

| Valor | Descrição |
|-------|-----------|
| `importError` | Erro ocorreu durante importação XLSX |
| `draftError` | Erro ocorreu durante sincronização de rascunho ou validação |
| `draft` | Edição criada, aguardando configuração |
| `lockedForBuy` | Edição bloqueada para fluxo de compra no commerce |
| `readyToMint` | Edição configurada e pronta para ser mintada on-chain |
| `minting` | Transação de minting submetida, aguardando confirmação |
| `minted` | Mintada com sucesso on-chain |
| `burning` | Transação de queima submetida, aguardando confirmação |
| `burned` | Queimada com sucesso on-chain |
| `burnFailure` | Transação de queima falhou |
| `transferring` | Transação de transferência submetida, aguardando confirmação |
| `transferred` | Transferida com sucesso para novo proprietário |
| `transferFailure` | Transação de transferência falhou |

### PublishValidationsEnum

Códigos de erro de validação retornados quando a validação de publicação falha.

| Valor | Descrição |
|-------|-----------|
| `IS_REQUIRED` | Um campo obrigatório está ausente |
| `GREATER_THAN_ZERO` | Valor deve ser maior que zero |
| `LESS_THAN_QUANTITY` | Valor deve ser menor que a quantidade total |
| `INVALID_RANGE` | Formato de intervalo é inválido |
| `INVALID_VALUE` | Valor inválido genérico |
| `INVALID_STRING` | Valor não é uma string válida |
| `INVALID_DATE` | Valor não é uma data válida |
| `INVALID_YEAR` | Valor de ano é inválido |
| `INVALID_NUMBER` | Valor não é um número válido |
| `INVALID_URL` | Valor não é uma URL válida |
| `INVALID_2D_DIMENSIONS` | Formato de dimensão 2D é inválido |
| `INVALID_3D_DIMENSIONS` | Formato de dimensão 3D é inválido |
| `INVALID_ARRAY` | Valor não é um array válido |
| `INVALID_BOOLEAN` | Valor não é um booleano válido |
| `INVALID_TYPE` | Tipo do valor não corresponde ao tipo esperado |
| `CONTRACT_HAS_NO_ADDRESS` | Contrato não tem endereço implantado |
| `CONTRACT_NO_ENABLE_MINTER` | Contrato não tem funcionalidade ADMIN_MINTER habilitada |
| `COMPANY_HAS_NO_DEFAULT_OWNER_ADDRESS` | Empresa não tem endereço de proprietário padrão configurado |
| `RFID_HAS_ALREADY_BEEN_USED` | RFID já está atribuído a outra edição ativa |
| `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` | RFIDs duplicados encontrados na lista submetida |
| `RFID_USED` | RFID está em uso |
| `JOB_RUNNING` | Um job assíncrono ainda está rodando para esta coleção |
| `EDITIONS_WITH_ERRORS` | Algumas edições têm status de erro (DRAFT_ERROR ou IMPORT_ERROR) |
| `EDITIONS_NOT_SYNC` | Edições não foram sincronizadas (sync-draft-tokens não chamado ou não concluído) |
| `YEAR_OUT_OF_RANGE` | Valor de ano está fora do intervalo aceitável |

---

## Entidades e Relacionamentos

```
TokenCollection (1) ---> (N) TokenEdition
    |
    +--- status: TokenCollectionStatus
    +--- companyId: uuid
    +--- contractId?: uuid (vincula à entidade Contract do módulo Contracts)
    +--- subcategoryId?: uuid (define o template de metadados do token)
    +--- name: string
    +--- description?: string
    +--- mainImage?: string (URL)
    +--- tokenData?: JSON (metadados correspondendo ao template da subcategoria)
    +--- publishedTokenTemplate?: JSON (snapshot do template no momento da publicação)
    +--- quantity: number (total de edições)
    +--- initialQuantityToMint: number
    +--- initialQuantity: number
    +--- quantityMinted: number
    +--- rfids: string[] (lista de RFIDs atribuídos às edições)
    +--- ownerAddress?: string (carteira padrão do proprietário)
    +--- similarTokens: boolean (todas as edições compartilham mesmos metadados)
    +--- pass: boolean (se esta coleção funciona como token-pass)
    +--- settings?: JSON (configuração de webhook, etc.)
    +--- rangeInitialToMint?: string (ex: "1-50" ou "1,5,10")

TokenEdition:
    +--- editionNumber: number (sequencial dentro da coleção)
    +--- tokenCollectionId: uuid
    +--- companyId: uuid
    +--- contractId?: uuid
    +--- status: TokenEditionStatusEnum
    +--- rfid?: string (globalmente único entre edições ativas)
    +--- contractAddress?: string
    +--- ownerAddress?: string
    +--- chainId?: number
    +--- tokenId?: number (ID do token on-chain)
    +--- mintedHash?: string (hash da transação)
    +--- mintedAt?: datetime
    +--- nftMintingId?: uuid
    +--- name?: string
    +--- description?: string
    +--- mainImage?: string (URL)
    +--- tokenData?: JSON
    +--- rawData?: JSON
    +--- errorFields?: JSON (erros de validação da publicação ou importação)
```

---

## DTOs

### CreateTokenCollectionDto

Usado ao criar um novo rascunho de coleção.

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `name` | string | Sim | -- | Nome de exibição da coleção |
| `description` | string | Não | null | Descrição da coleção |
| `mainImage` | string | Não | null | URL da imagem principal |
| `contractId` | uuid | Não | null | Contrato a associar (pode ser definido depois) |
| `subcategoryId` | uuid | Não | null | Subcategoria do template de metadados |
| `quantity` | number | Não | 0 | Número total de edições |
| `initialQuantityToMint` | number | Não | 0 | Quantas edições mintar na publicação |
| `tokenData` | object | Não | null | Metadados correspondendo ao template da subcategoria |
| `similarTokens` | boolean | Não | true | Se todas as edições compartilham mesmos metadados |
| `ownerAddress` | string | Não | null | Endereço de carteira do proprietário padrão |
| `rangeInitialToMint` | string | Não | null | Intervalo de edições a mintar (ex: "1-50") |

### UpdateTokenCollectionDto

Usado ao editar um rascunho de coleção. Mesmos campos que Create, todos opcionais, mais `rfids` (string[]) e `settings` (object).

### MintOnDemandDto

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenCollectionId` | uuid | Sim | ID da coleção |
| `editionNumbers` | number[] | Sim | Números das edições a mintar |
| `ownerAddress` | string | Sim | Endereço da carteira para receber os tokens mintados |

### UpdateTokenMetadataDto

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenCollectionId` | uuid | Sim | ID da coleção |
| `editionNumbers` | number[] | Sim | Números das edições a atualizar |
| `tokenData` | object | Sim | Novos valores de metadados |
| `mergeTokenData` | boolean | Não | Se true, mescla com dados existentes; se false, substitui inteiramente |
| `keepStatus` | boolean | Não | Se true, mantém status atual; se false, reseta para rascunho |

### BurnTokensDto

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokens` | object[] | Sim | Array de tokens a queimar |
| `tokens[].tokenCollectionId` | uuid | Sim | ID da coleção |
| `tokens[].editionNumber` | number | Sim | Número da edição a queimar |

### TransferTokensDto

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenCollectionId` | uuid | Sim | ID da coleção |
| `editionNumbers` | number[] | Sim | Números das edições a transferir |
| `toAddress` | string | Sim | Endereço da carteira de destino |

---

## Endpoints

### CRUD de Token Collection

#### POST /{companyId}/token-collections

Criar uma nova coleção de tokens em status DRAFT.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| POST | `/{companyId}/token-collections` | Bearer (Admin) | 201 Created |

**Requisição Mínima:**
```json
{ "name": "My NFT Collection" }
```

**Requisição Completa:**
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
  "tokenData": { "name": "Asset #", "description": "Unique digital asset", "image": "https://example.com/asset.png" }
}
```

**Notas:**
- `contractId` é opcional na criação mas obrigatório antes da publicação
- `subcategoryId` define o template de metadados para validação
- `quantity` padrão é 0 se não fornecido
- `similarTokens` padrão é true (todas as edições compartilham o mesmo tokenData)

---

#### GET /{companyId}/token-collections

Listar todas as coleções de tokens de uma empresa com paginação.

| Método | Caminho | Auth | Resposta |
|--------|---------|------|----------|
| GET | `/{companyId}/token-collections` | Bearer (Admin) | 200 OK |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | number | Não | Número da página (padrão: 1) |
| `limit` | number | Não | Itens por página (padrão: 10) |
| `search` | string | Não | Filtrar por nome (correspondência parcial) |
| `status` | string | Não | Filtrar por status (`draft` ou `published`) |
| `orderBy` | string | Não | `ASC` ou `DESC` |
| `sortBy` | string | Não | Campo para ordenação |

---

#### GET /{companyId}/token-collections/{id}

Obter uma coleção de tokens por ID.

#### PUT /{companyId}/token-collections/{id}

Editar um rascunho de coleção. Permitido apenas quando o status é DRAFT.

**Notas:**
- Permitido apenas quando o status da coleção é `draft`
- Alterar `quantity` pode requerer re-sincronização das edições

---

### Ciclo de Vida de Token Collection

#### PATCH /{companyId}/token-collections/{id}/sync-draft-tokens

Gerar ou re-sincronizar rascunhos de edições para uma coleção. Operação assíncrona.

**Resposta (200):**
```json
{ "jobId": "job-uuid" }
```

**Notas:**
- Cria `quantity` edições com `editionNumber` sequencial começando de 1
- Job assíncrono -- faça polling do status do job para conclusão
- Deve ser concluído antes da publicação ser permitida

---

#### PATCH /{companyId}/token-collections/publish/{id}

Publicar um rascunho de coleção na blockchain.

**Verificações de Validação (todas devem passar):**
- Contrato deve estar em status PUBLISHED
- Contrato deve ter funcionalidade ADMIN_MINTER habilitada
- Todas as edições devem estar em status DRAFT (sem estados de erro)
- Contagem de edições deve corresponder à quantidade da coleção
- Se `similarTokens` é true, tokenData é validado contra o template da subcategoria
- Nenhum job assíncrono rodando para esta coleção
- RFIDs devem ser únicos (sem duplicatas na lista, sem conflitos com outras coleções)
- Empresa deve ter endereço de proprietário padrão se `ownerAddress` não estiver definido

**Notas:**
- Na publicação, se `similarTokens` é true, o tokenData da coleção é aplicado a todas as edições
- Edições no intervalo `rangeInitialToMint` são definidas com status READY_TO_MINT
- O template da subcategoria é salvo como snapshot em `publishedTokenTemplate`
- Status muda de `draft` para `published` (irreversível)

---

#### DELETE /{companyId}/token-collections/{id}/draft

Excluir um rascunho de coleção e todas as suas edições. Apenas permitido quando status é `draft`.

#### DELETE /{companyId}/token-collections/{id}/burn

Queimar uma coleção publicada e todas as suas edições mintadas. Apenas permitido quando status é `published`.

#### PATCH /{companyId}/token-collections/{id}/increase-editions

Aumentar o número de edições em uma coleção publicada. Requer papel Integration.

#### PATCH /{companyId}/token-collections/{id}/pass/enable

Marcar uma coleção como token-pass. Token-passes têm comportamento especial no módulo commerce.

#### PATCH /{companyId}/token-collections/{id}/pass/disable

Desmarcar uma coleção como token-pass.

#### GET /{companyId}/token-collections/estimate-gas

Estimar custos de gas para publicação de uma coleção.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `contractId` | uuid | Sim | ID do contrato |
| `initialQuantityToMint` | number | Sim | Número de tokens a mintar |

---

### Importação/Exportação de Token Collection

#### GET /{companyId}/token-collections/{id}/export/xlsx

Exportar template Excel com dados atuais das edições.

**Notas:**
- Colunas do template incluem: editionNumber, name, description, rfid, mais campos customizados do template da subcategoria
- Usado como ponto de partida para importação em lote

#### POST /{companyId}/token-collections/{id}/import/xlsx

Importar edições de um arquivo Excel.

**Requisição:** `multipart/form-data` com campo de arquivo.

**Notas:**
- Operação assíncrona -- faça polling do status do job para conclusão
- Arquivo deve seguir a estrutura do template exportado
- Edições com erros recebem status IMPORT_ERROR
- Coleção deve estar em status DRAFT

---

### CRUD de Token Edition

#### GET /{companyId}/token-editions

Listar edições com paginação e filtros.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | number | Não | Número da página (padrão: 1) |
| `limit` | number | Não | Itens por página (padrão: 10) |
| `tokenCollectionId` | uuid | Não | Filtrar por coleção |
| `status` | string | Não | Filtrar por status da edição |
| `search` | string | Não | Buscar por nome ou número de edição |
| `orderBy` | string | Não | `ASC` ou `DESC` |
| `sortBy` | string | Não | Campo para ordenação |

#### GET /{companyId}/token-editions/{id}

Obter uma edição por ID.

#### PATCH /{companyId}/token-editions/{id}

Editar uma edição. Campos: `name`, `description`, `mainImage`, `tokenData`.

---

### Operações de Status de Token Edition

#### PATCH /{companyId}/token-editions/ready-to-mint

Marcar múltiplas edições como prontas para mintar (em lote).

**Requisição:**
```json
{ "tokenCollectionId": "collection-uuid", "editionNumbers": [1, 2, 3, 4, 5] }
```

#### PATCH /{companyId}/token-editions/{id}/ready-to-mint

Marcar uma única edição como pronta para mintar.

#### PATCH /{companyId}/token-editions/mint-on-demand

Mintar edições específicas sob demanda.

**Requisição:**
```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

**Notas:**
- Edições devem estar em status READY_TO_MINT ou DRAFT
- Define edições com status MINTING e submete transações blockchain
- Coleção deve estar PUBLISHED

#### PATCH /{companyId}/token-editions/locked-for-buy

Bloquear edições para fluxo de compra no commerce.

#### PATCH /{companyId}/token-editions/unlocked-for-buy

Desbloquear edições previamente bloqueadas para compra. Retorna edições ao status DRAFT.

#### PATCH /{companyId}/token-editions/notify-externally-minted

Notificar o sistema que edições foram mintadas externamente (fora da plataforma).

**Notas:**
- Marca edições como MINTED sem a plataforma executar a transação de minting
- Usado para integrações onde o minting acontece em um sistema externo

---

### Metadados de Token Edition

#### PATCH /{companyId}/token-editions/update-token-metadata

Atualizar metadados em uma ou mais edições. Opcionalmente aciona um webhook.

**Requisição Completa:**
```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "tokenData": { "name": "Updated Name", "description": "Updated description" },
  "mergeTokenData": true,
  "keepStatus": true
}
```

**Notas:**
- `mergeTokenData`: quando true, mescla o tokenData fornecido com dados existentes (merge superficial); quando false, substitui inteiramente
- `keepStatus`: quando true, mantém status atual da edição; quando false, pode resetar para draft
- Se `collection.settings.sendWebhookWhenTokenEditionIsUpdated` estiver habilitado, uma notificação de webhook é enviada após a atualização

---

### RFID de Token Edition

#### GET /{companyId}/token-editions/check-rfid

Verificar se um RFID está disponível para atribuição.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `rfid` | string | Sim | Valor do RFID a verificar |

**Notas:**
- RFIDs são globalmente únicos entre todas as edições ativas (excluindo excluídas e queimadas)

#### PATCH /{companyId}/token-editions/{id}/rfid

Atribuir um RFID a uma edição específica.

**Requisição:**
```json
{ "rfid": "RFID-001-ABC" }
```

**Notas:**
- O RFID deve ser globalmente único (verifique com `check-rfid` primeiro)
- Atribuir um novo RFID a uma edição que já possui um substituirá o RFID existente

---

### Transferência de Token Edition

#### PATCH /{companyId}/token-editions/transfer-token

Transferir múltiplas edições para um endereço de carteira (transferência admin em lote).

**Requisição:**
```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Notas:**
- Edições devem estar em status MINTED ou TRANSFERRED
- Define edições com status TRANSFERRING e submete transações blockchain

#### PATCH /{companyId}/token-editions/{id}/transfer-token

Transferir uma única edição para um endereço de carteira.

**Notas:**
- Usuários podem transferir seus próprios tokens (proprietário deve corresponder à carteira do usuário autenticado)

#### PATCH /{companyId}/token-editions/{id}/transfer-token/email

Transferir uma única edição para um usuário por endereço de email.

**Requisição:**
```json
{ "email": "recipient@example.com" }
```

**Notas:**
- O sistema resolve o email para um endereço de carteira
- Se o email não estiver associado a um usuário, a transferência pode ser adiada até o usuário se registrar

---

### Queima de Token Edition

#### DELETE /{companyId}/token-editions/burn

Queimar uma ou mais edições.

**Requisição:**
```json
{
  "tokens": [
    { "tokenCollectionId": "collection-uuid", "editionNumber": 1 },
    { "tokenCollectionId": "collection-uuid", "editionNumber": 2 }
  ]
}
```

**Notas:**
- Edições devem estar em status MINTED ou TRANSFERRED
- Define edições com status BURNING e submete transações blockchain de queima
- Usuários podem queimar seus próprios tokens (proprietário deve corresponder à carteira do usuário autenticado)

---

### Estimativa de Gas de Token Edition

#### GET /{companyId}/token-editions/{id}/estimate-gas/mint

Estimar custo de gas para mintar uma edição específica.

#### GET /{companyId}/token-editions/{id}/estimate-gas/transfer

Estimar custo de gas para transferir uma edição específica.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `toAddress` | string | Sim | Endereço da carteira de destino |

#### GET /{companyId}/token-editions/{id}/estimate-gas/burn

Estimar custo de gas para queimar uma edição específica.

Todos os endpoints de estimativa de gas retornam a mesma estrutura com tiers slow/standard/fast.

---

### Histórico de Transações de Token Edition

#### GET /{companyId}/token-editions/{id}/get-last/{type}

Obter a última transação de um dado tipo para uma edição.

**Parâmetros de Caminho:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `type` | string | Sim | Tipo de transação: `transfer` ou `burn` |

---

### Exportação Assíncrona de Token Edition

#### GET /{companyId}/token-editions/xls

Solicitar exportação assíncrona de edições como arquivo Excel.

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `tokenCollectionId` | uuid | Sim | Coleção a exportar |

**Notas:**
- Operação assíncrona -- faça polling do status do job para a URL de download
- Útil para coleções grandes onde exportação síncrona resultaria em timeout

---

### Reprocessamento de Token Edition

#### PATCH /{companyId}/token-editions/retry-bulk-by-collection

Reprocessar operações em lote que falharam para todas as edições em uma coleção.

**Requisição:**
```json
{ "tokenCollectionId": "collection-uuid" }
```

**Notas:**
- Reprocessa edições com status de erro (DRAFT_ERROR, IMPORT_ERROR, BURN_FAILURE, TRANSFER_FAILURE)
- Reseta o status para o estado anterior apropriado e re-submete a operação

---

## Referência de Erros

| Exceção | Status HTTP | Causa |
|---------|-------------|-------|
| `TokenCollectionNotFoundException` | 404 | ID da coleção não encontrado para a empresa fornecida |
| `TokenCollectionNotDraftException` | 400 | Operação requer status DRAFT mas coleção está PUBLISHED |
| `TokenCollectionNotPublishedException` | 400 | Operação requer status PUBLISHED mas coleção está DRAFT |
| `TokenCollectionPublishValidationException` | 400 | Validação de publicação falhou (ver PublishValidationsEnum para detalhes) |
| `TokenEditionNotFoundException` | 404 | ID da edição não encontrado para a empresa fornecida |
| `TokenEditionInvalidStatusException` | 400 | Edição não está no status requerido para a operação |
| `RfidAlreadyUsedException` | 400 | RFID já está atribuído a outra edição ativa |
| `RfidDuplicateException` | 400 | RFIDs duplicados encontrados na lista submetida |
| `ContractNotFoundException` | 404 | ID do contrato não encontrado |
| `ContractNotPublishedException` | 400 | Contrato não está em status PUBLISHED |
| `ContractNoMinterException` | 400 | Contrato não tem funcionalidade ADMIN_MINTER habilitada |
| `JobRunningException` | 400 | Um job assíncrono ainda está rodando para esta coleção |
| `InvalidRangeException` | 400 | Formato de rangeInitialToMint é inválido |
| `GasEstimationException` | 400 | Estimativa de gas falhou (problema com contrato ou rede) |
