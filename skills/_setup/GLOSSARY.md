---
id: GLOSSARY
title: "Glossário de Termos do Ecossistema W3Block"
module: setup
version: "1.0.0"
type: reference
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
tags:
  - glossário
  - referência
  - setup
---

# Glossário de Termos do Ecossistema W3Block

Definições dos termos específicos usados nas APIs e documentação da plataforma W3Block.

---

## Termos de Plataforma

| Termo | Definição |
|-------|-----------|
| **Tenant** | Organização/empresa cadastrada na plataforma W3Block. Cada tenant tem configurações, usuários e dados isolados. Identificado por um UUID (`tenantId` ou `companyId`). |
| **CompanyId** | UUID do tenant. Usado como path param nas APIs de negócio (KEY, Commerce, Pass). Sinônimo de `tenantId` no contexto de autenticação. |
| **TenantId** | UUID do tenant. Usado nos endpoints de autenticação (PixwayID) e no payload do JWT. Mesmo valor que `companyId`. |
| **PixwayID** | Serviço de identidade da W3Block. Responsável por autenticação, usuários, KYC, configurações de tenant e billing. URL: `pixwayid.w3block.io` |
| **KEY (Registry)** | Serviço de blockchain/registro da W3Block. Responsável por contratos, tokens, loyalty, tokenização e whitelist. URL: `api.w3block.io` |
| **Passwordless** | Modo de autenticação do tenant onde senha é opcional. O usuário pode fazer login via código OTP ou magic token sem precisar de senha. |

---

## Termos de Autenticação

| Termo | Definição |
|-------|-----------|
| **Access Token** | JWT RS256 de curta duração emitido no login. Enviado no header `Authorization: Bearer {token}` em requisições autenticadas. Contém `sub`, `roles`, `tenantId`, `exp`. |
| **Refresh Token** | JWT de longa duração usado para obter novos access tokens sem re-autenticar. Enviado no body de `POST /auth/refresh-token`. |
| **Magic Token** | UUID de uso único gerado por admin/integração para login direto de um usuário, sem senha. Tem TTL configurável (padrão: 30 min). |
| **Código OTP** | Código numérico de 6 dígitos enviado por email para login sem senha. Válido por tempo limitado. |
| **UserRole** | Papel do usuário dentro de um tenant: `superAdmin`, `admin`, `operator`, `user`, `loyaltyOperator`, `commerceOrderReceiver`, `kycApprover`, `keyErc20Receiver`. |
| **TenantRole** | Papel de acesso ao tenant: `application`, `administrator`, `integration`, `contextApplication`. Usado em JWTs de tenant (API key login). |

---

## Termos de Tokenização e Blockchain

| Termo | Definição |
|-------|-----------|
| **Collection** | Agrupamento de tokens (NFTs) dentro de um contrato. Uma collection tem metadados compartilhados e pode conter múltiplas editions. Equivale a um "produto digital". |
| **Edition** | Variante específica de um token dentro de uma collection. Cada edition tem quantidade (supply), preço e metadados próprios. Análogo a "SKU" no e-commerce tradicional. |
| **Token** | Ativo digital individual na blockchain. Pode ser NFT (ERC-721, único) ou fungível (ERC-20, divisível). Cada token tem um `tokenId` on-chain. |
| **Contrato (Contract)** | Smart contract na blockchain. Na W3Block, pode ser ERC-721 (NFTs), ERC-1155 (multi-token) ou ERC-20 (fungível/loyalty). O deploy é assíncrono e gerenciado pela plataforma. |
| **Mint** | Processo de criar/emitir um novo token on-chain. Requer contrato deployado e edition configurada. |
| **Burn** | Processo de destruir um token on-chain, removendo-o permanentemente da circulação. |
| **RFID** | Identificador de rádio-frequência vinculado a um token físico. Usado para conectar ativos digitais a objetos do mundo real. |

---

## Termos de Commerce

