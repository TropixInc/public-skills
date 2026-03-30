# Compartilhamento de Pass (Gift Card)

## Overview

O compartilhamento de pass permite enviar um token pass como gift card para outra pessoa. Existem duas formas de gerar share codes: **via checkout** (automático ao comprar produto com `passShareCodeConfig`) e **via API direta** (admin cria manualmente). O destinatário acessa o pass compartilhado via URL pública sem necessidade de autenticação.

---

## Prerequisites

### Para gerar share codes via Checkout:
- Produto configurado com `passShareCodeConfig.enabled: true`
- Campos customizáveis definidos em `passShareCodeConfig.dataFields[]`
- Fluxo de checkout concluído (pedido com status `concluded`)

### Para gerar share codes via API:
- Bearer token com role: `admin` ou `superAdmin`
- `contractAddress` e `chainId` do token
- `editionNumber` do token a compartilhar

### Para acessar share code:
- Nenhuma autenticação necessária (endpoint público)
- Código do share code (ex: `ABC123XYZ`)

---

## Fluxo A — Geração via Checkout

### Step 1: Configuração do Produto (Admin)

O admin configura o produto com share code habilitado. Isso é feito na configuração do produto na Commerce API:

```json
{
  "settings": {
    "passShareCodeConfig": {
      "enabled": true,
      "dataFields": [
        {
          "name": "destinationUserName",
          "type": "text",
          "label": "Nome do destinatário",
          "required": true
        },
        {
          "name": "message",
          "type": "textarea",
          "label": "Mensagem",
          "required": false
        },
        {
          "name": "destinationUserEmail",
          "type": "text",
          "label": "Email do destinatário",
          "required": false
        }
      ]
    }
  }
}
```

**Tipos de campo disponíveis:**

| Tipo | Descrição |
|------|-----------|
| `text` | Campo de texto simples (uma linha) |
| `textarea` | Campo de texto longo (múltiplas linhas) |

---

### Step 2: Preenchimento no Checkout

Durante o checkout, o usuário preenche os campos definidos em `dataFields`. O frontend envia esses dados no campo `passShareCodeData` do Create Order:

```json
{
  "orderProducts": [...],
  "passShareCodeData": {
    "destinationUserName": "Maria Silva",
    "message": "Feliz aniversário! Aproveite o evento!",
    "destinationUserEmail": "maria@example.com"
  },
  "payments": [...]
}
```

> **Nota:** Se o produto tem campos obrigatórios (`required: true`) em `dataFields` e eles não forem enviados, a API retorna erro `missing-pass-share-code-data-field` com os campos faltantes.

---

### Step 3: Geração Automática do Share Code

Após o pagamento ser confirmado e o pedido atingir status `waiting_delivery`:

1. O sistema gera automaticamente os share codes para cada produto do pedido
2. O status do share code no pedido começa como `pending`
3. Quando gerado com sucesso, muda para `generated`

**Status do share code no pedido:**

| Status | Descrição |
|--------|-----------|
| `pending` | Share code sendo gerado |
| `generated` | Share code pronto para compartilhar |
| `failed` | Falha na geração |

**Consulta via pedido:**

```
GET /companies/{companyId}/orders/{orderId}
```

Resposta inclui `passShareCodeInfo`:

```json
{
  "id": "order-uuid",
  "status": "concluded",
  "passShareCodeInfo": {
    "status": "generated",
    "data": {
      "destinationUserName": "Maria Silva",
      "message": "Feliz aniversário!"
    },
    "codes": [
      {
        "productTokenId": "token-uuid",
        "productId": "product-uuid",
        "code": "ABC123XYZ",
        "status": "generated"
      }
    ]
  }
}
```

---

## Fluxo B — Geração via API Direta

### Step 1: Criar Share Code

- **User Action**: Admin cria share code manualmente.

- **API Call**:

