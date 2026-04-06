---
id: FLOW_TOKENIZATION_BULK_IMPORT
title: "Tokenizacao - Importacao/Exportacao em Massa (XLSX)"
module: offpix/tokenization
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - tokenization
  - xlsx
  - import
  - export
  - bulk
depends_on:
  - TOKENIZATION_API_REFERENCE
  - FLOW_TOKENIZATION_COLLECTION_LIFECYCLE
---

# Importacao/Exportacao em Massa de Edicoes de Tokens

## Visao Geral

Este fluxo cobre o gerenciamento em massa de edicoes de tokens via planilhas Excel (XLSX). Inclui exportacao de um template de edicoes com a estrutura de colunas correta, importacao de dados de edicoes a partir de uma planilha preenchida, tratamento de erros de importacao, retentativa de operacoes falhadas e exportacoes assincronas de grandes conjuntos de edicoes. A importacao em massa e usada principalmente quando `similarTokens` e false e cada edicao precisa de metadados individuais (nome, descricao, RFID, campos customizados).

## Pre-requisitos

| Requisito | Descricao | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `companyId` | UUID da Empresa | Fluxo de autenticacao / configuracao do ambiente |
| Colecao em DRAFT | Colecao de tokens com edicoes sincronizadas | [Ciclo de Vida da Colecao](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Edicoes sincronizadas | `sync-draft-tokens` deve ter sido concluido | [Ciclo de Vida da Colecao](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |

---

## Fluxo: Exportar Template

Baixe um template Excel pre-populado com a estrutura de edicoes da colecao e campos de metadados.

### Passo 1: Exportar o Template XLSX

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-collections/{id}/export/xlsx` | Bearer (Admin) |

Nenhum parametro de query necessario.

**Resposta:** Download de arquivo XLSX binario (Content-Type: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`)

**Estrutura do Template:**

O arquivo exportado contem uma linha por edicao com as seguintes colunas:

| Coluna | Tipo | Descricao |
|--------|------|-----------|
| `editionNumber` | number | Numero sequencial da edicao (somente leitura, nao modifique) |
| `name` | string | Nome de exibicao da edicao |
| `description` | string | Descricao da edicao |
| `rfid` | string | Valor da tag RFID (deve ser globalmente unico) |
| Campos customizados... | varia | Colunas adicionais baseadas no template de subcategoria |

**Exemplo de conteudo do template:**

| editionNumber | name | description | rfid | rarity | color |
|---------------|------|-------------|------|--------|-------|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |
| ... | | | | | |
| 100 | | | | | |

**Observacoes:**
- A coluna `editionNumber` e pre-preenchida e nao deve ser alterada
- Colunas customizadas (ex.: `rarity`, `color`) sao derivadas dos campos do template de subcategoria
- Se as edicoes ja possuem dados (de uma importacao anterior ou edicao manual), os valores existentes sao incluidos na exportacao
- Use este template como ponto de partida para a importacao em massa para garantir a estrutura de colunas correta

---

## Fluxo: Importar Edicoes

Faca upload de um arquivo Excel preenchido para atualizar dados de edicoes em massa.

### Passo 1: Preparar a Planilha

Preencha o template exportado com os dados das edicoes:

**Exemplo de conteudo preenchido:**

| editionNumber | name | description | rfid | rarity | color |
|---------------|------|-------------|------|--------|-------|
| 1 | Golden Phoenix #1 | A rare golden phoenix | RFID-GP-001 | legendary | gold |
| 2 | Silver Dragon #2 | A majestic silver dragon | RFID-SD-002 | epic | silver |
| 3 | Bronze Griffin #3 | A sturdy bronze griffin | RFID-BG-003 | rare | bronze |

**Checklist de preparacao:**
- Nao modifique a coluna `editionNumber`
- Garanta que todos os RFIDs sejam unicos dentro do arquivo e na plataforma
- Preencha os campos obrigatorios definidos pelo template de subcategoria
- Salve o arquivo no formato `.xlsx` (nao `.xls` ou `.csv`)
- Mantenha o tamanho do arquivo razoavel (arquivos grandes com milhares de linhas podem causar timeout)

### Passo 2: Fazer Upload do Arquivo

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| POST | `/{companyId}/token-collections/{id}/import/xlsx` | Bearer (Admin) |

**Requisicao:** `multipart/form-data`

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `file` | file | Sim | O arquivo XLSX a importar |

**Exemplo cURL:**

```bash
curl -X POST \
  "https://api.w3block.io/{companyId}/token-collections/{collectionId}/import/xlsx" \
  -H "Authorization: Bearer {accessToken}" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@editions.xlsx"
```

**Resposta (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Passo 3: Consultar Conclusao do Job

A importacao e processada de forma assincrona. Consulte o endpoint de status do job para acompanhar a conclusao.

**Endpoint de Consulta:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Resposta (em andamento):**

```json
{
  "id": "job-uuid",
  "status": "processing",
  "progress": 35,
  "createdAt": "2026-04-02T12:00:00.000Z"
}
```

**Resposta (concluido com sucesso):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "totalRows": 100,
    "updated": 98,
    "errors": 2
  },
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:02:00.000Z"
}
```

**Resposta (falhou):**

```json
{
  "id": "job-uuid",
  "status": "failed",
  "progress": 0,
  "error": "Invalid file format",
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:00:05.000Z"
}
```

**Observacoes:**
- Consulte em intervalos razoaveis (a cada 2-5 segundos para arquivos pequenos, a cada 10-15 segundos para arquivos grandes)
- A contagem `result.errors` indica quantas linhas tiveram problemas de validacao
- Mesmo que algumas linhas falhem, as linhas validas ainda sao processadas
- A colecao deve estar no status DRAFT para que a importacao funcione

---

## Fluxo: Tratar Erros de Importacao

Quando algumas edicoes falham na validacao durante a importacao, elas sao marcadas com status de erro. Este fluxo cobre a identificacao e correcao desses erros.

### Passo 1: Listar Edicoes com Erros

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions` | Bearer (Admin) |

