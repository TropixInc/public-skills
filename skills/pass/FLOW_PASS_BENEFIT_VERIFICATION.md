---
id: FLOW_PASS_BENEFIT_VERIFICATION
title: "Pass - Verificacao de Beneficios"
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
  - verification
depends_on:
  - FLOW_PASS_OVERVIEW
  - PASS_API_REFERENCE
---

# Verificacao e Registro de Uso de Beneficio (Operador)

## Overview

Fluxo principal do operador: escanear o QR code apresentado pelo portador do token, verificar se o beneficio e valido, e registrar o uso. O operador acessa seus passes atribuidos, abre o scanner QR, escaneia o codigo, verifica via API, e confirma o registro de uso.

## Prerequisites

- Bearer token com role: `operator` (ou `admin` / `superAdmin`)
- Operador deve estar atribuido ao beneficio via `tokenPassBenefitOperators`
- Usuario deve apresentar QR code (fisico ou na tela do celular)
- `tenantId` disponivel via `useCompanyConfig()` (`companyId`)

---

## Steps

### Step 1: Page Load — Lista de Passes do Operador

- **Screen**: Grid de componentes `PassCard` mostrando passes atribuidos ao operador. Botao "Escanear QR Code" no topo da pagina. Rota: `/tokens/pass`.
- **User Action**: Nenhuma (automatico). O `PassesList` verifica se o usuario possui role `admin`, `superAdmin` ou `operator` via `useProfile()` antes de renderizar.

- **API Call**:

```
GET /token-passes/tenants/{tenantId}/users/{userId}
Authorization: Bearer <token>
```

- **Response Handling** (200 OK):

```json
{
  "items": [
    {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "tokenName": "VIP Pass 2024",
      "contractAddress": "0x1234567890abcdef1234567890abcdef12345678",
      "chainId": "137",
      "collectionId": "a1b2c3d4-5678-90ab-cdef-1234567890ab",
      "name": "Pass VIP Evento",
      "description": "Acesso exclusivo aos beneficios VIP do evento",
      "rules": "Valido para maiores de 18 anos",
      "imageUrl": "https://cdn.example.com/pass-vip.png",
      "tenantId": "tenant-uuid",
      "createdAt": "2024-01-10T10:00:00Z",
      "updatedAt": "2024-01-10T10:00:00Z"
    }
  ]
}
```

- **State Changes**: `passes` populado com `data?.data?.items || []`. Grid de `PassCard` renderizado com `id`, `imageUrl`, `name`, `tokenName`, `contractAddress`, `chainId`.

---

### Step 2: Selecionar Pass

- **Screen**: Cada `PassCard` exibe imagem, nome e nome do token. Botao "Ver Beneficios" navega para a pagina de detalhes.
- **User Action**: Clica no `PassCard` (botao "Ver Beneficios").
- **State Changes**: Navegacao para `/tokens/pass/[tokenPassId]?chainId={chainId}&contractAddress={contractAddress}`.

---

### Step 3: Ver Beneficios do Pass

- **Screen**: Header com imagem, nome e descricao do pass. Tabela `GenericTable` com colunas: Beneficio (nome com link), Local, Periodo, Status, Acao. Apenas exibe beneficios onde o operador esta atribuido (filtrado por `tokenPassBenefitOperators.userId === user.id`).
- **User Action**: Nenhuma (automatico no load).

- **API Call 1** (detalhes do pass):

```
GET /token-passes/tenants/{tenantId}/{tokenPassId}
Authorization: Bearer <token>
```

- **API Call 2** (beneficios com status):

```
GET /token-pass-benefits/tenants/{tenantId}?tokenPassId={tokenPassId}&chainId={chainId}&contractAddress={contractAddress}
Authorization: Bearer <token>
```

- **Response Handling** (200 OK):

```json
{
  "items": [
    {
      "id": "b1c2d3e4-5678-90ab-cdef-1234567890ab",
      "name": "Acesso Backstage",
      "description": "Entrada na area backstage",
      "type": "physical",
      "useLimit": 3,
      "eventStartsAt": "2024-06-15T18:00:00-03:00",
      "eventEndsAt": "2024-06-15T23:59:00-03:00",
      "status": "active",
      "allowSelfUse": false,
      "tokenPassId": "f47ac10b-...",
      "tokenPassBenefitAddresses": [
        {
          "name": "Arena Principal",
          "street": "Rua das Flores, 123",
          "number": 123,
          "city": "Sao Paulo",
          "state": "SP",
          "postalCode": "01234-567",
          "rules": "Apresentar documento com foto"
        }
      ]
    }
  ]
}
```

