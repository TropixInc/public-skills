---
id: TOKENS_API_REFERENCE
title: "Tokens - API Reference"
module: tokens
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - api
  - tokens
---

# Tokens API Reference

Referencia consolidada de todos os endpoints, schemas, enums e interfaces TypeScript do modulo de Tokens/NFT W3block.

---

## Base URLs

| Servico | Base URL | Descricao |
|---------|----------|-----------|
| Key API | `https://api.w3block.io` | NFTs, metadata, balance, transfers, blockchain |
| Identity API | `https://id.w3block.io` | WalletConnect, autenticacao |

> **Nota:** Em ambientes de staging/dev, substitua pela URL correspondente do ambiente.

---

## Autenticacao

Todos os endpoints (exceto os marcados como publicos) exigem:

```
Authorization: Bearer <access_token>
```

O `access_token` e obtido via login na Identity API (`https://id.w3block.io`). O token deve pertencer a um usuario com role `User` ou superior.

**Endpoints publicos (sem autenticacao):**
- `GET metadata/rfid/{rfid}`
- `GET metadata/address/{contractAddress}/{chainId}/{tokenId}`

---

## Enums

### ChainScan (IDs de Blockchain)

```typescript
export enum ChainScan {
  MAINNET = 1,
  ROPSTEN = 3,
  RINKEBY = 4,
  KOVAN = 42,
  LOCALHOST = 1337,
  MUMBAI = 80001,
  POLYGON = 137,
  ETHEREUM = 1,
}
```

| Valor | Rede | Explorer |
|-------|------|----------|
| `1` | Ethereum Mainnet | `https://etherscan.io` |
| `3` | Ropsten (testnet) | `https://ropsten.etherscan.io` |
| `4` | Rinkeby (testnet) | `https://rinkeby.etherscan.io` |
| `42` | Kovan (testnet) | `https://kovan.etherscan.io` |
| `80001` | Mumbai (Polygon testnet) | `https://mumbai.polygonscan.com` |
| `137` | Polygon Mainnet | `https://polygonscan.com` |

### W3blockAPI (APIs Internas)

```typescript
export enum W3blockAPI {
  ID,      // Identity API - autenticacao, WalletConnect
  KEY,     // Key API - tokens, metadata, balance, transfers
  COMMERCE,// Commerce API - pedidos, checkout
  POLL,    // Poll API - enquetes
  PASS,    // Pass API - token passes
}
```

### PixwayAPIRoutes (Rotas Relevantes)

```typescript
// Tokens / NFTs
NFTS_BY_WALLET = 'metadata/nfts/{address}/{chainId}'
METADATA_BY_RFID = 'metadata/rfid/{rfid}'
METADATA_BY_CHAINADDRESS_AND_TOKENID = 'metadata/address/{contractAddress}/{chainId}/{tokenId}'
METADATA_BY_COLLECTION_ID = 'metadata/{companyId}/{collectionId}'

// Transfers
TRANSFER_TOKEN = '{companyId}/token-editions/{id}/transfer-token'
TRANSFER_TOKEN_EMAIL = '{companyId}/token-editions/{id}/transfer-token/email'
GET_LAST_TRASNFER = '{companyId}/token-editions/{id}/get-last/transfer'
TRANSFER_COIN = '/{companyId}/erc20-tokens/{id}/transfer/user'

// Balance
BALANCE = 'blockchain/balance/{address}/{chainId}'

// WalletConnect
WALLET_CONNECT = '/blockchain/request-session-wallet-connect'
DISCONNECT_WALLET_CONNECT = '/blockchain/disconnect-session-wallet-connect'
```

---

## Interfaces TypeScript

### NFTByWalletDTO

Representa um NFT retornado pela listagem de NFTs por wallet.

