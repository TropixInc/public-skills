---
id: PDF_SKILL_INDEX
title: "Índice de Skills de PDF"
module: offpix/pdf
version: "1.0.0"
type: index
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
---

# Índice de Skills de PDF

Documentação do módulo de PDF da W3Block (w3block-pdf). Este é um **serviço interno** que gera documentos PDF a partir de templates HTML usando NestJS, Puppeteer e Handlebars. Ele produz certificados PDF para tokens blockchain (usando metadados on-chain) e PDFs de passes/ingressos com QR codes para resgate de benefícios. Embora o serviço não seja exposto diretamente aos usuários finais, sua API é útil para desenvolvedores que constroem aplicações personalizadas sobre a plataforma W3Block.

**Serviço:** PDF (w3block-pdf) | **Swagger:** https://pdf.w3block.io/docs

> **Serviço interno:** O serviço de PDF roda sem autenticação em seus endpoints. O acesso é controlado no nível de rede/firewall. Não exponha este serviço diretamente à internet pública.

---

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) | 1.0.0 | Todos os endpoints, DTOs, sintaxe de templates, configuração | Implementado | Referência de API a qualquer momento |
| 2 | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) | 1.0.0 | Gerar certificados a partir de metadados de token, pré-visualização, busca por query | Implementado | Construir funcionalidades de download/pré-visualização de certificados |
| 3 | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) | 1.0.0 | Gerar PDFs de passes/ingressos com benefícios e QR codes | Implementado | Construir funcionalidades de geração de PDFs de passes/ingressos |

---

## Guia Rápido

### Para geração de certificados:

```
1. Leia: FLOW_PDF_CERTIFICATE_GENERATION.md   -> Gerar certificado por token ID ou consulta de metadados
2. Consulte: PDF_API_REFERENCE.md              -> Para DTOs, variáveis de template, códigos de erro
```

### Para geração de passes/ingressos:

```
1. Leia: FLOW_PDF_PASS_GENERATION.md          -> Gerar PDF de passe com benefícios e QR codes
2. Consulte: PDF_API_REFERENCE.md              -> Para esquema do PassDto, sintaxe de template, helpers
```

### Implementação mínima orientada à API:

```
1. Gerar um certificado     -> GET /certification/{address}/{chainId}/{tokenId}
2. Gerar um passe/ingresso  -> POST /pass/generateTickets
3. Consultar detalhes da API -> PDF_API_REFERENCE.md
```

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|----------|---------|
| 1 | Geração de PDF expira após 30 segundos | Simplifique o template HTML, reduza o tamanho das imagens, evite carregamento de recursos externos nos templates. O serviço impõe um timeout global de 30 segundos |
| 2 | URLs de vídeo em `mainImage` produzem imagens quebradas | O serviço converte automaticamente URLs de vídeo do Cloudinary para thumbnails PNG (substitui a extensão por `.png`). Certifique-se de que os assets de vídeo estejam hospedados no Cloudinary para que isso funcione |
| 3 | Pré-visualização do template renderiza variáveis Handlebars literalmente | O endpoint de preview escapa `{{var}}` para `\{{var}}` para evitar renderização com dados vazios. Este é o comportamento esperado |
| 4 | Endpoints acessíveis sem autenticação | Isso é por design. O serviço de PDF é um serviço interno sem camada de autenticação. Proteja-o via políticas de rede e regras de firewall; nunca o exponha diretamente à internet pública |

---

## Tabela de Decisão

| Eu quero... | Leia isto |
|-------------|-----------|
| Gerar um certificado PDF para um token específico | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Encontrar um token por metadados e gerar seu certificado | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Pré-visualizar um template de certificado personalizado | [FLOW_PDF_CERTIFICATE_GENERATION](./FLOW_PDF_CERTIFICATE_GENERATION.md) |
| Gerar um PDF de passe/ingresso com QR codes | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) |
| Usar um template HTML personalizado para passes | [FLOW_PDF_PASS_GENERATION](./FLOW_PDF_PASS_GENERATION.md) |
| Entender DTOs, helpers Handlebars e variáveis de template | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |
| Verificar saúde e configuração do serviço | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |
| Entender códigos de erro e timeouts | [PDF_API_REFERENCE](./PDF_API_REFERENCE.md) |

---

## Matriz: Endpoints x Documentos

| Endpoint | Ref. API | Geração de Certificado | Geração de Passe |
|----------|:--------:|:----------------------:|:-----------------:|
| GET /certification/{address}/{chainId}/{tokenId} | X | X | |
| GET /certification/{address}/{chainId}/q? | X | X | |
| POST /certification/preview | X | X | |
| POST /pass/generateTickets | X | | X |
| GET /health | X | | |
| GET /health/liveness | X | | |
| GET /health/readiness | X | | |
| GET / | X | | |
