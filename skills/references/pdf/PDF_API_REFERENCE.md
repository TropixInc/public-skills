---
id: PDF_API_REFERENCE
title: "PDF - Referência da API"
module: offpix/pdf
version: "1.0.0"
type: api-reference
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - pdf
  - certificate
  - pass
  - puppeteer
  - handlebars
  - api-reference
---

# Referência da API de PDF

Referência completa de endpoints para o módulo PDF da W3Block (w3block-pdf). Este é um **serviço interno** construído com NestJS, Puppeteer e Handlebars que gera documentos PDF a partir de templates HTML. Ele atende dois casos de uso principais: certificados de tokens blockchain e PDFs de pass/ticket com QR codes.

> **Serviço interno:** Todos os endpoints não requerem autenticação. O serviço é projetado para rodar atrás de um firewall ou VPN e nunca deve ser exposto diretamente à internet pública. Desenvolvedores construindo aplicações customizadas podem chamar esses endpoints a partir de seus serviços backend.

## URLs Base

| Ambiente | URL |
|----------|-----|
| Produção | `https://pdf.w3block.io` |
| Swagger | https://pdf.w3block.io/docs |

## Autenticação

**Nenhuma necessária.** Este é um serviço interno. Todos os endpoints são abertos e não requerem nenhum header de autenticação. O controle de acesso é imposto no nível de rede/infraestrutura (firewall, VPN, service mesh).

---

## Configuração

| Variável | Descrição | Padrão |
|----------|-----------|--------|
| `KEY_URL` | URL base do serviço de metadados KEY (usado para buscar metadados de tokens) | `https://api.w3block.io` |
| `GLOBAL_TIMEOUT` | Timeout de requisição em milissegundos | `30000` (30 segundos) |

> **Timeout:** Todas as requisições estão sujeitas a um timeout global de 30 segundos. A geração de PDF para templates complexos com muitas imagens pode exceder este limite. Otimize os templates para ficar dentro da janela de timeout.

---

## Endpoints

### Endpoints de Certificado

#### GET /certification/{address}/{chainId}/{tokenId}

Gerar um certificado PDF para um token blockchain específico pelo seu token ID.

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/certification/{address}/{chainId}/{tokenId}` | None |

**Parâmetros de Caminho:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `address` | string | Sim | Endereço do contrato Ethereum (hex, ex: `0x1234...abcd`) |
| `chainId` | number | Sim | ID da cadeia blockchain (ex: `137` para Polygon, `80001` para Mumbai) |
| `tokenId` | number | Sim | ID do token dentro do contrato |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `preview` | any | Não | Se presente (qualquer valor), retorna PDF inline para exibição no navegador em vez de como download |

**Resposta:**

| Status | Content-Type | Descrição |
|--------|-------------|-----------|
| 200 | `application/pdf` | Stream PDF. Sem `preview`, inclui header `Content-Disposition: attachment` |
| 404 | `application/json` | Token não encontrado no address/chainId/tokenId fornecido |
| 500 | `application/json` | Geração de PDF falhou (erro de template, crash do Puppeteer) |
| 408 | `application/json` | Requisição excedeu o timeout de 30 segundos |

**Fluxo Interno:**

1. O serviço chama o endpoint de metadados KEY: `GET {KEY_URL}/metadata/address/{address}/{chainId}/{tokenId}`
2. Extrai `information` (título, descrição, mainImage), `certificateLink`, detalhes do token e dados da edição
3. Se `mainImage` for uma URL de vídeo do Cloudinary, converte para thumbnail PNG (substitui extensão do arquivo)
4. Renderiza o template HTML do certificado com Handlebars usando o contexto de metadados
5. Usa Puppeteer para converter o HTML renderizado em PDF
6. Retorna o stream PDF

**Exemplo de Requisição:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/42"
```

**Exemplo de Requisição (pré-visualização inline):**

```bash
curl "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/42?preview=true"
```

---

#### GET /certification/{address}/{chainId}/q?

