---
id: FLOW_PASS_OVERVIEW
title: "Pass - Visao Geral"
module: pass
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - pass
  - overview
depends_on:
  - PASS_API_REFERENCE
---

# Pass Module — Visão Geral

## Overview

O módulo Pass do W3block é um sistema de **benefícios e fidelidade digital/físico** baseado em NFTs. Ele permite criar token passes com benefícios associados, verificar utilizações via QR code, compartilhar passes (gift cards) e rastrear histórico de usos.

Os benefícios podem ser de dois tipos:
- **Digitais** — link, conteúdo, acesso exclusivo
- **Físicos** — localização, evento presencial, produto tangível

O sistema opera com **3 personas** distintas (Admin, Operador, Usuário), cada uma com permissões e fluxos próprios.

---

## Pré-requisitos

Para utilizar o módulo Pass, você precisa de:

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| **Autenticação** | Bearer token do usuário | Login via Identity API — consulte a [documentação da Identity API](https://id.w3block.io/api-docs) |
| **Roles** | Role compatível com a ação desejada | `admin`, `operator` ou `user` atribuídos no tenant |
| **Token Pass configurado** | Pass criado com benefícios associados | Admin cria via painel ou API |
| **Collection com tokens mintados** | Tokens NFT emitidos na collection do pass | Mint via API ou painel administrativo |

---

## Personas

| Persona | Roles | Pode fazer |
|---------|-------|------------|
| **Admin** | `superAdmin`, `admin` | CRUD de passes, benefícios, endereços, operadores. Ver tudo. Configurar templates e regras de uso. |
| **Operador** | `operator` | Listar passes atribuídos, verificar QR code, registrar uso de benefício, ver histórico de utilizações. |
| **Usuário** | `user` | Ver benefícios do token que possui, realizar self-use, gerar QR code para verificação por operador. |

---

## Fluxo Visual

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           FLUXO DO ADMIN                                    │
│                                                                              │
│   Cria Pass ──→ Adiciona Benefícios ──→ Atribui Operadores                  │
│       │              │                        │                              │
│       ▼              ▼                        ▼                              │
│   PassesList    BenefitDetails          ConfigPanel                          │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                          FLUXO DO USUÁRIO                                   │
│                                                                              │
│   Compra Token ──→ Vê Benefícios ──→ Gera QR Code ──→ Operador escaneia   │
│       │                │                   │                                 │
│       │                │                   ▼                                 │
│       │                │             QrCodeSection                           │
│       │                │                                                     │
│       │                └──→ Self-Use (uso direto sem operador)               │
│       │                          │                                           │
│       ▼                          ▼                                           │
│   DetailPass              usePostSelfUseBenefit                              │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                         FLUXO DO OPERADOR                                   │
│                                                                              │
│   Lista Passes ──→ Escaneia QR ──→ Verifica Benefício ──→ Registra Uso    │
│       │                │                   │                     │           │
│       ▼                ▼                   ▼                     ▼           │
│   PassesList     QrCodeSection      VerifyBenefit      usePostBenefitUse    │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                      FLUXO DE COMPARTILHAMENTO                              │
│                                                                              │
│   Checkout gera share code ──→ Usuário compartilha ──→ Destinatário acessa │
│              │                         │                        │            │
│              ▼                         ▼                        ▼            │
│   useGetTokenSharedCode          Link com código          SharedOrder        │
│                                                        (PassCodePage)       │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Rotas

### SDK (w3block-ui-sdk)

| Rota | Componente | Descrição |
|------|-----------|-----------|
| `/tokens/pass` | `PassPage` | Lista de passes do operador. Exibe todos os passes atribuídos com status e ações. |
| `/tokens/pass/[tokenPassId]` | `PassIdPage` | Detalhes de um pass específico com lista de benefícios, configurações e histórico. |
| `/pass/share/[code]` | `PassCodePage` | Visualização de pass compartilhado via share code. Acessível sem autenticação. |

### teste-skills (w3block-teste-skills)

Operator e user têm rotas totalmente separadas.

| Rota | Componente | Persona | Descrição |
|------|-----------|---------|-----------|
| `/operator` | `OperatorPassList` | Operador | Passes com cards (imagem, tokenName, descrição) + botão global de scan |
| `/operator/[id]` | `OperatorPassDetail` | Operador | Benefícios filtrados por operador + scan por benefício |
| `/operator/[id]/benefits/[benefitId]` | `OperatorBenefitDetail` | Operador | Detalhe do benefício + botão "Validar" (substitui QR code) |
| `/pass` | `PassList` | Usuário | Lista de passes do usuário |
| `/pass/[id]` | `PassDetail` | Usuário | Detalhes do pass com benefícios |
| `/pass/[id]/benefits/[benefitId]` | `BenefitDetail` | Usuário | Detalhe com QR code + self-use |
| `/my-passes` | redirect → `/pass` | Usuário | Redirecionamento |

---

## Componentes Principais

### SDK (w3block-ui-sdk)

| Componente | Descrição |
|-----------|-----------|
| `PassesList` | Lista de passes com filtros e paginação. Usado por Admin e Operador. |
| `PassesDetail` | Detalhes completos de um pass, incluindo benefícios e configuração. |
| `PassCard` | Card individual de um pass com resumo visual (título, imagem, status). |
| `BenefitDetails` | Detalhes de um benefício específico (tipo, regras de uso, limites). |
| `SharedOrder` | Visualização de pedido compartilhado via share code. |
| `VerifyBenefit` | Interface de verificação de benefício via dados do QR code. |
| `PassTemplate` | Template visual do pass para exibição e personalização. |
| `QrCodeSection` | Seção de geração e exibição de QR code para verificação. |
| `ConfigPanel` | Painel de configuração do pass (operadores, endereços, regras). |
| `BenefitUsesList` | Lista de utilizações de um benefício com histórico e detalhes. |
| `DetailPass` | Visualização detalhada de um pass do ponto de vista do usuário. |

### teste-skills (w3block-teste-skills)

| Componente | Descrição |
|-----------|-----------|
| `OperatorPassList` | Cards de passes do operador (imagem, tokenName, descrição) + scan global. Fluxo completo de verificação. |
| `OperatorPassDetail` | Benefícios filtrados por `tokenPassBenefitOperators` + scan por benefício. Paginação. |
| `OperatorBenefitDetail` | Detalhe do benefício com botão "Validar" (substitui QR code/self-use do user view). |
| `BenefitDetail` | Detalhe do benefício (user): info + `BenefitQrCode` + `SelfUseButton` + endereços. |
| `BenefitQrCode` | QR code com auto-refresh 15s (dinâmico). Formato: `editionNumber,userId,secret,benefitId`. |
| `SelfUseButton` | Botão + dialog de confirmação. Mostra usos restantes no toast de sucesso. |
| `QrCodeScanner` | Scanner de câmera via `html5-qrcode` em Dialog. Fallback de câmera. `min-h-[300px]`. |
| `VerifyBenefitModal` | Modal de verificação: loading → dados do usuário/benefício → "Confirmar Uso". |
| `QrResultModal` | Resultado: sucesso (checkmark + "Validar Outro") ou erro (X + mensagem). |
| `BenefitStatusBadge` | Badge: active (verde), inactive (cinza), unavailable (amarelo). |
| `BenefitTypeBadge` | Badge: digital / physical. |

---

## Hooks Principais

### SDK (w3block-ui-sdk)

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetPassByUser` | `useQuery` | Retorna os passes atribuídos ao operador autenticado. |
| `useGetPass` / `useTokenPass` | `useQuery` | Lista todos os passes disponíveis no tenant. |
| `useGetPassById` | `useQuery` | Retorna detalhes de um pass específico por ID. |
| `useGetPassBenefits` | `useQuery` | Lista todos os benefícios associados a um pass. |
| `useGetPassBenefitById` | `useQuery` | Retorna detalhes de um benefício específico por ID. |
| `useGetBenefitsByEditionNumber` | `useQuery` | Retorna benefícios filtrados por número de edição do token. |
| `useVerifyBenefit` | `useQuery` | Verifica a validade de um benefício a partir dos dados do QR code. |
| `useGetQRCodeSecret` | `useQuery` | Obtém o secret necessário para gerar o QR code de verificação. |
| `useGetTokenSharedCode` | `useQuery` | Consulta o share code associado a um token para compartilhamento. |
| `useGetBenefitUses` | `useQuery` | Retorna o histórico de utilizações de um benefício. |
| `useGetPassBenefitOperators` | `useQuery` | Lista os operadores atribuídos a um benefício específico. |
| `usePostBenefitUse` | `useMutation` | Registra o uso de um benefício (operador escaneia QR). |
| `usePostBenefitRegisterUse` | `useMutation` | Registra uso de benefício com dados completos (localização, metadata). |
| `usePostSelfUseBenefit` | `useMutation` | Registra self-use de benefício diretamente pelo usuário. |

### teste-skills (w3block-teste-skills)

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetOperatorPasses` | `useQuery` | Passes do operador (usa `getPassesByUser`, retorna `items`). |
| `useGetPasses` | `useQuery` | Lista passes do usuário (paginado). |
| `useGetPass` | `useQuery` | Detalhes de um pass por ID. |
| `useGetBenefits` | `useQuery` | Lista benefícios com filtros (tokenPassId, type, page, limit). |
| `useGetBenefit` | `useQuery` | Detalhes de um benefício por ID. |
| `useGetBenefitsByEdition` | `useQuery` | Benefícios por edição do token. |
| `useGetBenefitSecret` | `useQuery` | Secret para QR code (`staleTime: 0`, `gcTime: 0`, `initialData: null`). |
| `useUserEdition` | `useQuery` | Resolve `editionNumber` via Key API (collection tokens → public token data → `edition.currentNumber`). |
| `useVerifyBenefit` | `useQuery` | Verificação de benefício (enabled quando params fornecidos, `retry: false`). |
| `useSelfUseBenefit` | `useMutation` | Self-use: POST `/use`. Invalida queries de benefit/benefits. |
| `useRegisterBenefitUse` | `useMutation` | Registro de uso (operador): POST `/register-use`. Invalida queries de benefit/benefits. |
| `useBenefitStatus` | hook puro | Calcula `BenefitUseStatusEnum` a partir dos dados do benefício. Também exporta `calculateBenefitStatus`. |

---

## Mapa de Documentos

| Documento | Descrição | Quando usar |
|-----------|-----------|-------------|
| [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md) | Referência de endpoints, schemas, enums | Consulta de API em qualquer momento |
| [FLOW_PASS_MANAGEMENT.md](./FLOW_PASS_MANAGEMENT.md) | CRUD de passes e benefícios (Admin) | Implementar criação e edição de passes |
| [FLOW_PASS_BENEFIT_VERIFICATION.md](./FLOW_PASS_BENEFIT_VERIFICATION.md) | Verificação de benefícios via QR code (Operador) | Implementar fluxo de scan e verificação |
| [FLOW_PASS_BENEFITS_VIEW.md](./FLOW_PASS_BENEFITS_VIEW.md) | Visualização de benefícios (Operador + Usuário) | Implementar tela de benefícios do token |
| [FLOW_PASS_SELF_USE.md](./FLOW_PASS_SELF_USE.md) | Self-use de benefícios (Usuário) | Implementar uso direto sem operador |
| [FLOW_PASS_SHARE.md](./FLOW_PASS_SHARE.md) | Compartilhamento de passes (Gift Card) | Implementar fluxo de share code |
| [FLOW_PASS_BENEFIT_TRACKING.md](./FLOW_PASS_BENEFIT_TRACKING.md) | Histórico e rastreamento de usos | Implementar listagem de utilizações |
| [PASS_SKILL_INDEX.md](./PASS_SKILL_INDEX.md) | Índice master do módulo Pass | Ponto de entrada para qualquer implementação |
