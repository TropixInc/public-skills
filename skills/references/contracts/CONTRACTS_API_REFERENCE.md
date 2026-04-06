---
id: CONTRACTS_API_REFERENCE
title: "Contratos e Tokens - Referência da API"
module: offpix/contracts
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contracts
  - tokens
  - nft
  - erc20
  - collections
  - editions
  - royalty
  - blockchain
  - api-reference
---

# Referência da API de Contratos e Tokens

Referência completa de endpoints para o módulo de Contratos e Tokens da W3Block. Cobre contratos NFT (ERC721A), coleções e edições de tokens, tokens fungíveis ERC20, gerenciamento de royalties, estimativa de gas e operações blockchain. Todos servidos pelo backend Pixway Registry.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://api.w3block.io` |
| Staging | *(ambiente de staging disponível — use a URL base de staging)* |
| Swagger | https://api.w3block.io/docs/ |
| Certificados PDF | `https://pdf.w3block.io` |

## Autenticação

Todos os endpoints requerem Bearer token a menos que indicado:

```
Authorization: Bearer {accessToken}
```

Todos os endpoints possuem escopo por empresa via parâmetro de caminho `{companyId}`.

---

## Enums

### ContractStatus (NFT e ERC20)

| Valor | Descrição |
|-------|-----------|
| `draft` | Estado inicial, editável |
| `publishing` | Deploy na blockchain em andamento |
| `published` | Ativo na blockchain |
| `failed` | Deploy falhou (pode tentar novamente) |

### NftContractFeature

| Valor | Descrição |
|-------|-----------|
| `admin:minter` | Admin pode mintar novos tokens |
| `admin:burner` | Admin pode queimar tokens |
| `admin:mover` | Admin pode transferir tokens |
| `user:burner` | Usuários podem queimar seus próprios tokens |
| `user:mover` | Usuários podem transferir seus próprios tokens |

### ContractOperatorRole

| Valor | Descrição |
|-------|-----------|
| `minter` | Pode mintar novos tokens |
| `mover` | Pode transferir tokens |

### ERC20ContractType

| Valor | Label no Frontend | Descrição |
|-------|-------------------|-----------|
| `classic` | Classic | ERC20 padrão — transferências diretas na blockchain |
| `permissioned` | Permissioned | Regras de transferência impostas (taxa, limites, período) |
| `in_custody` | In Custody | Empresa mantém tokens em nome dos usuários |

### TokenCollectionStatus

| Valor | Descrição |
|-------|-----------|
| `draft` | Editável, ainda não publicado |
| `published` | Publicado na blockchain |

### TokenEditionStatusEnum

| Valor | Label no Frontend | Descrição |
|-------|-------------------|-----------|
| `draft` | Rascunho | Estado inicial |
| `draftError` | Erro de Rascunho | Erro de validação de dados |
| `importError` | Erro de Importação | Validação de importação XLS falhou |
| `readyToMint` | Pronto para Mintar | Validado, aguardando publicação |
| `lockedForBuy` | Bloqueado para Compra | Reservado para compra |
| `minting` | Mintando | Deploy na blockchain em andamento |
| `minted` | Mintado | Na blockchain com sucesso |
| `burning` | Queimando | Queima em andamento |
| `burned` | Queimado | Destruído |
| `burnFailure` | Falha na Queima | Transação de queima falhou |
| `transferring` | Transferindo | Transferência em andamento |
| `transferred` | Transferido | Transferido com sucesso |
| `transferFailure` | Falha na Transferência | Transação de transferência falhou |

### ERC20TransferConfigModel

| Valor | Descrição |
|-------|-----------|
| `FREE` | Sem restrições (ou política forbidden/erc20ReceiverOnly) |
| `FIXED` | Taxa fixa deduzida por transferência |
| `PERCENTAGE` | Taxa percentual deduzida por transferência |
| `PERIOD` | Máximo de transferências dentro de uma janela de tempo |
| `MAX_LIMIT` | Máximo total de transferências por usuário |

### ChainId (Blockchains Suportadas)