Gerar um certificado PDF consultando metadados de token com filtros customizados. Retorna um certificado para o primeiro token correspondente.

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/certification/{address}/{chainId}/q` | None |

**Parâmetros de Caminho:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `address` | string | Sim | Endereço do contrato Ethereum (hex) |
| `chainId` | number | Sim | ID da cadeia blockchain |

**Parâmetros de Query:**

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `preview` | any | Não | Se presente, retorna PDF inline para exibição no navegador |
| (filtros customizados) | any | Não | Quaisquer parâmetros de query adicionais são tratados como pares chave-valor de filtro de metadados |

**Resposta:**

| Status | Content-Type | Descrição |
|--------|-------------|-----------|
| 200 | `application/pdf` | Stream PDF para o primeiro token correspondente |
| 404 | `application/json` | Nenhum token corresponde aos filtros de metadados fornecidos |
| 500 | `application/json` | Geração de PDF falhou |
| 408 | `application/json` | Requisição excedeu o timeout de 30 segundos |

**Fluxo Interno:**

1. O serviço chama o endpoint de metadados KEY com parâmetros de query como filtros
2. Retorna os metadados do primeiro token correspondente
3. Se nenhuma correspondência for encontrada, lança `NotFoundException`
4. Gera o certificado PDF a partir dos metadados do token correspondente

**Exemplo de Requisição:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0x1234567890abcdef1234567890abcdef12345678/137/q?rarity=legendary&edition=1"
```

---

#### POST /certification/preview

Gerar um PDF de pré-visualização a partir de um template HTML customizado. Os delimitadores Handlebars são escapados para que as variáveis apareçam como texto literal (ex: `{{information.title}}` aparece como está no PDF).

| Método | Caminho | Auth |
|--------|---------|------|
| POST | `/certification/preview` | None |

**Corpo da Requisição:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `pdfTemplate` | string | Sim | String HTML completa com sintaxe Handlebars para o template do certificado |

**Requisição Mínima:**

```json
{
  "pdfTemplate": "<html><body><h1>{{information.title}}</h1><p>{{information.description}}</p><img src='{{information.mainImage}}' /><p>Token: {{token.address}} #{{token.tokenId}}</p></body></html>"
}
```

**Resposta:**

| Status | Content-Type | Descrição |
|--------|-------------|-----------|
| 200 | `application/pdf` | Stream PDF com variáveis Handlebars escapadas exibidas como texto literal |
| 400 | `application/json` | Corpo da requisição inválido (faltando `pdfTemplate`) |
| 500 | `application/json` | Geração de PDF falhou |

**Notas:**
- A pré-visualização escapa todos os delimitadores Handlebars (`{{` se torna `\{{`) para que as variáveis sejam exibidas como texto de placeholder em vez de renderizadas com dados vazios
- Útil para designers de templates verificarem layout e estilo antes de implantar um template

**Exemplo de Requisição:**

```bash
curl -X POST "https://pdf.w3block.io/certification/preview" \
  -H "Content-Type: application/json" \
  -d '{"pdfTemplate": "<html><body><h1>{{information.title}}</h1></body></html>"}'  \
  -o preview.pdf
```

---

### Endpoints de Pass

#### POST /pass/generateTickets

Gerar um PDF de pass/ticket contendo informações do token, detalhes de variante e benefícios com QR codes.

| Método | Caminho | Auth |
|--------|---------|------|
| POST | `/pass/generateTickets` | None |

