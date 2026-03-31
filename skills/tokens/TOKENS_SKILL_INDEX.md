---
id: TOKENS_SKILL_INDEX
title: "Tokens Skill Index"
module: tokens
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
---

# Tokens Skill Index

Indice master de toda a documentacao do modulo Tokens/NFT do W3block. Use este documento como ponto de entrada para implementar qualquer funcionalidade de tokens, NFTs, transferencias, saldo blockchain e WalletConnect.

> **Swagger (documentacao interativa):** Acesse `https://api.w3block.io/docs` para testar endpoints e ver schemas atualizados.

---

## Getting Started

Se voce esta implementando pela primeira vez, siga estes passos:

```
1. Autentique-se na Identity API (https://id.w3block.io) -> obtenha Bearer token
2. GET  metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true  -> Liste NFTs da wallet
3. GET  metadata/rfid/{rfid}  OU  metadata/address/{contract}/{chainId}/{tokenId}  -> Dados publicos do token
4. Escolha o fluxo que precisa implementar na Tabela de Decisao abaixo
```

> Para detalhes sobre endpoints, schemas e exemplos de codigo, consulte [TOKENS_API_REFERENCE.md](./TOKENS_API_REFERENCE.md).

---

## Documentos

| # | Documento | Versão | Descricao | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [TOKENS_API_REFERENCE.md](./TOKENS_API_REFERENCE.md) | 1.0.0 | Endpoints, schemas JSON, enums, interfaces TypeScript, erros | Referencia | Consulta de API em qualquer momento |
| 2 | [FLOW_TOKENS_NFT_WALLET.md](./FLOW_TOKENS_NFT_WALLET.md) | 1.0.0 | Listar NFTs da wallet do usuario autenticado | Implementado | Exibir colecao de NFTs do usuario |
| 3 | [FLOW_TOKENS_METADATA.md](./FLOW_TOKENS_METADATA.md) | 1.0.0 | Consultar metadados de tokens por RFID, contrato ou colecao | Implementado | Exibir detalhes de um token/NFT |
| 4 | [FLOW_TOKENS_TRANSFER.md](./FLOW_TOKENS_TRANSFER.md) | 1.0.0 | Transferir NFTs e ERC-20 (por address, email, entre usuarios) | Implementado | Transferencia de tokens |
| 5 | [FLOW_TOKENS_BALANCE.md](./FLOW_TOKENS_BALANCE.md) | 1.0.0 | Consultar saldo blockchain (ETH, MATIC) de uma wallet | Implementado | Exibir saldo nativo da wallet |
| 6 | [FLOW_TOKENS_WALLET_CONNECT.md](./FLOW_TOKENS_WALLET_CONNECT.md) | 1.0.0 | Conectar wallet externa via WalletConnect (ID API) | Implementado | Integrar wallet externa |

---

## Guia Rapido

### Para listar NFTs do usuario:

```
1. Obtenha o profile do usuario -> extraia mainWallet.address
2. GET metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true  -> Lista paginada de NFTs
```

### Para consultar dados de um token especifico:

```
Opcao A (por RFID):     GET metadata/rfid/{rfid}
Opcao B (por contrato): GET metadata/address/{contractAddress}/{chainId}/{tokenId}
Opcao C (por colecao):  GET metadata/{companyId}/{collectionId}
```

### Para transferir um NFT:

```
Opcao A (por address): PATCH {companyId}/token-editions/{id}/transfer-token       -> body: { toAddress }
Opcao B (por email):   PATCH {companyId}/token-editions/{id}/transfer-token/email -> body: { email }
```

### Para consultar saldo:

```
GET blockchain/balance/{address}/{chainId}  -> { balance, currency }
```

### Implementacao minima (API-first):

```
1. GET  metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true  -> Listar NFTs
2. GET  metadata/rfid/{rfid}                                         -> Detalhes do token
3. PATCH {companyId}/token-editions/{id}/transfer-token              -> Transferir
4. GET  blockchain/balance/{address}/{chainId}                       -> Saldo
```

---

## Armadilhas Comuns (leia antes de implementar!)

| # | Problema | Solucao |
|---|----------|---------|
| 1 | `chainId` deve ser numero, nao string | Sempre converter para `Number(chainId)` antes de usar na URL |
| 2 | `useGetNFTSByWallet` requer `address` e `chainId` definidos | Hook so dispara query quando ambos sao non-undefined. Verifique o profile antes |
| 3 | `usePublicTokenData` aceita RFID ou (contractAddress + chainId + tokenId) | Se `rfid` existe, ele tem prioridade. Nao passe ambos |
| 4 | Endpoint de transfer usa PATCH, nao POST | `transfer-token` e `transfer-token/email` sao PATCH |
| 5 | Balance retorna `balance` como string | Usar `parseFloat(balance)` para calculos numericos |
| 6 | WalletConnect usa ID API, nao Key API | O endpoint `/blockchain/request-session-wallet-connect` esta na Identity API |
| 7 | `onlyMintedByWeblock=true` e obrigatorio | Sem este query param, retorna NFTs de terceiros tambem |

---

## Tabela de Decisao: O que implementar?

