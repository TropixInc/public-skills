---
id: FLOW_TOKENIZATION_EDITION_OPERATIONS
title: "Tokenizacao - Operacoes de Edicao (Mintar, Transferir, Queimar, RFID, Metadados)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - editions
  - mint
  - transfer
  - burn
  - rfid
  - metadata
depends_on:
  - TOKENIZATION_API_REFERENCE
  - FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
---

# Operacoes de Edicao de Tokens

## Visao Geral

Este fluxo cobre todas as operacoes em edicoes individuais de tokens apos uma colecao ter sido publicada na blockchain. As operacoes incluem marcar edicoes como prontas para mintar, mintagem sob demanda, transferencia de tokens entre carteiras, queima de tokens, atualizacao de metadados on-chain, gerenciamento de atribuicoes de RFID, estimativa de custos de gas e bloqueio/desbloqueio de edicoes para fluxos de comercio.

## Pre-requisitos

| Requisito | Descricao | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin, Integration ou User (por endpoint) | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da Empresa | Fluxo de autenticacao / configuracao do ambiente |
| Colecao publicada | Colecao de tokens no status PUBLISHED | [Ciclo de Vida da Colecao](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |

---

## Maquina de Estados da Edicao

```
                    +---------------+
                    | IMPORT_ERROR  |
                    +---------------+
                           ^
                           | (falha na importacao xlsx)
                           |
+---------------+    +-----+-----+    +---------------+
| DRAFT_ERROR   |<---+   DRAFT   +--->| LOCKED_FOR_BUY|
+---------------+    +-----+-----+    +-------+-------+
                           |                  |
                           v                  | (unlock)
                    +------+--------+         |
                    | READY_TO_MINT |<--------+
                    +------+--------+
                           |
                           v
                    +------+--------+
                    |    MINTING    |
                    +------+--------+
                           |
                           v
                    +------+--------+
              +---->|    MINTED     |<----+
              |     +--+--------+--+     |
              |        |        |        |
              |        v        v        |
              |  +-----+--+ +--+------+ |
              |  |BURNING  | |TRANSFER-| |
              |  |         | |RING     | |
              |  +----+--++ +--+---+--+ |
              |       |  |     |   |    |
              |       v  |     v   |    |
              | +-----++ | +--+---++   |
              | |BURNED | | |TRANS- |   |
              | +-------+ | |FERRED +---+
              |           | +-------+
              |           v
              |    +------+--------+
              |    | BURN_FAILURE  |
              |    +---------------+
              |
              |    +---------------+
              +----+TRANSFER_      |
                   |FAILURE        |
                   +---------------+
```

**Transicoes de estado:**
- `DRAFT` --> `DRAFT_ERROR`: erro de validacao durante sincronizacao ou configuracao
- `DRAFT` --> `IMPORT_ERROR`: erro durante importacao XLSX
- `DRAFT` --> `READY_TO_MINT`: marcado como pronto (manual ou via rangeInitialToMint na publicacao)
- `DRAFT` --> `LOCKED_FOR_BUY`: bloqueado para uma compra no comercio
- `LOCKED_FOR_BUY` --> `READY_TO_MINT`: desbloqueado
- `READY_TO_MINT` --> `MINTING`: transacao de mintagem submetida
- `MINTING` --> `MINTED`: transacao de mintagem confirmada
- `MINTED` --> `TRANSFERRING`: transacao de transferencia submetida
- `MINTED` --> `BURNING`: transacao de queima submetida
- `TRANSFERRING` --> `TRANSFERRED`: transferencia confirmada
- `TRANSFERRING` --> `TRANSFER_FAILURE`: transferencia falhou
- `TRANSFERRED` --> `TRANSFERRING`: outra transferencia submetida
- `TRANSFERRED` --> `BURNING`: transacao de queima submetida
- `BURNING` --> `BURNED`: queima confirmada
- `BURNING` --> `BURN_FAILURE`: queima falhou
- `TRANSFER_FAILURE` --> `MINTED`: retentativa reseta para minted
- `BURN_FAILURE` --> `MINTED`: retentativa reseta para minted

---

## Fluxo: Marcar como Pronto para Mintar

Antes de mintar, as edicoes devem estar no status READY_TO_MINT. Edicoes no intervalo `rangeInitialToMint` sao automaticamente definidas para este status na publicacao. Para outras edicoes, use estes endpoints.

### Marcacao em Massa como Pronto

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/ready-to-mint` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [51, 52, 53, 54, 55]
}
```

**Resposta (200):**

```json
{
  "updated": 5
}
```

### Marcacao Individual como Pronto

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/{id}/ready-to-mint` | Bearer (Admin) |

Nenhum corpo de requisicao necessario.

**Resposta (200):** Retorna a TokenEditionEntity atualizada com status `readyToMint`.

**Observacoes:**
- Edicoes devem estar no status DRAFT para serem marcadas como prontas
- Edicoes com DRAFT_ERROR ou IMPORT_ERROR nao podem ser marcadas como prontas (corrija os erros primeiro)

---

## Fluxo: Mintagem sob Demanda

Minte edicoes especificas para um endereco de carteira.

### Mintar Edicoes

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/mint-on-demand` | Bearer (Admin/Integration) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "ownerAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

**Resposta (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "minting"
    },
    {
      "editionNumber": 2,
      "status": "minting"
    },
    {
      "editionNumber": 3,
      "status": "minting"
    }
  ]
}
```

**Observacoes:**
- Edicoes devem estar no status READY_TO_MINT
- O `ownerAddress` e a carteira que sera proprietaria dos NFTs mintados
- A mintagem e assincrona -- edicoes passam para MINTING e depois para MINTED apos a transacao blockchain ser confirmada
- A colecao deve estar no status PUBLISHED
- Cada edicao mintada recebe um `tokenId`, `mintedHash`, `mintedAt` e `contractAddress` da transacao blockchain
- Se a mintagem falhar, a edicao permanece no status MINTING (tente novamente via `retry-bulk-by-collection`)

---

## Fluxo: Transferir Tokens

Transfira tokens mintados para um novo proprietario. Tres metodos estao disponiveis: transferencia em massa pelo admin, transferencia individual e transferencia por e-mail.

### Transferencia em Massa (Admin)

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/transfer-token` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Resposta (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "transferring"
    },
    {
      "editionNumber": 2,
      "status": "transferring"
    },
    {
      "editionNumber": 3,
      "status": "transferring"
    }
  ]
}
```

### Transferencia Individual

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token` | Bearer (Admin/User) |

