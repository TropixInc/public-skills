---
id: FLOW_PDF_CERTIFICATE_GENERATION
title: "PDF - Geração de Certificados (Por Token ID, Por Query, Preview)"
module: offpix/pdf
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - pdf
  - certificate
  - metadata
  - blockchain
  - template
depends_on:
  - PDF_API_REFERENCE
---

# Geração de Certificados PDF

## Visão Geral

Este fluxo cobre a geração de certificados PDF para tokens blockchain. Os certificados são criados buscando metadados do token no serviço KEY e renderizando-os em um template HTML via Handlebars e Puppeteer. O serviço de PDF suporta três operações de certificado: geração por token ID exato, geração por consulta de metadados (encontrando o primeiro token correspondente) e pré-visualização de um layout de template personalizado.

> **Serviço interno:** O serviço de PDF não requer autenticação. É um serviço interno destinado a ser chamado por serviços backend ou ferramentas de desenvolvimento, não diretamente por frontends de usuários finais. Proteja o acesso no nível de rede.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Endereço do contrato | Endereço do contrato Ethereum onde o token reside | Da criação do token / módulo de Contracts |
| Chain ID | Identificador da rede blockchain (ex.: `137` para Polygon) | Da criação do token / módulo de Contracts |
| Token ID | ID específico do token dentro do contrato (para busca direta) | Do fluxo de minting / tokenização |
| Serviço KEY | O serviço de metadados KEY deve estar acessível na `KEY_URL` configurada | Infraestrutura / configuração de ambiente |

---

## Fluxo: Gerar Certificado por Token ID

O fluxo mais comum. Dado um endereço de contrato, chain ID e token ID específicos, gerar um certificado PDF.

### Passo 1: Solicitar o Certificado

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/certification/{address}/{chainId}/{tokenId}` | Nenhuma |

**Exemplo de Requisição (download):**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/1"
```

**Exemplo de Requisição (pré-visualização inline):**

```bash
curl "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/1?preview=true"
```

### Passo 2: Processamento Interno (automático)

O serviço executa os seguintes passos automaticamente:

1. **Buscar metadados** do serviço KEY:
   ```
   GET {KEY_URL}/metadata/address/0xAbC123.../137/1
   ```

2. **Extrair variáveis do template** da resposta de metadados:
   - `information.title` -- o título de exibição do token
   - `information.description` -- o texto de descrição do token
   - `information.mainImage` -- URL da imagem principal do token

3. **Tratar URLs de vídeo**: Se `mainImage` for uma URL de vídeo do Cloudinary (ex.: `.mp4`, `.mov`), o serviço substitui automaticamente a extensão por `.png` para usar a thumbnail do vídeo. Isso evita imagens quebradas no PDF.

4. **Construir o contexto do template**:
   ```json
   {
     "information": {
       "title": "My NFT Certificate",
       "description": "A unique digital collectible.",
       "mainImage": "https://res.cloudinary.com/.../image.png"
     },
     "certificateLink": "https://app.w3block.io/dash/certificate/0xAbC123.../137/1",
     "token": {
       "address": "0xAbC123def456789012345678901234567890ABCD",
       "chainId": 137,
       "tokenId": 1
     },
     "edition": {
       "number": 1,
       "total": 100
     }
   }
   ```

5. **Renderizar HTML** usando Handlebars com o template de certificado e os dados de contexto.

6. **Gerar PDF** usando Puppeteer (Chromium headless) e retornar o stream do PDF.

### Passo 3: Tratar a Resposta

**Sucesso (200):**

O corpo da resposta é um stream binário bruto de PDF com `Content-Type: application/pdf`.

- **Sem o parâmetro de query `preview`:** A resposta inclui o header `Content-Disposition: attachment`, solicitando download do arquivo em navegadores.
- **Com o parâmetro de query `preview`:** A resposta é inline, permitindo que navegadores exibam o PDF diretamente (ex.: em um iframe).

**Respostas de Erro:**

| Status | Causa | Ação |
|--------|-------|------|
| 404 | Token não encontrado no endereço/chainId/tokenId fornecido | Verifique se o token existe on-chain. Confira se o endereço do contrato, chain ID e token ID estão corretos |
| 500 | Falha na geração do PDF (erro de template, problema no Puppeteer, serviço KEY indisponível) | Verifique os logs do serviço. Confirme que o serviço KEY está acessível. Valide a sintaxe do template |
| 408 | Geração excedeu o timeout de 30 segundos | Simplifique o template. Reduza a quantidade e resolução das imagens |

---

## Fluxo: Gerar Certificado por Consulta de Metadados

Encontre um token por atributos de metadados e gere um certificado para a primeira correspondência. Útil quando o token ID exato não é conhecido, mas as propriedades dos metadados são.

### Passo 1: Requisição com Filtros de Metadados

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/certification/{address}/{chainId}/q` | Nenhuma |

**Exemplo de Requisição:**

```bash
curl -o certificate.pdf \
  "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/q?rarity=legendary&color=gold"
