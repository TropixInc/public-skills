---
id: FLOW_PASS_BENEFITS_VIEW
title: "Pass - Visualizacao de Beneficios"
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
depends_on:
  - FLOW_PASS_OVERVIEW
  - PASS_API_REFERENCE
---

# Visualização de Benefícios

## Overview

Visualização de benefícios de um token pass com duas perspectivas: **Operador/Admin** (lista passes atribuídos, vê benefícios que gerencia) e **Usuário** (vê benefícios do token que possui, com status de uso e QR code).

---

## Prerequisites

### Fluxo A — Operador/Admin
- Bearer token com role: `operator`, `admin` ou `superAdmin`
- Operador atribuído a pelo menos um benefício (via `token-pass-benefit-operators`)

### Fluxo B — Usuário
- Bearer token com role: `user`
- Possuir um token (NFT) da collection associada ao pass
- Conhecer o `editionNumber` do token

---

## Fluxo A — Operador/Admin

### Step 1: Listar Passes Atribuídos

- **Screen**: Grid de cards (`PassCard`) com imagem, nome e descrição de cada pass.
- **User Action**: Nenhuma (automático).

- **API Call**:

```
GET /token-passes/tenants/{tenantId}/users/{userId}
Authorization: Bearer <token>
```

- **Response Handling**: Lista paginada de `TokenPassEntityDto`. Renderiza um `PassCard` para cada pass com botão "Ver Benefícios".

---

### Step 2: Ver Benefícios do Pass

- **Screen**: Tabela de benefícios com colunas: Nome, Tipo, Período, Status.
- **User Action**: Click em "Ver Benefícios" no `PassCard`.

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}?tokenPassId={tokenPassId}
Authorization: Bearer <token>
```

- **Response Handling**: Lista paginada de `TokenPassBenefitEntityDto`.
- **Filtro por Operador**: Exibir apenas os benefícios atribuídos ao operador logado, filtrando pelo campo `tokenPassBenefitOperators` de cada benefício.

---

### Step 3: Detalhes de Benefício

- **Screen**: Detalhes completos do benefício selecionado.
- **User Action**: Click em um benefício da lista.

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}/{benefitId}
Authorization: Bearer <token>
```

- **Response Handling**: `TokenPassBenefitEntityDto` com todos os campos:

| Campo | Exibição |
|-------|----------|
| `name` | Título do benefício |
| `description` | Descrição completa |
| `rules` | Regras de uso |
| `type` | Badge: "Digital" ou "Físico" |
| `useLimit` | "Limite: X usos" (ou "Ilimitado" se null) |
| `eventStartsAt` / `eventEndsAt` | Período do evento formatado |
| `checkIn` | Horários de check-in por dia da semana |
| `linkUrl` | Link clicável (se digital) |
| `linkRules` | Regras do link |
| `dynamicQrCode` | Indicador de QR code dinâmico |
| `allowSelfUse` | Indicador de self-use habilitado |
| `tokenPassBenefitAddresses` | Lista de endereços (se físico) |

**Endereços (benefício físico):**

Para cada endereço em `tokenPassBenefitAddresses`:
- Nome do local
- Endereço completo (rua, número, cidade, estado, CEP)
- Regras do local

---

## Fluxo B — Usuário (Dono do Token)

### Step 1: Ver Benefícios por Edição do Token

- **Screen**: Lista de benefícios com status de uso, disponibilidade e QR code.
- **User Action**: Nenhuma (automático ao acessar página do token/pass).

- **API Call**:

```
GET /token-passes/tenants/{tenantId}/{tokenPassId}/token-editions/{editionNumber}/benefits
Authorization: Bearer <token>
```

- **Response Handling**: Array de `TokenPassBenefitsPublicDto`. Para cada benefício:

| Campo | Exibição |
|-------|----------|
| `name` | Nome do benefício |
| `description` | Descrição |
| `status` | Badge colorido: `active` (verde), `inactive` (cinza), `unavailable` (vermelho) |
| `statusMessage` | Mensagem de motivo (quando não ativo) |
| `useLimit` | Total de usos permitidos |
| `useAvailable` | Usos restantes |
| `type` | Tipo (digital/physical) |
| `eventStartsAt` / `eventEndsAt` | Período do evento |
| `nextCheckInDates` | Próximos horários de check-in |
| `tokenPassBenefitAddresses` | Endereços (se físico) |
| `linkUrl` | Link (se digital) |