**Query para encontrar edicoes com erro:**

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=importError&page=1&limit=50
```

Tambem verifique erros de rascunho:

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=draftError&page=1&limit=50
```

**Resposta (200):**

```json
{
  "items": [
    {
      "id": "edition-uuid-1",
      "editionNumber": 45,
      "tokenCollectionId": "collection-uuid",
      "status": "importError",
      "name": "Asset #45",
      "rfid": "RFID-DUPLICATE",
      "errorFields": {
        "rfid": "RFID_HAS_ALREADY_BEEN_USED"
      },
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:02:00.000Z"
    },
    {
      "id": "edition-uuid-2",
      "editionNumber": 78,
      "tokenCollectionId": "collection-uuid",
      "status": "importError",
      "name": "",
      "errorFields": {
        "name": "IS_REQUIRED",
        "image": "INVALID_URL"
      },
      "createdAt": "2026-04-02T12:00:00.000Z",
      "updatedAt": "2026-04-02T12:02:00.000Z"
    }
  ],
  "meta": {
    "totalItems": 2,
    "itemCount": 2,
    "itemsPerPage": 50,
    "totalPages": 1,
    "currentPage": 1
  }
}
```

### Passo 2: Corrigir Erros

Ha duas abordagens para corrigir erros de importacao:

**Opcao A: Corrigir individualmente via API**

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/{id}` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "name": "Fixed Asset #45",
  "rfid": "RFID-UNIQUE-045",
  "tokenData": {
    "image": "https://example.com/valid-image.png"
  }
}
```

**Resposta (200):** Retorna a TokenEditionEntity atualizada. O status pode mudar de `importError` para `draft` quando todos os campos com erro forem resolvidos.

**Opcao B: Corrigir na planilha e re-importar**

1. Exporte um novo template (inclui dados atuais e estados de erro)
2. Corrija as linhas problematicas na planilha
3. Re-importe o arquivo corrigido
4. A re-importacao sobrescreve os dados existentes da edicao para numeros de edicao correspondentes

### Passo 3: Verificar se Todos os Erros Foram Resolvidos

Antes de publicar, confirme que nenhuma edicao permanece em estados de erro:

```
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=importError&limit=1
GET /{companyId}/token-editions?tokenCollectionId={collectionId}&status=draftError&limit=1
```

Ambas as queries devem retornar `totalItems: 0`.

**Valores comuns de campos de erro:**

