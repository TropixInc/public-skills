# Checkout API Reference

Referência consolidada de todos os endpoints, schemas, enums e erros do módulo de checkout W3block.

---

## Base URLs

| Serviço | Base URL | Descrição |
|---------|----------|-----------|
| Commerce API | `https://commerce.w3block.io` | Pedidos, preview, pagamentos |
| Identity API (pixwayid) | `https://id.w3block.io` | Autenticação, tokens |
| Key API | `https://key.w3block.io` | Wallets, blockchain |

> **Nota:** Em ambientes de staging/dev, os domínios podem variar (ex: `commerce.stg.w3block.io`).

---

## Autenticação

Todos os endpoints (exceto os marcados como `@IsPublic()`) exigem:

```
Authorization: Bearer <access_token>
```

O `access_token` é obtido via login no pixwayid (Identity API). O token deve pertencer a um usuário com role `User` ou superior.

---

## Endpoints

### 1. Order Preview

Calcula preços, taxas, métodos de pagamento disponíveis e validações de carrinho **sem criar** um pedido.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/companies/{companyId}/orders/preview` |
| **Auth** | Opcional (público, mas retorna mais dados se autenticado) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "orderProducts": [
    {
      "productId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "productTokenId": "11111111-2222-3333-4444-555555555555",
      "variantIds": ["uuid-variante-1"],
      "quantity": "1",
      "selectBestPrice": false,
      "resellerId": null
    }
  ],
  "currencyId": "00000000-0000-0000-0000-000000000001",
  "acceptIncompleteCart": true,
  "couponCode": "DESCONTO10",
  "payments": [
    {
      "currencyId": "00000000-0000-0000-0000-000000000001",
      "amountType": "percentage",
      "amount": "100",
      "paymentMethod": "credit_card"
    }
  ],
  "anchorPaymentsCurrencyId": null,
  "destinationWalletAddress": "0x0000000000000000000000000000000000000000",
  "destinationUserId": null
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `orderProducts` | `OrderProductPreviewDto[]` | Sim | Produtos no carrinho (1-100 itens) |
| `orderProducts[].productId` | `uuid` | Sim | ID do produto |
| `orderProducts[].productTokenId` | `uuid` | Não | ID do token específico (NFT) |
| `orderProducts[].variantIds` | `uuid[]` | Não | IDs de variantes selecionadas |
| `orderProducts[].quantity` | `string` | Não | Quantidade (default: `"1"`) |
| `orderProducts[].selectBestPrice` | `boolean` | Não | Selecionar melhor preço entre revendedores |
| `orderProducts[].resellerId` | `uuid` | Não | ID do revendedor |
| `currencyId` | `uuid` | Não | Moeda para cálculo (deprecated, usar `payments`) |
| `acceptIncompleteCart` | `boolean` | Não | Aceitar carrinho mesmo com produtos com erro (default: `true`) |
| `couponCode` | `string` | Não | Código de cupom de desconto |
| `payments` | `OrderPreviewMultiPaymentSelectionDto[]` | Sim* | Seleção de pagamentos (*auto-preenchido por retro-compatibilidade) |
| `payments[].currencyId` | `uuid` | Sim | Moeda do pagamento |
| `payments[].amountType` | `MultiPaymentSelectionAmountType` | Sim | Tipo de valor: `percentage`, `fixed`, `all_remaining` |
| `payments[].amount` | `string` | Condicional | Valor (obrigatório se `amountType` != `all_remaining`) |
| `payments[].paymentMethod` | `PaymentMethod` | Não | Método de pagamento |
| `anchorPaymentsCurrencyId` | `uuid \| null` | Não | Moeda âncora para cálculos multi-currency |
| `destinationWalletAddress` | `string` | Não | Endereço da wallet de destino |
| `destinationUserId` | `uuid` | Não | ID do usuário de destino |

#### Response Body (200 OK)

```json
{
  "products": [
    {
      "id": "uuid-produto",
      "name": "Nome do Produto",
      "prices": [{ "amount": "100.00", "currency": "BRL" }]
    }
  ],
  "cartPrice": "100.00",
  "clientServiceFee": "5.00",
  "gasFee": { "signature": "...", "gasFee": "0.50" },
  "totalPrice": "105.50",
  "originalCartPrice": "120.00",
  "originalClientServiceFee": "6.00",
  "originalTotalPrice": "126.50",
  "appliedCoupon": "DESCONTO10",
  "productsErrors": [],
  "providersForSelection": [
    {
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "inputs": ["cpf_cnpj", "transparent_checkout", "credit_card_number", "credit_card_expiry", "credit_card_ccv", "credit_card_holder_name", "installments"],
      "availableInstallments": [
        { "amount": 1, "finalPrice": "105.50", "interest": 0, "installmentPrice": "105.50" },
        { "amount": 2, "finalPrice": "107.61", "interest": 1.0, "installmentPrice": "53.81" }
      ],
      "userCreditCards": [
        { "id": "uuid-card", "brand": "visa", "lastNumbers": "4242", "name": "Meu Cartão", "createdAt": "2024-01-01T00:00:00Z" }
      ],
      "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" }
    },
    {
      "paymentMethod": "pix",
      "paymentProvider": "asaas",
      "inputs": ["cpf_cnpj"],
      "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" }
    }
  ],
  "payments": [
    {
      "totalPrice": "105.50",
      "currencyId": "uuid-brl",
      "cartPrice": "100.00",
      "clientServiceFee": "5.00",
      "gasFee": "0.50",
      "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" },
      "amount": "105.50",
      "providersForSelection": []
    }
  ],
  "cashback": {
    "amount": "5.00",
    "cashbackAmount": "5.00",
    "currencyId": "uuid"
  },
  "variants": [],
  "totalAmount": [{ "amount": "105.50", "currencyId": "uuid" }],
  "currencyAmount": [{ "amount": "100.00", "currencyId": "uuid" }],
  "originalCurrencyAmount": [{ "amount": "120.00", "currencyId": "uuid" }],
  "originalTotalAmount": [{ "amount": "126.50", "currencyId": "uuid" }],
  "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" },
  "status": "ok",
  "currencyAllowanceState": null
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `products` | `Product[]` | Produtos validados com preços calculados |
| `cartPrice` | `string` | Preço do carrinho (com desconto, se aplicável) |
| `clientServiceFee` | `string` | Taxa de serviço |
| `gasFee` | `GasFee` | Taxa de gas (blockchain), inclui assinatura |
| `totalPrice` | `string` | Preço total (cart + service fee + gas) |
| `originalCartPrice` | `string` | Preço original sem desconto |
| `originalClientServiceFee` | `string` | Taxa original sem desconto |
| `originalTotalPrice` | `string` | Total original sem desconto |
| `appliedCoupon` | `string` | Cupom aplicado (se válido) |
| `productsErrors` | `ProductErrorInterface[]` | Erros por produto (ex: limite de compra) |
| `providersForSelection` | `PaymentMethodsAvaiable[]` | Métodos de pagamento disponíveis |
| `payments` | `PaymentsResponse[]` | Detalhamento de pagamentos (multi-payment) |
| `cashback` | `object` | Informações de cashback, se aplicável |
| `variants` | `Variants[]` | Variantes disponíveis |
| `totalAmount` | `{amount, currencyId}[]` | Totais por moeda |
| `currency` | `CurrencyResponse` | Moeda principal |
| `currencyAllowanceState` | `string` | Estado do allowance ERC-20 (para crypto) |