```typescript
interface Media {
  raw: string;
  gateway: string;
  thumbnail: string;
}

interface MetadataAtribute {
  value: string;
  trait_type: string;
}

export interface NFTByWalletDTO {
  contract: {
    address: string;
  };
  id: {
    tokenId: string;
    tokenMetadata: {
      tokenType: string; // ex: "ERC721", "ERC1155"
    };
  };
  balance: string;
  title: string;
  description: string;
  tokenUri: {
    raw: string;
    gateway: string;
  };
  media: Array<Media>;
  metadata: {
    image?: string;
    atributes?: Array<MetadataAtribute>;
    timeLastUpdated?: string;
    collectionData: {
      id: string;
      name: string;
      pass: boolean;
    };
    tokenEditionData: {
      editionNumber: number;
      id: string;
      rfid: string;
    };
  };
}
```

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `contract.address` | `string` | Endereco do contrato na blockchain |
| `id.tokenId` | `string` | ID do token no contrato |
| `id.tokenMetadata.tokenType` | `string` | Tipo do token (ERC721, ERC1155) |
| `balance` | `string` | Quantidade possuida (relevante para ERC1155) |
| `title` | `string` | Titulo do NFT |
| `description` | `string` | Descricao do NFT |
| `tokenUri.raw` | `string` | URI original dos metadados |
| `tokenUri.gateway` | `string` | URI via gateway (IPFS proxy) |
| `media` | `Media[]` | Imagens/midias do NFT |
| `metadata.image` | `string?` | URL da imagem principal |
| `metadata.atributes` | `MetadataAtribute[]?` | Atributos (trait_type + value) |
| `metadata.collectionData.id` | `string` | UUID da colecao W3block |
| `metadata.collectionData.name` | `string` | Nome da colecao |
| `metadata.collectionData.pass` | `boolean` | Se a colecao e um pass |
| `metadata.tokenEditionData.editionNumber` | `number` | Numero da edicao |
| `metadata.tokenEditionData.id` | `string` | UUID da edicao (editionId) |
| `metadata.tokenEditionData.rfid` | `string` | RFID vinculado |

### PublicTokenPageDTO

Dados publicos completos de um token, retornado por RFID ou por contrato+tokenId.

```typescript
export interface CompanyTheme {
  headerLogoUrl: string | null;
  headerBackgroundColor: string;
  bodyCardBackgroundColor: string;
}

export interface PublicTokenPageDTO {
  company: {
    id: string;
    name: string;
    theme: CompanyTheme;
  };
  group: {
    categoryName: string;
    categoryId: string;
    subcategoryName: string;
    subcategoryId: string;
    collectionId: string;
    collectionPass: boolean;
  };
  information: {
    title: string;
    mainImage: string | null;
    description: string;
    contractName: string;
  };
  dynamicInformation: {
    tokenData: Record<string, DynamicFormFieldValue>;
    publishedTokenTemplate: DynamicFormConfiguration;
  };
  edition: {
    total: number;
    currentNumber: string;
    rfid: string;
    isMultiple: boolean;
    mintedAt?: string;
    mintedHash?: string;
  };
  token?: {
    tokenId?: string;
    address?: string;
    chainId?: number;
    firstOwnerAddress?: string;
  };
  isMinted: boolean;
}
```

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `company.id` | `string` | UUID da empresa |
| `company.name` | `string` | Nome da empresa |
| `company.theme` | `CompanyTheme` | Tema visual (cores, logo) |
| `group.categoryName` | `string` | Nome da categoria |
| `group.collectionId` | `string` | UUID da colecao |
| `group.collectionPass` | `boolean` | Se a colecao tem pass vinculado |
| `information.title` | `string` | Titulo do token |
| `information.mainImage` | `string?` | URL da imagem principal |
| `information.description` | `string` | Descricao do token |
| `information.contractName` | `string` | Nome do contrato |
| `dynamicInformation.tokenData` | `Record<string, DynamicFormFieldValue>` | Dados dinamicos do token |
| `dynamicInformation.publishedTokenTemplate` | `DynamicFormConfiguration` | Template dos campos dinamicos |
| `edition.total` | `number` | Total de edicoes na colecao |
| `edition.currentNumber` | `string` | Numero da edicao atual |
| `edition.rfid` | `string` | RFID da edicao |
| `edition.isMultiple` | `boolean` | Se aceita multiplas edicoes |
| `edition.mintedAt` | `string?` | Data/hora de mintagem |
| `edition.mintedHash` | `string?` | Hash da transacao de mint |
| `token.tokenId` | `string?` | ID do token na blockchain |
| `token.address` | `string?` | Endereco do contrato |
| `token.chainId` | `number?` | ID da blockchain |
| `token.firstOwnerAddress` | `string?` | Endereco do primeiro dono |
| `isMinted` | `boolean` | Se o token ja foi mintado |