**Corpo da Requisicao:**

```json
{
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef"
}
```

**Resposta (200):** Retorna a TokenEditionEntity atualizada com status `transferring`.

### Transferencia por E-mail

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/{id}/transfer-token/email` | Bearer (Admin/User) |

**Corpo da Requisicao:**

```json
{
  "email": "recipient@example.com"
}
```

**Resposta (200):** Retorna a TokenEditionEntity atualizada com status `transferring`.

**Observacoes:**
- Edicoes devem estar no status MINTED ou TRANSFERRED
- A transferencia e assincrona -- edicoes passam para TRANSFERRING e depois para TRANSFERRED apos confirmacao
- Usuarios (nao-admin) so podem transferir tokens que possuem (o `ownerAddress` deve corresponder a carteira do usuario autenticado)
- Transferencia por e-mail resolve o e-mail do destinatario para o endereco de carteira dele
- Se o e-mail do destinatario nao estiver registrado, a transferencia pode ser adiada ate o usuario se registrar e ter uma carteira
- Em caso de sucesso, `ownerAddress` da edicao e atualizado para o novo proprietario
- Em caso de falha, a edicao passa para o status TRANSFER_FAILURE

---

## Fluxo: Queimar Tokens

Destrua permanentemente tokens mintados na blockchain.

### Queimar Edicoes

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| DELETE | `/{companyId}/token-editions/burn` | Bearer (Admin/User) |

**Corpo da Requisicao:**

```json
{
  "tokens": [
    {
      "tokenCollectionId": "collection-uuid",
      "editionNumber": 1
    },
    {
      "tokenCollectionId": "collection-uuid",
      "editionNumber": 2
    }
  ]
}
```

**Resposta (200):**

```json
{
  "editions": [
    {
      "editionNumber": 1,
      "status": "burning"
    },
    {
      "editionNumber": 2,
      "status": "burning"
    }
  ]
}
```

**Observacoes:**
- Edicoes devem estar no status MINTED ou TRANSFERRED
- A queima e assincrona -- edicoes passam para BURNING e depois para BURNED apos confirmacao
- Usuarios (nao-admin) so podem queimar tokens que possuem
- Tokens queimados sao permanentemente destruidos na blockchain
- RFIDs de edicoes queimadas ficam disponiveis para reutilizacao
- Em caso de falha, a edicao passa para o status BURN_FAILURE

---

## Fluxo: Atualizar Metadados de Token

Atualize os metadados de edicoes mintadas. Isto pode disparar uma atualizacao de metadados on-chain e notificacao opcional via webhook.

### Atualizar Metadados

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/update-token-metadata` | Bearer (Admin/Integration) |

