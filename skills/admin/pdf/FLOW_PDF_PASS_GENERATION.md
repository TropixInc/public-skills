---
id: FLOW_PDF_PASS_GENERATION
title: "PDF - Geração de Passes/Ingressos (Benefícios, QR Codes, Templates Personalizados)"
module: offpix/pdf
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - pdf
  - pass
  - tickets
  - benefits
  - qrcode
  - template
depends_on:
  - PDF_API_REFERENCE
---

# Geração de PDF de Passes/Ingressos

## Visão Geral

Este fluxo cobre a geração de PDFs de passes (ingressos) para detentores de token-pass. Um PDF de passe contém o nome do token, número da edição, atributos opcionais de variante (ex.: nível, assento, cor) e uma lista de benefícios -- cada um renderizado com um QR code para resgate em pontos de check-in físicos ou digitais. O serviço suporta tanto um template padrão embutido quanto templates HTML totalmente personalizados via Handlebars.

> **Serviço interno:** O serviço de PDF não requer autenticação. É um serviço interno chamado pelo backend de Pass para gerar PDFs para detentores de token-pass. Desenvolvedores construindo aplicações personalizadas também podem chamar este endpoint diretamente de seus serviços backend.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Dados do token-pass | Nome do token, número da edição e atributos de variante | Do módulo Pass & Benefits ou da lógica da sua aplicação |
| Dados dos benefícios | Array de benefícios com nomes, descrições, regras, datas e dados de QR code | Do módulo Pass & Benefits ou da lógica da sua aplicação |
| Dados de QR code | URL de resgate ou string de dados para cada benefício | Gerado pelo backend de Pass ou pela sua aplicação |

---

## Fluxo: Gerar PDF de Passe

### Passo 1: Preparar os Dados da Requisição

Construa o `PassDto` com todos os campos obrigatórios. No mínimo, você precisa de `tokenName`, `tokenEditionNumber`, um array de `tokenVariants` (pode ser vazio) e um array de `benefits` com pelo menos uma entrada.

**Requisição Mínima:**

```json
{
  "tokenName": "Summer Festival Pass",
  "tokenEditionNumber": "1",
  "tokenVariants": [],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Full-day access to the festival grounds.",
      "rules": "Valid for single entry. Present QR code at the gate.",
      "startDate": "2026-07-20T10:00:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/benefit-a1b2c3d4"
    }
  ]
}
```

**Requisição Completa (com todos os campos opcionais):**

```json
{
  "tokenName": "Summer Festival VIP Pass",
  "tokenEditionNumber": "42 of 100",
  "tokenVariants": [
    {
      "label": "Tier",
      "value": "VIP"
    },
    {
      "label": "Color",
      "value": "Gold"
    },
    {
      "label": "Zone",
      "value": "Front Stage"
    }
  ],
  "benefits": [
    {
      "name": "VIP Lounge Access",
      "description": "Exclusive access to the VIP lounge with complimentary drinks and food.",
      "rules": "Valid for the full event duration. QR code must be scanned at lounge entrance. No guests allowed.",
      "startDate": "2026-07-20T10:00:00.000Z",
      "endDate": "2026-07-20T23:59:00.000Z",
      "startCheckinTime": "2026-07-20T09:30:00.000Z",
      "endCheckinTime": "2026-07-20T22:00:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/benefit-vip-lounge-uuid"
    },
    {
      "name": "Meet & Greet",
      "description": "30-minute backstage session with the headlining artist.",
      "rules": "One-time use only. Arrive 15 minutes before start time. No recording allowed.",
      "startDate": "2026-07-20T18:00:00.000Z",
      "endDate": "2026-07-20T18:30:00.000Z",
      "startCheckinTime": "2026-07-20T17:45:00.000Z",
      "endCheckinTime": "2026-07-20T18:00:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/benefit-meet-greet-uuid"
    },
    {
      "name": "Afterparty Entry",
      "description": "Entry to the exclusive afterparty at the rooftop venue.",
      "rules": "Valid for single entry after 22:00. Must be 18+ with valid ID.",
      "startDate": "2026-07-20T22:00:00.000Z",
      "endDate": "2026-07-21T04:00:00.000Z",
      "startCheckinTime": "2026-07-20T21:30:00.000Z",
      "endCheckinTime": "2026-07-20T23:30:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/benefit-afterparty-uuid"
    }
  ]
}
```