### GetLastTransferAPIResponse

Resposta da consulta da ultima transferencia de um token.

```typescript
export interface GetLastTransferAPIResponse {
  sender: string;
  status: string;
  toAddress: string;
  txHash: string;
  chainId: number;
}
```

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `sender` | `string` | Endereco do remetente |
| `status` | `string` | Status da transferencia (ex: "pending", "completed", "failed") |
| `toAddress` | `string` | Endereco do destinatario |
| `txHash` | `string` | Hash da transacao na blockchain |
| `chainId` | `number` | ID da blockchain onde ocorreu |

### GetBalanceAPIResponse

Resposta da consulta de saldo blockchain.

```typescript
type Currency = 'ETH' | 'MATIC';

interface GetBalanceAPIResponse {
  balance: string;
  currency: Currency;
}
```

| Campo | Tipo | Descricao |
|-------|------|-----------|
| `balance` | `string` | Saldo em wei/unidade nativa (como string) |
| `currency` | `Currency` | Moeda nativa da rede (`ETH` ou `MATIC`) |

### RequestWalletConnectDTO

Payload para conectar wallet via WalletConnect.

```typescript
export interface RequestWalletConnectDTO {
  chainId: number;
  address: string;
  uri?: string;
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `chainId` | `number` | Sim | ID da blockchain |
| `address` | `string` | Sim | Endereco da wallet |
| `uri` | `string` | Nao | URI do WalletConnect (wc:...) |

### MetadataApiInterface

Interface generica para metadados de token retornados pela colecao.

```typescript
export interface MetadataApiInterface<T> {
  id: string;
  createdAt: string;
  updatedAt: string;
  editionNumber: number;
  tokenCollectionId: string;
  companyId: string;
  contractId: string;
  rfid: string;
  status: string;
  contractAddress: string;
  ownerAddress: string;
  chainId: number;
  tokenId: number;
  mintedHash: string;
  mintedAt: string;
  nftMintingId: string;
  name: string;
  description: string;
  mainImage: string;
  tokenData: T;
  errorFields: any;
  tokenCollection: TokenCollection<any>;
  contract: any;
  mintedAddress: string;
}

export interface TokenCollection<T> {
  id: string;
  createdAt: string;
  updatedAt: string;
  name: string;
  description: string;
  mainImage: string;
  companyId: string;
  contractId: string;
  subcategoryId: string;
  tokenData: T;
  publishedTokenTemplate: any;
  quantity: number;
  initialQuantityToMint: number;
  rangeInitialToMint: string;
  quantityMinted: number;
  rfids: string[];
  ownerAddress: string;
  similarTokens: boolean;
  pass: false;
}
```

---

## Endpoints

### 1. NFTs por Wallet

Lista todos os NFTs mintados pelo W3block em uma wallet especifica.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true` |
| **API** | Key API (`https://api.w3block.io`) |
| **Auth** | Bearer token (requer perfil com wallet) |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `address` | `string` | Endereco da wallet (0x...) |
| `chainId` | `number` | ID da blockchain (1, 137, 80001) |

#### Query Parameters

| Parametro | Tipo | Obrigatorio | Descricao |
|-----------|------|-------------|-----------|
| `onlyMintedByWeblock` | `boolean` | Sim | Filtrar apenas NFTs mintados pelo W3block (usar `true`) |

#### Response (200 OK)

```json
{
  "items": [
    {
      "contract": { "address": "0x1234...abcd" },
      "id": {
        "tokenId": "1",
        "tokenMetadata": { "tokenType": "ERC721" }
      },
      "balance": "1",
      "title": "Meu NFT #1",
      "description": "Descricao do NFT",
      "tokenUri": {
        "raw": "ipfs://...",
        "gateway": "https://gateway.ipfs.io/..."
      },
      "media": [
        {
          "raw": "ipfs://...",
          "gateway": "https://gateway.ipfs.io/...",
          "thumbnail": "https://gateway.ipfs.io/.../thumbnail"
        }
      ],
      "metadata": {
        "image": "https://...",
        "atributes": [
          { "value": "Raro", "trait_type": "Raridade" }
        ],
        "collectionData": {
          "id": "uuid-colecao",
          "name": "Minha Colecao",
          "pass": false
        },
        "tokenEditionData": {
          "editionNumber": 1,
          "id": "uuid-edition",
          "rfid": "RFID-001"
        }
      }
    }
  ],
  "meta": {
    "totalItems": 25,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 3,
    "currentPage": 1
  }
}
```