**Requisicao Minima:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1],
  "tokenData": {
    "name": "Updated Asset Name"
  }
}
```

**Requisicao Completa:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "tokenData": {
    "name": "Updated Asset Name",
    "description": "Updated description",
    "image": "https://example.com/updated-image.png",
    "attributes": [
      { "trait_type": "level", "value": "5" },
      { "trait_type": "experience", "value": "1500" }
    ]
  },
  "mergeTokenData": true,
  "keepStatus": true
}
```

**Resposta (200):**

```json
{
  "updated": 3
}
```

**Comportamento dos campos:**

| Campo | Padrao | Descricao |
|-------|--------|-----------|
| `mergeTokenData` | false | Quando `true`, o tokenData fornecido e mesclado superficialmente com os dados existentes (novas chaves adicionadas, chaves existentes sobrescritas). Quando `false`, o tokenData inteiro e substituido |
| `keepStatus` | false | Quando `true`, o status atual da edicao e preservado. Quando `false`, o status pode ser resetado para draft para reprocessamento |

**Observacoes:**
- Se a colecao tiver `settings.sendWebhookWhenTokenEditionIsUpdated` definido como `true`, uma notificacao via webhook e disparada apos a atualizacao
- Atualizacoes de metadados afetam os metadados off-chain; metadados on-chain podem exigir uma atualizacao adicional dependendo da implementacao do contrato
- A opcao `mergeTokenData` e util para atualizar campos individuais sem perder dados existentes

---

## Fluxo: Gerenciamento de RFID

Atribua e gerencie tags RFID em edicoes individuais. RFIDs sao identificadores globalmente unicos que vinculam itens fisicos a tokens digitais.

### Verificar Disponibilidade de RFID

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/check-rfid` | Bearer (Admin) |

**Query:**

```
GET /{companyId}/token-editions/check-rfid?rfid=RFID-001-ABC
```

**Resposta (disponivel):**

```json
{
  "available": true
}
```

**Resposta (em uso):**

```json
{
  "available": false,
  "usedBy": {
    "editionId": "edition-uuid",
    "tokenCollectionId": "collection-uuid",
    "editionNumber": 42
  }
}
```

### Atribuir RFID

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/{id}/rfid` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "rfid": "RFID-001-ABC"
}
```

**Resposta (200):** Retorna a TokenEditionEntity atualizada com o RFID atribuido.

**Observacoes:**
- RFIDs sao globalmente unicos em todas as edicoes ativas na plataforma (excluindo edicoes deletadas e queimadas)
- Sempre verifique a disponibilidade com `check-rfid` antes de atribuir para evitar conflitos
- Atribuir um novo RFID a uma edicao que ja possui um substitui o RFID existente (o antigo fica disponivel)
- RFIDs de edicoes queimadas sao liberados e ficam disponiveis para reutilizacao

---

## Fluxo: Estimativa de Gas

Estime custos de gas antes de executar operacoes on-chain (mintar, transferir, queimar).

### Estimar Gas de Mintagem

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/mint` | Bearer (Admin) |