### Passo 2: Enviar a Requisição

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| POST | `/pass/generateTickets` | Nenhuma |

**Exemplo de Requisição:**

```bash
curl -X POST "https://pdf.w3block.io/pass/generateTickets" \
  -H "Content-Type: application/json" \
  -d '{
    "tokenName": "Summer Festival Pass",
    "tokenEditionNumber": "1",
    "tokenVariants": [{"label": "Tier", "value": "VIP"}],
    "benefits": [{
      "name": "General Admission",
      "description": "Full-day access to the festival.",
      "rules": "Single entry. Present QR at gate.",
      "startDate": "2026-07-20T10:00:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/abc123"
    }]
  }' \
  -o pass.pdf
```

### Passo 3: Tratar a Resposta

**Sucesso (200):**

A resposta é um stream binário de PDF com os seguintes headers:

```
Content-Type: application/pdf
Content-Disposition: attachment; filename=pass.pdf
```

O arquivo é sempre retornado como attachment (download), diferente dos endpoints de certificado que suportam pré-visualização inline.

**Respostas de Erro:**

| Status | Causa | Ação |
|--------|-------|------|
| 400 | Campos obrigatórios ausentes (`tokenName`, `tokenEditionNumber`, `benefits`) ou DTO malformado | Valide o corpo da requisição contra o schema do PassDto. Certifique-se de que o array `benefits` não esteja vazio |
| 500 | Falha na geração do PDF (erro de template, crash do Puppeteer) | Verifique os logs do serviço. Verifique a sintaxe do template se estiver usando um template personalizado |
| 408 | Geração excedeu o timeout de 30 segundos | Reduza o número de benefícios. Simplifique templates personalizados. Certifique-se de que os dados de QR code não sejam excessivamente longos |

---

## Funcionalidades do Template Embutido

Quando nenhum `pdfTemplate` é fornecido, o serviço usa seu template embutido localizado em `/src/templates/template.html`. Este template fornece um layout completo de passe com as seguintes funcionalidades:

### Estrutura do Layout

1. **Seção de Cabeçalho**
   - Exibe `tokenName` de forma proeminente
   - Mostra `tokenEditionNumber` abaixo do nome

2. **Seção de Variantes**
   - Itera pelo array `tokenVariants`
   - Cada variante é exibida como um par rótulo/valor (ex.: "Tier: VIP", "Color: Gold")
   - Omitida inteiramente se `tokenVariants` for um array vazio

3. **Seção de Benefícios**
   - Itera por cada benefício no array `benefits`
   - Cada card de benefício exibe:
     - **QR Code**: Gerado no lado do cliente via QRCode.js a partir da string de dados `qrCode`
     - **Nome**: Título do benefício
     - **Descrição**: Texto completo da descrição
     - **Regras**: Regras de uso e restrições
     - **Datas**: Data de início e data de término (quando fornecidas), formatadas para legibilidade
     - **Horários de Check-in**: Janelas de início e término de check-in (quando fornecidas)

4. **Controle de Quebra de Página**
   - O template usa CSS `page-break-inside: avoid` nos cards de benefício
   - Isso evita que um único card de benefício seja dividido entre duas páginas
   - Cada benefício ocupa um bloco independente

### Geração de QR Code

O template embutido inclui uma seção JavaScript que gera QR codes no momento da renderização:

