---
id: USER_KYC_SKILL_INDEX
title: "Índice de Skills de KYC (Usuário)"
module: user/kyc
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
---

# Índice de Skills de KYC — Perspectiva do Usuário

Fluxos de submissão de documentos KYC do ponto de vista do usuário final.

**Serviço:** PixwayID | **Swagger:** https://pixwayid.w3block.io/docs/

---

## Documentos

| # | Documento | Descrição | Quando usar |
|---|-----------|-----------|-------------|
| 1 | [FLOW_KYC_SUBMISSION.md](./FLOW_KYC_SUBMISSION.md) | Fluxo de submissão KYC via SDK (buscar campos, enviar documentos, multi-etapas) | Implementar formulário de KYC para usuário |
| 2 | [FLOW_CONTACTS_KYC_SUBMISSION.md](./FLOW_CONTACTS_KYC_SUBMISSION.md) | Submissão KYC via API de Contacts (perspectiva alternativa) | Integração direta com API de Contacts |

> **Nota:** `FLOW_KYC_SUBMISSION.md` e `FLOW_CONTACTS_KYC_SUBMISSION.md` cobrem o mesmo fluxo com perspectivas diferentes. Para a maioria dos casos, use `FLOW_KYC_SUBMISSION.md`.

---

## Referências Relacionadas

| Documento | Descrição |
|-----------|-----------|
| [KYC_API_REFERENCE.md](../../references/kyc/KYC_API_REFERENCE.md) | Endpoints, validações por tipo de input, erros |
| [CONTACTS_API_REFERENCE.md](../../references/contacts/CONTACTS_API_REFERENCE.md) | Endpoints de usuários e KYC via serviço de Contacts |
| [admin/kyc/](../../admin/kyc/) | Skills de aprovação/revisão de KYC (perspectiva do admin) |
| [admin/configurations/](../../admin/configurations/) | Configuração de contexts e inputs de KYC |

---

## Guia Rápido

```
1. Buscar campos do formulário   → GET /contexts/{slug}/inputs
2. Exibir formulário ao usuário  → FLOW_KYC_SUBMISSION.md § Renderização
3. Enviar documentos             → POST /kyc/submissions
4. Tratar reenvio (se negado)    → FLOW_KYC_SUBMISSION.md § Reenvio
5. Verificar status              → GET /kyc/submissions/{userId}
```

---

## Armadilhas Comuns

| Armadilha | Solução |
|-----------|---------|
| Não tratar status `requiredReview` | Exibir mensagem pedindo correção dos documentos específicos |
| Upload sem validar tipo de arquivo | Verificar `allowedMimeTypes` do input antes de aceitar o arquivo |
| Tentar reenviar após aprovação | Verificar status antes de exibir formulário — bloquear se `approved` |