| Valor | Rede |
|-------|------|
| `1` | Ethereum Mainnet |
| `137` | Polygon Mainnet |
| `80001` | Polygon Mumbai (testnet) |
| `1284` | Moonbeam |
| `1337` | Localhost (dev) |

---

## Entidades e Relacionamentos

```
NftContract (1) ──→ (1) RoyaltyContract ──→ (N) RoyaltyEligible
     │                                              │
     │                                              └── ExternalContact ou User
     │
     └──→ (N) TokenCollection ──→ (N) TokenEdition
               │                        │
               │ name, quantity,         │ editionNumber, status,
               │ tokenData template,     │ ownerAddress, rfid,
               │ subcategory             │ tokenId, mintedHash
               │
               └── Subcategory (definição do template)

ERC20Contract (standalone)
     │
     └──→ transferConfig[] (regras de transferência)
     └──→ (N) ERC20TransferUserAction (trilha de auditoria)
```

---

## Endpoints: Contratos NFT

Caminho base: `/{companyId}/contracts`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar rascunho de contrato NFT |
| 2 | GET | `/` | Bearer | Admin | Listar contratos NFT da empresa |
| 3 | GET | `/:id` | Bearer | Admin | Obter detalhes do contrato |
| 4 | PATCH | `/:id` | Bearer | Admin | Atualizar contrato em rascunho |
| 5 | PATCH | `/:id/publish` | Bearer | Admin | Fazer deploy na blockchain |
| 6 | GET | `/:id/estimate-gas` | Bearer | Admin | Estimar gas do deploy (cache 1 min) |
| 7 | PATCH | `/has-role` | Bearer | Admin, Integration | Verificar se endereço tem papel |
| 8 | PATCH | `/grant-role` | Bearer | Admin, Integration | Conceder papel a endereço |

### POST /{companyId}/contracts

Criar um novo rascunho de contrato NFT com configuração de royalty.

**Requisição Mínima:**
```json
{
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "participants": []
}
```

