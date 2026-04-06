---
id: CONTRACTS_SKILL_INDEX
title: "Índice de Skills de Contratos & Tokens"
module: offpix/contracts
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Índice de Skills de Contratos & Tokens

Documentação do módulo de Contratos & Tokens da W3Block. Abrange deploy de contratos NFT (ERC721A), gerenciamento de coleções e edições de tokens, ciclo de vida de tokens fungíveis ERC20 com regras de transferência, configuração de royalties, estimativa de gas, rastreamento RFID, importação/exportação em massa, e endpoints públicos de metadados/certificados.

**Serviço:** Pixway Registry | **Swagger:** https://api.w3block.io/docs/

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [CONTRACTS_API_REFERENCE](./CONTRACTS_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos de entidades | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) | 1.0.0 | Criar, configurar, fazer deploy de contratos NFT com royalty | Implementado | Fazer deploy de novos contratos NFT |
| 3 | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) | 1.0.0 | Coleções, edições, templates, publicação, XLSX em massa | Implementado | Gerenciar coleções de tokens |
| 4 | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) | 1.0.0 | Mint, transferência, burn de tokens, RFID, metadados, gas, certificados | Implementado | Operar em tokens individuais |
| 5 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) | 1.0.0 | ERC20 criar, deploy, mint, regras de transferência (5 handlers), burn, histórico | Implementado | Construir funcionalidades de loyalty/tokens fungíveis |

---

## Guia Rápido

### Para implementação de NFT:

```
1. Read: FLOW_CONTRACTS_NFT_LIFECYCLE.md           → Deploy an NFT contract
2. Read: FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md   → Create and publish collections
3. Read: FLOW_CONTRACTS_TOKEN_OPERATIONS.md        → Mint, transfer, burn tokens
4. Consult: CONTRACTS_API_REFERENCE.md             → For enums, DTOs, edge cases
```

### Para implementação de ERC20/Loyalty:

```
1. Read: FLOW_CONTRACTS_ERC20_LIFECYCLE.md         → Deploy and operate ERC20 tokens
2. Consult: CONTRACTS_API_REFERENCE.md             → For transfer config models, DTOs
```

### Fluxo mínimo de NFT (5 chamadas):

```
1. POST   /{id}/contracts                           → Create NFT contract
2. PATCH  /{id}/contracts/{id}/publish               → Deploy to blockchain
3. POST   /{id}/token-collections                    → Create collection
4. PATCH  /{id}/token-collections/{id}/sync-draft-tokens  → Generate editions
5. PATCH  /{id}/token-collections/publish/{id}       → Publish & mint
```

### Fluxo mínimo de ERC20 (3 chamadas):

```
1. POST   /{id}/erc20-contracts                     → Create ERC20 contract
2. PATCH  /{id}/erc20-contracts/{id}/publish         → Deploy to blockchain
3. PATCH  /{id}/erc20-tokens/{id}/mint               → Mint tokens
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Contrato preso em `publishing` | A confirmação na blockchain leva tempo. Faça polling do status ou use webhooks |
| 2 | Não é possível criar coleção sem contrato | Faça deploy e publique um contrato NFT primeiro |
| 3 | Edições não aparecem após criar coleção | Chame `sync-draft-tokens` para gerar os registros de edição |
| 4 | Transferência de usuário ERC20 rejeitada | Verifique as regras de configuração de transferência (política FREE, MAX_LIMIT, PERIOD) |
| 5 | Estimativa de gas retorna 0 | Certifique-se de que o contrato/coleção possui configuração válida |
| 6 | RFID já em uso | RFIDs são globalmente únicos — verifique a disponibilidade antes de atribuir |
| 7 | Contratos publicados são imutáveis | Crie um novo contrato se precisar de configurações diferentes |
| 8 | Taxas de porcentagem/fixas criam 2 transações | Por design — a taxa vai para a carteira da empresa separadamente |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|--------------|-----------|
| Fazer deploy de um novo contrato NFT | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Configurar divisão de royalties | [FLOW_CONTRACTS_NFT_LIFECYCLE](./FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Criar uma coleção de tokens | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Importar edições em massa via XLSX | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Publicar uma coleção na blockchain | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) |
| Fazer mint de tokens sob demanda | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Transferir tokens entre carteiras | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Fazer burn de tokens | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Atribuir RFID a um token | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Gerar um certificado PDF | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Estimar custos de gas | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./FLOW_CONTRACTS_TOKEN_OPERATIONS.md) |
| Fazer deploy de um token ERC20 (loyalty) | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Configurar taxas/limites de transferência | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Fazer mint/transferência/burn de tokens ERC20 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |
| Visualizar histórico de transações ERC20 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./FLOW_CONTRACTS_ERC20_LIFECYCLE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref API | Ciclo NFT | Coleções | Ops Token | ERC20 |
|----------|:-------:|:---------:|:--------:|:---------:|:-----:|
| POST /contracts | X | X | | | |
| GET /contracts | X | X | | | |
| GET /contracts/:id | X | X | | | |
| PATCH /contracts/:id | X | X | | | |
| PATCH /contracts/:id/publish | X | X | | | |
| GET /contracts/:id/estimate-gas | X | X | | X | |
| PATCH /contracts/has-role | X | X | | | |
| PATCH /contracts/grant-role | X | X | | | |
| POST /token-collections | X | | X | | |
| GET /token-collections | X | | X | | |
| GET /token-collections/:id | X | | X | | |
| PUT /token-collections/:id | X | | X | | |
| PATCH /token-collections/publish/:id | X | | X | | |
| PATCH /token-collections/:id/sync-draft-tokens | X | | X | | |
| GET/POST /token-collections/:id/export-import/xlsx | X | | X | | |
| GET /token-collections/estimate-gas | X | | X | X | |
| GET /token-editions | X | | X | X | |
| PATCH /token-editions/ready-to-mint | X | | X | X | |
| PATCH /token-editions/mint-on-demand | X | | | X | |
| PATCH /token-editions/transfer-token | X | | | X | |
| DELETE /token-editions/burn | X | | | X | |
| PATCH /token-editions/update-token-metadata | X | | X | X | |
| GET /token-editions/check-rfid | X | | X | X | |
| PATCH /token-editions/:id/rfid | X | | X | X | |
| GET /token-editions/:id/estimate-gas/* | X | | | X | |
| POST /erc20-contracts | X | | | | X |
| GET /erc20-contracts | X | | | | X |
| PATCH /erc20-contracts/:id | X | | | | X |
| PATCH /erc20-contracts/:id/publish | X | | | | X |
| PATCH /erc20-contracts/:id/transfer-config | X | | | | X |
| PATCH /erc20-tokens/:id/mint | X | | | | X |
| PATCH /erc20-tokens/:id/transfer/* | X | | | | X |
| PATCH /erc20-tokens/:id/burn | X | | | | X |
| GET /erc20-tokens/:id/history/* | X | | | | X |
| POST/GET /contracts/royalty-eligible | X | X | | | |
| GET /metadata/* | X | | | X | |
| GET/POST /certification/* | X | | | X | |
| POST/GET /subcategories | X | | X | | |