- **State Changes**: Tabela renderizada. Beneficios expirados (`eventEndsAt < now`) nao exibem botao "Validar". Beneficios ativos exibem botao "Validar" na coluna Acao.

---

### Step 4: Abrir Scanner QR

- **Screen**: Modal `QrCodeReader` com camera ativa para leitura de QR code. Ocupa tela cheia.
- **User Action**: Clica no botao "Validar" em um beneficio especifico (na pagina de detalhes) OU clica no botao "Escanear QR Code" (na lista de passes).
- **Frontend Validation**: O scanner le a string do QR code e o frontend faz parse:

```typescript
const [editionNumber, userId, secret, benefitId] = qrCodeData.split(',');
```

Formato do QR code: `{editionNumber},{userId},{secret},{benefitId}`

Exemplo: `42,d4e5f6a7-8901-2345-bcde-f67890123456,a1b2c3d4e5f6g7h8i9j0,b1c2d3e4-5678-90ab-cdef-1234567890ab`

- **Validacoes pos-leitura**:
  - Se algum dos 4 componentes estiver ausente: exibe `QrCodeError` com mensagem "Formato invalido" (`TypeError.read`)
  - Se o `benefitId` do QR nao corresponde ao beneficio selecionado (no fluxo via `PassesDetail`): exibe `QrCodeError` com `TypeError.invalid`
  - Se todos os 4 componentes validos: seta `qrCodeData` no state e abre modal de verificacao

- **State Changes**: `showScan = false`, `qrCodeData` setado, `showVerify = true`.

---

### Step 5: Verificacao do Beneficio

- **Screen**: Modal `VerifyBenefit` em estado de loading (`Spinner`) enquanto a verificacao e processada.
- **User Action**: Nenhuma (automatico). A query `useVerifyBenefit` dispara quando `secret` esta disponivel.

- **API Call**:

```
GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/verify?userId={userId}&editionNumber={editionNumber}&secret={secret}
Authorization: Bearer <token>
```

- **Response Handling** (200 OK):