**Requisição Completa (exemplo de produção):**
```json
{
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "description": "A digital art collection",
  "image": "https://example.com/contract-image.png",
  "externalLink": "https://example.com",
  "participants": [
    {
      "name": "Artist",
      "payee": "0x1234567890abcdef1234567890abcdef12345678",
      "share": 5.0
    },
    {
      "name": "Platform",
      "payee": "0xabcdef1234567890abcdef1234567890abcdef12",
      "share": 2.5
    }
  ],
  "features": ["admin:minter", "admin:burner", "user:mover"],
  "transferWhitelistId": "whitelist-uuid",
  "minterWhitelistId": "whitelist-uuid",
  "maxSupply": "10000"
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `name` | string | Sim | — | Nome do contrato (latinizado, alfanumérico) |
| `symbol` | string | Sim | — | Símbolo do token (maiúsculas, alfanumérico) |
| `chainId` | ChainId | Sim | — | Rede blockchain de destino |
| `description` | string | Não | — | Descrição do contrato |
| `image` | URL | Não | — | Imagem do contrato |
| `externalLink` | URL | Não | — | Link externo |
| `participants` | array | Sim | `[]` | Destinatários de royalty (ver abaixo) |
| `features` | NftContractFeature[] | Não | — | Capacidades habilitadas |
| `transferWhitelistId` | UUID | Não | — | Whitelist para restrições de transferência |
| `minterWhitelistId` | UUID | Não | — | Whitelist para restrições de mint |
| `maxSupply` | string (bigint) | Não | `null` | Máximo de tokens mintáveis (null = ilimitado) |

**Objeto de participante:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do participante |
| `payee` | string | Sim | Endereço Ethereum para pagamentos de royalty |
| `share` | number | Sim | Porcentagem de royalty (0.01–10.0) |
| `contactId` | UUID | Não | Link para registro RoyaltyEligible |

**Resposta (201):**
```json
{
  "id": "contract-uuid",
  "companyId": "company-uuid",
  "name": "My NFT Collection",
  "symbol": "MNC",
  "chainId": 137,
  "address": null,
  "status": "draft",
  "features": ["admin:minter", "admin:burner", "user:mover"],
  "maxSupply": "10000",
  "royalty": {
    "id": "royalty-uuid",
    "fee": 7.5,
    "participants": [
      { "name": "Artist", "payee": "0x1234...", "share": 5.0 },
      { "name": "Platform", "payee": "0xabcd...", "share": 2.5 }
    ],
    "status": "draft"
  },
  "operators": [],
  "contractAction": null,
  "createdAt": "2026-04-01T10:00:00Z"
}
```

### PATCH /{companyId}/contracts/:id/publish

Faz deploy do contrato na blockchain. Funciona apenas com status `draft` ou `failed`.

**Resposta:** 204 No Content

**Transição de estado:** `draft` → `publishing` → (assíncrono) → `published` ou `failed`

O deploy cria uma entidade `ContractAction` que rastreia a transação blockchain. O deploy real é processado assincronamente — faça polling do status do contrato ou use webhooks.

### GET /{companyId}/contracts/:id/estimate-gas

Retorna estimativa de gas para deploy do contrato. Cache de 1 minuto.

**Resposta (200):**
```json
{
  "totalGas": 5000000,
  "totalGasPrice": {
    "fast": "25000000000",
    "proposed": "20000000000",
    "safe": "15000000000"
  }
}
```

---

## Endpoints: Coleções de Tokens

Caminho base: `/{companyId}/token-collections`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar rascunho de coleção |
| 2 | GET | `/` | Bearer | Admin | Listar coleções (paginado, filtrável) |
| 3 | GET | `/:id` | Bearer | Admin | Obter detalhes da coleção |
| 4 | PUT | `/:id` | Bearer | Admin | Atualizar coleção em rascunho |
| 5 | PATCH | `/publish/:id` | Bearer | Admin | Publicar coleção na blockchain |
| 6 | PATCH | `/:id/sync-draft-tokens` | Bearer | Admin | Criar edições em rascunho (assíncrono) |
| 7 | PATCH | `/:id/increase-editions` | Bearer | Integration | Aumentar quantidade de edições (assíncrono) |
| 8 | GET | `/estimate-gas` | Bearer | Admin | Estimar gas de publicação (cache 1 min) |
| 9 | PATCH | `/:id/pass/enable` | Bearer | Integration | Habilitar token pass |
| 10 | PATCH | `/:id/pass/disable` | Bearer | Integration | Desabilitar token pass |
| 11 | DELETE | `/:id/burn` | Bearer | Admin | Queimar coleção inteira |
| 12 | DELETE | `/:id/draft` | Bearer | Admin | Excluir coleção em rascunho |
| 13 | GET | `/:id/export/xlsx` | Bearer | Admin | Exportar edições como XLSX |
| 14 | POST | `/:id/import/xlsx` | Bearer | Admin | Importar edições de XLSX (assíncrono) |

### POST /{companyId}/token-collections

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
  "contractId": "contract-uuid",
  "subcategoryId": "subcategory-uuid",
  "name": "Genesis Collection",
  "description": "First edition collectibles",
  "mainImage": "https://example.com/collection.png",
  "tokenData": {
    "artist": "John Doe",
    "year": 2026,
    "medium": "Digital Art"
  },
  "quantity": 100,
  "rangeInitialToMint": "1-100",
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "similarTokens": true,
  "settings": {
    "sendWebhookWhenTokenEditionIsUpdated": true
  }
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `contractId` | UUID | Não | — | Contrato NFT associado |
| `subcategoryId` | UUID | Sim | — | Template/subcategoria da coleção |
| `name` | string | Sim | — | Nome da coleção |
| `description` | string | Não | — | Descrição |
| `mainImage` | URL | Não | — | Imagem de capa |
| `tokenData` | object | Não | `{}` | Campos de metadados customizados (definidos pelo template) |
| `quantity` | integer | Não | `0` | Número de edições |
| `rangeInitialToMint` | string | Não | — | Notação de intervalo para mints iniciais (ex: "1-100") |
| `ownerAddress` | string | Não | — | Endereço Ethereum para tokens mintados |
| `similarTokens` | boolean | Não | `true` | Se as edições compartilham mesmos metadados |
| `settings` | object | Não | — | Configurações da coleção |

### PATCH /{companyId}/token-collections/publish/:id

Publica a coleção — valida todas as edições, minta tokens marcados como `readyToMint`.

**Resposta (201):**
```json
{
  "tokenCollection": { "..." : "..." },
  "validationErrors": []
}
```

Se a validação falhar, retorna erros detalhados por campo usando `PublishValidationsEnum`.

### PATCH /{companyId}/token-collections/:id/sync-draft-tokens

Cria edições de tokens em rascunho para a coleção. Operação assíncrona via fila Bull.

**Resposta (201):**
```json
{
  "jobId": "job-uuid"
}
```

### GET/POST /{companyId}/token-collections/:id/export/xlsx / import/xlsx

- **Exportar:** Baixa template XLSX com dados atuais das edições
- **Importar:** Upload de XLSX com dados de edições (FormData, máx 10MB). Retorna `{ jobId }` para processamento assíncrono

---

## Endpoints: Edições de Tokens

Caminho base: `/{companyId}/token-editions`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | GET | `/` | Bearer | Admin | Listar edições (paginado, filtrável) |
| 2 | GET | `/:id` | Bearer | Admin | Obter detalhes da edição |
| 3 | GET | `/xls` | Bearer | Admin | Solicitar relatório XLS |
| 4 | GET | `/check-rfid` | Bearer | Admin | Verificar disponibilidade de RFID |
| 5 | PATCH | `/ready-to-mint` | Bearer | Admin | Marcar edições como prontas para mintar |
| 6 | PATCH | `/mint-on-demand` | Bearer | Admin, Integration, Application | Mintar edições específicas |
| 7 | PATCH | `/locked-for-buy` | Bearer | Admin, Integration, Application | Bloquear edições para compra |
| 8 | PATCH | `/unlocked-for-buy` | Bearer | Admin, Integration, Application | Desbloquear edições |
| 9 | PATCH | `/notify-externally-minted` | Bearer | Admin, Integration, Application | Registrar mint externo |
| 10 | PATCH | `/update-token-metadata` | Bearer | Admin, Integration, Application | Atualizar metadados da edição |
| 11 | PATCH | `/transfer-token` | Bearer | Admin | Transferir tokens (em lote) |
| 12 | PATCH | `/:id/transfer-token` | Bearer | Admin | Transferir token único |
| 13 | DELETE | `/burn` | Bearer | Admin | Queimar tokens |
| 14 | PATCH | `/:id/rfid` | Bearer | Admin | Atribuir/atualizar RFID |
| 15 | GET | `/:id/estimate-gas/mint` | Bearer | Admin | Estimar gas de mint |
| 16 | GET | `/:id/estimate-gas/transfer` | Bearer | Admin | Estimar gas de transferência |
| 17 | GET | `/:id/estimate-gas/burn` | Bearer | Admin | Estimar gas de queima |
| 18 | GET | `/:id/get-last/transfer` | Bearer | Admin | Obter último status de transferência |

### PATCH /{companyId}/token-editions/ready-to-mint

Marcar edições como prontas para mintar (transição de `draft` para `readyToMint`).

**Requisição:**
```json
{
  "editionId": ["edition-uuid-1", "edition-uuid-2"]
}
```

### PATCH /{companyId}/token-editions/mint-on-demand

Mintar edições específicas imediatamente.

**Requisição:**
```json
{
  "editionNumbers": [1, 2, 3],
  "collectionId": "collection-uuid",
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

### PATCH /{companyId}/token-editions/transfer-token

Transferir tokens entre carteiras.

**Requisição:**
```json
{
  "toAddress": "0xrecipient...",
  "editionId": ["edition-uuid-1", "edition-uuid-2"]
}
```

### DELETE /{companyId}/token-editions/burn

Queimar (destruir) edições de tokens.

**Requisição:**
```json
{
  "tokens": ["edition-uuid-1", "edition-uuid-2"]
}
```

### PATCH /{companyId}/token-editions/update-token-metadata

Atualizar metadados de edições publicadas.

**Requisição:**
```json
{
  "tokenEditionIds": ["edition-uuid-1"],
  "tokenData": {
    "artist": "Updated Artist Name",
    "year": 2027
  }
}
```

### GET /{companyId}/token-editions/check-rfid

Verificar se um RFID já está em uso.

**Query:** `?rfid=RFID-VALUE`

**Resposta:** `{ "used": true/false }`

---

## Endpoints: Contratos ERC20

Caminho base: `/{companyId}/erc20-contracts`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/` | Bearer | Admin | Criar rascunho de contrato ERC20 |
| 2 | GET | `/` | Bearer | Admin | Listar contratos ERC20 |
| 3 | GET | `/:id` | Bearer | Admin | Obter detalhes do contrato ERC20 |
| 4 | PATCH | `/:id` | Bearer | Admin | Atualizar ERC20 em rascunho |
| 5 | PATCH | `/:id/transfer-config` | Bearer | Admin | Atualizar regras de transferência |
| 6 | PATCH | `/:id/publish` | Bearer | Admin | Fazer deploy do ERC20 na blockchain |
| 7 | GET | `/:id/estimate-gas` | Bearer | Admin | Estimar gas do deploy |
| 8 | PATCH | `/has-role` | Bearer | Admin, Integration | Verificar papel |
| 9 | PATCH | `/grant-role` | Bearer | Admin, Integration | Conceder papel |

### POST /{companyId}/erc20-contracts

**Requisição Mínima:**
```json
{
  "name": "Loyalty Token",
  "symbol": "LYT",
  "chainId": 137,
  "type": "classic"
}
```

**Requisição Completa:**
```json
{
  "name": "Loyalty Token",
  "symbol": "LYT",
  "chainId": 137,
  "type": "permissioned",
  "initialAmount": "1000000",
  "initialOwner": "0x1234567890abcdef1234567890abcdef12345678",
  "isBurnable": true,
  "isWithdrawable": false,
  "transferConfig": [
    { "model": "PERCENTAGE", "value": "0.05" },
    { "model": "MAX_LIMIT", "value": "10" },
    { "model": "PERIOD", "value": "5", "period": "7d" }
  ]
}
```

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `name` | string | Sim | — | Nome do token (latinizado) |
| `symbol` | string | Sim | — | Símbolo do token (maiúsculas) |
| `chainId` | ChainId | Sim | — | Blockchain de destino |
| `type` | ERC20ContractType | Sim | — | `classic`, `permissioned` ou `in_custody` |
| `initialAmount` | string (bigint) | Não | — | Quantidade inicial de mint |
| `initialOwner` | string | Não | — | Endereço do proprietário inicial |
| `isBurnable` | boolean | Não | `false` | Habilitar capacidade de queima |
| `isWithdrawable` | boolean | Não | `false` | Habilitar saque |
| `transferConfig` | array | Não | `[]` | Regras de transferência (para permissioned/in_custody) |
| `maxSupply` | string (bigint) | Não | — | Suprimento máximo |

**Objeto de configuração de transferência:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `model` | ERC20TransferConfigModel | Sim | Tipo de regra |
| `value` | string | Sim | Valor da regra (percentual 0–1, contagem ou valor fixo) |
| `period` | string | Condicional | Janela de tempo para modelo PERIOD (ex: "7d", "24h") |

---

## Endpoints: Operações de Token ERC20

Caminho base: `/{companyId}/erc20-tokens`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | PATCH | `/:id/mint` | Bearer | Admin | Mintar novos tokens |
| 2 | PATCH | `/:id/transfer/admin` | Bearer | Admin | Transferência admin (ignora regras) |
| 3 | PATCH | `/:id/transfer/user` | Bearer | Admin, User | Transferência de usuário (regras aplicadas) |
| 4 | PATCH | `/:id/burn` | Bearer | Admin, User | Queimar tokens |
| 5 | GET | `/:id/history` | Bearer | Admin, LoyaltyOperator | Histórico de transações (cache 10s) |
| 6 | GET | `/:id/history/action/:actionId` | Bearer | Admin | Detalhes de ação específica |
| 7 | GET | `/:id/history/operator/:operatorId` | Bearer | Admin, LoyaltyOperator | Atividade do operador |
| 8 | GET | `/:id/history/:userId` | Bearer | Admin, User | Atividade do usuário |

### PATCH /{companyId}/erc20-tokens/:id/mint

**Requisição:**
```json
{
  "to": "0xrecipient...",
  "amount": "1000",
  "metadata": { "reason": "Welcome bonus" },
  "sendEmail": true,
  "available": "1s"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `to` | string | Sim | Endereço Ethereum do destinatário |
| `amount` | string | Sim | Quantidade a mintar |
| `metadata` | object | Não | Metadados customizados para a transação |
| `sendEmail` | boolean | Não | Enviar email de notificação (padrão: true) |
| `available` | string | Não | Expressão de atraso de disponibilidade (ex: "1s", "24h") |

### PATCH /{companyId}/erc20-tokens/:id/transfer/user

Transferência iniciada pelo usuário. Para contratos `permissioned` e `in_custody`, isso passa pela **cadeia de handlers de transferência**:

1. **FreeTransferHandler** — verifica se transferências são gratuitas, proibidas ou restritas
2. **MaxLimitTransferHandler** — verifica se o usuário não excedeu a contagem máxima de transferências
3. **PeriodTransferHandler** — verifica transferências dentro da janela de tempo
4. **PercentageTransferHandler** — deduz taxa percentual (cria 2 transações)
5. **FixedTransferHandler** — deduz taxa fixa (cria 2 transações)

Para contratos `classic`, a transferência é executada diretamente.

**Requisição:** Mesmo que transferência admin.

### PATCH /{companyId}/erc20-tokens/:id/burn

**Requisição:**
```json
{
  "from": "0xburner...",
  "amount": "500",
  "metadata": { "reason": "Redemption" },
  "sendEmail": true
}
```

---

## Endpoints: Elegibilidade a Royalty

Caminho base: `/{companyId}/contracts/royalty-eligible`

| # | Método | Caminho | Auth | Papéis | Descrição |
|---|--------|---------|------|--------|-----------|
| 1 | POST | `/create` | Bearer | Admin | Criar registro de elegibilidade a royalty |
| 2 | GET | `/` | Bearer | Admin | Listar elegíveis a royalty |
| 3 | GET | `/:id` | Bearer | Admin | Obter detalhes |
| 4 | PATCH | `/` | Bearer | Admin | Ativar/desativar |

### POST /{companyId}/contracts/royalty-eligible/create

```json
{
  "active": true,
  "displayName": "Artist Name",
  "userId": "user-uuid"
}
```

---

## Endpoints: Metadados e Certificados (Público)

| # | Método | Caminho | Auth | Descrição |
|---|--------|---------|------|-----------|
| 1 | GET | `/metadata/rfid/{rfid}` | None | Obter metadados do token por RFID |
| 2 | GET | `/metadata/address/{contractAddress}/{chainId}/{tokenId}` | None | Obter metadados por endereço de cadeia + token ID |
| 3 | GET | `/metadata/contract/{address}/{chainId}` | None | Obter metadados do contrato |
| 4 | GET | `/metadata/nfts/{walletAddress}/{chainId}` | None | Listar NFTs por carteira |
| 5 | GET | `/certification/{address}/{chainId}/{tokenId}` | None | Obter certificado PDF |
| 6 | POST | `/certification/preview` | Bearer | Pré-visualizar certificado PDF |

---

## Subcategorias (Templates de Coleção)

Caminho base: `/{companyId}/subcategories`

| # | Método | Caminho | Auth | Descrição |
|---|--------|---------|------|-----------|
| 1 | POST | `/` | Bearer (Admin) | Criar template |
| 2 | PATCH | `/:id` | Bearer (Admin) | Atualizar template |
| 3 | GET | `/` | Bearer (Admin) | Listar templates |
| 4 | GET | `/:id` | Bearer (Admin) | Obter template |

Templates definem os campos de metadados dinâmicos para coleções de tokens (tipos de campo: DATE, BOOLEAN, DIMENSIONS_2D, DIMENSIONS_3D, NUMERIC, RADIOGROUP, SELECT, IMAGE, TEXTAREA, TEXTFIELD, YEAR).