**Resposta (200):**

```json
{
  "slow": {
    "gasPrice": "30000000000",
    "estimatedCost": "0.003",
    "estimatedTime": "120s"
  },
  "standard": {
    "gasPrice": "50000000000",
    "estimatedCost": "0.005",
    "estimatedTime": "60s"
  },
  "fast": {
    "gasPrice": "80000000000",
    "estimatedCost": "0.008",
    "estimatedTime": "30s"
  }
}
```

### Estimar Gas de Transferencia

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/transfer?toAddress={address}` | Bearer (Admin/User) |

**Parametros de Query:**

| Parametro | Tipo | Obrigatorio | Descricao |
|-----------|------|-------------|-----------|
| `toAddress` | string | Sim | Endereco da carteira de destino para a estimativa |

**Resposta (200):** Mesma estrutura da estimativa de gas de mintagem (niveis slow/standard/fast).

### Estimar Gas de Queima

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/{id}/estimate-gas/burn` | Bearer (Admin/User) |

**Resposta (200):** Mesma estrutura da estimativa de gas de mintagem (niveis slow/standard/fast).

**Observacoes:**
- Estimativas de gas refletem as condicoes atuais da rede e podem mudar rapidamente
- Os tres niveis fornecem diferentes compromissos entre custo e velocidade de confirmacao
- Os custos sao denominados no token nativo da chain (ex.: MATIC para Polygon, ETH para Ethereum)
- Estimativas sao aproximacoes -- os custos reais podem variar levemente

---

## Fluxo: Bloquear/Desbloquear para Comercio

Bloqueie edicoes para fluxos de compra e desbloqueie quando a compra for cancelada ou concluida sem mintagem.

### Bloquear para Compra

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/locked-for-buy` | Bearer (Admin/Integration) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [10, 11, 12]
}
```

**Resposta (200):**

```json
{
  "updated": 3
}
```

### Desbloquear para Compra

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/unlocked-for-buy` | Bearer (Admin/Integration) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [10, 11, 12]
}
```

**Resposta (200):**

```json
{
  "updated": 3
}
```

**Observacoes:**
- Bloquear define edicoes de DRAFT ou READY_TO_MINT para o status LOCKED_FOR_BUY
- Desbloquear retorna edicoes para o status DRAFT
- Usado pelo modulo de comercio para reservar edicoes especificas durante um fluxo de compra
- Edicoes bloqueadas nao podem ser mintadas, transferidas ou queimadas ate serem desbloqueadas

---

## Fluxo: Notificar Mintagem Externa

Marque edicoes como mintadas quando a mintagem foi realizada fora da plataforma W3Block.

### Notificar Mintagem Externa

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/notify-externally-minted` | Bearer (Admin/Integration) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid",
  "editionNumbers": [1, 2, 3],
  "contractAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
  "chainId": 137
}
```

**Resposta (200):**

```json
{
  "updated": 3
}
```

**Observacoes:**
- Define edicoes para o status MINTED sem executar uma transacao de mintagem na plataforma
- Usado para integracoes onde a mintagem ocorre em plataformas externas ou smart contracts customizados
- O `contractAddress` e `chainId` sao registrados nas edicoes para referencia

---

## Fluxo: Obter Ultima Transacao

