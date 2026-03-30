---
id: FLOW_PASS_BENEFIT_TRACKING
title: "Pass - Rastreamento de Beneficios"
module: pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - pass
  - benefits
  - tracking
depends_on:
  - FLOW_PASS_OVERVIEW
  - PASS_API_REFERENCE
---

# Histórico e Rastreamento de Uso de Benefícios

## Overview

Permite que operadores e admins visualizem o histórico completo de uso de benefícios, com filtros por pass, benefício, edição e data. Inclui funcionalidade de exportação para relatório XLS com polling assíncrono para download.

---

## Prerequisites

- Bearer token com role: `operator`, `admin` ou `superAdmin`
- Pelo menos um benefício com usos registrados

---

## Steps

### Step 1: Acessar Histórico de Uso

- **Screen**: Lista paginada de usos com dados do usuário, benefício e data.
- **User Action**: Navegar para seção de histórico de uso no detalhe de um benefício ou na área de gestão.

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}/usages?benefitId={benefitId}&page=1&limit=10
Authorization: Bearer <token>
```

#### Query Params

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `page` | `number` | Não | Página (default: 1) |
| `limit` | `number` | Não | Itens por página (default: 10) |
| `search` | `string` | Não | Busca por nome/email do usuário |
| `sortBy` | `string[]` | Não | Campo(s) para ordenação |
| `orderBy` | `OrderByEnum` | Não | `ASC` ou `DESC` |
| `tokenPassId` | `uuid` | Não | Filtrar por token pass |
| `benefitId` | `uuid` | Não | Filtrar por benefício específico |
| `editionNumber` | `number` | Não | Filtrar por edição do token |
| `createdAt` | `datetime` | Não | Filtrar por data (ex: `2024-01-30T10:30:40-03:00`) |

- **Response Handling** (200 OK):

```json
{
  "items": [
    {
      "id": "use-uuid-1",
      "editionNumber": 42,
      "tokenPassBenefit": {
        "id": "b1c2d3e4-...",
        "name": "Acesso Backstage",
        "type": "physical",
        "useLimit": 3,
        "tokenPass": {
          "id": "f47ac10b-...",
          "name": "Pass VIP Evento"
        }
      },
      "tokenPassBenefitId": "b1c2d3e4-...",
      "uses": 2,
      "user": {
        "name": "João Silva",
        "email": "joao@example.com",
        "cpf": "12345678901",
        "phone": "11987654321"
      },
      "createdAt": "2024-06-15T19:30:00Z",
      "updatedAt": "2024-06-15T19:30:00Z"
    },
    {
      "id": "use-uuid-2",
      "editionNumber": 15,
      "tokenPassBenefit": {
        "id": "b1c2d3e4-...",
        "name": "Acesso Backstage",
        "type": "physical",
        "useLimit": 3
      },
      "tokenPassBenefitId": "b1c2d3e4-...",
      "uses": 1,
      "user": {
        "name": "Maria Santos",
        "email": "maria@example.com",
        "cpf": "98765432100",
        "phone": "21912345678"
      },
      "createdAt": "2024-06-15T18:45:00Z",
      "updatedAt": "2024-06-15T18:45:00Z"
    }
  ],
  "meta": {
    "itemCount": 2,
    "totalItems": 50,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  }
}
```

**Dados exibidos por uso:**

| Coluna | Campo | Descrição |
|--------|-------|-----------|
| Usuário | `user.name` | Nome do usuário que usou |
| Email | `user.email` | Email do usuário |
| Benefício | `tokenPassBenefit.name` | Nome do benefício |
| Edição | `editionNumber` | Número da edição do token |
| Usos | `uses` | Total de usos acumulados |
| Data | `createdAt` | Data/hora do registro de uso |

---

### Step 2: Filtrar Resultados

- **Screen**: Campos de filtro acima da lista.
- **User Action**: Aplicar filtros (por pass, benefício, edição, busca por nome, data).

- **Filtros disponíveis:**

| Filtro | Query Param | Descrição |
|--------|-------------|-----------|
| Token Pass | `tokenPassId` | Selecionar pass específico |
| Benefício | `benefitId` | Selecionar benefício específico |
| Edição | `editionNumber` | Filtrar por edição do token |
| Busca | `search` | Buscar por nome/email do usuário |
| Data | `createdAt` | Filtrar por data específica |
| Ordenação | `sortBy` + `orderBy` | Ordenar por campo (ex: `createdAt DESC`) |

---

### Step 3: Paginação

- **Screen**: Controles de paginação (anterior, próxima, número de página).
- **User Action**: Click em página ou botões de navegação.

- **API Call**: Mesmo endpoint com `page` atualizado.

---

## Exportação XLS

### Step 1: Solicitar Exportação

- **Screen**: Botão "Exportar XLS" na tela de histórico.
- **User Action**: Click em "Exportar XLS".

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}/usages/xls?benefitId={benefitId}
Authorization: Bearer <token>
```