> **⚠️ CUIDADO com `installmentPrice`:** A API pode retornar este campo como `number` ou `string`. **Sempre converta para número antes de usar `.toFixed()`**: `Number(inst.installmentPrice).toFixed(2)`. Usar `.toFixed()` diretamente em uma string causa `TypeError: inst.installmentPrice.toFixed is not a function`.

---

### 2. Create Order

Cria um pedido e inicia o processamento do pagamento.

| Campo | Valor |
|-------|-------|
| **Método** | `POST` |
| **Path** | `/companies/{companyId}/orders` |
| **Auth** | Bearer token (role: `User`) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "orderProducts": [
    {
      "productId": "aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "expectedPrice": "105.50",
      "quantity": 1,
      "variantIds": []
    }
  ],
  "destinationWalletAddress": "0x0000000000000000000000000000000000000000",
  "currencyId": "00000000-0000-0000-0000-000000000001",
  "paymentMethod": "credit_card",
  "providerInputs": {
    "cpf_cnpj": "123.456.789-00",
    "transparent_checkout": true,
    "credit_card_number": "4242 4242 4242 4242",
    "credit_card_expiry": "12/28",
    "credit_card_ccv": "123",
    "credit_card_holder_name": "JOAO SILVA",
    "credit_card_holder_cpf_cnpj": "123.456.789-00",
    "credit_card_holder_phone": "(11) 91234-5678",
    "credit_card_holder_postal_code": "12345-678",
    "installments": 1
  },
  "passShareCodeData": {},
  "successUrl": "https://tenant-hostname.com/wallet",
  "couponCode": "DESCONTO10",
  "payments": [
    {
      "currencyId": "00000000-0000-0000-0000-000000000001",
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {
        "cpf_cnpj": "123.456.789-00",
        "transparent_checkout": true,
        "credit_card_number": "4242 4242 4242 4242",
        "credit_card_expiry": "12/28",
        "credit_card_ccv": "123",
        "credit_card_holder_name": "JOAO SILVA",
        "credit_card_holder_cpf_cnpj": "123.456.789-00",
        "credit_card_holder_phone": "(11) 91234-5678",
        "credit_card_holder_postal_code": "12345-678",
        "installments": 1
      }
    }
  ]
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `orderProducts` | `OrderProduct[]` | Sim | Produtos com preço esperado (1-100) |
| `orderProducts[].productId` | `uuid` | Sim | ID do produto |
| `orderProducts[].expectedPrice` | `string` | Sim | Preço esperado (totalPrice do preview) |
| `orderProducts[].quantity` | `number` | Sim | Quantidade (sempre `1`) |
| `orderProducts[].variantIds` | `string[]` | Sim | IDs de variantes selecionadas (vazio `[]` se nenhuma) |
| `destinationWalletAddress` | `string` | Condicional | Wallet de destino (ou `destinationUserId`) |
| `currencyId` | `uuid` | Sim | ID da moeda de pagamento (top-level) |
| `paymentMethod` | `PaymentMethod` | Sim | Metodo de pagamento (top-level, espelha `payments[0].paymentMethod`) |
| `providerInputs` | `Record<string, unknown>` | Nao | Dados do provider (top-level, espelha `payments[0].providerInputs`) |
| `passShareCodeData` | `object` | Sim | Dados para gift card / pass share (enviar `{}` se nao aplicavel) |
| `successUrl` | `string` | Nao | URL de redirect apos sucesso (Stripe). Construida a partir do hostname do tenant: `https://{tenant-host}/wallet` |
| `couponCode` | `string \| null` | Nao | Codigo do cupom. Envia `null` (nao `undefined`) quando vazio |
| `payments` | `OrderMultiPaymentSelectionDto[]` | Sim | Detalhes de pagamento (ver secao Multi-Payment) |
| `acceptSimilarOrderInShortPeriod` | `boolean` | Nao | Pular validacao de pedido duplicado. **Apenas enviado como `true` no retry** apos erro `similar-order-not-accepted`. Nao enviado por padrao |