Recupere a transacao de transferencia ou queima mais recente de uma edicao.

### Obter Ultima Transacao

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/{id}/get-last/{type}` | Bearer (Admin) |

**Parametros de Caminho:**

| Parametro | Valores | Descricao |
|-----------|---------|-----------|
| `type` | `transfer`, `burn` | Tipo de transacao a recuperar |

**Exemplo:**

```
GET /{companyId}/token-editions/{id}/get-last/transfer
```

**Resposta (200):**

```json
{
  "id": "transaction-uuid",
  "type": "transfer",
  "transactionHash": "0xabc123def456...",
  "fromAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "toAddress": "0xrecipient1234567890abcdef1234567890abcdef",
  "status": "confirmed",
  "createdAt": "2026-04-02T14:00:00.000Z"
}
```

**Observacoes:**
- Retorna apenas a transacao mais recente do tipo especificado
- Util para verificar o status da transacao ou exibir detalhes da transacao na UI

---

## Tratamento de Erros

| Status | Erro | Causa | Resolucao |
|--------|------|-------|-----------|
| 400 | TokenEditionInvalidStatusException | Edicao nao esta no status exigido para a operacao | Verifique o diagrama de maquina de estados acima para transicoes validas |
| 400 | RfidAlreadyUsedException | RFID esta atribuido a outra edicao ativa | Use `check-rfid` antes de atribuir; escolha um RFID diferente |
| 400 | RfidDuplicateException | RFIDs duplicados na lista submetida | Remova duplicatas da requisicao |
| 404 | TokenEditionNotFoundException | Edicao nao encontrada para a empresa informada | Verifique o ID da edicao e o companyId |
| 400 | TokenCollectionNotPublishedException | Colecao nao esta publicada | Publique a colecao primeiro |
| 400 | GasEstimationException | Estimativa de gas falhou | Verifique o deploy do contrato e a conectividade de rede |
| 403 | Forbidden | Usuario tentando operar tokens que nao possui | Usuarios so podem transferir/queimar seus proprios tokens |

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Mintando edicoes que nao estao READY_TO_MINT | Marque as edicoes como prontas primeiro com o endpoint `ready-to-mint` |
| 2 | Usuario nao consegue transferir/queimar | Usuarios so podem operar tokens onde o `ownerAddress` corresponde a sua carteira. A role Admin pode operar em qualquer token |
| 3 | Transferencia/queima aparece como falha | Verifique o status da edicao (TRANSFER_FAILURE ou BURN_FAILURE). Use `retry-bulk-by-collection` para tentar novamente |
| 4 | Atualizacao de metadados nao dispara webhook | Garanta que `settings.sendWebhookWhenTokenEditionIsUpdated` esteja habilitado na colecao |
| 5 | Atribuicao de RFID rejeitada | Verifique a disponibilidade com `check-rfid` primeiro. RFIDs devem ser globalmente unicos em todas as edicoes ativas |
| 6 | Edicoes bloqueadas nao podem ser mintadas | Desbloqueie-as primeiro com `unlocked-for-buy`, depois marque como prontas e minte |
| 7 | Comportamento inesperado do mergeTokenData | Quando `mergeTokenData` e false (padrao), o tokenData inteiro e substituido. Use `true` para atualizar campos individuais |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida da Colecao | Obrigatorio: colecao deve ser publicada primeiro | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Importacao em Massa | Alternativa: configurar edicoes via Excel | [FLOW_TOKENIZATION_BULK_IMPORT](./FLOW_TOKENIZATION_BULK_IMPORT.md) |
| Referencia de API | Detalhes completos de endpoints e DTOs | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) |
| Ciclo de Vida de Contratos NFT | Obrigatorio: contrato deve existir para mintagem | [FLOW_CONTRACTS_NFT_LIFECYCLE](../contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) |
| Autenticacao Sign-In | Obrigatorio para Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
