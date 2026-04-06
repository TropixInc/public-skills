---
id: OFFPIX_SKILL_INDEX
title: "Índice de Skills Offpix"
module: offpix
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# Índice de Skills Offpix

Índice principal de toda a documentação da API backend Offpix. Estes documentos são voltados para desenvolvedores e usuários assistidos por IA que constroem aplicações customizadas na plataforma W3Block.

**Antes de escrever qualquer nova documentação, leia:** [OFFPIX_DOCUMENTATION_SPEC.md](./OFFPIX_DOCUMENTATION_SPEC.md)

---

## Módulos

| # | Módulo | Versão | Documentos | Status | Descrição |
|---|--------|--------|------------|--------|-----------|
| 0 | [Especificação da Documentação](./OFFPIX_DOCUMENTATION_SPEC.md) | 1.0.0 | 1 | Setup | Especificação do processo para geração de documentação |
| 1 | [Auth](./auth/AUTH_SKILL_INDEX.md) | 1.0.0 | 6 | Implementado | Sign-in (5 métodos), sign-up, reset de senha, OAuth, gerenciamento de tokens |
| 2 | [Commerce](./commerce/COMMERCE_SKILL_INDEX.md) | 1.0.0 | 6 | Implementado | Produtos, pedidos, pagamentos, promoções, splits, gateways, tags |
| 3 | [Configurations](./configurations/CONFIGURATIONS_SKILL_INDEX.md) | 1.0.0 | 5 | Implementado | Contextos, tenant contexts, campos de formulário, ativação de signup, config KYC |
| 4 | [Contacts](./contacts/CONTACTS_SKILL_INDEX.md) | 1.0.0 | 5 | Implementado | Gerenciamento de usuários, submissão e aprovação KYC, royalty, contatos externos |
| 5 | [Contracts & Tokens](./contracts/CONTRACTS_SKILL_INDEX.md) | 1.0.0 | 6 | Implementado | Contratos NFT, coleções, editions, ERC20, royalty, gas, RFID, XLSX |
| 6 | [Loyalty](./loyalty/LOYALTY_SKILL_INDEX.md) | 1.0.0 | 4 | Implementado | Contratos ERC20, programas de fidelidade, rewards, cashback, pagamentos, staking |
| 7 | [Whitelist](./whitelist/WHITELIST_SKILL_INDEX.md) | 1.0.0 | 3 | Implementado | Whitelists (grupos de usuários), entries, verificação de acesso, promoção on-chain |
| 8 | [Pass & Benefits](./pass/PASS_SKILL_INDEX.md) | 1.0.0 | 3 | Implementado | Token passes, benefícios digitais/físicos, QR check-in, operadores, share codes |
| 9 | [Settings & Billing](./settings/SETTINGS_SKILL_INDEX.md) | 1.0.0 | 4 | Implementado | Planos, cartão de crédito, ciclos de cobrança, uso, feature-gating, config de tenant |
| 10 | [Tokenization](./tokenization/TOKENIZATION_SKILL_INDEX.md) | 1.0.0 | 5 | Implementado | Coleções de tokens, editions, mint, transferência, burn, RFID, importação XLSX |
| 11 | [KYC](./kyc/KYC_SKILL_INDEX.md) | 1.0.0 | 5 | Implementado | Verificação de documentos, submissão, fluxos de aprovação, tipos de input, config |
| 12 | [PDF](./pdf/PDF_SKILL_INDEX.md) | 1.0.0 | 4 | Implementado | Geração de certificados, PDFs de pass/ticket, templates Handlebars (interno) |

---

