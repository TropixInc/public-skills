---
id: PASS_SKILL_INDEX
title: "Pass Skill Index"
module: pass
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
---

# Pass Skill Index

Índice master de toda a documentação do módulo Pass do W3block. Use este documento como ponto de entrada para implementar qualquer funcionalidade de token passes e benefícios.

> **Swagger (documentação interativa):** Acesse `https://pass.w3block.io/api-docs` para testar endpoints e ver schemas atualizados.

---

## Getting Started

Se você está implementando pela primeira vez, siga estes passos:

```
1. Autentique-se na Identity API (https://id.w3block.io) → obtenha Bearer token
2. GET  /token-passes/tenants/{tenantId}                              → Liste os passes disponíveis
3. GET  /token-passes/tenants/{tenantId}/{id}/token-editions/{ed}/benefits → Veja benefícios de um token
4. Escolha o fluxo que precisa implementar na Tabela de Decisão abaixo
```

> Para detalhes sobre autenticação, formato de erros e exemplos de código, consulte [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md).

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md) | 1.0.0 | Endpoints, schemas JSON, enums, erros, constantes | ✅ Referência | Consulta de API em qualquer momento |
| 2 | [FLOW_PASS_OVERVIEW.md](./FLOW_PASS_OVERVIEW.md) | 1.0.0 | Visão geral, personas, arquitetura, componentes SDK | ✅ Atualizado | Entender o módulo antes de implementar |
| 3 | [FLOW_PASS_MANAGEMENT.md](./FLOW_PASS_MANAGEMENT.md) | 1.0.0 | CRUD: criar pass, benefícios, endereços, operadores | ✅ Implementado | Setup inicial como admin |
| 4 | [FLOW_PASS_BENEFIT_VERIFICATION.md](./FLOW_PASS_BENEFIT_VERIFICATION.md) | 1.0.0 | Scan QR → verificar → registrar uso | ✅ Implementado | Operador verificando benefício |
| 5 | [FLOW_PASS_BENEFITS_VIEW.md](./FLOW_PASS_BENEFITS_VIEW.md) | 1.0.0 | Listar e visualizar benefícios (operador + usuário) | ✅ Implementado | Ver benefícios de um pass/token |
| 6 | [FLOW_PASS_SELF_USE.md](./FLOW_PASS_SELF_USE.md) | 1.0.0 | Usuário registra uso próprio de benefício | ✅ Implementado | Self-use (allowSelfUse: true) |
| 7 | [FLOW_PASS_SHARE.md](./FLOW_PASS_SHARE.md) | 1.0.0 | Gift card: gerar, compartilhar, acessar | ✅ Implementado | Compartilhar pass como presente |
| 8 | [FLOW_PASS_BENEFIT_TRACKING.md](./FLOW_PASS_BENEFIT_TRACKING.md) | 1.0.0 | Histórico de uso + exportação XLS | ✅ Implementado | Rastrear quem usou o quê |

---

## Guia Rápido

### Para implementar verificação de benefício (API-first):

```
1. GET  /token-pass-benefits/.../verify?userId=...&editionNumber=...&secret=...  → Verificar
2. POST /token-pass-benefits/.../{id}/register-use                                → Registrar uso
```

### Para implementar visualização de benefícios do usuário:

```
1. GET /token-passes/.../token-editions/{editionNumber}/benefits  → Listar benefícios com status
2. GET /token-pass-benefits/.../{id}/{editionNumber}/secret       → Obter secret para QR code
```

### Para setup completo como admin:

```
1. POST /token-passes/tenants/{tenantId}                    → Criar pass
2. POST /token-pass-benefits/tenants/{tenantId}              → Criar benefício
3. POST /token-pass-benefit-addresses/tenants/{tenantId}     → Adicionar endereço (se físico)
4. POST /token-pass-benefit-operators/tenants/{tenantId}     → Atribuir operador
```

### Para compartilhar pass (gift card):

```
1. POST /token-pass-share-codes/tenants/{tenantId}           → Criar share code
2. GET  /token-pass-share-codes/tenants/{tenantId}/{code}    → Consultar (público)
```

---

## Implementacoes

O modulo Pass possui duas implementacoes frontend:

| Projeto | Descricao | Rotas Operador | Rotas Usuario |
|---------|-----------|----------------|---------------|
| **w3block-ui-sdk** | SDK React original | `/tokens/pass`, `/tokens/pass/[id]` | Dentro do PassTemplate |
| **w3block-teste-skills** | Next.js 15 App Router | `/operator`, `/operator/[id]`, `/operator/[id]/benefits/[benefitId]` | `/pass`, `/pass/[id]`, `/pass/[id]/benefits/[benefitId]` |