- Usa a biblioteca QRCode.js (incluída no template)
- A string `qrCode` de cada benefício é codificada em uma imagem de QR code
- QR codes são renderizados como elementos canvas e convertidos em imagens inline
- O tamanho do QR code é otimizado para leitura por scanners

---

## Fluxo: Gerar Passe com Template Personalizado

Substitua o template embutido fornecendo um campo `pdfTemplate` no corpo da requisição.

### Passo 1: Criar o Template Personalizado

Escreva um documento HTML completo usando sintaxe Handlebars. As seguintes variáveis estão disponíveis:

| Variável | Tipo | Descrição |
|----------|------|-----------|
| `tokenName` | string | Nome de exibição do token-pass |
| `tokenEditionNumber` | string | Número ou rótulo da edição |
| `tokenVariants` | array | Array de objetos `{ label, value }` |
| `benefits` | array | Array de objetos de benefício (veja BenefitsDto) |

**Exemplo de Template Personalizado:**

```html
<html>
<head>
  <style>
    body {
      font-family: 'Helvetica Neue', Arial, sans-serif;
      margin: 0;
      padding: 30px;
      color: #1a1a1a;
    }
    .header {
      text-align: center;
      border-bottom: 2px solid #c9a54e;
      padding-bottom: 20px;
      margin-bottom: 30px;
    }
    .header h1 {
      font-size: 32px;
      margin: 0;
    }
    .header .edition {
      font-size: 16px;
      color: #888;
      margin-top: 5px;
    }
    .variants {
      display: flex;
      justify-content: center;
      gap: 20px;
      margin-bottom: 30px;
    }
    .variant {
      background: #f5f5f5;
      padding: 8px 16px;
      border-radius: 4px;
    }
    .variant .label {
      font-weight: bold;
      color: #666;
    }
    .benefit {
      border: 1px solid #e0e0e0;
      border-radius: 8px;
      padding: 20px;
      margin-bottom: 20px;
      page-break-inside: avoid;
    }
    .benefit h2 {
      margin-top: 0;
      color: #c9a54e;
    }
    .benefit .rules {
      background: #fff8e1;
      padding: 10px;
      border-radius: 4px;
      font-size: 13px;
    }
    .qr-container {
      text-align: center;
      margin: 15px 0;
    }
  </style>
  <script src="https://cdn.jsdelivr.net/npm/qrcodejs@1.0.0/qrcode.min.js"></script>
</head>
<body>
  <div class="header">
    <h1>{{tokenName}}</h1>
    <div class="edition">Edition {{tokenEditionNumber}}</div>
  </div>

  {{#if tokenVariants.length}}
  <div class="variants">
    {{#each tokenVariants}}
    <div class="variant">
      <span class="label">{{this.label}}:</span> {{this.value}}
    </div>
    {{/each}}
  </div>
  {{/if}}

  {{#each benefits}}
  <div class="benefit">
    <h2>{{this.name}}</h2>
    <p>{{this.description}}</p>
    <div class="rules">{{this.rules}}</div>
    <p>
      <strong>From:</strong> {{dateFormat this.startDate "DD/MM/YYYY HH:mm"}}
      {{#if this.endDate}}
        <strong>To:</strong> {{dateFormat this.endDate "DD/MM/YYYY HH:mm"}}
      {{/if}}
    </p>
    {{#if this.startCheckinTime}}
    <p>
      <strong>Check-in:</strong>
      {{dateFormat this.startCheckinTime "HH:mm"}} - {{dateFormat this.endCheckinTime "HH:mm"}}
    </p>
    {{/if}}
    <div class="qr-container" id="qr-{{@index}}"></div>
  </div>
  {{/each}}

  <script>
    document.addEventListener('DOMContentLoaded', function() {
      var benefits = {{{json benefits}}};
      benefits.forEach(function(benefit, index) {
        new QRCode(document.getElementById('qr-' + index), {
          text: benefit.qrCode,
          width: 180,
          height: 180
        });
      });
    });
  </script>
</body>
</html>
```