**Corpo da Requisição (PassDto):**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenName` | string | Sim | Nome de exibição do token pass |
| `tokenEditionNumber` | string | Sim | Número ou identificador da edição |
| `tokenVariants` | TokenVariantsDto[] | Sim | Array de pares label/valor de variantes (pode ser vazio `[]`) |
| `benefits` | BenefitsDto[] | Sim | Array de benefícios a incluir no pass |
| `pdfTemplate` | string | Não | Template HTML customizado com sintaxe Handlebars. Se omitido, o template padrão embutido é usado |

**Requisição Mínima:**

```json
{
  "tokenName": "VIP Concert Pass",
  "tokenEditionNumber": "42",
  "tokenVariants": [],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Access to the main event area",
      "rules": "Valid for one entry only. No re-entry allowed.",
      "startDate": "2026-06-15T18:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-1234"
    }
  ]
}
```

**Requisição Completa:**

```json
{
  "tokenName": "VIP Concert Pass",
  "tokenEditionNumber": "42",
  "tokenVariants": [
    {
      "label": "Tier",
      "value": "Platinum"
    },
    {
      "label": "Seat",
      "value": "A12"
    }
  ],
  "benefits": [
    {
      "name": "General Admission",
      "description": "Access to the main event area with reserved seating.",
      "rules": "Valid for one entry only. No re-entry allowed. Must present QR code at gate.",
      "startDate": "2026-06-15T18:00:00.000Z",
      "endDate": "2026-06-15T23:59:00.000Z",
      "startCheckinTime": "2026-06-15T17:00:00.000Z",
      "endCheckinTime": "2026-06-15T20:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-1234"
    },
    {
      "name": "Backstage Meet & Greet",
      "description": "Exclusive backstage access to meet the performers.",
      "rules": "Limited to 30 minutes. No photography allowed.",
      "startDate": "2026-06-15T22:00:00.000Z",
      "endDate": "2026-06-15T22:30:00.000Z",
      "startCheckinTime": "2026-06-15T21:45:00.000Z",
      "endCheckinTime": "2026-06-15T22:00:00.000Z",
      "qrCode": "https://example.com/redeem/benefit-uuid-5678"
    }
  ],
  "pdfTemplate": "<html><body><h1>{{tokenName}}</h1><p>Edition: {{tokenEditionNumber}}</p>{{#each benefits}}<div><h2>{{this.name}}</h2><p>{{this.description}}</p></div>{{/each}}</body></html>"
}
```

**Resposta:**

| Status | Content-Type | Descrição |
|--------|-------------|-----------|
| 200 | `application/pdf` | Stream PDF com `Content-Disposition: attachment; filename=pass.pdf` |
| 400 | `application/json` | Corpo da requisição inválido (campos obrigatórios ausentes, DTO malformado) |
| 500 | `application/json` | Geração de PDF falhou |
| 408 | `application/json` | Requisição excedeu o timeout de 30 segundos |

**Exemplo de Requisição:**

```bash
curl -X POST "https://pdf.w3block.io/pass/generateTickets" \
  -H "Content-Type: application/json" \
  -d '{
    "tokenName": "VIP Pass",
    "tokenEditionNumber": "1",
    "tokenVariants": [],
    "benefits": [{
      "name": "Entry",
      "description": "General admission",
      "rules": "Single use",
      "startDate": "2026-06-15T18:00:00.000Z",
      "qrCode": "https://example.com/redeem/abc123"
    }]
  }' \
  -o pass.pdf
```

---

### Endpoints de Saúde

#### GET /health

Verificação completa de saúde retornando métricas de uso de memória.

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/health` | None |

**Resposta (200):**

```json
{
  "status": "ok",
  "info": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  },
  "error": {},
  "details": {
    "memory_heap": { "status": "up" },
    "memory_rss": { "status": "up" }
  }
}
```

---

#### GET /health/liveness

Sonda de liveness para orquestração de contêineres (Kubernetes).

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/health/liveness` | None |

**Resposta:**

| Status | Descrição |
|--------|-----------|
| 204 | Serviço está ativo |
| 500 | Serviço não está respondendo |

---

#### GET /health/readiness

Sonda de readiness para orquestração de contêineres (Kubernetes).

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/health/readiness` | None |

**Resposta:**

| Status | Descrição |
|--------|-----------|
| 204 | Serviço está pronto para aceitar requisições |
| 503 | Serviço não está pronto (dependências indisponíveis) |

---

#### GET /

