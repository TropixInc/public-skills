---
id: OFFPIX_DOCUMENTATION_SPEC
title: "Especificação do Processo de Documentação Offpix"
module: offpix
version: "1.0.0"
type: setup
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - process
  - meta
  - documentation
---

# Especificação do Processo de Documentação Offpix

Este documento define o processo para gerar documentação da API W3Block voltada para **desenvolvedores e usuários assistidos por IA** que desejam consumir o backend da W3Block para construir seus próprios dashboards e aplicações. Consulte esta especificação toda vez que iniciar uma nova tarefa de documentação.

---

## 1. Público-Alvo e Objetivo

**Quem:** Desenvolvedores e usuários assistidos por IA construindo frontends customizados sobre a plataforma W3Block. Eles têm acesso aos docs do Swagger mas precisam de orientação sobre como orquestrar chamadas de API em fluxos funcionais.

**O que eles precisam:**
- Entender o domínio (entidades, relacionamentos, regras de negócio)
- Conhecer a sequência correta de chamadas de API para cada fluxo
- Ter payloads reais que possam copiar e adaptar
- Evitar armadilhas comuns que não são óbvias apenas pelo Swagger

**O que eles já possuem:**
- Documentação Swagger para todos os 6 serviços
- Entendimento básico de REST APIs e autenticação

**O que esta documentação NÃO é:**
- Uma cópia do Swagger (não duplique listagens de endpoints sem valor agregado)
- Específica de frontend (sem componentes React, hooks ou padrões de UI — foco na API)
- Um guia interno de desenvolvimento (assuma que o leitor não tem acesso ao código-fonte)

---

## 2. Material de Referência

### 2.1 Serviços Backend (Fonte Primária)

Use os serviços backend para entender lógica de negócio, validações, relacionamentos entre entidades e fluxos que o Swagger não expõe.

| Serviço | Swagger | Framework |
|---------|---------|-----------|
| **ID** (Auth/Identity) | https://pixwayid.w3block.io/docs/ | NestJS + TypeORM |
| **KEY** (Main API / Registry) | https://api.w3block.io/docs/ | NestJS + TypeORM |
| **Commerce** | https://commerce.w3block.io/docs | NestJS + TypeORM |
| **Pass** | https://pass.w3block.io/docs | NestJS + TypeORM |
| **PDF** | https://pdf.w3block.io/docs | NestJS + Puppeteer |

> **Nota:** O serviço PDF é interno e não é exposto aos usuários finais. No entanto, seu Swagger pode fornecer insights sobre a lógica de geração de documentos e padrões de templates.

**O que procurar nos codebases backend NestJS:**
- Diretórios de módulos — entender limites de domínio e relacionamentos entre entidades
- Arquivos de entidade — modelos de dados, tipos de colunas, relações, enums
- Arquivos de serviço — lógica de negócio, validações, efeitos colaterais, cadeias de eventos
- Arquivos de controller — assinaturas de endpoints, guards, decorators, DTOs de request/response
- Arquivos de DTO — formatos de request/response com decorators de validação
- Migrations — evolução do schema, entender o estado atual
- Arquivos de enum — valores de status, tipos, categorias

### 2.2 Referência do Frontend (Fonte Secundária — Nomes, Fluxos e Hierarquia)

O frontend Offpix serve como uma **implementação de referência** — prova de que as sequências de API funcionam em produção. Além de rastrear chamadas, é uma fonte crítica para **nomenclatura e hierarquia de fluxos**. Nomes de campos da API são frequentemente pouco intuitivos ou técnicos; o frontend usa labels amigáveis e uma hierarquia de navegação clara (o que vem antes do quê, relacionamentos pai-filho, ordenação de etapas) que devem ser refletidos na documentação. Use-o para:

- **Adotar a nomenclatura do frontend** ao documentar campos e conceitos — se o frontend chama um campo da API por um nome diferente e mais claro, prefira o nome do frontend na documentação e indique o nome do campo na API entre parênteses
- **Mapear a hierarquia de fluxos** — entender quais telas/etapas vêm antes de outras, quais são os relacionamentos pai-filho, e documentar fluxos na mesma ordem que o usuário os vivencia
- Rastrear a ordem real das chamadas de API para um dado fluxo
- Extrair payloads reais sendo enviados (de hooks, mutations, chamadas SDK)
- Entender quais campos são realmente obrigatórios vs. opcionais na prática
- Identificar padrões de tratamento de erros e casos de borda