> **Nota:** Os mesmos query params de filtro do endpoint de listagem podem ser usados para filtrar o conteúdo do relatório.

- **Response** (201 Created):

```json
{
  "id": "export-uuid",
  "tenantId": "tenant-uuid",
  "type": "benefit_usages_report",
  "status": "pending",
  "readyForDownloadDate": null,
  "expiresIn": null,
  "assetId": "asset-uuid",
  "asset": {
    "id": "asset-uuid",
    "tenantId": "tenant-uuid",
    "type": "document",
    "status": "waiting_upload",
    "directLink": null
  },
  "params": { "benefitId": "b1c2d3e4-..." },
  "createdAt": "2024-06-15T19:30:00Z"
}
```

- **State Changes**: Salvar `exportId` para polling.

---

### Step 2: Polling de Status da Exportação

- **Screen**: Indicador de progresso/loading enquanto relatório é gerado.
- **User Action**: Nenhuma (automático).

- **API Call** (polling periódico — intervalo recomendado: **3 a 5 segundos**, máximo de **60 tentativas**):

```
GET /exports/tenants/{tenantId}/{exportId}
Authorization: Bearer <token>
```

- **Response Handling**:

| Status | Ação |
|--------|------|
| `pending` | Continua polling. Exibe "Preparando relatório..." |
| `generating` | Continua polling. Exibe "Gerando relatório..." |
| `ready_for_download` | Para polling. Exibe botão de download |
| `failed` | Para polling. Exibe "Falha na geração. Tente novamente." |
| `expired` | Para polling. Exibe "Link expirado. Gere um novo relatório." |

---

### Step 3: Download do Relatório

- **Screen**: Botão "Baixar Relatório" habilitado.
- **User Action**: Click em "Baixar Relatório".

Quando `status === "ready_for_download"`:

```json
{
  "id": "export-uuid",
  "status": "ready_for_download",
  "readyForDownloadDate": "2024-06-15T19:31:00Z",
  "expiresIn": "2024-06-22T19:31:00Z",
  "asset": {
    "id": "asset-uuid",
    "type": "document",
    "status": "associated",
    "directLink": "https://<asset-provider-url>/benefit-usages-report.xlsx"
  }
}
```

- **Ação**: Abrir `asset.directLink` para download do arquivo XLS.
- **Nota**: O link de download tem data de expiração (`expiresIn`). Após expirar, é necessário gerar novo relatório.

---

## API Sequence

### Visualizar Histórico:
1. `GET /token-pass-benefits/tenants/{tenantId}/usages` — Listar usos (com filtros)

### Exportar XLS:
1. `GET /token-pass-benefits/tenants/{tenantId}/usages/xls` — Solicitar exportação
2. `GET /exports/tenants/{tenantId}/{exportId}` — Polling até `ready_for_download`
3. Acessar `asset.directLink` — Download do arquivo

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Nenhum uso registrado | Lista vazia — exibir mensagem "Nenhum uso registrado" |
| Filtro sem resultados | Lista vazia — sugerir remover filtros |
| Exportação falhou | Exibir erro + botão "Tentar novamente" |
| Link de download expirado | Exibir "Link expirado" + botão "Gerar novo relatório" |
| Erro de rede no polling | Continuar tentando silenciosamente |

> **Referência:** Para schemas completos de request/response e formato padrão de erro, consulte [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md).

---

## React SDK

### Componentes

| Componente | Responsabilidade |
|-----------|-----------------|
| `BenefitUsesList` | Lista paginada de usos com dados do usuário |
| `BenefitDetails` | Detalhes do benefício com seção de histórico |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetBenefitUses` | `useQuery` | Listar usos com paginação e filtros |
