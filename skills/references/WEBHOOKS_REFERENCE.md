---
id: WEBHOOKS_REFERENCE
title: "Referência de Webhooks"
module: references
version: "1.0.0"
type: reference
status: implemented
last_updated: "2026-04-06"
authors:
  - fernandodevpascoal
tags:
  - webhooks
  - eventos
  - integração
---

# Referência de Webhooks

Documentação dos eventos de webhook da plataforma W3Block. Webhooks são notificações HTTP enviadas ao endpoint configurado no tenant quando eventos específicos ocorrem.

---

## Configuração

Configure o endpoint de webhook no painel de administração do tenant ou via API de settings. Recomendações:

- **HTTPS obrigatório** — não aceita URLs HTTP em produção
- **Timeout:** a plataforma aguarda até **10 segundos** pela resposta
- **Resposta esperada:** HTTP 2xx. Qualquer outro status é tratado como falha
- **Idempotência:** implemente verificação por `eventId` para evitar processamento duplicado em retries

---

## Estrutura do Payload

Todo webhook tem o mesmo envelope:

```json
{
  "eventId": "uuid-único-do-evento",
  "eventType": "order.concluded",
  "tenantId": "uuid-do-tenant",
  "timestamp": "2026-04-06T12:00:00.000Z",
  "data": {
    // payload específico do evento
  }
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `eventId` | UUID | Identificador único do evento. Use para deduplicação |
| `eventType` | string | Tipo do evento (ver lista abaixo) |
| `tenantId` | UUID | ID do tenant que originou o evento |
| `timestamp` | ISO 8601 | Data/hora do evento |
| `data` | object | Payload específico do tipo de evento |

---

## Eventos por Módulo

### Commerce — Pedidos

#### `order.concluded`

Disparado quando um pedido é concluído com sucesso (pagamento confirmado e tokens entregues).

```json
{
  "eventId": "uuid",
  "eventType": "order.concluded",
  "tenantId": "uuid-tenant",
  "timestamp": "2026-04-06T12:00:00.000Z",
  "data": {
    "orderId": "uuid-order",
    "userId": "uuid-user",
    "products": [
      {
        "productId": "uuid-product",
        "editionId": "uuid-edition",
        "tokenId": "string-token-id",
        "quantity": 1
      }
    ],
    "totalAmount": "150.00",
    "currency": "BRL",
    "paymentMethod": "pix"
  }
}
```

#### `order.failed`

Disparado quando um pedido falha (pagamento não processado, timeout ou erro).

```json
{
  "eventType": "order.failed",
  "data": {
    "orderId": "uuid-order",
    "userId": "uuid-user",
    "reason": "payment_timeout",
    "failedAt": "2026-04-06T12:30:00.000Z"
  }
}
```

#### `order.cancelled`

Disparado quando um pedido é cancelado (pelo usuário ou pelo admin).

```json
{
  "eventType": "order.cancelled",
  "data": {
    "orderId": "uuid-order",
    "cancelledBy": "user | admin",
    "reason": "string (opcional)"
  }
}
```

---

### KYC — Verificação de Identidade

#### `kyc.submitted`

Disparado quando um usuário envia documentos KYC para revisão.

```json
{
  "eventType": "kyc.submitted",
  "data": {
    "userId": "uuid-user",
    "submissionId": "uuid-submission",
    "contextId": "uuid-context"
  }
}
```

#### `kyc.approved`

Disparado quando uma submissão KYC é aprovada pelo operador.

```json
{
  "eventType": "kyc.approved",
  "data": {
    "userId": "uuid-user",
    "submissionId": "uuid-submission",
    "approvedBy": "uuid-operator",
    "approvedAt": "2026-04-06T12:00:00.000Z"
  }
}
```

#### `kyc.denied`

Disparado quando uma submissão KYC é negada.

```json
{
  "eventType": "kyc.denied",
  "data": {
    "userId": "uuid-user",
    "submissionId": "uuid-submission",
    "reason": "Documento ilegível",
    "deniedBy": "uuid-operator"
  }
}
```

---

### Loyalty — Fidelidade

#### `loyalty.reward.distributed`

Disparado quando recompensas são distribuídas por uma regra de loyalty (cashback, multiplicador, etc.).

```json
{
  "eventType": "loyalty.reward.distributed",
  "data": {
    "programId": "uuid-program",
    "userId": "uuid-user",
    "amount": "50.00",
    "ruleType": "cashback | add | multiply | split",
    "triggeredBy": "order | manual | staking",
    "orderId": "uuid-order (se triggeredBy = order)"
  }
}
```

#### `loyalty.balance.updated`

Disparado quando o saldo de tokens de loyalty de um usuário muda (mint, burn, transfer).

```json
{
  "eventType": "loyalty.balance.updated",
  "data": {
    "userId": "uuid-user",
    "contractId": "uuid-contract",
    "operation": "mint | burn | transfer",
    "amount": "100.00",
    "newBalance": "350.00"
  }
}
```

---

### Pass — Token Passes

#### `pass.benefit.used`

Disparado quando um usuário usa um benefício do pass (validado pelo operador).

```json
{
  "eventType": "pass.benefit.used",
  "data": {
    "tokenPassId": "uuid-pass",
    "benefitId": "uuid-benefit",
    "userId": "uuid-user",
    "tokenId": "string-token-id",
    "operatorId": "uuid-operator",
    "usedAt": "2026-04-06T12:00:00.000Z",
    "remainingUses": 2
  }
}
```

---

### Contratos (Blockchain)

#### `contract.deployed`

Disparado quando um smart contract é deployado com sucesso na blockchain.

```json
{
  "eventType": "contract.deployed",
  "data": {
    "contractId": "uuid-contract",
    "contractType": "ERC721 | ERC1155 | ERC20",
    "contractAddress": "0x...",
    "chainId": 137,
    "deployedAt": "2026-04-06T12:00:00.000Z"
  }
}
```

#### `token.minted`

Disparado quando um ou mais tokens são mintados.

```json
{
  "eventType": "token.minted",
  "data": {
    "contractId": "uuid-contract",
    "editionId": "uuid-edition",
    "tokenIds": ["1", "2", "3"],
    "recipientWallet": "0x...",
    "recipientUserId": "uuid-user"
  }
}
```

---

## Política de Retry

| Tentativa | Aguarda antes de retentar |
|-----------|--------------------------|
| 1ª falha | 30 segundos |
| 2ª falha | 5 minutos |
| 3ª falha | 30 minutos |
| 4ª falha | 2 horas |
| 5ª falha | 24 horas |

Após 5 falhas consecutivas, o evento é descartado e registrado no log de erros do tenant.

---

## Verificação de Autenticidade

Webhooks incluem um header de assinatura para validar que a requisição veio da W3Block:

```
X-W3Block-Signature: sha256={hmac-sha256-hex}
```

Valide a assinatura no seu endpoint:

```typescript
import crypto from 'crypto';

function verifyWebhookSignature(
  rawBody: string,
  signature: string,
  secret: string
): boolean {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return `sha256=${expected}` === signature;
}
```

O `secret` é configurado junto com a URL do webhook no painel do tenant.

---

## Armadilhas Comuns

| Armadilha | Solução |
|-----------|---------|
| Processar o mesmo evento múltiplas vezes | Salvar `eventId` processados e ignorar duplicatas |
| Responder 500 ao receber evento desconhecido | Responder sempre 200 para eventos desconhecidos (ignorar silenciosamente) |
| Processar de forma síncrona (lentidão > 10s) | Responder 200 imediatamente, processar em background |
| Não validar assinatura | Sempre verificar `X-W3Block-Signature` antes de processar |