| Codigo de Erro | Campo | Significado |
|----------------|-------|-------------|
| `IS_REQUIRED` | Qualquer campo obrigatorio | O campo e obrigatorio mas estava vazio ou ausente |
| `INVALID_URL` | image, mainImage | O formato da URL e invalido |
| `INVALID_STRING` | name, description | O valor nao e uma string valida |
| `INVALID_NUMBER` | campos numericos | O valor nao e um numero valido |
| `INVALID_DATE` | campos de data | O formato da data e invalido |
| `INVALID_YEAR` | campos de ano | O ano esta fora do intervalo aceitavel |
| `RFID_HAS_ALREADY_BEEN_USED` | rfid | O RFID ja esta atribuido a outra edicao ativa |
| `RFID_HAS_ITEMS_DUPLICATED_ON_LIST` | rfid | Valores de RFID duplicados dentro do mesmo arquivo de importacao |
| `INVALID_2D_DIMENSIONS` | campos de dimensao | Formato de dimensao 2D esta incorreto |
| `INVALID_3D_DIMENSIONS` | campos de dimensao | Formato de dimensao 3D esta incorreto |
| `INVALID_BOOLEAN` | campos booleanos | Valor nao e um booleano valido |
| `INVALID_ARRAY` | campos de array | Valor nao e um array valido |
| `INVALID_TYPE` | qualquer campo tipado | Tipo do valor nao corresponde ao tipo esperado |

---

## Fluxo: Retentar Operacoes Falhadas

Apos corrigir erros, retente todas as edicoes falhadas de uma colecao de uma vez.

### Retentativa em Massa

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| PATCH | `/{companyId}/token-editions/retry-bulk-by-collection` | Bearer (Admin) |

**Corpo da Requisicao:**

```json
{
  "tokenCollectionId": "collection-uuid"
}
```

**Resposta (200):**

```json
{
  "retried": 5
}
```

**Observacoes:**
- Retenta edicoes com status de erro: `importError`, `draftError`, `burnFailure`, `transferFailure`
- Reseta edicoes para o estado anterior apropriado e as reprocessa
- Para erros de importacao/rascunho: reseta para `draft` e revalida
- Para falhas de queima/transferencia: reseta para `minted` e reenvia a transacao blockchain
- A contagem `retried` indica quantas edicoes foram afetadas

---

## Fluxo: Exportacao Assincrona

Para colecoes grandes, use o endpoint de exportacao assincrona para gerar um arquivo Excel sem risco de timeout.

### Solicitar Exportacao Assincrona

**Endpoint:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/{companyId}/token-editions/xls` | Bearer (Admin) |

**Parametros de Query:**

| Parametro | Tipo | Obrigatorio | Descricao |
|-----------|------|-------------|-----------|
| `tokenCollectionId` | uuid | Sim | Colecao a exportar |

**Exemplo:**

```
GET /{companyId}/token-editions/xls?tokenCollectionId={collectionId}
```

**Resposta (200):**

```json
{
  "jobId": "job-uuid"
}
```

### Consultar Conclusao da Exportacao

**Endpoint de Consulta:**

| Metodo | Caminho | Autenticacao |
|--------|---------|--------------|
| GET | `/jobs/{jobId}` | Bearer (Admin) |

**Resposta (concluido):**

```json
{
  "id": "job-uuid",
  "status": "completed",
  "progress": 100,
  "result": {
    "downloadUrl": "https://storage.w3block.io/exports/editions-export-uuid.xlsx",
    "expiresAt": "2026-04-03T12:00:00.000Z"
  },
  "createdAt": "2026-04-02T12:00:00.000Z",
  "completedAt": "2026-04-02T12:01:00.000Z"
}
```

**Observacoes:**
- A URL de download e temporaria e expira (tipicamente dentro de 24 horas)
- Use este endpoint para colecoes com centenas ou milhares de edicoes onde a exportacao sincrona pode causar timeout
- O arquivo exportado segue a mesma estrutura de colunas da exportacao de template

---

## Resumo do Fluxo de Importacao/Exportacao

```
1. Criar rascunho da colecao
   POST /{companyId}/token-collections

