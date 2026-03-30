# Self-Use de Benefício (Usuário)

## Overview

Permite que o próprio dono do token registre o uso de um benefício diretamente, sem necessidade de um operador escanear QR code. Disponível apenas para benefícios configurados com `allowSelfUse: true`.

---

## Prerequisites

- Bearer token com role: `user`
- Usuário é dono do token (NFT) na blockchain
- Benefício com `allowSelfUse: true`
- Benefício com status `active`
- Usos disponíveis restantes > 0

---

## Steps

### Step 1: Visualizar Benefícios do Token

- **Screen**: Lista de benefícios por edição do token com status e usos disponíveis.
- **User Action**: Nenhuma (automático).

- **API Call**:

```
GET /token-passes/tenants/{tenantId}/{tokenPassId}/token-editions/{editionNumber}/benefits
Authorization: Bearer <token>
```

- **Response Handling**: Array de `TokenPassBenefitsPublicDto`. Filtrar benefícios com `allowSelfUse: true` e `status: "active"` para exibir botão "Usar Benefício".

| Condição | Exibição |
|----------|----------|
| `allowSelfUse: true` + `status: "active"` | Botão "Usar Benefício" habilitado |
| `allowSelfUse: true` + `status: "inactive"` | Botão desabilitado + `statusMessage` |
| `allowSelfUse: true` + `status: "unavailable"` | Botão desabilitado + "Limite atingido" |
| `allowSelfUse: false` | Sem botão — requer operador |

---

### Step 2: Confirmar Uso

- **Screen**: Detalhes do benefício com informações de uso.
- **User Action**: Click em "Usar Benefício".

**Dados exibidos antes de confirmar:**

| Campo | Valor |
|-------|-------|
| Nome do benefício | `name` |
| Descrição | `description` |
| Usos disponíveis | `useAvailable` de `useLimit` |
| Período do evento | `eventStartsAt` — `eventEndsAt` |
| Tipo | `digital` ou `physical` |

- **Frontend Validation**: Verificar que `status === "active"` e `useAvailable > 0` antes de permitir envio.

---

### Step 3: Registrar Uso

- **Screen**: Loading/spinner durante o envio.
- **User Action**: Nenhuma (automático após confirmação).

- **API Call**:

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/use
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456",
  "editionNumber": 42
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuário (deve ser o dono do token) |
| `editionNumber` | `number` | Sim | Número da edição do token |

- **Response Handling** (201 Created):

```json
{
  "id": "use-uuid",
  "editionNumber": 42,
  "tokenPassBenefit": {
    "id": "b1c2d3e4-...",
    "name": "Desconto 20% Restaurante",
    "type": "digital",
    "useLimit": 5
  },
  "tokenPassBenefitId": "b1c2d3e4-...",
  "uses": 3,
  "createdAt": "2024-06-15T19:30:00Z",
  "updatedAt": "2024-06-15T19:30:00Z"
}
```

- **State Changes**:
  - `useAvailable` decrementa 1 (ex: 3 → 2)
  - Se `useAvailable === 0`: status muda para `unavailable`
  - Atualizar lista de benefícios para refletir novo estado

---

### Step 4: Feedback de Sucesso

- **Screen**: Mensagem de sucesso com detalhes do uso registrado.

| Informação | Valor |
|-----------|-------|
| Mensagem | "Benefício utilizado com sucesso!" |
| Benefício | `name` |
| Usos restantes | `useLimit - uses` |
| Data do uso | `createdAt` formatado |

- **User Action**: Fechar modal/toast → volta à lista de benefícios atualizada.

---

### Step 4 (alternativo): Erro

Se o registro falhar, exibir mensagem de erro adequada.

- **Error States**:

