# Pagamento via PIX

## Overview

Pagamento via PIX com QR code e copia-e-cola. Suporta dois caminhos: **Asaas** (QR code nativo na resposta da API) e **Pagar.me** (iframe com URL de pagamento). Após gerar o PIX, o frontend faz polling a cada 3 segundos até detectar confirmação do pagamento.

## Prerequisites

- Step 1 (Cart Confirmation) concluído
- `product_cart_info_key` no localStorage com `choosedPayment.paymentMethod` = `pix`
- `choosedPayment.paymentProvider` = `asaas` ou `pagar_me`

---

## Steps

### Step 1: Page Load — Fase A (Formulário CPF)

- **Screen**: Dentro do `PixPaymentView`, Fase A: formulário com campo CPF/CNPJ e botão "Gerar PIX" (com ícone QrCode).
- **User Action**: Nenhuma (automático). O `CheckoutPayment` carrega o cache do localStorage e detecta que o método é PIX.
- **State Changes**: `PixPaymentView` renderiza com `pixData=null`, mostrando o formulário de CPF.

---

### Step 2: Preenchimento do CPF e Submit

- **Screen**: Campo CPF/CNPJ formatado + botão "Gerar pagamento".
- **User Action**: Preenche CPF e clica "Gerar pagamento".
- **Frontend Validation**: `isValidCpfOrCnpj()` — valida formato (`123.456.789-00` ou `12.345.678/0001-00`) e dígitos verificadores.

---

### Step 3: Criação do Pedido

- **API Call**:

```
POST /companies/{companyId}/orders
Authorization: Bearer <token>
Content-Type: application/json
```

**Request Body:**

```json
{
  "orderProducts": [
    {
      "productId": "uuid-produto",
      "expectedPrice": "105.50",
      "quantity": 1,
      "variantIds": []
    }
  ],
  "currencyId": "uuid-brl",
  "paymentMethod": "pix",
  "providerInputs": {
    "cpf_cnpj": "123.456.789-00",
    "transparent_checkout": true
  },
  "destinationWalletAddress": "0x...",
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": null,
  "passShareCodeData": {},
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "pix",
      "paymentProvider": "asaas",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {
        "cpf_cnpj": "123.456.789-00",
        "transparent_checkout": true
      }
    }
  ]
}
```

> **Nota:** `orderProducts` agora inclui `quantity: 1` e `variantIds: []`. Campos `signedGasFee`/`signedGasFees` nao sao enviados. `couponCode` envia `null` quando vazio. `successUrl` usa hostname do tenant via `useTenantBaseUrl()`. Campos top-level `currencyId`, `paymentMethod`, `providerInputs` e `passShareCodeData: {}` sao obrigatorios.

- **Response Handling** (201 Created):

**Caminho 1 — Asaas (QR code nativo):**

A resposta inclui dados PIX diretamente:

```json
{
  "id": "uuid-order",
  "status": "pending",
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "pix",
      "paymentProvider": "asaas",
      "publicData": {
        "pix": {
          "encodedImage": "iVBORw0KGgoAAAANS...",
          "expirationDate": "2024-01-15T11:00:00Z",
          "payload": "00020126580014br.gov.bcb.pix..."
        }
      }
    }
  ]
}
```

O frontend extrai:
- `pixImage` = `payments[brl].publicData.pix.encodedImage` (QR code base64)
- `pixPayload` = `payments[brl].publicData.pix.payload` (código copia-e-cola)
- Inicia polling de status

**Caminho 2 — Pagar.me (iframe):**

A resposta inclui URL de pagamento:

```json
{
  "id": "uuid-order",
  "status": "pending",
  "payments": [
    {
      "publicData": {
        "paymentUrl": "https://api.pagar.me/checkout/pix/abc123"
      }
    }
  ]
}
```

O frontend extrai:
- `iframeLink` = `payments[brl].publicData.paymentUrl`
- Renderiza iframe com a URL

- **State Changes**: `orderResponse` salvo no localStorage. `pixImage`/`pixPayload` setados (Asaas) ou `iframeLink` setado (Pagar.me). Polling iniciado.

---

### Step 4: Exibição do QR Code (Asaas)

- **Screen** (`PixPaymentView`):
  - Mensagem explicativa sobre entrega
  - Countdown de expiração do PIX
  - QR code (imagem base64 ou SVG gerado pelo `QRCodeSVG`)
  - Código PIX copia-e-cola (clicável)
  - Botão de copiar + feedback "Código copiado!"

- **User Action**: Escaneia QR code ou copia/cola o código PIX no app do banco.

### Step 4 (alternativo): Iframe PIX (Pagar.me)

- **Screen** (`IframePaymentView`):
  - Mensagem explicativa
  - Iframe em tela cheia com URL do Pagar.me
  - Countdown de expiração