> **Nota sobre campos removidos:** Os campos `signedGasFee`, `signedGasFees`, `expectedPrices` (array), `selectBestPrice`, `destinationUserId`, `addressId` e `utmParams` **nao sao enviados** pela implementacao atual. O `orderProducts` tambem nao inclui `productTokenId`.

#### Response Body (201 Created)

```json
{
  "id": "uuid-order-id",
  "companyId": "uuid-company",
  "userId": "uuid-user",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z",
  "destinationWalletAddress": "0xd3304...",
  "currencyId": "uuid-brl",
  "currencyAmount": "100.00",
  "status": "pending",
  "paymentProvider": "asaas",
  "paymentMethod": "credit_card",
  "providerTransactionId": "pay_abc123",
  "deliverDate": null,
  "expiresIn": "2024-01-15T10:45:00Z",
  "gasFee": "0.50",
  "clientServiceFee": "5.00",
  "totalAmount": "105.50",
  "deliverId": "uuid-deliver",
  "failReason": null,
  "paymentInfo": {
    "paymentUrl": null,
    "pix": null,
    "clientSecret": null,
    "publicKey": null
  },
  "payments": [
    {
      "currencyId": "uuid-brl",
      "paymentMethod": "credit_card",
      "paymentProvider": "asaas",
      "amountType": "percentage",
      "amount": "100",
      "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" },
      "publicData": null
    }
  ],
  "currency": { "id": "uuid", "code": "BRL", "symbol": "R$" },
  "passShareCodeInfo": null,
  "products": []
}
```

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | `uuid` | ID do pedido criado |
| `status` | `OrderStatus` | Status inicial (geralmente `pending` ou `confirming_payment`) |
| `paymentInfo.paymentUrl` | `string \| null` | URL de pagamento externo (Pagar.me PIX) |
| `paymentInfo.pix` | `PixInterface \| null` | Dados do PIX (QR code, payload) |
| `paymentInfo.pix.encodedImage` | `string` | QR code em base64 |
| `paymentInfo.pix.payload` | `string` | Código PIX copia-e-cola |
| `paymentInfo.pix.expirationDate` | `string` | Data de expiração do PIX |
| `paymentInfo.clientSecret` | `string \| null` | Client secret do Stripe |
| `paymentInfo.publicKey` | `string \| null` | Public key do Stripe |
| `paymentUrl` | `string` | URL de pagamento (Pagar.me) |
| `deliverId` | `string` | ID de entrega (usado na tela de conclusão) |
| `payments[].publicData.pix` | `PixInterface` | PIX data no formato multi-payment |
| `payments[].publicData.paymentUrl` | `string` | URL de pagamento no formato multi-payment |
| `passShareCodeInfo` | `object \| null` | Info de gift card (status, códigos) |

