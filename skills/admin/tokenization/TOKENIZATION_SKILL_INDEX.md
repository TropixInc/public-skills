---
id: TOKENIZATION_SKILL_INDEX
title: "Indice de Skills de Tokenizacao"
module: offpix/tokenization
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# Indice de Skills de Tokenizacao

Documentacao para o modulo de Tokenizacao da W3Block. Cobre o ciclo de vida completo de colecoes de tokens (DRAFT ate PUBLISHED) e operacoes de edicoes incluindo mintagem, transferencia, queima, atualizacao de metadados, atribuicao de RFID e importacao/exportacao em massa via XLSX. Este modulo lida com tudo **apos** o deploy do contrato -- o modulo de Contratos cobre o deploy em si.

**Servico:** KEY | **Swagger:** https://api.w3block.io/docs/

---

## Documentos

| # | Documento | Versao | Descricao | Status | Quando usar |
|---|----------|--------|-----------|--------|-------------|
| 1 | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos entre entidades | Implementado | Referencia de API a qualquer momento |
| 2 | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) | 1.0.0 | Criar rascunho, sincronizar edicoes, configurar, publicar na blockchain | Implementado | Construir fluxos de criacao e publicacao de colecoes |
| 3 | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) | 1.0.0 | Mintar, transferir, queimar, atualizar metadados, RFID, estimativa de gas | Implementado | Gerenciar edicoes individuais de tokens apos publicacao |
| 4 | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) | 1.0.0 | Exportacao de template XLSX, importacao em massa de edicoes, rastreamento de jobs assincronos | Implementado | Gerenciar edicoes em massa via planilhas Excel |

---

## Guia Rapido

### Criar e publicar uma colecao de tokens:

```
1. Leia: FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md  -> Criar rascunho, sincronizar edicoes, configurar, publicar
2. Consulte: TOKENIZATION_API_REFERENCE.md           -> Para enums, DTOs, regras de validacao
```

### Gerenciar edicoes (mintar/transferir/queimar):

```
1. Leia: FLOW_TOKENIZATION_EDITION_OPERATIONS.md    -> Mintar, transferir, queimar, RFID, metadados
2. Consulte: TOKENIZATION_API_REFERENCE.md           -> Para enums de status, codigos de erro
```

### Importacao em massa via Excel:

```
1. Leia: FLOW_TOKENIZATION_BULK_IMPORT.md           -> Exportar template, importar, tratar erros
2. Consulte: TOKENIZATION_API_REFERENCE.md           -> Para detalhes dos endpoints de importacao/exportacao
```

### Consultar detalhes da API:

```
1. Leia: TOKENIZATION_API_REFERENCE.md              -> Referencia completa de endpoints, DTOs, enums
```

---

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Publicar sem sincronizar edicoes primeiro | Deve chamar `sync-draft-tokens` e aguardar o job assincrono completar antes de chamar publish |
| 2 | Contrato nao publicado | O contrato deve estar no status PUBLISHED com a funcionalidade ADMIN_MINTER habilitada antes da publicacao da colecao |
| 3 | Status da edicao nao e DRAFT na publicacao | Todas as edicoes devem estar no status DRAFT (sem estados de erro como DRAFT_ERROR ou IMPORT_ERROR) |
| 4 | Violacao de unicidade de RFID | RFIDs sao globalmente unicos em todas as edicoes ativas (excluindo deletadas/queimadas). Verifique a disponibilidade com `check-rfid` antes de atribuir |
| 5 | Formato invalido de rangeInitialToMint | Deve ser um intervalo valido como `"1-50"` ou separado por virgulas como `"1,5,10"` |

---

## Tabela de Decisao

| Eu quero... | Leia isto |
|-------------|-----------|
| Criar uma colecao de tokens | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Sincronizar/gerar edicoes | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Publicar uma colecao na blockchain | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Mintar tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Transferir tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Queimar tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Atualizar metadados de tokens | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Atribuir RFID a um token | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Estimar custos de gas | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Importar edicoes do Excel | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| Exportar template de edicoes | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Ciclo de Vida da Colecao | Operacoes de Edicao | Importacao em Massa |
|----------|:-------:|:--------------------:|:------------------:|:-----------:|
| POST /{companyId}/token-collections | X | X | | |
| GET /{companyId}/token-collections | X | X | | |
| GET /{companyId}/token-collections/{id} | X | X | | |
| PUT /{companyId}/token-collections/{id} | X | X | | |
| PATCH /{companyId}/token-collections/publish/{id} | X | X | | |
| DELETE /{companyId}/token-collections/{id}/draft | X | X | | |
| DELETE /{companyId}/token-collections/{id}/burn | X | X | | |
| PATCH /{companyId}/token-collections/{id}/sync-draft-tokens | X | X | | |
| PATCH /{companyId}/token-collections/{id}/increase-editions | X | X | | |
| PATCH /{companyId}/token-collections/{id}/pass/enable | X | | | |
| PATCH /{companyId}/token-collections/{id}/pass/disable | X | | | |
| GET /{companyId}/token-collections/estimate-gas | X | X | | |
| GET /{companyId}/token-collections/{id}/export/xlsx | X | | | X |
| POST /{companyId}/token-collections/{id}/import/xlsx | X | | | X |
| GET /{companyId}/token-editions | X | | X | X |
| GET /{companyId}/token-editions/{id} | X | | X | |
| PATCH /{companyId}/token-editions/{id} | X | | X | |
| PATCH /{companyId}/token-editions/ready-to-mint | X | | X | |
| PATCH /{companyId}/token-editions/{id}/ready-to-mint | X | | X | |
| PATCH /{companyId}/token-editions/mint-on-demand | X | | X | |
| PATCH /{companyId}/token-editions/locked-for-buy | X | | X | |
| PATCH /{companyId}/token-editions/unlocked-for-buy | X | | X | |
| PATCH /{companyId}/token-editions/update-token-metadata | X | | X | |
| PATCH /{companyId}/token-editions/notify-externally-minted | X | | X | |
| GET /{companyId}/token-editions/check-rfid | X | | X | |
| PATCH /{companyId}/token-editions/{id}/rfid | X | | X | |
| DELETE /{companyId}/token-editions/burn | X | | X | |
| PATCH /{companyId}/token-editions/transfer-token | X | | X | |
| PATCH /{companyId}/token-editions/{id}/transfer-token | X | | X | |
| PATCH /{companyId}/token-editions/{id}/transfer-token/email | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/mint | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/transfer | X | | X | |
| GET /{companyId}/token-editions/{id}/estimate-gas/burn | X | | X | |
| GET /{companyId}/token-editions/{id}/get-last/{type} | X | | X | |
| GET /{companyId}/token-editions/xls | X | | | X |
| PATCH /{companyId}/token-editions/retry-bulk-by-collection | X | | | X |