```json
{
  "id": "usage-uuid",
  "editionNumber": 42,
  "tokenPassBenefit": {
    "id": "b1c2d3e4-5678-90ab-cdef-1234567890ab",
    "name": "Acesso Backstage",
    "description": "Entrada na area backstage",
    "type": "physical",
    "useLimit": 3,
    "dynamicQrCode": true,
    "allowSelfUse": false,
    "tokenPass": {
      "id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
      "name": "Pass VIP Evento"
    }
  },
  "tokenPassBenefitId": "b1c2d3e4-...",
  "uses": 1,
  "user": {
    "name": "Joao Silva",
    "email": "joao@example.com",
    "cpf": "12345678901",
    "phone": "11987654321"
  },
  "secret": "a1b2c3d4e5f6g7h8i9j0",
  "shareCodes": null,
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

- **Error States**:

| HTTP | Codigo | Descricao | Exibicao para o Operador | Acao de Recuperacao |
|------|--------|-----------|--------------------------|---------------------|
| 400 | `invalid-secret-hash` | Secret do QR code invalido | "QR code invalido" | Solicitar novo QR code ao usuario |
| 400 | `secret-expired` | Secret expirado (QR code dinamico) | "QR code expirado" | Usuario deve gerar novo QR code no app |
| 400 | `undefined-benefit-start-date` | Beneficio sem data de inicio definida | "Beneficio mal configurado" | Admin deve configurar `eventStartsAt` |
| 400 | `benefit-not-started` | Beneficio ainda nao comecou | "Beneficio ainda nao esta ativo" | Aguardar data de inicio do evento |
| 400 | `benefit-expired` | Beneficio expirado | "Beneficio expirado" | Nao e possivel usar — periodo encerrado |
| 400 | `temporary-usage-limit-reached` | Limite temporario atingido (usageRule) | "Limite temporario atingido" | Aguardar proximo periodo permitido |
| 400 | `usage-limit-reached` | Limite total de uso atingido | "Todos os usos ja foram consumidos" | Nao e possivel usar — limite total atingido |
| 400 | `wrong-checkin-time` | Fora do horario de check-in | "Fora do horario de check-in" | Aguardar horario de check-in configurado |
| 400 | `undefined-checkin-times` | Horarios de check-in nao configurados | "Horarios de check-in nao configurados" | Admin deve configurar check-in times |
| 403 | `invalid-token-owner` | Usuario nao e dono do token | "Usuario nao possui este token" | Verificar se o usuario possui o NFT |
| 403 | `self-use-not-allowed` | Self-use nao habilitado | "Uso proprio nao permitido" | Beneficio requer verificacao por operador |
| 403 | `unsatisfied-benefit-requirements` | Requisitos nao atendidos | "Requisitos do beneficio nao atendidos" | Verificar requisitos do beneficio |
| 403 | `unauthorized-operator` | Operador nao autorizado para este beneficio | "Voce nao tem permissao para este beneficio" | Operador nao esta atribuido via `tokenPassBenefitOperators` |
| 404 | `benefit-not-found` | Beneficio nao encontrado | "Beneficio nao encontrado" | Verificar `benefitId` |
| 404 | `token-not-found` | Token nao encontrado | "Token nao encontrado" | Verificar `editionNumber` e contrato |

> **Nota:** Em caso de erro na verificacao (`verifyError = true`), o modal `VerifyBenefit` exibe icone de erro com mensagem "QR code invalido" e botao "Voltar". A `ErrorBox` tambem renderiza a mensagem de erro da API quando disponivel.
>
> **Referência:** Para a tabela completa de erros e o formato padrão de resposta de erro, consulte [PASS_API_REFERENCE.md — Endpoint 14 e Formato de Erro](./PASS_API_REFERENCE.md).

- **State Changes**: `verifyBenefit` populado com dados do usuario e beneficio. Modal `VerifyBenefit` transiciona de loading para exibicao dos dados.

---

### Step 6: Modal de Confirmacao

- **Screen**: Modal `VerifyBenefit` (tela cheia, fundo branco) exibindo:
  - Nome do pass (ex: "Pass VIP Evento") em azul (`#295BA6`)
  - Nome do beneficio (ex: "Beneficio: Acesso Backstage")
  - Descricao do beneficio (renderizada como HTML)
  - Regras do beneficio (renderizada como HTML)
  - Dados do usuario: Username e E-mail
  - Enderecos fisicos (se `type === 'physical'`): nome, rua, cidade, estado, regras do local
  - Botao "Confirmar Uso" (primario)
  - Botao "Cancelar" (secundario)
  - `ErrorBox` para erros do registro de uso

- **User Action**: Clica em "Confirmar Uso" para registrar o uso do beneficio.

- **Frontend Validation**: No fluxo via `PassesDetail`, verifica se o `benefitId` do QR code corresponde ao beneficio selecionado na tabela (`benefitId === benefitIdQR`). Se nao corresponde, exibe `QrCodeError` com `TypeError.invalid`.

- **State Changes**: Ao clicar "Confirmar Uso", dispara `registerUse()` (mutation). Botoes ficam desabilitados durante o loading (`isLoading = true`).

---

### Step 7: Registro de Uso

- **Screen**: Modal `VerifyBenefit` com botoes desabilitados durante o processamento.
- **User Action**: Nenhuma (automatico apos clique em "Confirmar Uso").

- **API Call**:

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456",
  "editionNumber": 42,
  "secret": "a1b2c3d4e5f6g7h8i9j0"
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `userId` | `uuid` | Sim | ID do usuario dono do token |
| `editionNumber` | `number` | Sim | Numero da edicao do token |
| `secret` | `string` | Sim | Secret obtido do QR code |

> **Nota:** O hook `usePostBenefitRegisterUse` tambem envia `benefitId` no body, mas ele e usado apenas para construir a URL do endpoint (`.replace('{id}', body.benefitId)`).

- **Response Handling** (201 Created):

```json
{
  "id": "use-uuid",
  "editionNumber": 42,
  "tokenPassBenefit": {
    "id": "b1c2d3e4-...",
    "name": "Acesso Backstage"
  },
  "tokenPassBenefitId": "b1c2d3e4-...",
  "uses": 2,
  "createdAt": "2024-06-15T19:30:00Z",
  "updatedAt": "2024-06-15T19:30:00Z"
}
```