---

### 3. Get Order by ID

Busca o estado atual de um pedido. Usado para polling de status.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/companies/{companyId}/orders/{orderId}` |
| **Auth** | Bearer token (owner do pedido) |
| **Query Params** | `fetchNewestStatus=true` |

#### Response Body (200 OK)

Mesmo schema de `CreateOrderResponse` (ver endpoint Create Order), com `status` atualizado.

---

### 4. Pay Order (Complete Payment)

Completa o pagamento de um pedido existente (usado pelo Stripe, que cria o pedido antes do pagamento).

| Campo | Valor |
|-------|-------|
| **Método** | `PATCH` |
| **Path** | `/companies/{companyId}/orders/{orderId}/pay` |
| **Auth** | Bearer token (owner do pedido) |
| **Content-Type** | `application/json` |

#### Request Body

```json
{
  "payments": [
    {
      "currencyId": "00000000-0000-0000-0000-000000000001",
      "paymentMethod": "credit_card",
      "paymentProvider": "stripe",
      "amountType": "percentage",
      "amount": "100",
      "providerInputs": {}
    }
  ],
  "successUrl": "https://meusite.com/checkout/sucesso"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `payments` | `OrderMultiPaymentSelectionDto[]` | Sim | Pelo menos 1 método de pagamento |
| `successUrl` | `string` | Não | URL de callback pós-pagamento |

#### Response Body (200 OK)

Mesmo schema de `CreateOrderResponse`.

---

### 5. Get Order by Deliver ID (Público)

Busca um pedido pelo `deliverId` (sem autenticação).

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/companies/{companyId}/orders/get-by-deliver-id/{deliverId}` |
| **Auth** | Nenhuma (público) |

#### Response Body (200 OK)

Versão pública do pedido (`PublicOrderEntity`) — campos sensíveis removidos.

---

### 6. List User Orders

Lista todos os pedidos do usuário autenticado.

| Campo | Valor |
|-------|-------|
| **Método** | `GET` |
| **Path** | `/companies/{companyId}/orders` |
| **Auth** | Bearer token (role: `User`) |
| **Query Params** | Paginação padrão (`page`, `limit`) |

#### Response Body (200 OK)

```json
{
  "items": [ /* OrderEntity[] */ ],
  "meta": {
    "totalItems": 50,
    "itemCount": 10,
    "itemsPerPage": 10,
    "totalPages": 5,
    "currentPage": 1
  }
}
```

---

## Schemas de Multi-Payment

### OrderMultiPaymentSelectionDto

Usado nos endpoints **Create Order** e **Pay Order**.

```json
{
  "currencyId": "uuid",
  "amountType": "percentage",
  "amount": "100",
  "paymentMethod": "credit_card",
  "paymentProvider": "asaas",
  "providerInputs": {
    "cpf_cnpj": "123.456.789-00",
    "transparent_checkout": true
  }
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `currencyId` | `uuid` | Sim | ID da moeda para este pagamento |
| `amountType` | `MultiPaymentSelectionAmountType` | Sim | Como calcular o valor |
| `amount` | `string \| null` | Condicional | Valor (não necessário se `amountType = all_remaining`) |
| `paymentMethod` | `PaymentMethod` | Não | Método de pagamento |
| `paymentProvider` | `PaymentProvider` | Não | Provedor de pagamento |
| `providerInputs` | `Record<string, unknown>` | Não | Dados específicos do provedor |

### OrderPreviewMultiPaymentSelectionDto

Usado no endpoint **Order Preview** (sem `paymentProvider` nem `providerInputs`).

```json
{
  "currencyId": "uuid",
  "amountType": "percentage",
  "amount": "100",
  "paymentMethod": "credit_card"
}
```

---

## Enums

### PaymentProvider (Backend)

Identifica o **provedor** que processa o pagamento.

| Valor | Descrição |
|-------|-----------|
| `pagar_me` | Pagar.me — cartão de crédito e PIX (Brasil) |
| `asaas` | Asaas — cartão de crédito e PIX (Brasil) |
| `stripe` | Stripe — cartão internacional |
| `paypal` | PayPal — não implementado no frontend |
| `braza` | Braza — bridge fiat-to-crypto |
| `crypto` | Crypto — pagamento ERC-20 nativo |
| `transfer` | Transfer — transferência bancária manual |
| `free` | Free — pedidos gratuitos (valor zero) |

### PaymentMethod (Backend)

Identifica o **método** de pagamento.

| Valor | Descrição |
|-------|-----------|
| `credit_card` | Cartão de crédito |
| `debit_card` | Cartão de débito |
| `pix` | PIX (transferência instantânea Brasil) |
| `crypto` | Criptomoeda / token ERC-20 |
| `transfer` | Transferência bancária |
| `billet` | Boleto bancário |
| `google_pay` | Google Pay |
| `apple_pay` | Apple Pay |

### PaymentMethod (Frontend SDK enum)

Identifica o **provedor** no frontend (atenção: no SDK frontend, `PaymentMethod` mapeia para providers, não métodos).

| Valor | Mapeamento Backend |
|-------|-------------------|
| `pagar_me` | PaymentProvider.PAGAR_ME |
| `stripe` | PaymentProvider.STRIPE |
| `paypal` | PaymentProvider.PAYPAL |
| `asaas` | PaymentProvider.ASAAS |
| `braza` | PaymentProvider.BRAZA |
| `transfer` | PaymentProvider.TRANSFER |

### OrderStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Pedido criado, aguardando pagamento |
| `confirming_payment` | Pagamento sendo confirmado |
| `waiting_delivery` | Pagamento confirmado, aguardando entrega (blockchain) |
| `delivering` | Entrega em andamento (mint/transfer na blockchain) |
| `concluded` | Pedido concluído com sucesso |
| `failed` | Pedido falhou |
| `cancelling` | Em processo de cancelamento |
| `cancelled` | Pedido cancelado |
| `partially_cancelled` | Parcialmente cancelado |
| `expired` | Pedido expirado (tempo de pagamento esgotado) |

### MultiPaymentSelectionAmountType

| Valor | Descrição | Uso |
|-------|-----------|-----|
| `percentage` | Porcentagem do total | `"100"` = 100% do valor |
| `fixed` | Valor fixo na moeda | `"50.00"` = exatamente R$ 50,00 |
| `all_remaining` | Todo o restante | `amount` é ignorado/null |

### OrderPassShareCodeStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Gift card sendo gerado |
| `generated` | Gift card pronto para compartilhar |
| `failed` | Falha na geração do gift card |

### INPUTS_POSSIBLE

Campos que o frontend pode enviar como `providerInputs`, determinados pelo array `inputs` retornado no `providersForSelection`.

| Valor | Descrição | Tipo |
|-------|-----------|------|
| `cpf_cnpj` | CPF ou CNPJ do comprador | `string` (formatado: `123.456.789-00`) |
| `transparent_checkout` | Checkout transparente (cartão) | `boolean` |
| `credit_card_number` | Número do cartão | `string` (com espaços: `"4242 4242 4242 4242"`) |
| `credit_card_expiry` | Validade do cartão | `string` (`MM/YY`) |
| `credit_card_ccv` | Código de segurança (CVV) | `string` (3-4 dígitos) |
| `credit_card_holder_name` | Nome no cartão | `string` (min 3 chars) |
| `credit_card_holder_cpf_cnpj` | CPF/CNPJ do titular do cartão | `string` (formatado) |
| `credit_card_holder_postal_code` | CEP do titular | `string` (`12345-678`) |
| `credit_card_holder_phone` | Telefone do titular | `string` (`(11) 91234-5678`) |
| `installments` | Número de parcelas | `number` |
| `credit_card_id` | ID do cartão salvo | `string` (uuid) |
| `save_credit_card` | Salvar cartão para uso futuro | `boolean` |
| `save_credit_card_name` | Nome para o cartão salvo | `string` |
| `phone` | Telefone do comprador | `string` |
| `postal_code` | CEP do comprador | `string` |
| `quote_id` | ID da cotação Braza | `string` |

### CommonPaymentInput (Backend)

Inputs comuns reconhecidos pelo backend.

| Valor | Descrição |
|-------|-----------|
| `installments` | Número de parcelas |
| `credit_card_id` | ID do cartão salvo |
| `save_credit_card` | Flag para salvar cartão |
| `save_credit_card_name` | Nome do cartão a salvar |

### PaymentStatus

| Valor | Descrição |
|-------|-----------|
| `pending` | Pagamento pendente |
| `processing` | Em processamento |
| `concluded` | Pagamento concluído |
| `failed` | Pagamento falhou |
| `cancelling` | Em cancelamento |
| `cancelled` | Cancelado |
| `expired` | Expirado |
| `refunded` | Reembolsado |

---

## Validações Frontend

### Cartão de Crédito
- **Número**: Algoritmo de Luhn (via `card-validator`)
- **Validade**: Formato `MM/YY`, não pode estar expirado
- **CVV**: 3 ou 4 dígitos numéricos
- **Nome**: Mínimo 3 caracteres

### CPF/CNPJ
- **CPF**: Formato `123.456.789-00`, validação de dígitos verificadores
- **CNPJ**: Formato `12.345.678/0001-00`, validação de dígitos verificadores
- Biblioteca: `cpf-cnpj-validator`

### CEP
- Formato: `12345-678`

### Telefone
- Formato: `(11) 91234-5678` ou `(11) 1234-5678`

---

## Erros Comuns

| HTTP Status | Código/Situação | Descrição | Ação de Recuperação |
|-------------|----------------|-----------|---------------------|
| `400` | Validation error | Campos inválidos no body | Verificar campos e re-enviar |
| `400` | Similar order in short period | Pedido duplicado detectado | Enviar com `acceptSimilarOrderInShortPeriod: true` |
| `400` | Product limit exceeded | Limite de compra por usuário excedido | Exibir `productsErrors[].error.limit` ao usuário |
| `400` | Invalid expected price | Preço mudou entre preview e criação | Refazer preview e usar preço atualizado |
| `401` | Unauthorized | Token ausente ou inválido | Re-autenticar o usuário |
| `403` | Forbidden | Usuário sem permissão para esta operação | Verificar roles do usuário |
| `404` | Order not found | Pedido não encontrado | Verificar orderId e companyId |
| `404` | Product not found | Produto não existe | Verificar productId |
| `422` | Payment processing failed | Falha no processamento do pagamento | Tentar outro método de pagamento |
| `422` | Coupon not valid | Cupom inválido ou expirado | Remover cupom e tentar sem desconto |
| `409` | Coupon already redeemed | Cupom já utilizado | Exibir mensagem ao usuário |

---

## Constantes Importantes

| Constante | Valor | Descrição |
|-----------|-------|-----------|
| `PIX_STATUS_POLL_INTERVAL_MS` | `3000` | Intervalo de polling de status PIX (3s) |
| `STRIPE_CACHE_VALIDITY_MINUTES` | `5` | Tempo de validade do cache Stripe (5 min) |
| `ORDER_PREVIEW_POLL_INTERVAL_MS` | `20000` | Intervalo de re-preview (20s) |
| `FREE_ORDER_DEBOUNCE_MS` | `4000` | Debounce para pedidos gratuitos/transfer (4s) |
| `BRL_CURRENCY_IDS` | Array de UUIDs | IDs de moedas BRL conhecidas (obtidos via API de currencies) |

---

## Fluxo de providerInputs por Provider

Cada `paymentProvider` espera `providerInputs` específicos. O array `inputs` no `providersForSelection` indica quais campos enviar.

| Provider | paymentMethod | providerInputs típicos |
|----------|---------------|----------------------|
| `asaas` / `pagar_me` | `credit_card` | `cpf_cnpj`, `transparent_checkout`, `credit_card_number` (com espaços), `credit_card_expiry`, `credit_card_ccv`, `credit_card_holder_name`, `credit_card_holder_cpf_cnpj`, `credit_card_holder_phone`, `credit_card_holder_postal_code`, `installments` |
| `asaas` / `pagar_me` | `pix` | `cpf_cnpj`, `transparent_checkout` |
| `stripe` | `credit_card` | `{}` (vazio — Stripe Elements lida com os dados) |
| `braza` | `crypto` | `cpf_cnpj`, `quote_id` |
| `crypto` | `crypto` | `{}` (vazio — wallet do usuário) |
| `transfer` | `transfer` | `{}` (vazio — processamento manual) |
| `free` | (qualquer) | `{}` (vazio — valor zero) |

> **Nota:** Na implementação atual, o formulário de cartão de crédito **sempre** envia os campos `credit_card_holder_cpf_cnpj`, `credit_card_holder_phone` e `credit_card_holder_postal_code`, independente do provider (Asaas ou Pagar.me). O campo `credit_card_number` é enviado **com espaços** (ex: `"4242 4242 4242 4242"`), não limpo.

---

## Gotchas / Armadilhas Comuns

Problemas reais encontrados durante implementação. Verifique esses pontos para evitar erros em runtime:

| # | Problema | Onde ocorre | Solução |
|---|----------|-------------|---------|
| 1 | `installmentPrice.toFixed()` falha com TypeError | CreditCardForm ao renderizar parcelas | `installmentPrice` pode vir como string da API. Sempre usar `Number(inst.installmentPrice).toFixed(2)` |
| 2 | Tipos da API não correspondem ao TypeScript | Interfaces de `Installment`, campos numéricos | Declarar tipos flexíveis: `installmentPrice: number \| string` |
| 3 | `providersForSelection` pode estar vazio | Order Preview response sem métodos disponíveis | Sempre verificar `providers?.length > 0` antes de auto-selecionar |
| 4 | Preços retornados como string, não number | Todos os campos de preço (`cartPrice`, `totalPrice`, etc.) | Todos os preços são `string`. Usar `parseFloat()` para cálculos |
