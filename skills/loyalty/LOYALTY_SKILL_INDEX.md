# Loyalty Skill Index

Indice master de toda a documentacao de loyalty/recompensas do W3block. Use este documento como ponto de entrada para implementar qualquer parte do sistema de fidelidade.

> **Swagger (documentacao interativa):** Acesse `https://api.w3block.io/docs` para testar endpoints e ver schemas atualizados.

---

## Documentos

| # | Documento | Descricao | Status | Quando usar |
|---|-----------|-----------|--------|-------------|
| 1 | [LOYALTY_API_REFERENCE.md](./LOYALTY_API_REFERENCE.md) | Endpoints, schemas JSON, enums, DTOs, constantes | Referencia | Consulta de API em qualquer momento |
| 2 | [FLOW_LOYALTY_BALANCE.md](./FLOW_LOYALTY_BALANCE.md) | Consulta de saldo do usuario e listagem de loyalties da empresa | Implementado | Exibir saldo e programas de fidelidade |
| 3 | [FLOW_LOYALTY_PAYMENT.md](./FLOW_LOYALTY_PAYMENT.md) | Preview e criacao de pagamentos com pontos de fidelidade | Implementado | Pagar com pontos de loyalty |
| 4 | [FLOW_LOYALTY_STAKING.md](./FLOW_LOYALTY_STAKING.md) | Resumo de staking e resgate de pontos travados | Implementado | Staking e resgate de pontos |
| 5 | [FLOW_LOYALTY_DEFERRED.md](./FLOW_LOYALTY_DEFERRED.md) | Recompensas diferidas por usuario, listagem geral e exportacao XLSX | Implementado | Gerenciar cashback agendado/diferido |
| 6 | [FLOW_LOYALTY_ADMIN_REPORTS.md](./FLOW_LOYALTY_ADMIN_REPORTS.md) | Relatorios administrativos e por operador de historico ERC20 | Implementado | Relatorios admin e operador |

---

## Guia Rapido

### Para consultar saldo e programas de fidelidade:

```
1. Leia: FLOW_LOYALTY_BALANCE.md              -> Buscar saldo do usuario e listar loyalties
2. Consulte: LOYALTY_API_REFERENCE.md         -> Para detalhes de schemas/enums
```

### Para realizar pagamento com pontos:

```
1. Leia: FLOW_LOYALTY_BALANCE.md              -> Entender saldo disponivel
2. Leia: FLOW_LOYALTY_PAYMENT.md              -> Preview + criar pagamento
3. Consulte: LOYALTY_API_REFERENCE.md         -> DTOs e campos obrigatorios
```

### Para gerenciar staking:

```
1. Leia: FLOW_LOYALTY_STAKING.md              -> Consultar resumo e resgatar
2. Consulte: LOYALTY_API_REFERENCE.md         -> SummaryDto e parametros
```

### Para gerenciar recompensas diferidas:

```
1. Leia: FLOW_LOYALTY_DEFERRED.md             -> Por usuario, geral, ou exportar XLSX
2. Consulte: LOYALTY_API_REFERENCE.md         -> Filtros de status e paginacao
```

### Para relatorios administrativos:

```
1. Leia: FLOW_LOYALTY_ADMIN_REPORTS.md        -> Admin geral ou por operador
2. Consulte: LOYALTY_API_REFERENCE.md         -> Controle de acesso por role
```

### Implementacao minima (API-first):

```
1. GET  /{companyId}/loyalties/admin?limit=49                       -> Listar programas de fidelidade
2. GET  /{companyId}/loyalties/users/balance/{userId}               -> Consultar saldo
3. PATCH /{companyId}/loyalties/rewards/payment/preview             -> Pre-visualizar pagamento
4. PATCH /{companyId}/loyalties/rewards/payment                     -> Executar pagamento
```

---

## Tabela de Decisao: Por Role