| Termo | Definição |
|-------|-----------|
| **Product** | Item à venda no marketplace. Vinculado a uma ou mais token editions. Tem slug, preços, imagens e configurações de pagamento. |
| **Order** | Pedido de compra. Criado a partir de um product, passa por estados: `pending` → `confirming` → `delivering` → `concluded` (ou `failed`/`cancelled`). |
| **Order Preview** | Simulação de pedido antes da criação real. Retorna preço final com promoções aplicadas, sem criar a order. |
| **Split** | Divisão de receita de vendas entre múltiplos destinatários (ex: criador recebe 70%, plataforma 30%). Configurado por product. |
| **Gateway** | Provedor de pagamento configurado no tenant (ex: Stripe, PagSeguro, PIX). Cada tenant pode ter múltiplos gateways ativos. |
| **Promoção (Promotion)** | Desconto aplicável a products. Tipos: percentual, valor fixo, frete grátis. Pode ter condições (whitelist, período, quantidade). |

---

## Termos de Loyalty

| Termo | Definição |
|-------|-----------|
| **Programa de Fidelidade** | Configuração de regras de recompensa vinculada a um contrato ERC-20. Define como pontos são ganhos e gastos. |
| **Reward Rule** | Regra que define quando e quanto de pontos são distribuídos. Tipos: `add` (fixo), `multiply` (multiplicador), `split` (divisão), `cashback` (% de compras com comissões multinível). |
| **Staking** | Bloqueio de tokens por um período em troca de recompensas. O usuário deposita tokens que ficam travados até o prazo configurado. |
| **Cashback Multinível** | Sistema de comissões em cascata onde referenciadores ganham percentuais das compras de seus indicados, em múltiplos níveis. |

---

## Termos de Pass e Benefícios

| Termo | Definição |
|-------|-----------|
| **Token Pass** | Configuração que vincula benefícios a posse de tokens específicos. O usuário que possui o token tem acesso aos benefícios do pass. |
| **Benefit (Benefício)** | Vantagem vinculada a um token pass. Pode ter uso limitado (ex: 3 usos/mês), período de validade e regras de verificação. |
| **Operator (Operador)** | Usuário com papel de operador que pode verificar e validar o uso de benefícios (ex: porteiro de evento, atendente de loja). |
| **Share Code** | Código único gerado para compartilhar/presentear um token pass. Pode ser usado para acesso público sem autenticação. |
| **QR Code** | Código gerado para verificação de benefícios. Formato: `tokenPassBenefitId:userId:tokenId`. Escaneado pelo operador para validar uso. |

---

## Termos de KYC

| Termo | Definição |
|-------|-----------|
| **KYC (Know Your Customer)** | Processo de verificação de identidade do usuário. Envolve envio de documentos e aprovação por operador/admin. |
| **Context** | Formulário configurável do tenant. Define quais campos (inputs) são exibidos em telas de KYC, signup ou perfil. Cada context tem um `slug` único. |
| **Input** | Campo individual dentro de um context. Tipos: `text`, `email`, `phone`, `cpf`, `file` (upload), `select`, `multiselect`, `url`, `number`, etc. |
| **KYC Status** | Estados de uma submissão KYC: `pending` (aguardando revisão), `approved` (aprovado), `denied` (negado), `requiredReview` (necessita correção). |

---

## Termos de Configuração

| Termo | Definição |
|-------|-----------|
| **Billing Plan** | Plano de assinatura do tenant na W3Block. Define limites de uso, funcionalidades disponíveis e preço mensal. |
| **Feature Gate** | Controle de funcionalidades baseado no plano de billing. Verifica se o tenant tem acesso a uma funcionalidade específica antes de exibi-la. |
| **Whitelist** | Lista de controle de acesso. Endereços/usuários na whitelist têm permissão para determinadas ações (ex: compra de tokens exclusivos, acesso antecipado). |

---

## Termos Técnicos Gerais

| Termo | Definição |
|-------|-----------|
| **Slug** | Identificador legível (URL-friendly) de um recurso. Ex: `meu-produto-123`. Usado em vez de UUID em URLs públicas. |
| **UUID** | Identificador único universal no formato `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. Usado como ID primário de todas as entidades. |
| **Webhook** | Notificação HTTP enviada pela plataforma quando um evento ocorre (ex: pedido concluído, KYC aprovado). Configurado pelo tenant. |
| **Interceptor** | Middleware do Axios que adiciona automaticamente o header `Authorization` e trata tokens expirados. Conceito do SDK React. |