Cada documento de fluxo possui secoes separadas para SDK e teste-skills com componentes, hooks e rotas especificos.

---

## Armadilhas Comuns (leia antes de implementar!)

| # | Problema | Solução |
|---|----------|---------|
| 1 | QR code usa vírgula no SDK mas ponto-e-vírgula na API `register-use-by-qrcode` | Frontend SDK monta: `editionNumber,userId,secret,benefitId`. Endpoint `register-use-by-qrcode` espera: `editionNumber;userId;secret;benefitId`. Converter separador se usar API direta |
| 2 | `verify` é GET, não POST | Os dados (`userId`, `editionNumber`, `secret`) vão como **query params**, não no body |
| 3 | `allowSelfUse` precisa estar `true` | Endpoint `/use` retorna 403 `self-use-not-allowed` se `allowSelfUse: false`. Verificar config do benefício |
| 4 | Operator não pode criar passes | Roles para CRUD: `superAdmin`, `admin`. Operator só pode verificar/registrar uso |
| 5 | Check-in times usam timezone do benefício | Default: `America/Sao_Paulo`. Configurar `timezoneOrUtcOffset` se diferente. Erros `wrong-checkin-time` consideram a timezone |
| 6 | `dynamicQrCode: true` gera secrets que expiram | O usuário deve gerar novo QR code antes de cada uso. Secret expirado retorna 400 `secret-expired` |
| 7 | Filtrar benefícios por operador no frontend | Use o campo `tokenPassBenefitOperators` de cada benefício para exibir apenas os benefícios atribuídos ao operador logado |

---

## Tabela de Decisão: O que implementar?

| Quero... | Sou... | Use este documento |
|----------|--------|-------------------|
| Criar passes e benefícios do zero | Admin | [FLOW_PASS_MANAGEMENT](./FLOW_PASS_MANAGEMENT.md) |
| Configurar horários de check-in | Admin | [FLOW_PASS_MANAGEMENT](./FLOW_PASS_MANAGEMENT.md) (Step 4) |
| Atribuir operadores a benefícios | Admin | [FLOW_PASS_MANAGEMENT](./FLOW_PASS_MANAGEMENT.md) (Step 6) |
| Verificar benefício escaneando QR code | Operador | [FLOW_PASS_BENEFIT_VERIFICATION](./FLOW_PASS_BENEFIT_VERIFICATION.md) |
| Registrar uso de benefício sem QR | Operador | [FLOW_PASS_BENEFIT_VERIFICATION](./FLOW_PASS_BENEFIT_VERIFICATION.md) (Métodos Alternativos) |
| Ver benefícios que gerencio | Operador | [FLOW_PASS_BENEFITS_VIEW](./FLOW_PASS_BENEFITS_VIEW.md) (Fluxo A) |
| Ver benefícios do meu token/NFT | Usuário | [FLOW_PASS_BENEFITS_VIEW](./FLOW_PASS_BENEFITS_VIEW.md) (Fluxo B) |
| Usar benefício sem operador | Usuário | [FLOW_PASS_SELF_USE](./FLOW_PASS_SELF_USE.md) |
| Gerar QR code para meu benefício | Usuário | [FLOW_PASS_BENEFITS_VIEW](./FLOW_PASS_BENEFITS_VIEW.md) (Fluxo B, Step 2) |
| Compartilhar pass como gift card | Admin/Sistema | [FLOW_PASS_SHARE](./FLOW_PASS_SHARE.md) |
| Ver histórico de usos | Operador/Admin | [FLOW_PASS_BENEFIT_TRACKING](./FLOW_PASS_BENEFIT_TRACKING.md) |
| Exportar relatório de usos em XLS | Operador/Admin | [FLOW_PASS_BENEFIT_TRACKING](./FLOW_PASS_BENEFIT_TRACKING.md) (Exportação) |
| Consultar detalhes de um endpoint | Qualquer | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Método | Management | Verification | Benefits View | Self-Use | Share | Tracking |
|----------|--------|:-:|:-:|:-:|:-:|:-:|:-:|
| `POST .../token-passes` | Criar pass | **X** | | | | | |
| `GET .../token-passes` | Listar passes | **X** | | | | | |
| `GET .../token-passes/users/{userId}` | Passes do operador | | **X** | **X** | | | |
| `GET .../token-editions/{ed}/benefits` | Benefícios por edição | | | **X** | **X** | | |
| `POST .../token-pass-benefits` | Criar benefício | **X** | | | | | |
| `GET .../token-pass-benefits` | Listar benefícios | | **X** | **X** | | | |
| `GET .../token-pass-benefits/{id}` | Detalhes de benefício | | | **X** | | | |
| `GET .../{id}/{ed}/secret` | Secret QR code | | | **X** | | | |
| `GET .../{id}/verify` | Verificar benefício | | **X** | | | | |
| `POST .../{id}/register-use` | Registrar uso (secret) | | **X** | | | | |
| `POST .../{id}/register-use-by-user` | Registrar uso (user/CPF) | | **X** | | | | |
| `POST .../register-use-by-qrcode` | Registrar uso (QR string) | | **X** | | | | |
| `POST .../{id}/use` | Self-use | | | | **X** | | |
| `GET .../usages` | Listar usos | | | | | | **X** |
| `GET .../usages/xls` | Exportar XLS | | | | | | **X** |
| `POST .../token-pass-share-codes` | Criar share code | | | | | **X** | |
| `GET .../token-pass-share-codes/{code}` | Consultar share code | | | | | **X** | |
| `POST .../benefit-addresses` | Criar endereço | **X** | | | | | |
| `POST .../benefit-operators` | Criar operador | **X** | | | | | |
| `GET .../exports/{id}` | Status da exportação | | | | | | **X** |