### Passo 2: Incluir Template na Requisição

```json
{
  "tokenName": "Summer Festival VIP Pass",
  "tokenEditionNumber": "42 of 100",
  "tokenVariants": [
    { "label": "Tier", "value": "VIP" }
  ],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Full access to the festival.",
      "rules": "Single entry only.",
      "startDate": "2026-07-20T10:00:00.000Z",
      "qrCode": "https://app.w3block.io/redeem/abc123"
    }
  ],
  "pdfTemplate": "<html>... (your full HTML template) ...</html>"
}
```

### Passo 3: Tratar Resposta

Mesmo que o fluxo padrão -- um stream binário de PDF com `Content-Disposition: attachment; filename=pass.pdf`.

---

## Integração com o Módulo de Pass

O serviço de geração de PDF de passe é tipicamente chamado pelo backend de Pass & Benefits, não diretamente pelos usuários finais. O fluxo de integração é:

```
Usuário compra       Backend de Pass      Serviço de PDF
token-pass      -->  cria benefícios  -->  gera PDF
                     com QR codes          com QR codes
                                       <--  retorna stream do PDF
                <--  entrega PDF
                     ao usuário
```

### Como o Backend de Pass Chama o Serviço de PDF

1. O backend de Pass coleta os dados do token-pass (nome, edição, variantes) e os dados de benefícios (incluindo URLs de resgate de QR code)
2. Ele envia uma requisição `POST /pass/generateTickets` para o serviço de PDF
3. O serviço de PDF gera o PDF e o retorna como um stream binário
4. O backend de Pass entrega o PDF ao usuário (via link de download, anexo de e-mail, etc.)

### Conteúdo do QR Code

O campo `qrCode` de cada benefício tipicamente contém uma URL de resgate que codifica:
- O ID do benefício
- A edição do token-pass
- Um token ou assinatura de verificação

Quando escaneada em um ponto de check-in, a URL redireciona para o fluxo de resgate de benefícios da plataforma W3Block.

---

## Referência de Variáveis de Template

Lista completa de variáveis disponíveis no contexto Handlebars durante a geração de PDF de passe:

| Caminho da Variável | Tipo | Descrição | Valor de Exemplo |
|----------------------|------|-----------|------------------|
| `tokenName` | string | Nome de exibição do token-pass | `"Summer Festival VIP Pass"` |
| `tokenEditionNumber` | string | Identificador da edição | `"42 of 100"` |
| `tokenVariants` | array | Atributos de variante | `[{ "label": "Tier", "value": "VIP" }]` |
| `tokenVariants[n].label` | string | Nome do atributo de variante | `"Tier"` |
| `tokenVariants[n].value` | string | Valor do atributo de variante | `"VIP"` |
| `benefits` | array | Benefícios com QR codes | Veja BenefitsDto |
| `benefits[n].name` | string | Nome do benefício | `"General Admission"` |
| `benefits[n].description` | string | Descrição do benefício | `"Full-day access..."` |
| `benefits[n].rules` | string | Regras de uso | `"Single entry only"` |
| `benefits[n].startDate` | string | Data de início (ISO 8601) | `"2026-07-20T10:00:00.000Z"` |
| `benefits[n].endDate` | string | Data de término (ISO 8601, opcional) | `"2026-07-20T23:59:00.000Z"` |
| `benefits[n].startCheckinTime` | string | Início do check-in (ISO 8601, opcional) | `"2026-07-20T09:30:00.000Z"` |
| `benefits[n].endCheckinTime` | string | Término do check-in (ISO 8601, opcional) | `"2026-07-20T22:00:00.000Z"` |
| `benefits[n].qrCode` | string | String de dados do QR code | `"https://app.w3block.io/redeem/abc123"` |

---

## Tratamento de Erros

### Cenários Comuns de Erro

