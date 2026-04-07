---
id: USER_PASS_SKILL_INDEX
title: "Índice de Skills de Pass (Usuário)"
module: user/pass
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
---

# Índice de Skills de Pass — Perspectiva do Usuário

Fluxos de uso de token passes e resgate de benefícios do ponto de vista do usuário final.

**Serviço:** Pass | **Swagger:** https://pass.w3block.io/docs/

---

## Documentos

| # | Documento | Descrição | Quando usar |
|---|-----------|-----------|-------------|
| 1 | [FLOW_PASS_SELF_USE.md](./FLOW_PASS_SELF_USE.md) | Visualizar passes, benefícios disponíveis e gerar QR Code para uso | Tela "Meus Benefícios" do usuário |
| 2 | [FLOW_PASS_BENEFIT_REDEMPTION.md](./FLOW_PASS_BENEFIT_REDEMPTION.md) | Resgate de benefício — validação pelo operador via QR Code | Fluxo de validação presencial/digital de benefício |

---

## Referências Relacionadas

| Documento | Descrição |
|-----------|-----------|
| [PASS_API_REFERENCE.md](../../references/pass/PASS_API_REFERENCE.md) | Endpoints completos da API de Pass |
| [PASS_FRONTEND_API_REFERENCE.md](../../references/pass/PASS_FRONTEND_API_REFERENCE.md) | Referência orientada ao frontend (34 endpoints, schemas, paginação) |
| [FLOW_PASS_OVERVIEW.md](../../references/pass/FLOW_PASS_OVERVIEW.md) | Visão geral do módulo Pass — personas, rotas, componentes |
| [admin/pass/PASS_SKILL_INDEX.md](../../admin/pass/PASS_SKILL_INDEX.md) | Skills de configuração de passes e benefícios (perspectiva do admin) |

---

## Guia Rápido

```
Ver meus passes          → FLOW_PASS_SELF_USE.md § Listagem
Ver benefícios de um pass → FLOW_PASS_SELF_USE.md § Benefícios
Gerar QR Code para uso   → FLOW_PASS_SELF_USE.md § QR Code
Validar benefício (operador) → FLOW_PASS_BENEFIT_REDEMPTION.md
```

## Formato do QR Code

```
{tokenPassBenefitId}:{userId}:{tokenId}
```

O operador escaneia esse QR Code para validar o uso do benefício via endpoint de verificação.

---

## Armadilhas Comuns

| Armadilha | Solução |
|-----------|---------|
| Exibir benefícios sem verificar `remainingUses` | Verificar `remainingUses > 0` antes de permitir geração do QR Code |
| QR Code sem validade controlada | QR Codes não expiram automaticamente — implementar TTL no frontend se necessário |
| Usuário sem token não vê passes | Verificar se o usuário possui o token vinculado ao pass antes de exibir |