- **Error States**:

| Situacao | Comportamento |
|----------|--------------|
| `ERR_BAD_REQUEST` (400) | `QrCodeError` com `TypeError.use` — indica que o uso nao pode ser registrado (limite atingido, expirado, etc.) |
| Outro erro | `QrCodeError` generico com mensagem vazia |

- **State Changes**: `onSuccess`: `showSuccess = true`, `showVerify = false`, `showScan = false`. `onError`: `showError = true`, `showVerify = false`, `showScan = false`.

---

### Step 8: Sucesso

- **Screen**: Modal `QrCodeValidated` exibindo:
  - Confirmacao de uso validado com sucesso
  - Nome do beneficio
  - Tipo do beneficio (`physical` / `digital`)
  - Enderecos (se fisico)
  - E-mail e nome do usuario
  - Botao "Validar Outro" — reabre o scanner QR
  - Botao "Fechar" — fecha o modal

- **User Action**: Clica "Validar Outro" para escanear outro QR code, ou "Fechar" para retornar a tela anterior.

- **State Changes**: Ao fechar, `showSuccess = false`. Ao "Validar Outro", `showScan = true` (reabre o scanner).

---

## Metodos Alternativos de Registro

### Registro por userId/CPF (sem QR code)

Permite registrar o uso identificando o usuario por `userId` ou `cpf`, sem necessidade de QR code.

```
POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use-by-user
Authorization: Bearer <token>
Content-Type: application/json
```

**Opcao 1 — por userId:**

```json
{
  "userId": "d4e5f6a7-8901-2345-bcde-f67890123456"
}
```

**Opcao 2 — por CPF:**

```json
{
  "cpf": "12345678901"
}
```

> **Nota:** Enviar `userId` OU `cpf` — pelo menos um e obrigatorio. CPF deve ser 11 digitos sem pontuacao.

### Registro por QR Code string

Envia a string completa do QR code para o backend processar.

```
POST /token-pass-benefits/tenants/{tenantId}/register-use-by-qrcode
Authorization: Bearer <token>
Content-Type: application/json
```

```json
{
  "qrcode": "42;d4e5f6a7-8901-2345-bcde-f67890123456;a1b2c3d4e5f6g7h8i9j0;b1c2d3e4-5678-90ab-cdef-1234567890ab"
}
```

> **Nota:** Este endpoint usa ponto-e-virgula (`;`) como separador na string do qrcode, diferente do formato usado pelo frontend SDK que usa virgula (`,`). O formato e: `{editionNumber};{userId};{secret};{benefitId}`.

---

## API Sequence

1. `GET /token-passes/tenants/{tenantId}/users/{userId}` — Lista passes atribuidos ao operador
2. `GET /token-pass-benefits/tenants/{tenantId}?tokenPassId={id}&chainId={chainId}&contractAddress={contractAddress}` — Lista beneficios do pass com status
3. `GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/verify?userId={userId}&editionNumber={editionNumber}&secret={secret}` — Verifica validade do beneficio
4. `POST /token-pass-benefits/tenants/{tenantId}/{benefitId}/register-use` — Registra uso do beneficio

---

## Error Recovery

| Situacao | Comportamento |
|----------|--------------|
| QR code com formato invalido (menos de 4 campos) | `QrCodeError` com `TypeError.read` — "Formato invalido" |
| `benefitId` do QR nao corresponde ao beneficio selecionado | `QrCodeError` com `TypeError.invalid` — "Beneficio invalido" |
| Erro de verificacao (`verifyError`) | `VerifyBenefit` exibe icone de erro + "QR code invalido" + botao "Voltar" |
| `invalid-secret-hash` / `secret-expired` | Solicitar novo QR code ao usuario |
| `benefit-not-started` | Aguardar data de inicio do evento |
| `benefit-expired` | Nao e possivel usar — periodo encerrado |
| `usage-limit-reached` | Nao e possivel usar — limite total atingido |
| `temporary-usage-limit-reached` | Aguardar proximo periodo permitido |
| `wrong-checkin-time` | Aguardar horario de check-in configurado |
| `unauthorized-operator` | Operador nao esta atribuido a este beneficio |
| `invalid-token-owner` | Verificar se o usuario possui o NFT |
| `ERR_BAD_REQUEST` no registro | `QrCodeError` com `TypeError.use` |
| Erro de rede | `ErrorBox` renderiza mensagem de erro da API |