| Cenário | Erro | Resolução |
|---------|------|-----------|
| Array `benefits` vazio | 400 BadRequestException | Pelo menos um benefício deve ser fornecido na requisição |
| `tokenName` ou `tokenEditionNumber` ausente | 400 BadRequestException | Ambos os campos são strings obrigatórias |
| Benefício sem campo `qrCode` | 400 BadRequestException | Todo benefício deve incluir uma string `qrCode` |
| Template personalizado com sintaxe Handlebars inválida | 500 InternalServerErrorException | Valide a sintaxe do template. Verifique se há blocos não fechados (`{{#if}}`, `{{#each}}`) |
| Geração de PDF excede timeout de 30 segundos | 408 RequestTimeoutException | Reduza a quantidade de benefícios, simplifique o template ou comprima recursos embutidos |
| QRCode.js falha ao carregar no template personalizado | 500 InternalServerErrorException | Inclua a biblioteca QRCode.js via CDN ou insira-a inline no template |

### Considerações de Timeout

O timeout global de 30 segundos se aplica a todo o processo de geração de PDF, incluindo:
- Renderização do template com Handlebars
- Geração de QR code via execução de JavaScript
- Renderização de página e conversão para PDF pelo Puppeteer

Passes com muitos benefícios (10+) ou templates personalizados complexos com CSS pesado podem se aproximar desse limite. Estratégias para ficar dentro do timeout:

1. Limite os benefícios a um número razoável por PDF (recomendado: abaixo de 15)
2. Use layouts CSS simples (evite flexbox/grid complexos com muitos elementos aninhados)
3. Otimize tamanhos de QR code (o padrão de 180x180px é um bom equilíbrio)
4. Evite carregar recursos externos (fontes, imagens) de CDNs lentas

---

## Armadilhas Comuns

| # | Armadilha | Detalhe |
|---|-----------|---------|
| 1 | String de dados do QR code muito longa | URLs ou strings de dados extremamente longas produzem QR codes densos que podem ser difíceis de escanear. Mantenha os dados do QR abaixo de 500 caracteres. Use URLs curtas ou UUIDs em vez de payloads de dados completos |
| 2 | Muitos benefícios produzem um PDF grande | Cada benefício adiciona aproximadamente uma página. Um passe com 20 benefícios gera um PDF de 20+ páginas, que demora mais para gerar e pode atingir o timeout. Divida os benefícios em PDFs separados se necessário |
| 3 | Template personalizado sem script de QR code | Se você usar um `pdfTemplate` personalizado, deve incluir o JavaScript de geração de QR code por conta própria. O QRCode.js embutido é incluído apenas no template padrão. Inclua-o via CDN ou insira a biblioteca inline |
| 4 | Quebras de página dividindo cards de benefício | Use `page-break-inside: avoid` nos containers de cards de benefício para evitar que um único benefício seja dividido entre duas páginas. O template embutido já trata isso |
| 5 | Formatação de datas em templates personalizados | Use o helper Handlebars `dateFormat` para exibição de datas (ex.: `{{dateFormat this.startDate "DD/MM/YYYY HH:mm"}}`). Strings ISO 8601 brutas não são amigáveis para o usuário |
| 6 | Array `tokenVariants` vazio ainda é obrigatório | Mesmo que não haja variantes, o campo deve estar presente como um array vazio `[]`. Omiti-lo completamente causa um erro de validação |

---

## Fluxos Relacionados

| Fluxo | Módulo | Relacionamento |
|-------|--------|----------------|
| Gerenciamento de Pass & Benefits | [Pass](../pass/PASS_SKILL_INDEX.md) | O backend de Pass é o principal consumidor deste endpoint de PDF. Eventos do ciclo de vida do pass acionam a geração de PDF |
| Geração de Certificado | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) | Tipo alternativo de PDF para certificados de tokens blockchain (baseado em metadados) |
| Referência de API | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) | Schemas completos de DTOs, helpers de template e códigos de erro |