| HTTP | Código | Mensagem para o Usuário | Ação |
|------|--------|------------------------|------|
| 400 | `benefit-not-started` | "Este benefício ainda não está disponível" | Aguardar data de início |
| 400 | `benefit-expired` | "Este benefício expirou" | Não é possível usar |
| 400 | `usage-limit-reached` | "Limite de usos atingido" | Não é possível usar mais |
| 400 | `temporary-usage-limit-reached` | "Limite temporário atingido. Tente novamente mais tarde" | Aguardar próximo período |
| 400 | `wrong-checkin-time` | "Fora do horário de check-in" | Verificar horários permitidos |
| 403 | `self-use-not-allowed` | "Este benefício não permite uso direto" | Procurar um operador |
| 403 | `invalid-token-owner` | "Você não possui este token" | Verificar wallet |
| 403 | `unsatisfied-benefit-requirements` | "Requisitos do benefício não atendidos" | Verificar requisitos |
| 404 | `benefit-not-found` | "Benefício não encontrado" | Verificar dados |
| 404 | `token-not-found` | "Token não encontrado" | Verificar editionNumber |

> **Referência:** Para o formato padrão de resposta de erro e detalhes dos schemas, consulte [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md).

---

## API Sequence

1. `GET /token-passes/tenants/{tenantId}/{id}/token-editions/{editionNumber}/benefits` — Ver benefícios com status
2. `POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/use` — Registrar uso

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Benefício sem `allowSelfUse` | Não exibir botão de uso — benefício requer verificação por operador |
| Limite de usos atingido | Desabilitar botão + exibir "Limite atingido" |
| Fora do horário de check-in | Exibir `nextCheckInDates` com próximo horário disponível |
| Limite temporário atingido | Exibir mensagem + aguardar próximo período (baseado em `usageRule`) |
| Token não pertence ao usuário | Exibir erro — verificar se o NFT está na wallet do usuário |

---

## Implementacao — React SDK (w3block-ui-sdk)

### Componentes

| Componente | Responsabilidade |
|-----------|-----------------|
| `PassTemplate` | Template de pass do usuário com lista de benefícios |
| `DetailPass` | Detalhe do benefício com botão de uso |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetBenefitsByEditionNumber` | `useQuery` | Benefícios por edição com status |
| `usePostSelfUseBenefit` | `useMutation` | Registrar self-use |

---

## Implementacao — teste-skills (w3block-teste-skills)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `BenefitDetail` | `components/pass/benefit-detail.tsx` | Detalhe do beneficio com info, QR code e `SelfUseButton`. |
| `SelfUseButton` | `components/pass/self-use-button.tsx` | Botao "Usar Beneficio" + Dialog de confirmacao. So renderiza se `allowSelfUse: true`. Desabilitado se nao `active`. Toast com usos restantes no sucesso. Mapeia erros da API via `pass.apiErrors`. |

### Hooks

| Hook | Arquivo | Tipo | Descricao |
|------|---------|------|-----------|
| `useSelfUseBenefit` | `hooks/use-self-use-benefit.ts` | `useMutation` | POST `/use`. Body: `{ userId, editionNumber }`. Invalida queries benefit + benefits no sucesso. |
| `useUserEdition` | `hooks/use-user-edition.ts` | `useQuery` | Resolve `editionNumber` via Key API (necessario para o body do self-use). |
| `useBenefitStatus` | `hooks/use-benefit-status.ts` | hook puro | Calcula status para habilitar/desabilitar o botao. |

### Fluxo

```
[BenefitDetail]
    |
    | benefit.allowSelfUse === true && status === 'active'
    v
[SelfUseButton] -- clique --> Dialog de confirmacao
    |
    | clique "Confirmar"
    v
useSelfUseBenefit.mutate({ userId, editionNumber })
    |
    |--- onSuccess --> toast.success("Beneficio utilizado!") + usos restantes
    |--- onError --> toast.error(mensagem traduzida via pass.apiErrors)
```

### Paginas Next.js

| Pagina | Rota | Componente |
|--------|------|-----------|
| Detalhe do Beneficio | `/pass/[id]/benefits/[benefitId]` | `BenefitDetail` (inclui `SelfUseButton`) |