---

## Implementacao — React SDK (w3block-ui-sdk)

### Componentes

| Componente | Responsabilidade |
|-----------|-----------------|
| `PassesList` | Lista de passes do operador. Verifica role (`admin`/`superAdmin`/`operator`). Botao "Escanear QR Code" global. Orquestra scan, verify e register. |
| `PassesDetail` | Detalhes do pass com tabela de beneficios filtrados por operador. Botao "Validar" por beneficio. Orquestra scan, verify e register com validacao de `benefitId`. |
| `VerifyBenefit` | Modal fullscreen. Exibe dados do beneficio e usuario. Botoes "Confirmar Uso" e "Cancelar". States: loading, erro, dados. |
| `PassCard` | Card individual do pass com imagem, nome e link para detalhes. |
| `QrCodeReader` | Modal com scanner de camera para leitura de QR code. |
| `QrCodeValidated` | Modal de sucesso apos validacao. Exibe nome, tipo, endereco, dados do usuario. |
| `QrCodeError` | Modal de erro. Tipos: `read` (formato invalido), `use` (erro no registro), `invalid` (beneficio nao corresponde). |

### Hooks

| Hook | Tipo | Descricao |
|------|------|-----------|
| `useGetPassByUser()` | `usePrivateQuery` | Lista passes do operador. Rota: `GET /token-passes/tenants/{tenantId}/users/{userId}`. Habilitado apenas para roles admin/superAdmin/operator. |
| `useGetPassBenefits()` | `usePrivateQuery` | Lista beneficios de um pass. Rota: `GET /token-pass-benefits/tenants/{tenantId}`. Params: `tokenPassId`, `chainId`, `contractAddress`. |
| `useGetPassById()` | `usePrivateQuery` | Detalhes de um pass por ID. Rota: `GET /token-passes/tenants/{tenantId}/{id}`. Retorna `tokenPassBenefits` com `tokenPassBenefitOperators`. |
| `useVerifyBenefit()` | `usePrivateQuery` | Verifica beneficio. Rota: `GET /token-pass-benefits/tenants/{tenantId}/{benefitId}/verify`. Habilitado quando `secret` e `tenantId` estao disponiveis. |
| `usePostBenefitRegisterUse()` | `useMutation` (TanStack Query) | Registra uso do beneficio. Rota: `POST /token-pass-benefits/tenants/{tenantId}/{id}/register-use`. Callbacks `onSuccess`/`onError` gerenciam transicao de modais. |

### Fluxo de Estado dos Modais

```
[Lista/Detalhe]
    |
    | clique "Escanear QR Code" / "Validar"
    v
[QrCodeReader] showScan=true
    |
    | scan do QR code
    |--- formato invalido --> [QrCodeError] showError=true (TypeError.read)
    |--- benefitId diferente --> [QrCodeError] showError=true (TypeError.invalid)
    |
    | formato valido
    v
[VerifyBenefit] showVerify=true
    |
    | useVerifyBenefit() loading...
    |--- erro na API --> VerifyBenefit exibe icone de erro
    |
    | dados carregados
    |--- clique "Cancelar" --> fecha modal
    |--- clique "Confirmar Uso" --> registerUse()
         |
         |--- onSuccess --> [QrCodeValidated] showSuccess=true
         |--- onError --> [QrCodeError] showError=true (TypeError.use)
```

### Paginas Next.js

| Pagina | Rota | Componente |
|--------|------|-----------|
| Lista de Passes | `/tokens/pass` | `PassPage` -> `PassesList` |
| Detalhes do Pass | `/tokens/pass/[tokenPassId]` | `PassIdPage` -> `PassesDetail` |

---