Endpoint de informações da aplicação retornando metadados do serviço.

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/` | None |

**Resposta (200):**

```json
{
  "name": "w3block-pdf",
  "version": "0.0.1",
  "node_env": "production"
}
```

---

## DTOs

### CertificationDto

Usado internamente ao resolver requisições de certificado por token ID.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `address` | string | Sim | Endereço do contrato Ethereum (formato hex, ex: `0x1234...abcd`) |
| `chainId` | number | Sim | ID da cadeia blockchain (ex: `137` para Polygon) |
| `tokenId` | number | Sim | ID do token dentro do contrato |

### CertificationPreviewDto

Corpo da requisição para o endpoint de pré-visualização de certificado.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `pdfTemplate` | string | Sim | String HTML completa contendo o template do certificado com sintaxe Handlebars |

### PassDto

Corpo da requisição para geração de PDF de pass/ticket.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `tokenName` | string | Sim | Nome de exibição do token pass |
| `tokenEditionNumber` | string | Sim | Número ou rótulo da edição (ex: `"42"`, `"Limited #5"`) |
| `tokenVariants` | TokenVariantsDto[] | Sim | Array de pares label/valor de variantes. Passe `[]` se não houver variantes |
| `benefits` | BenefitsDto[] | Sim | Array de benefícios com QR codes |
| `pdfTemplate` | string | Não | Template HTML customizado. Se omitido, o template padrão embutido é usado |

### TokenVariantsDto

Descreve um único atributo de variante de um token pass.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `label` | string | Sim | Nome do atributo de variante (ex: `"Color"`, `"Tier"`, `"Seat"`) |
| `value` | string | Sim | Valor do atributo de variante (ex: `"Blue"`, `"Platinum"`, `"A12"`) |

### BenefitsDto

Descreve uma única entrada de benefício dentro de um pass, incluindo os dados do QR code.

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Sim | Nome de exibição do benefício |
| `description` | string | Sim | Texto de descrição do benefício |
| `rules` | string | Sim | Regras e restrições de uso |
| `startDate` | string | Sim | Data/hora de início do benefício (ISO 8601) |
| `endDate` | string | Não | Data/hora de término do benefício (ISO 8601) |
| `startCheckinTime` | string | Não | Início da janela de check-in (ISO 8601) |
| `endCheckinTime` | string | Não | Fim da janela de check-in (ISO 8601) |
| `qrCode` | string | Sim | Dados a codificar no QR code (tipicamente uma URL de resgate) |

---

## Sintaxe de Template

O serviço de PDF usa [Handlebars](https://handlebarsjs.com/) como motor de templates. Os templates são documentos HTML completos que são renderizados para PDF via Puppeteer.

### Uso Básico do Handlebars

```html
<!-- Saída de variável -->
<h1>{{information.title}}</h1>
<p>{{information.description}}</p>
<img src="{{information.mainImage}}" />