---

### 2. Metadados por Colecao

Lista edicoes de token de uma colecao especifica.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `metadata/{companyId}/{collectionId}` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `uuid` | UUID da empresa |
| `collectionId` | `uuid` | UUID da colecao |

#### Query Parameters

| Parametro | Tipo | Default | Descricao |
|-----------|------|---------|-----------|
| `page` | `number` | `1` | Pagina |
| `limit` | `number` | `10` | Itens por pagina |
| `sortBy` | `string` | `createdAt` | Campo de ordenacao |
| `orderBy` | `string` | `DESC` | Direcao (ASC/DESC) |
| `walletAddresses` | `string` | - | Filtrar por enderecos de wallet |

#### Response (200 OK)

```json
{
  "items": [
    {
      "id": "uuid-edition",
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "editionNumber": 1,
      "tokenCollectionId": "uuid-colecao",
      "companyId": "uuid-empresa",
      "contractId": "uuid-contrato",
      "rfid": "RFID-001",
      "status": "minted",
      "contractAddress": "0x1234...abcd",
      "ownerAddress": "0xabcd...1234",
      "chainId": 137,
      "tokenId": 1,
      "mintedHash": "0xhash...",
      "mintedAt": "2024-01-15T10:30:00Z",
      "name": "Token #1",
      "description": "Descricao",
      "mainImage": "https://..."
    }
  ]
}
```

---

### 3. Metadados por RFID

Consulta dados publicos de um token a partir do seu RFID.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `metadata/rfid/{rfid}` |
| **API** | Key API |
| **Auth** | Publico (sem autenticacao) |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `rfid` | `string` | Identificador RFID do token |

#### Response (200 OK)

Retorna `PublicTokenPageDTO` (ver interface completa acima).

---

### 4. Metadados por Contrato + Token ID

Consulta dados publicos de um token pelo endereco do contrato, chainId e tokenId.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `metadata/address/{contractAddress}/{chainId}/{tokenId}` |
| **API** | Key API |
| **Auth** | Publico (sem autenticacao) |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `contractAddress` | `string` | Endereco do contrato (0x...) |
| `chainId` | `string` | ID da blockchain |
| `tokenId` | `string` | ID do token no contrato |

#### Response (200 OK)

Retorna `PublicTokenPageDTO` (ver interface completa acima).

---

### 5. Transferir Token (por Address)

Transfere um NFT para outro endereco de wallet.

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `{companyId}/token-editions/{id}/transfer-token` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `uuid` | UUID da empresa |
| `id` | `uuid` | UUID da edicao do token (editionId) |

#### Request Body