## Documentos do Módulo Auth

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [AUTH_SKILL_INDEX](./auth/AUTH_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [AUTH_API_REFERENCE](./auth/AUTH_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, schemas |
| 3 | [FLOW_AUTH_SIGNIN](./auth/FLOW_AUTH_SIGNIN.md) | Fluxo | Sign-in por senha, OTP, magic token, API key do tenant |
| 4 | [FLOW_AUTH_SIGNUP](./auth/FLOW_AUTH_SIGNUP.md) | Fluxo | Registro de usuário + onboarding de tenant |
| 5 | [FLOW_AUTH_PASSWORD_RESET](./auth/FLOW_AUTH_PASSWORD_RESET.md) | Fluxo | Solicitar reset + definir nova senha |
| 6 | [FLOW_AUTH_OAUTH](./auth/FLOW_AUTH_OAUTH.md) | Fluxo | Google & Apple OAuth (redirect + direto) |

## Documentos do Módulo Commerce

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [COMMERCE_SKILL_INDEX](./commerce/COMMERCE_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [COMMERCE_API_REFERENCE](./commerce/COMMERCE_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, schemas |
| 3 | [FLOW_COMMERCE_PRODUCT_LIFECYCLE](./commerce/FLOW_COMMERCE_PRODUCT_LIFECYCLE.md) | Fluxo | Criar, atualizar, publicar, cancelar produtos; variantes, order rules, tags |
| 4 | [FLOW_COMMERCE_ORDER_PURCHASE](./commerce/FLOW_COMMERCE_ORDER_PURCHASE.md) | Fluxo | Preview de pedido, criação, multi-pagamento, entrega, reembolsos |
| 5 | [FLOW_COMMERCE_PROMOTIONS](./commerce/FLOW_COMMERCE_PROMOTIONS.md) | Fluxo | Cupons, descontos, escopo por produto/usuário, requisitos |
| 6 | [FLOW_COMMERCE_SPLITS_GATEWAYS](./commerce/FLOW_COMMERCE_SPLITS_GATEWAYS.md) | Fluxo | Splits de receita, config Stripe/ASAAS, seleções de provedor |

---

## Domínios Disponíveis (Ainda Não Documentados)

| Domínio | Swagger | Status |
|---------|---------|--------|
| ~~Commerce~~ | https://commerce.w3block.io/docs | **Documentado** |
| ~~Configuration~~ | https://pixwayid.w3block.io/docs/ | **Documentado** |
| ~~Tokens & NFTs~~ | https://api.w3block.io/docs/ | **Documentado** (no módulo Contracts) |
| ~~Tokenization~~ | https://api.w3block.io/docs/ | **Documentado** (módulo dedicado) |
| ~~Pass & Benefits~~ | https://pass.w3block.io/docs | **Documentado** |
| ~~Loyalty~~ | https://api.w3block.io/docs/ | **Documentado** |
| ~~Contacts~~ | https://pixwayid.w3block.io/docs/ | **Documentado** |
| ~~KYC~~ | https://pixwayid.w3block.io/docs/ | **Documentado** (módulo dedicado) |
| ~~Settings & Billing~~ | https://pixwayid.w3block.io/docs/ | **Documentado** |
| ~~Whitelist~~ | https://pixwayid.w3block.io/docs/ | **Documentado** |
| ~~PDF~~ | https://pdf.w3block.io/docs | **Documentado** (serviço interno) |

---

## Documentos do Módulo Configurations

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [CONFIGURATIONS_SKILL_INDEX](./configurations/CONFIGURATIONS_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [CONFIGURATIONS_API_REFERENCE](./configurations/CONFIGURATIONS_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) | Fluxo | Criar, atualizar, listar formulários (contexts + tenant contexts) |
| 4 | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) | Fluxo | Adicionar, atualizar, obter campos de formulário (19 tipos de input) |
| 5 | [FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION](./configurations/FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION.md) | Fluxo | Toggle de registro de signup, configurar formulário de signup |

## Documentos do Módulo Contacts

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [CONTACTS_SKILL_INDEX](./contacts/CONTACTS_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [CONTACTS_API_REFERENCE](./contacts/CONTACTS_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_CONTACTS_USER_MANAGEMENT](./contacts/FLOW_CONTACTS_USER_MANAGEMENT.md) | Fluxo | Listar, convidar, editar usuários, roles, royalty, contatos externos |
| 4 | [FLOW_CONTACTS_KYC_SUBMISSION](./contacts/FLOW_CONTACTS_KYC_SUBMISSION.md) | Fluxo | Buscar campos de formulário, submeter documentos, multi-etapa, resubmissão |
| 5 | [FLOW_CONTACTS_KYC_APPROVAL](./contacts/FLOW_CONTACTS_KYC_APPROVAL.md) | Fluxo | Listar, revisar, aprovar/rejeitar/solicitar revisão de submissões KYC |

## Documentos do Módulo Contracts & Tokens

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [CONTRACTS_SKILL_INDEX](./contracts/CONTRACTS_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [CONTRACTS_API_REFERENCE](./contracts/CONTRACTS_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_CONTRACTS_NFT_LIFECYCLE](./contracts/FLOW_CONTRACTS_NFT_LIFECYCLE.md) | Fluxo | Criar, configurar, fazer deploy de contratos NFT com royalty |
| 4 | [FLOW_CONTRACTS_COLLECTION_MANAGEMENT](./contracts/FLOW_CONTRACTS_COLLECTION_MANAGEMENT.md) | Fluxo | Coleções, editions, templates, publicar, bulk XLSX |
| 5 | [FLOW_CONTRACTS_TOKEN_OPERATIONS](./contracts/FLOW_CONTRACTS_TOKEN_OPERATIONS.md) | Fluxo | Mint, transferir, burn de tokens, RFID, metadata, gas, certificados |
| 6 | [FLOW_CONTRACTS_ERC20_LIFECYCLE](./contracts/FLOW_CONTRACTS_ERC20_LIFECYCLE.md) | Fluxo | ERC20 criar, deploy, mint, regras de transferência (5 handlers), burn |

## Documentos do Módulo Loyalty

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [LOYALTY_SKILL_INDEX](./loyalty/LOYALTY_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [LOYALTY_API_REFERENCE](./loyalty/LOYALTY_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_LOYALTY_CONTRACT_SETUP](./loyalty/FLOW_LOYALTY_CONTRACT_SETUP.md) | Fluxo | Criar contrato ERC20, deploy, criar programa de fidelidade, configurar regras |
| 4 | [FLOW_LOYALTY_BALANCE_OPERATIONS](./loyalty/FLOW_LOYALTY_BALANCE_OPERATIONS.md) | Fluxo | Mint, transferir, burn de tokens; verificar saldos e histórico |
| 5 | [FLOW_LOYALTY_REWARDS_PAYMENTS](./loyalty/FLOW_LOYALTY_REWARDS_PAYMENTS.md) | Fluxo | Depósitos, cashback, comissões multinível, pagamentos, staking, rollbacks |

## Documentos do Módulo Pass & Benefits

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [PASS_SKILL_INDEX](./pass/PASS_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [PASS_API_REFERENCE](./pass/PASS_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, códigos de erro, schemas de check-in |
| 3 | [FLOW_PASS_LIFECYCLE](./pass/FLOW_PASS_LIFECYCLE.md) | Fluxo | Criar pass, adicionar benefícios, config de check-in, operadores, share codes |
| 4 | [FLOW_PASS_BENEFIT_REDEMPTION](./pass/FLOW_PASS_BENEFIT_REDEMPTION.md) | Fluxo | Geração de QR, 4 métodos de resgate, cadeia de validação, rastreamento de uso |

## Documentos do Módulo Settings & Billing

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [SETTINGS_SKILL_INDEX](./settings/SETTINGS_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [SETTINGS_API_REFERENCE](./settings/SETTINGS_API_REFERENCE.md) | Referência da API | Todos os 16 endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_SETTINGS_BILLING_MANAGEMENT](./settings/FLOW_SETTINGS_BILLING_MANAGEMENT.md) | Fluxo | Seleção de plano, cartão de crédito, ciclos de cobrança, uso, simulação, cancelamento |
| 4 | [FLOW_SETTINGS_TENANT_CONFIGURATION](./settings/FLOW_SETTINGS_TENANT_CONFIGURATION.md) | Fluxo | Perfil do tenant, provedores de auth, KYC, passwordless, email, verificação de features |

## Documentos do Módulo Tokenization

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [TOKENIZATION_SKILL_INDEX](./tokenization/TOKENIZATION_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [TOKENIZATION_API_REFERENCE](./tokenization/TOKENIZATION_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_TOKENIZATION_COLLECTION_LIFECYCLE](./tokenization/FLOW_TOKENIZATION_COLLECTION_LIFECYCLE.md) | Fluxo | Criar rascunho, sincronizar editions, configurar, publicar na blockchain |
| 4 | [FLOW_TOKENIZATION_EDITION_OPERATIONS](./tokenization/FLOW_TOKENIZATION_EDITION_OPERATIONS.md) | Fluxo | Mint, transferir, burn, atualização de metadata, RFID, estimativa de gas |
| 5 | [FLOW_TOKENIZATION_BULK_IMPORT](./tokenization/FLOW_TOKENIZATION_BULK_IMPORT.md) | Fluxo | Exportação de template XLSX, importação em massa de editions, rastreamento de job assíncrono |

## Documentos do Módulo KYC

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [KYC_SKILL_INDEX](./kyc/KYC_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [KYC_API_REFERENCE](./kyc/KYC_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, catálogo de tipos de input |
| 3 | [FLOW_KYC_SUBMISSION](./kyc/FLOW_KYC_SUBMISSION.md) | Fluxo | Buscar formulário, upload de arquivos, submeter documentos, multi-etapa, resubmissão |
| 4 | [FLOW_KYC_APPROVAL](./kyc/FLOW_KYC_APPROVAL.md) | Fluxo | Listar pendentes, revisar, aprovar/rejeitar/solicitar revisão, trilha de auditoria |
| 5 | [FLOW_KYC_CONFIGURATION](./kyc/FLOW_KYC_CONFIGURATION.md) | Fluxo | Setup de tenant context, auto-aprovação, whitelists de aprovadores, notificações |

## Documentos do Módulo PDF

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [PDF_SKILL_INDEX](./pdf/PDF_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [PDF_API_REFERENCE](./pdf/PDF_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, sintaxe de templates, configuração |
| 3 | [FLOW_PDF_CERTIFICATE_GENERATION](./pdf/FLOW_PDF_CERTIFICATE_GENERATION.md) | Fluxo | Gerar certificados a partir de metadata de tokens, preview, busca por query |
| 4 | [FLOW_PDF_PASS_GENERATION](./pdf/FLOW_PDF_PASS_GENERATION.md) | Fluxo | Gerar PDFs de pass/ticket com benefícios e QR codes |

## Documentos do Módulo Whitelist

| # | Documento | Tipo | Descrição |
|---|-----------|------|-----------|
| 1 | [WHITELIST_SKILL_INDEX](./whitelist/WHITELIST_SKILL_INDEX.md) | Índice | Ponto de entrada, tabela de decisão, matriz de endpoints |
| 2 | [WHITELIST_API_REFERENCE](./whitelist/WHITELIST_API_REFERENCE.md) | Referência da API | Todos os endpoints, DTOs, enums, relacionamentos de entidades |
| 3 | [FLOW_WHITELIST_MANAGEMENT](./whitelist/FLOW_WHITELIST_MANAGEMENT.md) | Fluxo | Criar, editar, deletar whitelists; adicionar/remover entries; verificar acesso; on-chain |

---

## Documentos de Referência Cross-Module

Documentos que cobrem temas transversais a todos os módulos. **Leia antes de iniciar qualquer integração.**

| # | Documento | Descrição |
|---|-----------|-----------|
| 1 | [AUTH_FLOW_INTEGRATION.md](../references/AUTH_FLOW_INTEGRATION.md) | Guia de autenticação sem SDK: tokens, headers, refresh, tenantId vs companyId, rate limits |
| 2 | [COMMON_ERROR_REFERENCE.md](../references/COMMON_ERROR_REFERENCE.md) | Estrutura de erro padrão, códigos HTTP, erros por módulo, retry policy |
| 3 | [WEBHOOKS_REFERENCE.md](../references/WEBHOOKS_REFERENCE.md) | Eventos de webhook por módulo, payloads, retry policy, verificação de assinatura |
| 4 | [ARCHITECTURE_OVERVIEW.md](../_setup/ARCHITECTURE_OVERVIEW.md) | Diagrama de serviços, URLs base, mapa de módulos, dependências |
| 5 | [GLOSSARY.md](../_setup/GLOSSARY.md) | Definições de termos do ecossistema (tenant, edition, collection, context, etc.) |

---

## Início Rápido

```
1. Arquitetura      → Leia _setup/ARCHITECTURE_OVERVIEW.md
2. Autenticar       → Leia references/AUTH_FLOW_INTEGRATION.md (sem SDK) ou admin/auth/AUTH_SKILL_INDEX.md (com SDK)
3. Escolher o fluxo → Consulte a tabela "Domínios Disponíveis" acima
4. Implementar      → Siga o SKILL_INDEX do módulo → docs de FLOW → API_REFERENCE
5. Erros            → Consulte references/COMMON_ERROR_REFERENCE.md
```
