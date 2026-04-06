---
id: WHITELIST_SKILL_INDEX
title: "Indice de Skills de Whitelist"
module: offpix/whitelist
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Indice de Skills de Whitelist

Documentacao para o modulo de Whitelist da W3Block. Cobre criacao e gerenciamento de whitelists (grupos de usuarios) e suas entradas, utilizados para controlar acesso a produtos, promocoes e operacoes blockchain. Os tipos de entrada incluem e-mail, endereco de carteira, ID de usuario, detentor de colecao, contexto aprovado por KYC, detentor de colecao KEY e detentor de ERC-20 KEY.

**Servico:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

> **Nomenclatura no frontend:** O frontend Offpix rotula whitelists como **"Grupos de Usuarios"** na navegacao e breadcrumbs.

---

## Documentos

| # | Documento | Versao | Descricao | Status | Quando usar |
|---|----------|--------|-----------|--------|-------------|
| 1 | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos entre entidades | Implementado | Referencia de API a qualquer momento |
| 2 | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) | 1.0.0 | Criar, editar, excluir whitelists; adicionar/remover entradas; verificar acesso de usuario; promover on-chain | Implementado | Construir funcionalidades de gerenciamento de whitelist (grupo de usuarios) |

---

## Guia Rapido

### Para uma implementacao basica de whitelist:

```
1. Leia: FLOW_WHITELIST_MANAGEMENT.md      -> Criar uma whitelist, adicionar entradas, verificar acesso
2. Consulte: WHITELIST_API_REFERENCE.md     -> Para enums, DTOs, paginacao, casos especiais
```

### Implementacao minima via API (3 chamadas):

```
1. POST /whitelists/{tenantId}                    -> Criar uma whitelist
2. POST /whitelists/{tenantId}/{id}/entries        -> Adicionar entradas (repetir por entrada)
3. GET  /whitelists/{tenantId}/{id}/check-user     -> Verificar acesso do usuario
```

---

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Erro de entrada duplicada ao adicionar | A combinacao de `whitelistId` + `type` + `value` e unica (exceto para `collection_holder` e `key_collection_holder`). Verifique antes de adicionar |
| 2 | `value` e convertido para minusculas silenciosamente | Todos os valores de entrada sao convertidos para minusculas ao salvar via um transformer do banco de dados. Armazene e compare em minusculas |
| 3 | Entrada de ID de usuario falha com "User not found" | O usuario deve pertencer ao mesmo tenant. IDs de usuarios de outros tenants sao rejeitados |
| 4 | `additionalData` obrigatorio para tipos holder | `collection_holder` requer `chainId`; `key_collection_holder` e `key_erc20_holder` aceitam `amount`. Omitir `additionalData` para estes tipos causa erros de validacao |
| 5 | Promover on-chain falha com "already on chain" | Uma whitelist so pode ter um wallet group por `chainId`. Verifique wallet groups existentes antes de promover |

---

## Tabela de Decisao

| Eu quero... | Leia isto |
|-------------|-----------|
| Criar uma nova whitelist (grupo de usuarios) | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Adicionar entradas (e-mails, carteiras, usuarios) a uma whitelist | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Verificar se um usuario tem acesso via whitelist | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Promover uma whitelist on-chain | [FLOW_WHITELIST_MANAGEMENT](./FLOW_WHITELIST_MANAGEMENT.md) |
| Entender tipos de entrada e schemas de additionalData | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) |
| Filtrar e paginar entradas de whitelist | [WHITELIST_API_REFERENCE](./WHITELIST_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Gerenciamento de Whitelist |
|----------|:-------:|:--------------------:|
| POST /whitelists/{tenantId} | X | X |
| GET /whitelists/{tenantId} | X | X |
| GET /whitelists/{tenantId}/{id} | X | X |
| PATCH /whitelists/{tenantId}/{id} | X | X |
| DELETE /whitelists/{tenantId}/{id} | X | X |
| POST /whitelists/{tenantId}/{id}/entries | X | X |
| GET /whitelists/{tenantId}/{id}/entries | X | X |
| DELETE /whitelists/{tenantId}/{id}/entries/{entryId} | X | X |
| GET /whitelists/{tenantId}/{id}/check-user | X | X |
| GET /whitelists/{tenantId}/check-user | X | X |
| PATCH /whitelists/{tenantId}/{id}/promote-on-chain | X | X |
