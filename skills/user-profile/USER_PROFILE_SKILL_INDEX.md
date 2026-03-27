# User Profile Skill Index (KEY API)

Indice do dominio de perfil de usuario que utiliza a KEY API (`https://api.w3block.io`). Este skill cobre exclusivamente as operacoes de **saque (withdrawals)** — que sao os unicos endpoints de user-profile na KEY API.

> **Swagger (documentacao interativa):** Acesse `https://api.w3block.io/docs` para testar endpoints e ver schemas atualizados.

> **Nota:** Endpoints de perfil, KYC, wallets e notificacoes usam a Identity API (`https://id.w3block.io`) e nao estao cobertos aqui.

---

## Documentos

| # | Documento | Descricao | Status | Quando usar |
|---|-----------|-----------|--------|-------------|
| 1 | [USER_PROFILE_API_REFERENCE.md](./USER_PROFILE_API_REFERENCE.md) | Endpoints KEY API de withdrawals, schemas, enums, erros | Referencia | Consulta de API de saques |
| 2 | [FLOW_PROFILE_WITHDRAWALS.md](./FLOW_PROFILE_WITHDRAWALS.md) | Saques: solicitacao, fluxo admin (escrow/concluir/recusar) | Implementado | Gerenciamento de saques |

---

## Guia Rapido

### Para implementar fluxo de saques (API-first):

```
1. POST /{companyId}/withdraws                        -> Solicitar saque
2. GET  /{companyId}/withdraws/{id}                   -> Consultar status do saque
3. PATCH /{companyId}/withdraws/admin/{id}/escrow      -> (Admin) Reter recursos
4. PATCH /{companyId}/withdraws/admin/{id}/conclude    -> (Admin) Concluir com comprovante
5. PATCH /{companyId}/withdraws/admin/{id}/refuse      -> (Admin) Recusar com motivo
```

### Armadilhas Comuns

| # | Problema | Solucao |
|---|----------|---------|
| 1 | Withdraw usa KEY API, nao ID API | `useRequestWithdraw` e acoes admin usam `W3blockAPI.KEY` (`https://api.w3block.io`) |
| 2 | `amount` de withdraw vem como string | Sempre usar `parseFloat(amount).toFixed(2)` para exibicao |
| 3 | Metodos de saque (withdraw-accounts) ficam na ID API | Listar/criar contas de saque usa `W3blockAPI.ID`, nao KEY |

---

## Tabela de Decisao

| Se o usuario quer... | Use este documento | Hooks principais |
|----------------------|-------------------|------------------|
| Solicitar saque | [Withdrawals](./FLOW_PROFILE_WITHDRAWALS.md) | `useRequestWithdraw` |
| Ver status de saque | [Withdrawals](./FLOW_PROFILE_WITHDRAWALS.md) | `useGetSpecificWithdraw` |
| Aprovar/recusar saque (admin) | [Withdrawals](./FLOW_PROFILE_WITHDRAWALS.md) | `useConcludeWithdraw`, `useRefuseWithdraw` |
| Reter recursos (admin) | [Withdrawals](./FLOW_PROFILE_WITHDRAWALS.md) | `useEscrowWithdraw` |

---

## Matriz: Endpoints x Documentos

| Endpoint | Metodo | API | Withdrawals |
|----------|--------|-----|:-:|
| `POST /{companyId}/withdraws` | Request Withdraw | KEY | **X** |
| `GET /{companyId}/withdraws/{id}` | Get Withdraw | KEY | **X** |
| `GET /{companyId}/withdraws/admin/{id}` | Get Withdraw (Admin) | KEY | **X** |
| `PATCH /{companyId}/withdraws/admin/{id}/conclude` | Conclude | KEY | **X** |
| `PATCH /{companyId}/withdraws/admin/{id}/escrow` | Escrow | KEY | **X** |
| `PATCH /{companyId}/withdraws/admin/{id}/refuse` | Refuse | KEY | **X** |

---

## Armadilhas Comuns (leia antes de implementar!)

| # | Problema | Solucao |
|---|----------|---------|
| 1 | Withdraw usa KEY API, metodos de saque usam ID API | `POST /{companyId}/withdraws` vai na KEY API (`api.w3block.io`), mas `GET /users/{tenantId}/withdraw-accounts/{userId}` vai na ID API (`id.w3block.io`) |
| 2 | `amount` vem como string | Sempre usar `parseFloat(amount).toFixed(2)` para exibicao |
| 3 | Escrow e obrigatorio antes de conclude | O fluxo admin e: request → escrow → conclude. Nao pode pular escrow |
| 4 | Refuse precisa de `reason` no body | PATCH refuse retorna 400 se nao enviar o campo `reason` |
| 5 | Status do withdraw nao e editavel | O status muda automaticamente conforme as acoes admin (escrow/conclude/refuse) |

---

## Glossario Rapido

| Termo | Descricao |
|-------|-----------|
| `companyId` / `tenantId` | UUID do tenant (empresa) na plataforma W3block |
| `W3blockAPI.KEY` | Key API — `https://api.w3block.io` — saques, operacoes blockchain |
| `withdrawAccountId` | UUID da conta de saque cadastrada pelo usuario (gerenciada na ID API) |
| `vault` | Wallet custodiada pela plataforma (vs `metamask` = wallet externa) |
| `WithdrawStatus` | Status do saque: pending, escrowing_resources, ready_to_transfer_funds, concluded, failed, refused |