**Áreas-chave para examinar no frontend:**
- Enums de rotas da API — todos os 170+ padrões de rotas da API
- Configuração do cliente da API
- Hooks por módulo — padrões de fetching de dados por domínio
- Implementações de chamadas de API por módulo
- Implementação do fluxo de autenticação (ex.: route handlers do NextAuth)

### 2.3 Documentação Existente (Fonte Terciária)

Verifique a documentação de skills existente para fluxos já documentados. Não duplique — referencie e estenda.

---

## 3. Processo de Pesquisa

Antes de escrever qualquer documentação, siga esta sequência:

```
Etapa 1: Escopo
  └─ Usuário informa qual fluxo/domínio documentar
  └─ Verificar se já existe na documentação de skills existente
  └─ Identificar qual(is) serviço(s) backend estão envolvidos

Etapa 2: Aprofundamento no Backend
  └─ Ler o(s) módulo(s) relevante(s) no repositório backend
  └─ Mapear entidades, relacionamentos e enums
  └─ Rastrear a lógica do serviço: validações, efeitos colaterais, cadeias de eventos
  └─ Ler DTOs para entender contratos de request/response
  └─ Verificar guards/decorators (requisitos de auth, roles, permissões)

Etapa 3: Verificação no Swagger
  └─ Cruzar referência do código backend com o output do Swagger
  └─ Anotar quaisquer discrepâncias (Swagger pode estar desatualizado em relação ao código)
  └─ Capturar paths exatos dos endpoints, métodos e status codes

Etapa 4: Rastreamento no Frontend
  └─ Encontrar como o frontend Offpix consome esses endpoints
  └─ Extrair a sequência de chamadas (qual endpoint primeiro, segundo, etc.)
  └─ Capturar payloads reais dos hooks/mutations
  └─ Anotar tratamento de erros e casos de borda

Etapa 5: Escrever
  └─ Seguir o template de output (Seção 4)
  └─ Usar payloads progressivos (Seção 5)
  └─ Aplicar checklist de qualidade (Seção 7)
```

---

## 4. Template de Output

Todo arquivo de documentação segue esta estrutura. Escale cada seção conforme sua complexidade — um fluxo simples pode ter uma visão geral curta, um complexo pode precisar de diagramas.

### 4.1 Frontmatter YAML (Obrigatório)

```yaml
---
id: FLOW_OFFPIX_{DOMAIN}_{FLOW_NAME}
title: "{Domain} - {Flow Name}"
module: offpix
version: "1.0.0"
type: flow                    # flow | api-reference | index
status: draft                 # draft | review | implemented | deprecated
last_updated: "YYYY-MM-DD"
authors:
  - rafaelmhp
tags:
  - domain-tag
  - flow-tag
depends_on:
  - DEPENDENCY_SKILL_ID
---
```

### 4.2 Corpo do Documento

```markdown
# {Título do Fluxo}

## Visão Geral

Breve descrição do que este fluxo realiza, quando usá-lo e qual é
o resultado final. 2-4 frases no máximo.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| `companyId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Bearer token | JWT access token | Autenticação no serviço ID |
| ... | ... | ... |

## Entidades e Relacionamentos

Descreva o modelo de domínio relevante para este fluxo. Inclua:
- Nomes das entidades e seu propósito
- Campos-chave e seus tipos
- Relacionamentos entre entidades (1:N, N:M, etc.)
- Enums/valores de status relevantes

Use diagramas ASCII para relacionamentos complexos:
```
Entity A (1) ──→ (N) Entity B ──→ (N) Entity C
```

## Fluxo: {Nome do Fluxo}

### Etapa 1: {Descrição da Ação}

**Endpoint:**
| Método | Path | Auth | Content-Type |
|--------|------|------|-------------|
| POST | `/path/{param}` | Bearer token | application/json |

**Request Mínimo:**
```json
{
  "requiredField": "value"
}
```

**Request Completo (exemplo de produção):**
```json
{
  "requiredField": "value",
  "optionalField": "value",
  "nestedObject": {
    "field": "value"
  }
}
```

