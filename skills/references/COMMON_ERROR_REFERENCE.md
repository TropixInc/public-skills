---
id: COMMON_ERROR_REFERENCE
title: "Referência de Erros Comuns"
module: references
version: "1.0.0"
type: reference
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
tags:
  - erros
  - error-handling
  - cross-module
---

# Referência de Erros Comuns

Referência unificada de erros das APIs W3Block. Cobre estrutura padrão, códigos HTTP e erros específicos por módulo.

---

## Estrutura Padrão de Erro

Todas as APIs W3Block retornam erros no mesmo formato:

```json
{
  "statusCode": 400,
  "message": "Descrição do erro",
  "error": "Bad Request"
}
```

Em alguns casos, o campo `message` pode ser um array de strings (erros de validação):

```json
{
  "statusCode": 422,
  "message": [
    "email must be an email",
    "password must be longer than or equal to 8 characters"
  ],
  "error": "Unprocessable Entity"
}
```

---

## Códigos HTTP Globais

| Status | Nome | Causa Comum | O que fazer |
|--------|------|-------------|-------------|
| `400` | Bad Request | Body malformado, JSON inválido | Verificar sintaxe do body da requisição |
| `401` | Unauthorized | Token ausente, expirado ou inválido | Renovar token via `POST /auth/refresh-token` |
| `403` | Forbidden | Token válido, mas sem permissão para o recurso | Verificar `roles` do usuário no JWT |
| `404` | Not Found | Recurso não existe ou foi deletado | Confirmar o ID/slug do recurso |
| `409` | Conflict | Conflito de estado (ex: email em múltiplos tenants) | Ver detalhes no body da resposta |
| `422` | Unprocessable Entity | Falha de validação de campos | Verificar `message[]` para ver quais campos falharam |
| `429` | Too Many Requests | Rate limit excedido | Aguardar e retentar com exponential backoff |
| `500` | Internal Server Error | Erro inesperado no servidor | Reportar para suporte com timestamp e request ID |
| `503` | Service Unavailable | Serviço temporariamente indisponível | Retentar após alguns minutos |

---

## Erros por Módulo

### Auth (PixwayID)

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| `Unauthorized` | 401 | Email/senha incorretos | Verificar credenciais |
| `Conflict` + `tenants[]` | 409 | Email existe em múltiplos tenants, sem `tenantId` | Usar `GET /auth/user-tenants` para resolver tenant, então reenviar com `tenantId` |
| `email must be an email` | 422 | Email inválido | Validar formato de email no frontend |
| Token expirado | 401 | `exp` do JWT ultrapassado | Chamar `POST /auth/refresh-token` |
| Refresh token inválido | 401 | Refresh token expirado ou já usado | Redirecionar usuário para login |

### Commerce / Checkout

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| Produto não encontrado | 404 | Slug ou ID inválido | Verificar slug do produto |
| Edition sem estoque | 409 | `quantity` da edition esgotada | Exibir "Esgotado" ao usuário |
| Pedido não pode ser pago | 409 | Order em status que não permite pagamento | Verificar status da order antes de pagar |
| Gateway não configurado | 422 | Tenant sem gateway para o método de pagamento | Configurar gateway no painel de settings |
| PIX expirado | 409 | QR Code PIX venceu (15-30 min) | Criar nova order |

### Contratos & Tokenização (KEY)

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| Contrato não deployado | 422 | Tentativa de mint em contrato pendente | Aguardar conclusão do deploy (assíncrono) |
| Edition sem supply | 409 | Tentativa de mint além do `maxSupply` | Verificar `currentSupply` vs `maxSupply` |
| Token não encontrado | 404 | `tokenId` ou `editionId` inválido | Confirmar ID do token/edition |
| Permissão insuficiente | 403 | Operação requer role `admin` ou `superAdmin` | Verificar roles do usuário |

### KYC

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| Campo obrigatório ausente | 422 | Input `required: true` não preenchido | Retornar formulário com erro no campo |
| Tipo de arquivo inválido | 422 | Upload com MIME type não aceito | Exibir tipos aceitos ao usuário |
| Submissão já aprovada | 409 | Tentativa de reenvio após aprovação | Bloquear reenvio para usuários já aprovados |
| KYC não configurado | 404 | Tenant sem context de KYC | Configurar context via admin/configurations |

### Loyalty (KEY)

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| Saldo insuficiente | 422 | Tentativa de gasto além do saldo | Verificar saldo antes de debitar |
| Contrato ERC-20 não deployado | 422 | Loyalty sem contrato on-chain | Aguardar deploy do contrato |
| Programa não encontrado | 404 | ID do programa inválido | Verificar ID do loyalty program |

### Pass

| Erro | Status | Causa | Solução |
|------|--------|-------|---------|
| Token não possui pass | 404 | Token sem token pass vinculado | Verificar vínculo token ↔ pass |
| Benefício esgotado | 409 | Uso máximo do benefício atingido | Exibir "Limite de uso atingido" |
| QR Code inválido | 422 | Formato incorreto ou token adulterado | Solicitar novo QR Code |
| Operador não autorizado | 403 | Usuário sem role de operador para o pass | Atribuir role `operator` ao usuário |

---

## Rate Limiting

Ao exceder o limite, a API retorna:

```json
{
  "statusCode": 429,
  "message": "Too Many Requests"
}
```

**Estratégia de retry recomendada (exponential backoff com jitter):**

```typescript
async function fetchWithRetry(fn: () => Promise<any>, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (err.statusCode !== 429 || attempt === maxRetries - 1) throw err;
      const delay = Math.pow(2, attempt) * 1000 + Math.random() * 500;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

Limites por serviço: ver [AUTH_FLOW_INTEGRATION.md — Rate Limiting](./AUTH_FLOW_INTEGRATION.md#6-rate-limiting).

---

## Erros de Validação — Padrão de Resposta

Quando a API retorna 422 com múltiplos erros de validação, itere sobre `message[]` para mapear erros por campo:

```typescript
function mapValidationErrors(apiError: { message: string[] }): Record<string, string> {
  const fieldMap: Record<string, string> = {};
  for (const msg of apiError.message) {
    // Formato típico: "fieldName must be..."
    const field = msg.split(' ')[0];
    fieldMap[field] = msg;
  }
  return fieldMap;
}
```

---

## Diagnóstico de Erros 500

Ao encontrar um erro 500, colete:
1. Timestamp da requisição (ISO 8601)
2. Endpoint e método HTTP
3. Body da requisição (sem dados sensíveis)
4. `x-request-id` do header de resposta (se presente)
5. Ambiente (produção / staging)

Reporte ao suporte W3Block com essas informações.