```

Todos os parâmetros de query exceto `preview` são tratados como pares chave-valor de filtro de metadados. O serviço os passa para o endpoint de metadados do KEY para encontrar tokens correspondentes.

### Passo 2: Processamento Interno (automático)

1. **Consultar metadados** do serviço KEY com os parâmetros de filtro
2. **Selecionar o primeiro token correspondente** dos resultados
3. Se **nenhuma correspondência for encontrada**, o serviço retorna um `404 NotFoundException`
4. Para o token encontrado, o serviço segue o mesmo pipeline de renderização do fluxo por token ID (Passos 2-3 acima)

### Passo 3: Tratar a Resposta

Mesmo formato de resposta do fluxo por token ID. O PDF é retornado como um stream binário.

**Exemplo com pré-visualização inline:**

```bash
curl "https://pdf.w3block.io/certification/0xAbC123def456789012345678901234567890ABCD/137/q?rarity=legendary&preview=true"
```

**Respostas de Erro:**

| Status | Causa | Ação |
|--------|-------|------|
| 404 | Nenhum token corresponde aos filtros de metadados fornecidos | Verifique se os parâmetros de filtro correspondem a metadados existentes de tokens. Confira as chaves e valores de metadados disponíveis |
| 500 | Falha na geração do PDF | Verifique os logs do serviço e a sintaxe do template |
| 408 | Geração excedeu o timeout de 30 segundos | Simplifique o template |

---

## Fluxo: Pré-visualizar um Template de Certificado Personalizado

Teste um template HTML personalizado sem precisar de um token real. O serviço escapa todos os delimitadores Handlebars para que os nomes das variáveis apareçam como texto literal no PDF de preview.

### Passo 1: Enviar o Template

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| POST | `/certification/preview` | Nenhuma |

**Requisição Mínima:**

```json
{
  "pdfTemplate": "<html><head><style>body { font-family: Arial; padding: 40px; } h1 { color: #333; }</style></head><body><h1>{{information.title}}</h1><p>{{information.description}}</p></body></html>"
}
```

**Requisição Completa (com layout completo de certificado):**

```json
{
  "pdfTemplate": "<html><head><style>body { font-family: 'Helvetica Neue', sans-serif; margin: 0; padding: 40px; } .certificate { border: 2px solid #c9a54e; padding: 60px; text-align: center; } .title { font-size: 28px; color: #1a1a1a; margin-bottom: 20px; } .image { max-width: 400px; border-radius: 8px; } .details { margin-top: 30px; font-size: 14px; color: #666; } .edition { font-size: 16px; color: #c9a54e; margin-top: 10px; }</style></head><body><div class='certificate'><h1 class='title'>{{information.title}}</h1><img class='image' src='{{information.mainImage}}' /><p>{{information.description}}</p><div class='details'><p>Contract: {{token.address}}</p><p>Chain: {{token.chainId}} | Token: {{token.tokenId}}</p></div><p class='edition'>Edition {{edition.number}} of {{edition.total}}</p><a href='{{certificateLink}}'>View Certificate</a></div></body></html>"
}
```

### Passo 2: Processamento Interno (automático)

1. **Escapar delimitadores Handlebars**: Todos os `{{` são substituídos por `\{{` para que os nomes das variáveis sejam renderizados como texto literal no preview
2. **Renderizar HTML** com o template escapado (nenhum contexto de dados é injetado)
3. **Gerar PDF** usando Puppeteer
4. Retornar o stream do PDF

### Passo 3: Revisar o Preview

O PDF retornado mostra o layout do template com os nomes das variáveis Handlebars exibidos como estão (ex.: `{{information.title}}` aparece literalmente no PDF). Isso permite que designers de templates verifiquem:

- Layout e espaçamento da página
- Estilização CSS e fontes
- Posicionamento dos elementos
- Onde os dados dinâmicos aparecerão

**Observações:**
- Imagens referenciadas por variáveis Handlebars (ex.: `{{information.mainImage}}`) aparecerão como imagens quebradas no preview, já que nenhum dado é injetado
- Para testar com imagens reais, insira URLs de imagens diretamente no template durante o preview e depois substitua por variáveis Handlebars para produção

---

## Referência de Variáveis de Template

Lista completa de variáveis disponíveis no contexto Handlebars durante a geração de certificados:

| Caminho da Variável | Tipo | Descrição | Valor de Exemplo |
|----------------------|------|-----------|------------------|
| `information.title` | string | Título dos metadados do token | `"Genesis Collection #42"` |
| `information.description` | string | Descrição dos metadados do token | `"A unique digital artwork..."` |
| `information.mainImage` | string | URL da imagem principal do token (URLs de vídeo convertidas automaticamente para PNG) | `"https://res.cloudinary.com/.../image.png"` |
| `certificateLink` | string | URL para a página de certificado no frontend | `"https://app.w3block.io/dash/certificate/0xAbC.../137/1"` |
| `token.address` | string | Endereço do contrato | `"0xAbC123...ABCD"` |
| `token.chainId` | number | Chain ID da blockchain | `137` |
| `token.tokenId` | number | Token ID | `1` |
| `edition` | object | Metadados da edição vindos do serviço KEY | `{ "number": 1, "total": 100 }` |

---

## Integração com o Frontend

### Padrão de URL da Página de Certificado

O frontend Offpix exibe certificados em:

```
/dash/certificate/{contractAddress}/{chainId}/{tokenId}
```

### Padrão de Download (JavaScript/TypeScript)

```typescript
import axios from 'axios';

async function downloadCertificate(
  address: string,
  chainId: number,
  tokenId: number
): Promise<void> {
  const response = await axios.get(
    `https://pdf.w3block.io/certification/${address}/${chainId}/${tokenId}`,
    { responseType: 'arraybuffer' }
  );

  const blob = new Blob([response.data], { type: 'application/pdf' });
  const url = window.URL.createObjectURL(blob);

  const link = document.createElement('a');
  link.href = url;
  link.download = `certificate-${tokenId}.pdf`;
  link.click();

  window.URL.revokeObjectURL(url);
}
```

### Padrão de Preview (iframe)

```html
<iframe
  src="https://pdf.w3block.io/certification/0xAbC123.../137/1?preview=true"
  width="100%"
  height="800px"
  style="border: none;"
></iframe>
```

> **Nota de segurança:** Como o serviço de PDF é interno, aplicações frontend devem fazer proxy das requisições de certificado através de seu próprio backend, em vez de chamar o serviço de PDF diretamente do navegador.

---

## Tratamento de Erros

### Cenários Comuns de Erro

| Cenário | Erro | Resolução |
|---------|------|-----------|
| Token não existe on-chain | 404 NotFoundException | Verifique se o token foi mintado. Confira o endereço do contrato, chain ID e token ID |
| Serviço de metadados KEY está inacessível | 500 InternalServerErrorException | Verifique a saúde do serviço KEY. Confirme a configuração de `KEY_URL` |
| Template contém sintaxe Handlebars inválida | 500 InternalServerErrorException | Valide a sintaxe do template. Verifique se há blocos `{{#if}}` ou `{{#each}}` não fechados |
| Geração de PDF excede 30 segundos | 408 RequestTimeoutException | Reduza a complexidade do template. Minimize o carregamento de recursos externos. Comprima imagens |
| Metadados possuem URL de vídeo para mainImage | Sem erro (tratado automaticamente) | O serviço converte automaticamente URLs de vídeo do Cloudinary para thumbnails PNG. URLs de vídeo fora do Cloudinary podem renderizar como imagens quebradas |

### Estratégia de Retry

O serviço de PDF não implementa retries internos. Serviços consumidores devem implementar sua própria lógica de retry com backoff exponencial para falhas transitórias (500, 408).

```typescript
async function getCertificateWithRetry(
  url: string,
  maxRetries: number = 3
): Promise<ArrayBuffer> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const response = await axios.get(url, { responseType: 'arraybuffer' });
      return response.data;
    } catch (error) {
      if (attempt === maxRetries) throw error;
      const delay = Math.pow(2, attempt) * 1000;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## Armadilhas Comuns

| # | Armadilha | Detalhe |
|---|-----------|---------|
| 1 | URL de vídeo em mainImage produz imagem quebrada | Apenas URLs de vídeo do Cloudinary são convertidas automaticamente. Se seu vídeo está hospedado em outro lugar, o PDF conterá uma imagem quebrada. Hospede no Cloudinary ou forneça uma URL de imagem direta nos metadados |
| 2 | Erros de sintaxe do template falham silenciosamente | Handlebars pode renderizar strings vazias para variáveis indefinidas em vez de lançar erros. Use guards `{{#if variable}}` para tratar dados ausentes |
| 3 | Preview escapa todos os Handlebars | O endpoint de preview não é uma funcionalidade de "renderizar com dados de exemplo". Ele escapa todos os delimitadores `{{`. Para testar com dados reais, use o endpoint por token ID com um token de teste |
| 4 | Imagens grandes causam timeout | Imagens embutidas no template são carregadas pelo Puppeteer durante a renderização. Imagens grandes (especialmente de alta resolução) podem ultrapassar o timeout de 30 segundos |
| 5 | Fontes externas podem não carregar | Puppeteer roda em um ambiente isolado (sandbox). URLs de fontes externas (Google Fonts, etc.) podem falhar ao carregar. Insira fontes como base64 inline ou use fontes do sistema |

---

## Fluxos Relacionados

| Fluxo | Módulo | Relacionamento |
|-------|--------|----------------|
| Criação / Minting de Token | [Contracts](../contracts/CONTRACTS_SKILL_INDEX.md) | Tokens devem existir on-chain antes que certificados possam ser gerados |
| Metadados do Token | [Tokenization](../tokenization/TOKENIZATION_SKILL_INDEX.md) | Dados do certificado (título, descrição, imagem) vêm dos metadados do token |
| Geração de Passe | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) | Tipo alternativo de PDF para casos de uso de passes/ingressos |