**Response (200):**
```json
{
  "id": "uuid",
  "status": "created",
  "...": "..."
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `requiredField` | string | Sim | ... |
| `optionalField` | string | Não | ... |

**Observações:**
- Regras de negócio, validações ou efeitos colaterais importantes
- O que acontece internamente após esta chamada (eventos disparados, etc.)

### Etapa 2: {Próxima Ação}
(repetir padrão)

## Tratamento de Erros

| Status | Código de Erro | Causa | Resolução |
|--------|---------------|-------|-----------|
| 400 | VALIDATION_ERROR | Campo obrigatório ausente | Verificar campos obrigatórios |
| 409 | CONFLICT | Entidade duplicada | ... |
| ... | ... | ... | ... |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Descrição da armadilha | Como corrigir/evitar |
| 2 | ... | ... |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|-----------|
| Auth | Necessário antes deste fluxo | [FLOW_SIGNUP](../auth/FLOW_SIGNUP.md) |
| ... | ... | ... |
```

### 4.3 Template de Referência da API

Para arquivos `*_API_REFERENCE.md` (um por domínio):

```markdown
# Referência da API {Domínio}

## URLs Base

| Ambiente | Serviço | URL |
|----------|---------|-----|
| Produção | {Serviço} | https://{service}.w3block.io |
| Swagger | {Serviço} | https://{service}.w3block.io/docs |

## Autenticação

Todos os endpoints autenticados requerem:
```
Authorization: Bearer {accessToken}
```

## Enums

### {EnumName}
| Valor | Descrição |
|-------|-----------|
| `value1` | ... |

## Endpoints

(Agrupar por recurso, listar todos os endpoints com método, path, descrição, requisito de autenticação)
```

### 4.4 Template do Índice de Skills

Para `OFFPIX_SKILL_INDEX.md`:

```markdown
# Índice de Skills Offpix

## Documentos

| # | Documento | Versão | Descrição | Status | Quando usar |
|---|-----------|--------|-----------|--------|-------------|
| 1 | [NOME_DOC](./path) | 1.0.0 | ... | Status | ... |

## Guia Rápido

(Etapas numeradas para o caminho de implementação mais comum)

## Armadilhas Comuns

(Principais problemas em todos os fluxos)

## Tabela de Decisão

(Mapear casos de uso para documentos)

## Matriz: Endpoints x Documentos

(Mapear endpoints para skills usando marcações X)
```

---

## 5. Estratégia de Payloads: Exemplos Progressivos

Toda exemplo de endpoint DEVE incluir duas versões de payload:

### Mínimo (faz o trabalho)
- Apenas campos obrigatórios
- Valores válidos mais simples
- Anotar com: `// required` para cada campo

### Completo (nível de produção)
- Todos os campos que o frontend Offpix realmente envia
- Valores reais extraídos do codebase do frontend
- Anotar campos opcionais com: `// optional — {propósito}`

**Exemplo:**

```json
// Mínimo
{
  "name": "My Token",           // required
  "contractId": "uuid-here"     // required
}

// Completo (baseado no uso em produção do Offpix)
{
  "name": "My Token",           // required
  "contractId": "uuid-here",    // required
  "description": "A collectible token",  // optional — exibido na vitrine
  "imageUrl": "https://...",             // optional — miniatura do token
  "metadata": {                          // optional — metadados on-chain
    "attributes": []
  }
}
```

---

## 6. Convenções de Linguagem e Nomenclatura

### Idioma
- **Português (PT-BR)** para todo o texto da documentação
- **Termos de domínio W3Block preservados como estão:** `tenantId`, `companyId`, `tokenPass`, `deliverId`, `passShareCode`, `providersForSelection`, etc.
- NÃO traduza ou parafraseie identificadores específicos da W3Block

### Estrutura de Arquivos e Pastas
Cada módulo possui sua própria subpasta espelhando a organização de módulos do frontend:

```
skills/offpix/
├── OFFPIX_DOCUMENTATION_SPEC.md        ← este arquivo
├── OFFPIX_SKILL_INDEX.md               ← índice global
├── auth/
│   ├── AUTH_SKILL_INDEX.md
│   ├── AUTH_API_REFERENCE.md
│   └── FLOW_AUTH_*.md
├── commerce/
│   ├── COMMERCE_SKILL_INDEX.md
│   ├── COMMERCE_API_REFERENCE.md
│   └── FLOW_COMMERCE_*.md
├── tokens/
│   └── ...
└── (uma pasta por módulo)
```

### Nomenclatura de Arquivos
- Documentos de fluxo: `FLOW_{MODULE}_{FLOW_NAME}.md`
- Referências de API: `{MODULE}_API_REFERENCE.md`
- Índice do módulo: `{MODULE}_SKILL_INDEX.md`
- Índice global: `OFFPIX_SKILL_INDEX.md`