2. Definir quantidade e sincronizar edicoes
   PUT /{companyId}/token-collections/{id}  (definir quantity)
   PATCH /{companyId}/token-collections/{id}/sync-draft-tokens  (gerar edicoes)
   GET /jobs/{jobId}  (consultar ate concluir)

3. Exportar template
   GET /{companyId}/token-collections/{id}/export/xlsx

4. Preencher planilha com dados das edicoes
   (etapa offline)

5. Importar planilha preenchida
   POST /{companyId}/token-collections/{id}/import/xlsx
   GET /jobs/{jobId}  (consultar ate concluir)

6. Verificar e corrigir erros
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=importError
   PATCH /{companyId}/token-editions/{editionId}  (corrigir cada uma)
   -- ou --
   Re-exportar, corrigir na planilha, re-importar

7. Verificar que nenhum erro permanece
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=importError&limit=1
   GET /{companyId}/token-editions?tokenCollectionId={id}&status=draftError&limit=1

8. Publicar colecao
   PATCH /{companyId}/token-collections/publish/{id}
```

---

## Tratamento de Erros

| Status | Erro | Causa | Resolucao |
|--------|------|-------|-----------|
| 400 | Formato de arquivo invalido | Arquivo nao e um XLSX valido | Salve o arquivo no formato `.xlsx` (nao `.xls` ou `.csv`) |
| 400 | TokenCollectionNotDraftException | Colecao ja esta publicada | Importacao so e permitida em colecoes DRAFT |
| 400 | Incompatibilidade de numero de edicao | Planilha contem numeros de edicao fora do intervalo da colecao | Garanta que os numeros de edicao correspondam as edicoes sincronizadas (1 ate quantity) |
| 400 | JobRunningException | Outro job de importacao ou sincronizacao ainda esta em execucao | Aguarde o job atual concluir |
| 400 | Arquivo muito grande | O arquivo enviado excede os limites de tamanho | Divida em multiplos arquivos menores ou reduza o volume de dados |
| 408 | Timeout da requisicao | Exportacao sincrona expirou para colecao grande | Use o endpoint de exportacao assincrona (`/token-editions/xls`) em vez disso |

## Armadilhas Comuns

| # | Problema | Solucao |
|---|---------|---------|
| 1 | Importacao de arquivo grande causa timeout | A importacao em si e assincrona (retorna jobId), mas o upload pode expirar para arquivos muito grandes. Mantenha tamanhos de arquivo razoaveis (abaixo de 10MB) |
| 2 | Conflitos de RFID na importacao | Garanta que todos os RFIDs na planilha sejam unicos tanto dentro do arquivo quanto na plataforma. Verifique com `check-rfid` para valores criticos |
| 3 | Problemas de codificacao de caracteres | Salve o arquivo Excel com codificacao UTF-8. Caracteres especiais podem ser corrompidos com outras codificacoes |
| 4 | Coluna editionNumber ausente | A coluna `editionNumber` deve estar presente e corresponder as edicoes sincronizadas. Nao delete ou reordene esta coluna |
| 5 | Modificando valores de editionNumber | Numeros de edicao sao atribuidos durante a sincronizacao e nao devem ser alterados. A importacao corresponde linhas pelo editionNumber |
| 6 | Importacao em colecao publicada | Importacao so e permitida quando a colecao esta no status DRAFT. Use o endpoint de atualizacao de metadados para colecoes publicadas |
| 7 | Re-importacao nao limpa dados anteriores | Re-importar sobrescreve apenas os campos presentes na planilha. Celulas vazias nao limpam valores existentes -- defina-os explicitamente como strings vazias se necessario |
| 8 | Colunas do template mudam apos atualizacao de subcategoria | Se o template de subcategoria for alterado, re-exporte o template para obter a estrutura de colunas atualizada |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida da Colecao | Obrigatorio: colecao deve ser criada e edicoes sincronizadas primeiro | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) |
| Operacoes de Edicao | Apos importacao: gerenciar edicoes individuais, mintar, transferir | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./FLOW_TOKENIZATION_EDITION_OPERATIONS.md) |
| Referencia de API | Detalhes completos de endpoints e DTOs | [TOKENIZATION_API_REFERENCE](./TOKENIZATION_API_REFERENCE.md) |
| Autenticacao Sign-In | Obrigatorio para Bearer token | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
