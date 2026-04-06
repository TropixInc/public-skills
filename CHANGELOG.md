# Changelog

Todas as mudanças relevantes deste projeto serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) e este projeto adere ao [Semantic Versioning](https://semver.org/lang/pt-BR/).

---

## [2.0.0] - 2026-04-06

### Reestruturação completa de pastas

Reorganização da pasta `skills/` por audiência, substituindo a estrutura flat + `offpix/` aninhada:

- **`user/`** — Documentação voltada ao usuário final (auth, checkout, tokens, pass, kyc)
- **`admin/`** — Documentação voltada a admins e integradores (13 módulos: auth, commerce, configurations, contacts, contracts, kyc, loyalty, pass, pdf, settings, tokenization, user-profile, whitelist)
- **`references/`** — API references e documentos compartilhados (user + admin)
- **`_setup/`** — Bootstrap de projeto e padrões de API
- **`_meta/`** — Gap analysis e índice legado offpix

### Mudanças principais

- Eliminada a pasta `offpix/` — módulos movidos para `admin/` e `references/`
- Eliminadas pastas raiz duplicadas (`auth/`, `checkout/`, `pass/`, `tokens/`, `user-profile/`, `setup/`) — movidas para `user/` ou `admin/`
- Resolvida duplicação de domínio (ex: `pass/` e `offpix/pass/` agora são `user/pass/` e `admin/pass/`)
- `OFFPIX_DOCUMENTATION_SPEC.md` renomeado para `_setup/DOCUMENTATION_SPEC.md`
- `OFFPIX_API_GAP_ANALYSIS.md` movido para `_meta/API_GAP_ANALYSIS.md`

### Tradução para PT-BR

- 66 arquivos traduzidos de inglês para português brasileiro
- Termos técnicos mantidos em inglês (endpoints, HTTP methods, JSON keys, enum values, nomes de produto)
- Todos os 95 arquivos agora estão em PT-BR

### Novos módulos (via offpix → admin)

- **commerce v1.0.0** — Produtos, pedidos, pagamentos, promoções, splits, gateways
- **configurations v1.0.0** — Contextos, formulários dinâmicos, signup activation
- **contacts v1.0.0** — Gestão de usuários, submissão e aprovação KYC
- **contracts v1.0.0** — Contratos NFT/ERC20, collections, editions, operações de token
- **kyc v1.0.0** — Configuração, submissão e aprovação de documentos KYC
- **loyalty v1.0.0** — Programas de fidelidade, rewards, cashback, pagamentos
- **pdf v1.0.0** — Geração de certificados e passes em PDF
- **settings v1.0.0** — Faturamento, planos, configuração de tenant
- **tokenization v1.0.0** — Collections, editions, mint/burn/transfer, bulk import XLSX
- **whitelist v1.0.0** — Listas de acesso, grupos de usuários

**Autor:** Fernando (`fernandodevpascoal@gmail.com`)

---

## [1.1.0] - 2026-03-30

### pass v1.0.0 — Novo módulo

- Adicionado módulo completo de gestão de passes (9 skills)
  - `FLOW_PASS_OVERVIEW.md` — Visão geral do fluxo de passes
  - `FLOW_PASS_MANAGEMENT.md` — Gerenciamento de passes (criação, edição, exclusão)
  - `FLOW_PASS_BENEFITS_VIEW.md` — Visualização de benefícios do pass
  - `FLOW_PASS_BENEFIT_TRACKING.md` — Rastreamento de uso de benefícios
  - `FLOW_PASS_BENEFIT_VERIFICATION.md` — Verificação e validação de benefícios
  - `FLOW_PASS_SELF_USE.md` — Fluxo de uso próprio do pass
  - `FLOW_PASS_SHARE.md` — Compartilhamento de passes entre usuários
  - `PASS_API_REFERENCE.md` — Referência completa da API de passes
  - `PASS_SKILL_INDEX.md` — Índice do módulo

### setup v1.0.0 — Novo módulo

- Adicionado módulo de configuração e bootstrap de projetos (3 skills)
  - `SETUP_PROJECT_BOOTSTRAP.md` — Bootstrap inicial de projetos
  - `SETUP_API_PATTERNS.md` — Padrões e convenções de API
  - `SETUP_SKILL_INDEX.md` — Índice do módulo

### tokens v1.0.0 — Novo módulo

- Adicionado módulo completo de tokens e wallets (7 skills)
  - `FLOW_TOKENS_BALANCE.md` — Consulta de saldo de tokens
  - `FLOW_TOKENS_METADATA.md` — Metadados de tokens
  - `FLOW_TOKENS_NFT_WALLET.md` — Wallet de NFTs
  - `FLOW_TOKENS_TRANSFER.md` — Transferência de tokens entre wallets
  - `FLOW_TOKENS_WALLET_CONNECT.md` — Conexão de wallets externas
  - `TOKENS_API_REFERENCE.md` — Referência completa da API de tokens
  - `TOKENS_SKILL_INDEX.md` — Índice do módulo

### user-profile v1.0.0 — Novo módulo

- Adicionado módulo de perfil de usuário (3 skills)
  - `FLOW_PROFILE_WITHDRAWALS.md` — Fluxo de saques/withdrawals
  - `USER_PROFILE_API_REFERENCE.md` — Referência da API de perfil
  - `USER_PROFILE_SKILL_INDEX.md` — Índice do módulo

**Autor:** Fernando (`fernandodevpascoal@gmail.com`)

---

## [1.0.0] - 2026-03-24

### auth v1.0.0 — Novo módulo

- Adicionado módulo de autenticação (1 skill)
  - `FLOW_SIGNUP.md` — Fluxo completo de cadastro/signup de usuários

### checkout v1.0.0 — Novo módulo

- Adicionado módulo completo de checkout e pagamentos (11 skills)
  - `FLOW_CHECKOUT_OVERVIEW.md` — Visão geral do fluxo de checkout
  - `FLOW_CHECKOUT_CART_CONFIRMATION.md` — Confirmação de carrinho
  - `FLOW_CHECKOUT_COMPLETION.md` — Finalização do checkout
  - `FLOW_CHECKOUT_PAYMENT_CREDIT_CARD.md` — Pagamento via cartão de crédito
  - `FLOW_CHECKOUT_PAYMENT_CRYPTO.md` — Pagamento via criptomoedas
  - `FLOW_CHECKOUT_PAYMENT_PIX.md` — Pagamento via PIX
  - `FLOW_CHECKOUT_PAYMENT_STRIPE.md` — Pagamento via Stripe
  - `FLOW_CHECKOUT_PAYMENT_TRANSFER.md` — Pagamento via transferência bancária
  - `FLOW_MY_ORDERS.md` — Visualização de pedidos do usuário
  - `CHECKOUT_API_REFERENCE.md` — Referência completa da API de checkout
  - `CHECKOUT_SKILL_INDEX.md` — Índice do módulo

**Autor:** rafaelmhp (`rafael.mh.perez@gmail.com`)