### Frontmatter
- `module: offpix` (sempre)
- `id:` corresponde ao nome do arquivo sem `.md`
- `version:` começa em `1.0.0`, segue SemVer

---

## 7. Checklist de Qualidade

Antes de marcar um documento como completo, verifique:

| # | Verificação | Descrição |
|---|-------------|-----------|
| 1 | **Endpoints verificados** | Cada path de endpoint testado contra o Swagger ou controller do backend |
| 2 | **Payloads validados** | Payload mínimo inclui todos os campos obrigatórios; payload completo corresponde ao uso do frontend |
| 3 | **Sequência correta** | Etapas do fluxo estão na ordem certa; dependências são explícitas |
| 4 | **Auth documentado** | Cada endpoint indica se auth é necessário e qual role/permissão |
| 5 | **Erros cobertos** | Respostas de erro comuns documentadas com causas e resoluções |
| 6 | **Sem duplicação do Swagger** | Documento agrega valor além do que o Swagger já mostra (sequência, regras de negócio, armadilhas) |
| 7 | **Referências cruzadas** | Fluxos relacionados linkados; pré-requisitos referenciam docs existentes |
| 8 | **Frontmatter completo** | Todos os campos YAML obrigatórios presentes e precisos |
| 9 | **Payloads progressivos** | Ambos os exemplos mínimo e completo presentes |
| 10 | **Armadilhas documentadas** | Pelo menos 2-3 erros comuns identificados a partir de validações do backend ou tratamento de erros do frontend |

---

## 8. Domínios Disponíveis

Referência de quais domínios podem ser documentados e seu mapeamento de fontes:

| Domínio | Serviço Backend | Swagger | Entidades-Chave |
|---------|----------------|---------|-----------------|
| **Auth & Identity** | ID Service | pixwayid.w3block.io/docs/ | Users, Tenants, Sessions, Roles |
| **Commerce** | Commerce Service | commerce.w3block.io/docs | Products, Orders, Prices, Discounts, Payments |
| **Tokens & NFTs** | Registry Service | api.w3block.io/docs/ | Contracts, Collections, Editions, Wallets |
| **Tokenization** | Registry Service | api.w3block.io/docs/ | Fluxos de tokenização dinâmica |
| **Pass & Benefits** | Pass Service | pass.w3block.io/docs | TokenPasses, Benefits, Operators, Addresses |
| **Loyalty** | Registry Service | api.w3block.io/docs/ | Loyalty contracts, transações ERC20 |
| **Contacts** | ID Service | pixwayid.w3block.io/docs/ | Contacts, Users, Imports |
| **KYC** | ID Service | pixwayid.w3block.io/docs/ | Documentos KYC, verificação |
| **PDF** | PDF Service | pdf.w3block.io/docs | Templates PDF, geração (apenas interno) |
| **Settings & Billing** | ID Service | pixwayid.w3block.io/docs/ | Plans, Balance, Billing, Emails |
| **Whitelist** | Registry Service | api.w3block.io/docs/ | Entradas de whitelist |
| **Configuration** | ID Service | pixwayid.w3block.io/docs/ | Tenant contexts, configuração de Menu |

---

## 9. Como Usar Esta Especificação

Quando o usuário solicitar documentação para um fluxo:

1. **Leia esta especificação** para carregar o processo e os templates
2. **Identifique o domínio** na tabela da Seção 8
3. **Siga o processo de pesquisa** da Seção 3 (backend → swagger → frontend)
4. **Escreva usando o template de output** da Seção 4
5. **Aplique payloads progressivos** conforme a Seção 5
6. **Valide contra o checklist de qualidade** da Seção 7
7. **Salve o arquivo** em `skills/offpix/` com a nomenclatura correta (Seção 6)
8. **Atualize `OFFPIX_SKILL_INDEX.md`** com a entrada do novo documento

**Exemplo de solicitação do usuário:**
> "Documente o fluxo de criação de coleção de tokens"

**Sua resposta:**
1. Domínio: Tokens & NFTs → Registry Service + `api.w3block.io/docs/`
2. Revisar o módulo backend para entidades de coleção, serviços, DTOs
3. Rastrear os hooks de token do frontend para a sequência de chamadas
4. Escrever `FLOW_OFFPIX_TOKENS_CREATE_COLLECTION.md`
5. Atualizar `OFFPIX_SKILL_INDEX.md`