**Status do Benefício:**

| Status | Significado | Visual |
|--------|------------|--------|
| `active` | Disponível para uso | Verde — pode gerar QR / usar |
| `inactive` | Fora do período/horário | Cinza — aguardar período válido |
| `unavailable` | Limite de usos atingido | Vermelho — não pode mais usar |

---

### Step 2: Gerar QR Code para Uso

Disponível apenas para benefícios com status `active`.

- **Screen**: QR code renderizado na tela do usuário.
- **User Action**: Click em "Gerar QR Code" ou "Mostrar QR Code".

#### Se `dynamicQrCode: true`:

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/{editionNumber}/secret
Authorization: Bearer <token>
```

- **Response**:

```json
{
  "secret": "a1b2c3d4e5f6g7h8i9j0"
}
```

- **Montagem do QR Code**: Construir a string:

```
{editionNumber},{userId},{secret},{benefitId}
```

Exemplo:

```
42,d4e5f6a7-8901-2345-bcde-f67890123456,a1b2c3d4e5f6g7h8i9j0,b1c2d3e4-5678-90ab-cdef-1234567890ab
```

Renderizar como QR code usando biblioteca como `qrcode.react` (`QRCodeSVG`).

> **Nota:** O secret dinâmico expira após um período. Se o operador escanear um QR code com secret expirado, receberá erro `secret-expired`. O usuário deve gerar um novo QR code.

#### Se `dynamicQrCode: false`:

O QR code é estático — o secret não muda e pode ser gerado uma única vez.

---

### Step 3: Usar Benefício Digital

Para benefícios do tipo `digital` com `linkUrl`:

- **Screen**: Link clicável para o conteúdo digital.
- **User Action**: Click no link.
- O link abre em nova aba.
- Se `linkRules` existe, exibir regras antes do link.

---

## API Sequence

### Fluxo A (Operador):
1. `GET /token-passes/tenants/{tenantId}/users/{userId}` — Listar passes
2. `GET /token-pass-benefits/tenants/{tenantId}?tokenPassId={id}` — Benefícios do pass
3. `GET /token-pass-benefits/tenants/{tenantId}/{benefitId}` — Detalhes (opcional)

### Fluxo B (Usuário):
1. `GET /token-passes/tenants/{tenantId}/{id}/token-editions/{editionNumber}/benefits` — Benefícios com status de uso
2. `GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/{editionNumber}/secret` — Secret para QR code (se dinâmico)

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Nenhum pass atribuído ao operador | Lista vazia — admin deve atribuir passes |
| Nenhum benefício no pass | Lista vazia — admin deve criar benefícios |
| Benefício `inactive` | Exibe `statusMessage` com motivo |
| Benefício `unavailable` | Exibe "Limite de usos atingido" |
| Secret expirado | Gerar novo QR code (chamar endpoint de secret novamente) |
| Usuário não é dono do token | Erro 403 `invalid-token-owner` — verificar wallet |

> **Referência:** Para schemas completos, formato de erro padrão e todos os endpoints, consulte [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md).

---

## Implementacao — React SDK (w3block-ui-sdk)

### Componentes

| Componente | Responsabilidade |
|-----------|-----------------|
| `PassesList` | Grid de passes do operador com QR scanner |
| `PassesDetail` | Tabela de benefícios de um pass (filtrada por operador) |
| `PassCard` | Card de exibição de um pass |
| `BenefitDetails` | Detalhes completos de um benefício |
| `PassTemplate` | Template de pass do usuário com QR code |
| `QrCodeSection` | Componente de QR code (geração + exibição) |
| `DetailPass` | Detalhe do benefício dentro do PassTemplate |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetPassByUser` | `useQuery` | Passes do operador |
| `useGetPassBenefits` | `useQuery` | Benefícios de um pass |
| `useGetPassBenefitById` | `useQuery` | Detalhes de benefício |
| `useGetBenefitsByEditionNumber` | `useQuery` | Benefícios por edição (com status) |
| `useGetQRCodeSecret` | `useQuery` | Secret para QR code dinâmico |
| `useGetPassBenefitOperators` | `useQuery` | Operadores de um benefício |