## Implementacao — teste-skills (w3block-teste-skills)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `OperatorPassList` | `components/pass/operator-pass-list.tsx` | Cards de passes (imagem, tokenName, descricao). Botao "Escanear QR Code" global (sem `expectedBenefitId`). Orquestra scan → verify → register → result. |
| `OperatorPassDetail` | `components/pass/operator-pass-detail.tsx` | Header do pass + beneficios filtrados por operador via `tokenPassBenefitOperators` (Set de IDs). Botao "Validar" por beneficio com `expectedBenefitId`. Paginacao. |
| `OperatorBenefitDetail` | `components/pass/operator-benefit-detail.tsx` | Detalhe do beneficio com botao "Validar" full-width (substitui QR code do user view). Scan com `expectedBenefitId`. |
| `QrCodeScanner` | `components/pass/qr-code-scanner.tsx` | Modal Dialog com camera via `html5-qrcode`. Fallback de camera (environment → first available). Wait DOM via requestAnimationFrame + 350ms. `min-h-[300px]`. Limpa innerHTML antes de criar scanner. |
| `VerifyBenefitModal` | `components/pass/verify-benefit-modal.tsx` | Modal: loading (Spinner) → dados (nome pass/beneficio, usuario, email, usos) → "Confirmar Uso". Erro: icone X + mensagem traduzida via `pass.apiErrors`. |
| `QrResultModal` | `components/pass/qr-result-modal.tsx` | Sucesso: checkmark verde + "Validar Outro". Erro: X vermelho + mensagem. Botao "Fechar". |

### Hooks

| Hook | Arquivo | Tipo | Descricao |
|------|---------|------|-----------|
| `useGetOperatorPasses` | `hooks/use-get-operator-passes.ts` | `useQuery` | Passes do operador. Usa `getPassesByUser`, retorna `data.items`. |
| `useGetPass` | `hooks/use-get-pass.ts` | `useQuery` | Detalhes do pass. Retorna `tokenPassBenefits` com `tokenPassBenefitOperators`. |
| `useGetBenefits` | `hooks/use-get-benefits.ts` | `useQuery` | Lista beneficios paginados (tokenPassId, type, page, limit). |
| `useGetBenefit` | `hooks/use-get-benefit.ts` | `useQuery` | Detalhes do beneficio por ID. |
| `useVerifyBenefit` | `hooks/use-verify-benefit.ts` | `useQuery` | Verifica beneficio. `enabled` quando params != null. `retry: false`, `staleTime: 0`. |
| `useRegisterBenefitUse` | `hooks/use-register-benefit-use.ts` | `useMutation` | POST `/register-use`. Invalida queries benefit + benefits. |

### Filtro de Beneficios por Operador

```typescript
// operator-pass-detail.tsx
const passBenefits = pass.tokenPassBenefits ?? pass.benefits ?? [];
const operatorBenefitIds = new Set(
  passBenefits
    .filter((b) => (b.tokenPassBenefitOperators ?? []).some((op) => op.userId === userId))
    .map((b) => b.id)
);
const allBenefits = benefitsData?.items ?? [];
const benefits = allBenefits.filter((b) => operatorBenefitIds.has(b.id));
```

### Fluxo de Estado dos Modais

```
[OperatorPassList / OperatorPassDetail / OperatorBenefitDetail]
    |
    | clique "Escanear QR Code" / "Validar"
    v
[QrCodeScanner] scanOpen=true
    |
    | scan do QR code (html5-qrcode)
    |--- formato invalido (parts.length !== 4) --> handleScanError --> [QrResultModal] error
    |--- benefitId diferente (expectedBenefitId) --> handleScanError --> [QrResultModal] error
    |
    | formato valido
    v
[VerifyBenefitModal] verifyOpen=true
    |
    | useVerifyBenefit() loading...
    |--- erro na API --> VerifyBenefitModal exibe XCircle + mensagem traduzida
    |
    | dados carregados
    |--- clique "Cancelar" --> fecha modal
    |--- clique "Confirmar Uso" --> useRegisterBenefitUse.mutate()
         |
         |--- onSuccess --> handleVerifySuccess --> [QrResultModal] success
         |--- onError --> handleVerifyError --> [QrResultModal] error
```

### Paginas Next.js

| Pagina | Rota | Componente |
|--------|------|-----------|
| Lista de Passes | `/operator` | `OperatorPage` → `OperatorPassList` |
| Detalhes do Pass | `/operator/[id]` | `OperatorPassDetailPage` → `OperatorPassDetail` |
| Detalhes do Beneficio | `/operator/[id]/benefits/[benefitId]` | `OperatorBenefitDetailPage` → `OperatorBenefitDetail` |