<!-- Renderização condicional -->
{{#if information.description}}
  <p>{{information.description}}</p>
{{/if}}

<!-- Iteração sobre arrays -->
{{#each benefits}}
  <div>
    <h2>{{this.name}}</h2>
    <p>{{this.description}}</p>
  </div>
{{/each}}
```

### Helpers Customizados

O serviço de PDF registra dois helpers Handlebars customizados:

#### dateFormat

Formata uma string de data usando um padrão de formato especificado.

```handlebars
{{dateFormat startDate "DD/MM/YYYY"}}
{{dateFormat startDate "YYYY-MM-DD HH:mm"}}
{{dateFormat createdAt "MMM D, YYYY"}}
```

#### xIf

Helper condicional estendido que suporta operadores de comparação.

```handlebars
{{#xIf edition "==" "Limited"}}
  <span class="badge">Limited Edition</span>
{{/xIf}}

{{#xIf tokenId ">" 100}}
  <span>High token ID</span>
{{/xIf}}

{{#xIf status "!=" "expired"}}
  <span>Active</span>
{{/xIf}}

{{#xIf value "typeof" "string"}}
  <span>{{value}}</span>
{{/xIf}}
```

**Operadores suportados:** `==`, `===`, `!=`, `<`, `>`, `<=`, `>=`, `typeof`

### Variáveis do Template de Certificado

Ao gerar certificados, as seguintes variáveis estão disponíveis no contexto Handlebars:

| Variável | Tipo | Descrição |
|----------|------|-----------|
| `information.title` | string | Título dos metadados do token |
| `information.description` | string | Descrição dos metadados do token |
| `information.mainImage` | string | URL para a imagem principal do token (URLs de vídeo são convertidas automaticamente para PNG) |
| `certificateLink` | string | URL completa para a página de certificado no frontend |
| `token.address` | string | Endereço do contrato |
| `token.chainId` | number | ID da cadeia blockchain |
| `token.tokenId` | number | ID do token |
| `edition` | object | Detalhes da edição do serviço de metadados |

### Variáveis do Template de Pass

Ao gerar PDFs de pass/ticket, as seguintes variáveis estão disponíveis:

| Variável | Tipo | Descrição |
|----------|------|-----------|
| `tokenName` | string | Nome de exibição do token pass |
| `tokenEditionNumber` | string | Número da edição |
| `tokenVariants` | array | Array de objetos `{ label, value }` |
| `tokenVariants[].label` | string | Nome do atributo de variante |
| `tokenVariants[].value` | string | Valor do atributo de variante |
| `benefits` | array | Array de objetos de benefício |
| `benefits[].name` | string | Nome do benefício |
| `benefits[].description` | string | Descrição do benefício |
| `benefits[].rules` | string | Regras do benefício |
| `benefits[].startDate` | string | Data de início (ISO 8601) |
| `benefits[].endDate` | string | Data de término (ISO 8601, pode estar ausente) |
| `benefits[].startCheckinTime` | string | Início do check-in (ISO 8601, pode estar ausente) |
| `benefits[].endCheckinTime` | string | Fim do check-in (ISO 8601, pode estar ausente) |
| `benefits[].qrCode` | string | Dados do QR code |

---

## Referência de Erros

| Status | Exceção | Causa | Resolução |
|--------|---------|-------|-----------|
| 400 | `BadRequestException` | DTO inválido: endereço Ethereum malformado, chain ID inválido, campos obrigatórios ausentes | Verifique se o corpo da requisição corresponde ao schema do DTO. Endereços Ethereum devem ser strings hex válidas |
| 404 | `NotFoundException` | Nenhum token encontrado correspondendo ao address/chainId/tokenId ou filtro de metadados fornecido | Verifique se o token existe on-chain. Confira os valores de address, chainId e tokenId. Para consultas baseadas em filtros, verifique os parâmetros de filtro |
| 408 | `RequestTimeoutException` | Geração de PDF excedeu o timeout global de 30 segundos | Simplifique o template HTML. Reduza a quantidade e o tamanho das imagens. Evite carregar recursos externos no template |
| 500 | `InternalServerErrorException` | Geração de PDF falhou devido a erro de renderização de template, crash do Puppeteer ou indisponibilidade do serviço KEY | Verifique a sintaxe do template para erros Handlebars. Verifique se o serviço KEY está acessível. Confira os logs do serviço para erros do Puppeteer |

---

## Tratamento de URL de Vídeo

O fluxo de geração de certificados inclui conversão automática de vídeo para thumbnail para URLs `mainImage`. Quando o serviço de metadados retorna uma URL de vídeo do Cloudinary, o serviço de PDF converte para uma thumbnail PNG estática:

| URL Original | URL Convertida |
|-------------|----------------|
| `https://res.cloudinary.com/.../video.mp4` | `https://res.cloudinary.com/.../video.png` |
| `https://res.cloudinary.com/.../clip.mov` | `https://res.cloudinary.com/.../clip.png` |
| `https://example.com/image.png` | `https://example.com/image.png` (sem alteração) |

Esta conversão se aplica apenas a URLs de vídeo hospedadas no Cloudinary. URLs não-Cloudinary e URLs de não-vídeo são passadas sem alteração.