---

## Implementacao — teste-skills (w3block-teste-skills)

### Fluxo A (Operador) — Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `OperatorPassList` | `components/pass/operator-pass-list.tsx` | Cards de passes (imagem, tokenName, descricao). Clique navega para `/operator/{id}`. |
| `OperatorPassDetail` | `components/pass/operator-pass-detail.tsx` | Header do pass + beneficios filtrados por operador. Colunas: Nome, Local, Periodo, Status, Acao. Clique no nome navega para detalhe. |
| `OperatorBenefitDetail` | `components/pass/operator-benefit-detail.tsx` | Detalhe do beneficio com info, enderecos e botao "Validar". |

### Fluxo B (Usuario) — Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `PassList` | `components/pass/pass-list.tsx` | Lista paginada de passes do usuario. |
| `PassDetail` | `components/pass/pass-detail.tsx` | Header do pass + tabela de beneficios com status. |
| `BenefitDetail` | `components/pass/benefit-detail.tsx` | Detalhe: info + `BenefitQrCode` + `SelfUseButton` + enderecos. |
| `BenefitQrCode` | `components/pass/benefit-qr-code.tsx` | QR code via `react-qr-code`. Auto-refresh 15s (dinamico). Formato: `editionNumber,userId,secret,benefitId`. |
| `BenefitStatusBadge` | `components/pass/benefit-status-badge.tsx` | Badge: active (verde), inactive (cinza), unavailable (amarelo). |
| `BenefitTypeBadge` | `components/pass/benefit-type-badge.tsx` | Badge: digital / physical. |

### Hooks

| Hook | Arquivo | Tipo | Descricao |
|------|---------|------|-----------|
| `useGetOperatorPasses` | `hooks/use-get-operator-passes.ts` | `useQuery` | Passes do operador |
| `useGetPasses` | `hooks/use-get-passes.ts` | `useQuery` | Passes do usuario (paginado) |
| `useGetPass` | `hooks/use-get-pass.ts` | `useQuery` | Detalhes do pass por ID |
| `useGetBenefits` | `hooks/use-get-benefits.ts` | `useQuery` | Beneficios com filtros |
| `useGetBenefit` | `hooks/use-get-benefit.ts` | `useQuery` | Detalhes do beneficio |
| `useGetBenefitsByEdition` | `hooks/use-get-benefits-by-edition.ts` | `useQuery` | Beneficios por edicao |
| `useGetBenefitSecret` | `hooks/use-get-benefit-secret.ts` | `useQuery` | Secret para QR code (`staleTime: 0`, `initialData: null`) |
| `useUserEdition` | `hooks/use-user-edition.ts` | `useQuery` | Resolve `editionNumber` via Key API |
| `useBenefitStatus` | `hooks/use-benefit-status.ts` | hook puro | Calcula status do beneficio |

### Resolucao do editionNumber (Key API)

O teste-skills segue o mesmo padrao do SDK para resolver o `editionNumber`:

```
1. useUserWallet() → walletAddress
2. getCollectionTokens(companyId, tokenPassId, walletAddress) → tokens[]
3. getPublicTokenData(contractAddress, chainId, tokenId) → edition.currentNumber
4. Fallback: token.editionNumber do passo 2
```

Encapsulado em `useUserEdition(tokenPassId)`. Usa `tokenPassId` como `collectionId` (mesmo padrao do SDK).

### Paginas Next.js

| Pagina | Rota | Componente |
|--------|------|-----------|
| Passes Operador | `/operator` | `OperatorPassList` |
| Detalhes Pass (Op) | `/operator/[id]` | `OperatorPassDetail` |
| Detalhes Beneficio (Op) | `/operator/[id]/benefits/[benefitId]` | `OperatorBenefitDetail` |
| Passes Usuario | `/pass` | `PassList` |
| Detalhes Pass (User) | `/pass/[id]` | `PassDetail` |
| Detalhes Beneficio (User) | `/pass/[id]/benefits/[benefitId]` | `BenefitDetail` |