| Se o usuario tem a role... | Pode acessar... | Documento |
|----------------------------|-----------------|-----------|
| `admin` ou `superAdmin` | Listar loyalties da empresa | [Balance](./FLOW_LOYALTY_BALANCE.md) |
| `admin` ou `superAdmin` | Relatorios admin (todas transacoes) | [Admin Reports](./FLOW_LOYALTY_ADMIN_REPORTS.md) |
| `loyaltyOperator` | Listar loyalties da empresa | [Balance](./FLOW_LOYALTY_BALANCE.md) |
| `loyaltyOperator` | Relatorios por operador (suas transacoes) | [Admin Reports](./FLOW_LOYALTY_ADMIN_REPORTS.md) |
| Qualquer autenticado | Consultar proprio saldo | [Balance](./FLOW_LOYALTY_BALANCE.md) |
| Qualquer autenticado | Fazer pagamento com pontos | [Payment](./FLOW_LOYALTY_PAYMENT.md) |
| Qualquer autenticado | Consultar staking | [Staking](./FLOW_LOYALTY_STAKING.md) |
| Qualquer autenticado | Ver recompensas diferidas | [Deferred](./FLOW_LOYALTY_DEFERRED.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Metodo | Balance | Payment | Staking | Deferred | Admin Reports |
|----------|--------|:-------:|:-------:|:-------:|:--------:|:-------------:|
| `GET /{companyId}/loyalties/admin` | Listar Loyalties | **X** | | | | |
| `GET /{companyId}/loyalties/users/balance/{userId}` | Saldo | **X** | | | | |
| `PATCH /{companyId}/loyalties/rewards/payment/preview` | Preview | | **X** | | | |
| `PATCH /{companyId}/loyalties/rewards/payment` | Pagamento | | **X** | | | |
| `GET /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}` | Staking Summary | | | **X** | | |
| `PATCH /{companyId}/loyalties/users/{userId}/staking/{loyaltyId}/redeem` | Redeem | | | **X** | | |
| `GET /{companyId}/loyalties/users/deferred/{userId}` | Deferred por User | | | | **X** | |
| `GET /{companyId}/loyalties/users/deferred` | Deferred Geral | | | | **X** | |
| `GET /{companyId}/loyalties/users/deferred/xlsx` | Exportar XLSX | | | | **X** | |
| `GET /{companyId}/erc20-tokens/{loyaltyId}/history/{userId}` | Historico User | | | | | **X** |
| `GET /{companyId}/erc20-tokens/{loyaltyId}/history` | Historico Admin | | | | | **X** |
| `GET /{companyId}/erc20-tokens/{loyaltyId}/history/operator/{operatorId}` | Historico Operador | | | | | **X** |

---

## Fluxo Visual

```
+------------------------------------------------------------------------+
|                       LOYALTY W3BLOCK                                  |
|                                                                        |
|  +----------------+    +----------------+    +------------------+      |
|  |   Consulta     |    |   Pagamento    |    |    Staking       |      |
|  |   Saldo        |--->|   com Pontos   |    |    & Resgate     |      |
|  +----------------+    +----------------+    +------------------+      |
|        |                      |                      |                 |
|        v                      v                      v                 |
|  +----------+          +-----------+           +----------+            |
|  | Balance  |          | Preview   |           | Summary  |            |
|  | API      |          | + Create  |           | + Redeem |            |
|  | Loyalty  |          | Payment   |           | Staking  |            |
|  | List     |          |           |           |          |            |
|  +----------+          +-----------+           +----------+            |
|                                                                        |
|  +----------------+    +------------------+                            |
|  |   Deferred     |    |   Admin          |                            |
|  |   Rewards      |    |   Reports        |                            |
|  +----------------+    +------------------+                            |
|        |                      |                                        |
|        v                      v                                        |
|  +-----------+          +-----------+                                  |
|  | By User   |          | All Admin |                                  |
|  | All       |          | By Oper.  |                                  |
|  | XLSX      |          |           |                                  |
|  +-----------+          +-----------+                                  |
+------------------------------------------------------------------------+
```

---

## Armadilhas Comuns (leia antes de implementar!)

| # | Problema | Solucao |
|---|----------|---------|
| 1 | `contractId` nao e o mesmo que `id` do loyalty | Use `contractId` (campo do objeto loyalty) para endpoints de transfer e balance, nao o `id` |
| 2 | `useGetUserBalance` usa `mutateAsync`, nao e query | O hook de saldo usa `useMutation` internamente — chame `mutateAsync()` e trate o resultado |
| 3 | Limite fixo `limit=49` em `useGetCompanyLoyalties` | O hook busca no maximo 49 loyalties — se o tenant tiver mais, implemente paginacao manual |
| 4 | `amount` vem como string nos endpoints de balance | Sempre usar `parseFloat(amount)` para calculos e `.toFixed(2)` para exibicao |
| 5 | Staking summary pode retornar vazio | Se o usuario nao tiver staking ativo, o endpoint retorna array vazio — trate o caso |
| 6 | Payment preview e obrigatorio antes de criar pagamento | Sempre chame o preview primeiro para validar e obter o valor correto antes de POST no create |

---

## Glossario Rapido

| Termo | Descricao |
|-------|-----------|
| `companyId` | UUID do tenant (empresa) na plataforma W3block |
| `loyaltyId` | UUID do programa de fidelidade |
| `contractId` | ID do contrato ERC-20 associado ao programa de fidelidade |
| `userId` | UUID do usuario na plataforma |
| `operatorId` | UUID do operador de loyalty (role `loyaltyOperator`) |
| `userCode` | Codigo temporario do usuario para identificacao em pagamentos presenciais |
| `LoyaltyInterface` | Interface TypeScript que descreve um programa de fidelidade |
| `SummaryDto` | Interface TypeScript que descreve um registro de staking |
| `CreatePaymentDTO` | DTO para criar pagamento com pontos de fidelidade |
| `PaymentPreviewDTO` | DTO para pre-visualizar pagamento antes de confirmar |
| `ErcTokenHistoryInterfaceResponse` | Resposta paginada de historico de transacoes ERC-20 |
| `pointPrecision` | Precisao dos pontos: `integer` ou `decimal` |
| `compensateRule` | Regra de compensacao com periodo (ex: cashback diferido) |
| `withdrawBlockedOnly` | Filtro para recompensas com saque bloqueado (agendadas) |
| `W3blockAPI.KEY` | Base URL da Key API: `https://api.w3block.io` |
| `W3blockAPI.ID` | Base URL da ID API (usada para buscar usuario por codigo) |