- **User Action**: Interage com o iframe do Pagar.me para completar o pagamento.
- **Detecção de conclusão (iframe):** Quando o iframe redireciona de volta para o hostname da app, detecta via `onLoad` e navega para completion.

> **Implementação Atual:** A versão Next.js implementa ambos os caminhos: Asaas (QR code nativo via `encodedImage`) e Pagar.me (iframe via `paymentUrl`). O `CheckoutPayment` extrai os dados da response e roteia para `PixPaymentView` (QR code) ou `IframePaymentView` (iframe) conforme o provider.

---

### Step 5: Polling de Status

- **Screen**: Mesma tela do QR code/iframe. Polling acontece em background.

- **API Call** (a cada 3 segundos):

```
GET /companies/{companyId}/orders/{orderId}?fetchNewestStatus=true
Authorization: Bearer <token>
```

- **Response Handling**:

| Status retornado | Ação |
|-----------------|------|
| `pending` | Inicia countdown com `expiresIn`. Continua polling. |
| `concluded`, `delivering`, `waiting_delivery` | Pagamento confirmado! Envia tracking `purchase`. Navega para Step 3 (completion). |
| `failed`, `cancelled` | Para polling. Exibe erro "Erro inesperado no pagamento". |
| `expired` | Para polling. Exibe "Código PIX expirado". |

**Intervalo:** `PIX_STATUS_POLL_INTERVAL_MS = 3000` (3 segundos).

- **State Changes**: Ao detectar pagamento, navega para `/checkout/completed`.

---

## API Sequence

1. `POST /companies/{companyId}/orders` — Criação do pedido com PIX
2. `GET /companies/{companyId}/orders/{orderId}?fetchNewestStatus=true` — Polling a cada 3s (via `useOrderStatusPolling`)

---

## Error Recovery

| Situação | Comportamento |
|----------|--------------|
| CPF inválido | Mensagem inline "CPF ou CNPJ inválido" |
| PIX expirado | Mensagem "Código PIX expirado" + usuário deve voltar e tentar novamente |
| Pagamento falhou | Mensagem "Erro inesperado" |
| Pedido duplicado | Modal de confirmação (mesmo fluxo de cartão) |
| Erro de rede no polling | Polling continua tentando silenciosamente |

---

## Implementação Real (Next.js)

### Componentes

| Componente | Arquivo | Responsabilidade |
|-----------|---------|-----------------|
| `CheckoutPayment` | `checkout-payment.tsx` | Orquestrador compartilhado — detecta PIX vs Credit Card vs Stripe vs Transfer, gerencia criação do pedido, extrai dados PIX da response |
| `PixPaymentView` | `pix-payment-view.tsx` | Duas fases: A) Formulário CPF/CNPJ, B) QR code + countdown + polling |
| `IframePaymentView` | `iframe-payment-view.tsx` | Iframe para Pagar.me PIX (paymentUrl) com detecção de redirect |

### Hooks

| Hook | Tipo | Descrição |
|------|------|-----------|
| `useCreateOrder()` | `useMutation` | Cria pedido (compartilhado com credit card) |
| `useOrderStatusPolling()` | `useQuery` | Polling a cada 3s. Ativado após orderId disponível |
| `useCountdown()` | custom hook | Timer de expiração. Retorna `formattedTime`, `isActive`, `startCountdown` |

### Extração de Dados PIX

O `CheckoutPayment` extrai os dados PIX da response assim:

```typescript
const brlPayment = response.payments.find(
  (p) => p.paymentMethod === PaymentMethodEnum.PIX
);
const pix = brlPayment?.publicData?.pix ?? response.paymentInfo?.pix ?? null;
```

Busca primeiro em `payments[].publicData.pix` (multi-payment), depois fallback para `paymentInfo.pix` (formato legado).

### Fluxo do PixPaymentView

**Fase A (sem pixData):**
1. Exibe formulário com campo CPF/CNPJ + botão "Gerar PIX"
2. Validação local: `cpf.isValid(digits) || cnpj.isValid(digits)`
3. Submit chama: `onSubmitCpf(cpfValue)` → pai cria pedido com `providerInputs: { cpf_cnpj, transparent_checkout: true }`

**Fase B (com pixData):**
1. Exibe QR code como `<img>` com `src="data:image/png;base64,{encodedImage}"`
2. Exibe código PIX (payload) com botão "Copiar" usando `navigator.clipboard.writeText()`
3. Countdown de expiração via `useCountdown()`
4. Spinner "Aguardando pagamento" enquanto polling ativo
5. Se status ∈ [CONCLUDED, DELIVERING, WAITING_DELIVERY] → `router.push('/checkout/completed')`
6. Se status ∈ [FAILED, CANCELLED, ...] → exibe mensagem de erro
7. Se countdown expira → exibe "Código PIX expirado"

### Toast para Clipboard

Usa `sonner` (toast) para feedback visual ao copiar código PIX.