---

## Fluxo Visual

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MÓDULO PASS W3BLOCK                               │
│                                                                             │
│  ┌─────────────┐                                                            │
│  │    ADMIN     │                                                            │
│  │             │                                                            │
│  │ Cria Pass   │──→ Cria Benefícios ──→ Atribui Operadores                  │
│  │ (CRUD)      │    (digital/físico)    (userId + benefitId)                │
│  └─────────────┘                                                            │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────┐                    │
│  │  OPERADOR   │    │   USUÁRIO    │    │  GIFT CARD   │                    │
│  │             │    │              │    │              │                    │
│  │ Lista Passes│    │ Compra Token │    │ Checkout com │                    │
│  │      │      │    │ (NFT)        │    │ shareCode    │                    │
│  │      ▼      │    │      │       │    │      │       │                    │
│  │ Escaneia QR │    │      ▼       │    │      ▼       │                    │
│  │      │      │    │ Vê Benefícios│    │ Gera código  │                    │
│  │      ▼      │    │ (por edition)│    │      │       │                    │
│  │ Verifica    │    │      │       │    │      ▼       │                    │
│  │      │      │    │      ▼       │    │ Compartilha  │                    │
│  │      ▼      │    │ ┌─────────┐  │    │ URL pública  │                    │
│  │ Registra    │    │ │Self-Use │  │    │      │       │                    │
│  │ Uso         │    │ │(se hab.)│  │    │      ▼       │                    │
│  │      │      │    │ └─────────┘  │    │ Destinatário │                    │
│  │      ▼      │    │      │       │    │ acessa pass  │                    │
│  │ Histórico   │    │ Gera QR Code │    └──────────────┘                    │
│  │ + Export XLS│    │ (p/ operador)│                                        │
│  └─────────────┘    └──────────────┘                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Glossário Rápido

| Termo | Descrição |
|-------|-----------|
| `tenantId` | UUID da empresa (tenant) na plataforma W3block |
| `tokenPassId` | UUID do token pass (agrupa benefícios) |
| `benefitId` | UUID de um benefício individual |
| `editionNumber` | Número da edição do token/NFT (identifica qual unidade) |
| `secret` | String de autenticação para validação de QR code |
| `chainId` | ID da blockchain (137 = Polygon, 1 = Ethereum) |
| `contractAddress` | Endereço do contrato NFT na blockchain |
| `collectionId` | UUID da collection de tokens (usado ao criar pass) |
| `dynamicQrCode` | Se `true`, QR code usa secret renovável (expira periodicamente) |
| `allowSelfUse` | Se `true`, usuário pode registrar uso próprio sem operador |
| `checkIn` | Configuração de horários de check-in por dia da semana |
| `usageRule` | Regra de limite temporário de uso (`timestamp_relative`, `day_relative`) |
| `useLimit` | Número máximo de usos por token (null = ilimitado) |
| `useAvailable` | Usos restantes disponíveis para uma edição específica |
| `passShareCodeData` | Dados customizáveis enviados no checkout para gift card |
| `operator` | Usuário com role de operador, responsável por verificar/registrar usos |
| `BenefitUseStatusEnum` | Status calculado: `active` (disponível), `inactive` (fora do período), `unavailable` (sem usos) |
| `TokenPassBenefitTypeEnum` | Tipo do benefício: `digital` (link/conteúdo) ou `physical` (local/evento) |
