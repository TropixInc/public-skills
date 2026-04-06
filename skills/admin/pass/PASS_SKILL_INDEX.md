---
id: PASS_SKILL_INDEX
title: "Índice de Skills de Pass e Benefícios"
module: offpix/pass
module_version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
---

# Índice de Skills de Pass e Benefícios

Documentação para o módulo de Pass e Benefícios do W3Block. Cobre token passes baseados em NFT, benefícios digitais e físicos, gerenciamento de operadores, check-in via QR code, rastreamento de uso com regras temporais, códigos de compartilhamento e requisitos de elegibilidade baseados em metadados.

**Serviço:** Pass | **Swagger:** https://pass.w3block.io/docs

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|----------|---------|-----------|--------|-------------|
| 1 | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, enums, relacionamentos de entidades, códigos de erro | Implementado | Referência da API a qualquer momento |
| 2 | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) | 1.0.0 | Criar pass, adicionar benefícios (digitais/físicos), config de check-in, operadores, códigos de compartilhamento | Implementado | Construir funcionalidades de gerenciamento de pass |
| 3 | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) | 1.0.0 | Visualizar benefícios, geração de QR, 4 métodos de resgate, rastreamento de uso, verificação | Implementado | Construir fluxos de resgate de benefícios |

---

## Guia Rápido

### Para admin (gerenciamento de pass):

```
1. Leia: FLOW_PASS_LIFECYCLE.md            → Criar passes e configurar benefícios
2. Consulte: PASS_API_REFERENCE.md          → Para schemas de check-in, regras de uso, requisitos
```

### Para usuário/operador (resgate):

```
1. Leia: FLOW_PASS_BENEFIT_REDEMPTION.md   → Todos os fluxos de resgate e cadeia de validação
2. Consulte: PASS_API_REFERENCE.md          → Para códigos de erro e casos especiais
```

### Implementação mínima API-first:

```
1. POST /token-passes/tenants/{id}                              → Criar pass
2. POST /token-pass-benefits/tenants/{id}                       → Adicionar benefício
3. POST /token-pass-benefit-operators/tenants/{id}              → Atribuir operador
4. GET  /token-pass-benefits/tenants/{id}/{bid}/{ed}/secret     → Gerar QR
5. POST /token-pass-benefits/tenants/{id}/register-use-by-qrcode → Escanear e registrar
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Coleção deve estar publicada | Criação de pass requer uma coleção publicada com endereço de contrato |
| 2 | QR dinâmico expira em 45s | Regenere antes de escanear -- não faça cache |
| 3 | `allowSelfUse` tem padrão `false` | Habilite para benefícios digitais, mantenha desabilitado para físicos (operador obrigatório) |
| 4 | Uso não reinicia | Configure `usageRule` (dia/semana/mês/timestamp) para reinícios periódicos |
| 5 | Incompatibilidade de timezone no check-in | Defina `timezoneOrUtcOffset` para corresponder à localização física |
| 6 | Requisitos não correspondem | Comparação de metadados de token é case-insensitive para igualdade de strings |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Criar um novo token pass | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Adicionar benefícios digitais (links) | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Adicionar benefícios físicos (locais) | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Configurar agendas de check-in | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) + [Referência da API (schema de check-in)](./PASS_API_REFERENCE.md) |
| Configurar limites de uso com reinícios | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) + [Referência da API (regras de uso)](./PASS_API_REFERENCE.md) |
| Atribuir operadores para check-in | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Criar códigos de compartilhamento públicos | [FLOW_PASS_LIFECYCLE](./FLOW_PASS_LIFECYCLE.md) |
| Gerar QR codes para usuários | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Registrar uso de benefício (operador) | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Auto-resgate pelo usuário | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Visualizar/exportar relatórios de uso | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Entender a cadeia de validação | [FLOW_PASS_BENEFIT_REDEMPTION](./FLOW_PASS_BENEFIT_REDEMPTION.md) |
| Requisitos de metadados de token | [PASS_API_REFERENCE](./PASS_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Ciclo de Vida | Resgate |
|----------|:-------:|:---------:|:----------:|
| POST /token-passes | X | X | |
| GET /token-passes | X | X | |
| GET /token-passes/:id | X | X | |
| GET /token-passes/:id/token-editions/:ed/benefits | X | | X |
| GET /token-passes/users/:userId | X | X | |
| PATCH /token-passes/:id | X | X | |
| DELETE /token-passes/:id | X | X | |
| POST /token-pass-benefits | X | X | |
| GET /token-pass-benefits | X | X | |
| GET /token-pass-benefits/:id | X | | X |
| GET /token-pass-benefits/:id/:ed/secret | X | | X |
| GET /token-pass-benefits/:id/verify | X | | X |
| PATCH /token-pass-benefits/:id | X | X | |
| DELETE /token-pass-benefits/:id | X | X | |
| POST /token-pass-benefits/:id/register-use | X | | X |
| POST /token-pass-benefits/:id/register-use-by-user | X | | X |
| POST /token-pass-benefits/register-use-by-qrcode | X | | X |
| POST /token-pass-benefits/:id/use | X | | X |
| GET /token-pass-benefits/usages | X | | X |
| GET /token-pass-benefits/usages/xls | X | | X |
| POST /token-pass-benefit-addresses | X | X | |
| GET /token-pass-benefit-addresses | X | X | |
| POST /token-pass-benefit-operators | X | X | |
| GET /token-pass-benefit-operators | X | X | |
| GET /token-pass-benefit-operators/benefits/:id | X | X | |
| DELETE /token-pass-benefit-operators/:id | X | X | |
| POST /token-pass-share-codes | X | X | |
| GET /token-pass-share-codes/:code | X | | X |