| Quero... | Use este documento |
|----------|-------------------|
| Listar todos os NFTs da wallet do usuario | [FLOW_TOKENS_NFT_WALLET](./FLOW_TOKENS_NFT_WALLET.md) |
| Ver detalhes/metadados de um token especifico | [FLOW_TOKENS_METADATA](./FLOW_TOKENS_METADATA.md) |
| Buscar tokens por RFID (etiqueta fisica) | [FLOW_TOKENS_METADATA](./FLOW_TOKENS_METADATA.md) |
| Buscar tokens por colecao | [FLOW_TOKENS_METADATA](./FLOW_TOKENS_METADATA.md) |
| Transferir NFT para outro endereco | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Transferir NFT por email do destinatario | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Transferir ERC-20 entre usuarios | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Consultar status da ultima transferencia | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Consultar saldo nativo (ETH/MATIC) | [FLOW_TOKENS_BALANCE](./FLOW_TOKENS_BALANCE.md) |
| Consultar saldo de multiplas wallets | [FLOW_TOKENS_BALANCE](./FLOW_TOKENS_BALANCE.md) |
| Conectar wallet externa via WalletConnect | [FLOW_TOKENS_WALLET_CONNECT](./FLOW_TOKENS_WALLET_CONNECT.md) |
| Consultar detalhes de um endpoint | [TOKENS_API_REFERENCE](./TOKENS_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Metodo | NFT Wallet | Metadata | Transfer | Balance | WalletConnect |
|----------|--------|:-:|:-:|:-:|:-:|:-:|
| `GET metadata/nfts/{address}/{chainId}` | Listar NFTs | **X** | | | | |
| `GET metadata/{companyId}/{collectionId}` | Metadados colecao | | **X** | | | |
| `GET metadata/rfid/{rfid}` | Metadados por RFID | | **X** | | | |
| `GET metadata/address/{contract}/{chainId}/{tokenId}` | Metadados por contrato | | **X** | | | |
| `PATCH {companyId}/token-editions/{id}/transfer-token` | Transferir NFT | | | **X** | | |
| `PATCH {companyId}/token-editions/{id}/transfer-token/email` | Transferir por email | | | **X** | | |
| `GET {companyId}/token-editions/{id}/get-last/transfer` | Ultima transferencia | | | **X** | | |
| `PATCH /{companyId}/erc20-tokens/{id}/transfer/user` | Transferir ERC-20 | | | **X** | | |
| `GET blockchain/balance/{address}/{chainId}` | Saldo blockchain | | | | **X** | |
| `POST /blockchain/request-session-wallet-connect` | WalletConnect (ID API) | | | | | **X** |

---

## Fluxo Visual

```
+---------------------------------------------------------------------------------+
|                          MODULO TOKENS W3BLOCK                                  |
|                                                                                 |
|  +----------------+                                                             |
|  |   USUARIO      |                                                             |
|  |                |                                                             |
|  | Autentica      |-> Obtem profile -> mainWallet.address                       |
|  +----------------+                                                             |
|        |                                                                        |
|        v                                                                        |
|  +----------------+    +------------------+    +------------------+              |
|  | NFT WALLET     |    |   METADATA       |    |   BALANCE        |             |
|  |                |    |                  |    |                  |              |
|  | Lista NFTs     |    | Por RFID         |    | Saldo ETH/MATIC  |             |
|  | por wallet +   |    | Por Contrato     |    | por wallet +     |             |
|  | chainId        |    | Por Colecao      |    | chainId          |             |
|  +----------------+    +------------------+    +------------------+              |
|        |                       |                                                |
|        v                       v                                                |
|  +----------------+    +------------------+    +------------------+              |
|  | TRANSFER       |    | WALLET CONNECT   |    |                  |             |
|  |                |    |                  |    |                  |              |
|  | Por Address    |    | Conectar wallet  |    |                  |              |
|  | Por Email      |    | externa via URI  |    |                  |              |
|  | ERC-20         |    | (ID API)         |    |                  |              |
|  | Status ultima  |    +------------------+    +------------------+              |
|  +----------------+                                                             |
+---------------------------------------------------------------------------------+
```

---

## Glossario Rapido

| Termo | Descricao |
|-------|-----------|
| `companyId` | UUID do tenant (empresa) na plataforma W3block |
| `chainId` | ID numerico da blockchain (1 = Ethereum, 137 = Polygon, 80001 = Mumbai) |
| `address` | Endereco da wallet na blockchain (formato 0x...) |
| `contractAddress` | Endereco do contrato NFT na blockchain |
| `tokenId` | ID numerico do token dentro do contrato |
| `editionId` | UUID interno da edicao do token no W3block |
| `editionNumber` | Numero sequencial da edicao dentro de uma colecao |
| `rfid` | Identificador RFID (etiqueta fisica) vinculado a um token |
| `collectionId` | UUID da colecao de tokens |
| `NFTByWalletDTO` | Interface TypeScript que representa um NFT retornado pela listagem por wallet |
| `PublicTokenPageDTO` | Interface TypeScript com dados publicos completos de um token |
| `GetLastTransferAPIResponse` | Interface TypeScript da resposta da ultima transferencia |
| `ChainScan` | Enum com IDs de blockchains suportadas (MAINNET=1, POLYGON=137, MUMBAI=80001) |
| `W3blockAPI.KEY` | API Key (`https://api.w3block.io`) usada para endpoints de tokens/metadata/balance |
| `W3blockAPI.ID` | API Identity (`https://id.w3block.io`) usada para WalletConnect |
| `ERC-20` | Padrao de token fungivel na blockchain (moedas, pontos) |
| `ERC-721` / `ERC-1155` | Padroes de tokens nao-fungiveis (NFTs) |