```
POST /token-pass-share-codes/tenants/{tenantId}
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "contractAddress": "0x1234567890abcdef1234567890abcdef12345678",
  "chainId": "137",
  "editionNumber": 42,
  "data": {
    "destinationUserName": "Maria Silva",
    "message": "Feliz aniversário!",
    "destinationUserEmail": "maria@example.com"
  },
  "expiresIn": "2024-12-31T23:59:59Z"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `contractAddress` | `string` | Sim | Endereço do contrato do token |
| `chainId` | `string` | Sim | ID da blockchain (default: `"137"` — Polygon) |
| `editionNumber` | `number` | Sim | Número da edição do token |
| `data` | `object` | Não | Dados customizáveis (mensagem, destinatário) |
| `expiresIn` | `datetime` | Não | Data de expiração do share code |

- **Response** (201 Created):

```json
{
  "id": "share-uuid",
  "tenantId": "tenant-uuid",
  "code": "ABC123XYZ",
  "data": {
    "destinationUserName": "Maria Silva",
    "message": "Feliz aniversário!"
  },
  "editionNumber": 42,
  "expiresIn": "2024-12-31T23:59:59Z",
  "tokenPassId": "f47ac10b-...",
  "createdAt": "2024-06-15T10:00:00Z"
}
```

---

## Fluxo C — Acesso ao Share Code (Público)

### Step 1: Acesso via URL

- **Screen**: Página de share code com dados do pass e benefícios.
- **User Action**: Destinatário acessa URL `/pass/share/{code}`.

- **API Call** (público, sem autenticação):

```
GET /token-pass-share-codes/tenants/{tenantId}/{code}
```

- **Response Handling**: Retorna share code com benefícios e metadados do token pass:
  - Imagem do pass (`imageUrl`)
  - Nome e descrição do pass
  - Lista de benefícios associados
  - Dados customizáveis (`data`) — mensagem do remetente, nome do destinatário
  - Data de expiração

---

### Step 2: Visualização do Pass

- **Screen**: Componente `SharedOrder` renderiza:

| Seção | Conteúdo |
|-------|----------|
| Imagem | `imageUrl` do token pass |
| Título | Nome do pass |
| Descrição | Descrição do pass |
| Mensagem | `data.message` (do remetente) |
| De/Para | `data.destinationUserName` |
| Benefícios | Lista de benefícios com nome, tipo, período |

---

### Step 3: QR Code para Resgate

- **Screen**: Seção de QR code para uso dos benefícios.
- **User Action**: Click em "Usar Benefício" → exibe QR code section.
- O QR code contém os dados necessários para o operador verificar e registrar uso.

---

## API Sequence

### Via Checkout:
1. `POST /companies/{companyId}/orders` — Criar pedido com `passShareCodeData`
2. `GET /companies/{companyId}/orders/{orderId}` — Polling até `passShareCodeInfo.status === "generated"`

### Via API Direta:
1. `POST /token-pass-share-codes/tenants/{tenantId}` — Criar share code

### Acesso Público:
1. `GET /token-pass-share-codes/tenants/{tenantId}/{code}` — Consultar share code

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| Share code não encontrado | 404 — verificar código e tenantId |
| Share code expirado | Exibir mensagem de expiração |
| `passShareCodeInfo.status === "failed"` | Falha na geração — contatar admin |
| `passShareCodeInfo.status === "pending"` | Aguardar geração — fazer polling do pedido |
| Campo obrigatório faltando em `passShareCodeData` | Erro 400 `missing-pass-share-code-data-field` — preencher campo |
| Email de destinatário inválido | Validar formato de email antes de enviar |

> **Referência:** Para schemas completos de request/response e formato padrão de erro, consulte [PASS_API_REFERENCE.md](./PASS_API_REFERENCE.md).

---

## React SDK

### Componentes

| Componente | Responsabilidade |
|-----------|-----------------|
| `SharedOrder` | Renderiza pass compartilhado com imagem, mensagem, QR code |
| `PassTemplate` | Template visual do pass |
| `QrCodeSection` | Seção de QR code para resgate |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useGetTokenSharedCode` | `useQuery` | Consulta share code por código |