```json
{
  "toAddress": "0xdestinatario..."
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `toAddress` | `string` | Sim | Endereco da wallet de destino |

#### Response (200 OK)

Retorna dados da transferencia iniciada.

---

### 6. Transferir Token (por Email)

Transfere um NFT para um usuario identificado por email.

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `{companyId}/token-editions/{id}/transfer-token/email` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `uuid` | UUID da empresa |
| `id` | `uuid` | UUID da edicao do token (editionId) |

#### Request Body

```json
{
  "email": "destinatario@exemplo.com"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `email` | `string` | Sim | Email do destinatario |

#### Response (200 OK)

Retorna dados da transferencia iniciada.

---

### 7. Consultar Ultima Transferencia

Consulta o status da ultima transferencia de uma edicao de token.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `{companyId}/token-editions/{id}/get-last/transfer` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `uuid` | UUID da empresa |
| `id` | `uuid` | UUID da edicao do token (editionId) |

#### Response (200 OK)

```json
{
  "sender": "0xremetente...",
  "status": "completed",
  "toAddress": "0xdestinatario...",
  "txHash": "0xhash-da-transacao...",
  "chainId": 137
}
```

---

### 8. Saldo Blockchain

Consulta o saldo nativo (ETH/MATIC) de uma wallet em uma blockchain.

| Campo | Valor |
|-------|-------|
| **Metodo** | `GET` |
| **Path** | `blockchain/balance/{address}/{chainId}` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `address` | `string` | Endereco da wallet (0x...) |
| `chainId` | `number` | ID da blockchain |

#### Response (200 OK)

```json
{
  "balance": "1500000000000000000",
  "currency": "MATIC"
}
```

> **Nota:** O `balance` e retornado em wei (menor unidade). Para converter para unidade legivel: `parseFloat(balance) / 1e18`.

---

### 9. Transferir ERC-20

Transfere tokens ERC-20 (moedas/pontos) entre usuarios.

| Campo | Valor |
|-------|-------|
| **Metodo** | `PATCH` |
| **Path** | `/{companyId}/erc20-tokens/{id}/transfer/user` |
| **API** | Key API |
| **Auth** | Bearer token |

#### Path Parameters

| Parametro | Tipo | Descricao |
|-----------|------|-----------|
| `companyId` | `uuid` | UUID da empresa |
| `id` | `uuid` | UUID do token ERC-20 (loyaltyId) |

#### Request Body

```json
{
  "to": "0xdestinatario...",
  "from": "0xremetente...",
  "amount": "100",
  "metadata": {
    "description": "Transferencia de pontos"
  },
  "sendEmail": true
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `to` | `string` | Sim | Endereco da wallet de destino |
| `from` | `string` | Sim | Endereco da wallet de origem |
| `amount` | `string` | Sim | Quantidade a transferir |
| `metadata.description` | `string` | Sim | Descricao da transferencia |
| `sendEmail` | `boolean` | Nao | Se deve enviar email de notificacao (default: true) |

---

### 10. WalletConnect - Solicitar Sessao

Inicia uma sessao WalletConnect para conectar wallet externa.

| Campo | Valor |
|-------|-------|
| **Metodo** | `POST` |
| **Path** | `/blockchain/request-session-wallet-connect` |
| **API** | Identity API (`https://id.w3block.io`) |
| **Auth** | Bearer token |

> **IMPORTANTE:** Este e o unico endpoint de tokens que usa a Identity API (W3blockAPI.ID), nao a Key API.

#### Request Body

```json
{
  "chainId": 137,
  "address": "0xminha-wallet...",
  "uri": "wc:a1b2c3@1?bridge=https://bridge.walletconnect.org&key=abc123"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `chainId` | `number` | Sim | ID da blockchain |
| `address` | `string` | Sim | Endereco da wallet do usuario |
| `uri` | `string` | Nao | URI do WalletConnect (formato wc:...) |

---

### 11. WalletConnect - Desconectar Sessao

Desconecta uma sessao WalletConnect ativa.

| Campo | Valor |
|-------|-------|
| **Metodo** | - |
| **Path** | `/blockchain/disconnect-session-wallet-connect` |
| **API** | Identity API (`https://id.w3block.io`) |
| **Auth** | Bearer token |

---

## Formato Padrao de Erros

Todas as APIs W3block retornam erros no seguinte formato:

```json
{
  "statusCode": 400,
  "message": "mensagem-de-erro",
  "error": "Bad Request"
}
```

Quando `message` e um array:

```json
{
  "statusCode": 400,
  "message": ["campo-x must be a string", "campo-y should not be empty"],
  "error": "Bad Request"
}
```

### Exemplos de Respostas de Erro

| Status | Exemplo `message` | Causa |
|--------|-------------------|-------|
| `400` | `"chainId must be a number"` | Parametro com tipo errado |
| `401` | `"Unauthorized"` | Token ausente ou expirado |
| `403` | `"Forbidden"` | Sem permissao (role insuficiente ou nao e dono do token) |
| `404` | `"Not Found"` | Recurso inexistente (editionId, rfid invalido) |
| `409` | `"Transfer already in progress"` | Transferencia anterior ainda nao concluiu |

---

## Erros Comuns

| HTTP Status | Erro | Descricao | Solucao |
|-------------|------|-----------|---------|
| `401` | Unauthorized | Token ausente ou expirado | Renovar access_token via refresh-token |
| `403` | Forbidden | Sem permissao para esta operacao | Verificar role do usuario e propriedade do token |
| `404` | Not Found | Recurso nao encontrado | Verificar IDs (companyId, editionId, rfid) |
| `400` | Bad Request | Parametros invalidos | Verificar formato dos parametros (chainId numerico, address 0x...) |
| `409` | Conflict | Transferencia ja em andamento | Aguardar conclusao da transferencia anterior |
