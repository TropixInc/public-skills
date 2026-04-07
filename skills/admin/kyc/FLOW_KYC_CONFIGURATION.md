---
id: FLOW_KYC_CONFIGURATION
title: "KYC - Configuração"
module: offpix/kyc
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-02"
authors:
  - rafaelmhp
tags:
  - kyc
  - configuration
  - tenant-context
  - notifications
  - input-types
depends_on:
  - KYC_API_REFERENCE
---

# Configuração KYC

## Visão Geral

Este fluxo cobre como configurar contextos KYC para um tenant, incluindo fluxos de aprovação, configurações de notificação e o catálogo completo de tipos de input. A configuração determina como as submissões são processadas, quem pode aprová-las e quais notificações são enviadas ao longo do ciclo de vida.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de autenticação / configuração do ambiente |
| Contexto criado | Um contexto KYC deve existir | [Ciclo de Vida do Contexto](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Inputs definidos | Campos de formulário devem ser criados | [Gerenciamento de Inputs](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |

---

## Fluxo: Visualizar Configurações KYC

Recupera todas as configurações de tenant context para ver as configurações atuais de aprovação e notificação.

**Endpoint:**

| Método | Caminho | Autenticação |
|--------|---------|--------------|
| GET | `/tenant-context/{tenantId}` | Bearer (Admin) |

**Resposta (200):**
```json
[
  {
    "id": "tc-uuid-1",
    "tenantId": "tenant-uuid",
    "contextId": "ctx-uuid-signup",
    "active": true,
    "data": null,
    "approverWhitelistIds": null,
    "requireSpecificApprover": false,
    "autoApprove": false,
    "blockCommerceDeliver": false,
    "notificationsConfig": {
      "KYC_APPROVAL_REQUEST": { "enabled": true },
      "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
      "KYC_USER_APPROVED": { "enabled": true },
      "KYC_ADMIN_APPROVED": { "enabled": true },
      "KYC_USER_REJECTED": { "enabled": true },
      "KYC_ADMIN_REJECTED": { "enabled": true }
    }
  },
  {
    "id": "tc-uuid-2",
    "tenantId": "tenant-uuid",
    "contextId": "ctx-uuid-onboarding",
    "active": true,
    "data": null,
    "approverWhitelistIds": ["whitelist-uuid-1", "whitelist-uuid-2"],
    "requireSpecificApprover": true,
    "autoApprove": false,
    "blockCommerceDeliver": true,
    "notificationsConfig": {
      "KYC_APPROVAL_REQUEST": { "enabled": true },
      "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
      "KYC_USER_APPROVED": { "enabled": true },
      "KYC_ADMIN_APPROVED": { "enabled": false },
      "KYC_USER_REJECTED": { "enabled": true },
      "KYC_ADMIN_REJECTED": { "enabled": false }
    }
  }
]
```

---

## Fluxo: Configurar Fluxo de Aprovação

O TenantContextEntity controla como as submissões KYC são processadas. As principais opções de configuração são:

### autoApprove

| Valor | Comportamento |
|-------|---------------|
| `true` | Submissões são automaticamente aprovadas na criação. Nenhuma revisão manual é necessária. Todos os documentos e o user context são definidos como `APPROVED` imediatamente. Emails de aprovação são enviados automaticamente. |
| `false` | Submissões requerem revisão manual por um Admin ou KycApprover. O status transiciona para `CREATED` e permanece até que um aprovador tome ação. |

**Caso de uso:** Habilite aprovação automática para contextos de baixo risco (ex.: cadastro em newsletter) onde a verificação de documentos não é crítica. Desabilite para contextos de alto risco (ex.: verificação de identidade para transações financeiras).

### requireSpecificApprover

| Valor | Comportamento |
|-------|---------------|
| `true` | Usuários devem especificar um `approverUserId` ao enviar documentos. Apenas esse aprovador específico pode revisar e agir na submissão. |
| `false` | Qualquer Admin ou KycApprover autorizado pode revisar a submissão. |

**Caso de uso:** Habilite quando submissões devem ser direcionadas a uma pessoa específica (ex.: um gerente regional que conhece o candidato, ou um oficial de compliance designado).

**Impacto na submissão:** Quando habilitado, o campo `approverUserId` se torna obrigatório no DTO `AttachDocumentsToUser`. Submissões sem ele falharão.

### approverWhitelistIds

| Valor | Comportamento |
|-------|---------------|
| `null` ou vazio | Qualquer KycApprover pode revisar submissões deste contexto |
| `UUID[]` | Apenas KycApprovers que são membros de uma das whitelists especificadas podem revisar. Admins e SuperAdmins não são afetados por esta restrição. |

**Caso de uso:** Restrinja a aprovação a uma equipe específica. Por exemplo, apenas membros da equipe de compliance (na whitelist `compliance-team-uuid`) podem aprovar contextos de verificação de identidade.

**Exemplo de configuração:**
```json
{
  "approverWhitelistIds": ["whitelist-uuid-compliance", "whitelist-uuid-senior-reviewers"]
}
```

Com esta configuração, um KycApprover deve ser membro da whitelist `compliance` ou `senior-reviewers` para ver e agir nas submissões.

### blockCommerceDeliver

| Valor | Comportamento |
|-------|---------------|
| `true` | A entrega de pedidos de commerce é bloqueada até que o KYC do usuário seja aprovado para este contexto. Quando o KYC é aprovado, o serviço de commerce é notificado e os pedidos pendentes são desbloqueados. |
| `false` | Pedidos de commerce não são afetados pelo status do KYC. |

**Caso de uso:** Habilite para contextos onde a entrega de produtos requer identidade verificada (ex.: produtos com restrição de idade, bens regulamentados).

### Combinações de Configuração

| autoApprove | requireSpecific | whitelistIds | Resultado |
|:-----------:|:---------------:|:------------:|-----------|
| `true` | `false` | `null` | Aprovação instantânea, sem revisão |
| `false` | `false` | `null` | Qualquer admin/aprovador pode revisar |
| `false` | `true` | `null` | Apenas o aprovador atribuído pode revisar |
| `false` | `false` | `[ids]` | Apenas aprovadores na whitelist podem revisar |
| `false` | `true` | `[ids]` | Apenas o aprovador atribuído, que deve estar na whitelist |
| `true` | `true` | qualquer | Aprovação automática prevalece; sem revisão manual |

---

## Fluxo: Configurar Notificações

O campo `notificationsConfig` no TenantContextEntity controla quais notificações por email são enviadas durante o ciclo de vida do KYC.

### Tipos de Notificação

#### KYC_APPROVAL_REQUEST

| Campo | Valor |
|-------|-------|
| **Destinatário** | Aprovadores (Admin, KycApprover) |
| **Gatilho** | Nova submissão criada com status `CREATED` |
| **Conteúdo** | Notificação de que uma nova submissão KYC está pronta para revisão |
| **Caso de uso** | Manter aprovadores informados sobre trabalho pendente |

#### KYC_REQUIRE_USER_REVIEW

| Campo | Valor |
|-------|-------|
| **Destinatário** | Usuário que enviou |
| **Gatilho** | Aprovador solicita revisão via endpoint `require-review` |
| **Conteúdo** | Notificação especificando quais campos precisam de reenvio e o motivo |
| **Caso de uso** | Solicitar ao usuário que corrija e reenvie documentos específicos |

#### KYC_USER_APPROVED

| Campo | Valor |
|-------|-------|
| **Destinatário** | Usuário que enviou |
| **Gatilho** | Submissão aprovada |
| **Conteúdo** | Confirmação de que a verificação KYC está completa |
| **Caso de uso** | Informar o usuário que sua identidade foi verificada |

#### KYC_ADMIN_APPROVED

| Campo | Valor |
|-------|-------|
| **Destinatário** | Admins |
| **Gatilho** | Submissão aprovada |
| **Conteúdo** | Notificação de que uma submissão foi aprovada (inclui qual admin aprovou) |
| **Caso de uso** | Trilha de auditoria para ciência da equipe admin |

#### KYC_USER_REJECTED

| Campo | Valor |
|-------|-------|
| **Destinatário** | Usuário que enviou |
| **Gatilho** | Submissão rejeitada |
| **Conteúdo** | Notificação de rejeição com motivo |
| **Caso de uso** | Informar o usuário que sua submissão foi negada |

#### KYC_ADMIN_REJECTED

| Campo | Valor |
|-------|-------|
| **Destinatário** | Admins |
| **Gatilho** | Submissão rejeitada |
| **Conteúdo** | Notificação de que uma submissão foi rejeitada (inclui qual admin rejeitou e o motivo) |
| **Caso de uso** | Trilha de auditoria para ciência da equipe admin |

### Exemplo de Configuração de Notificações

**Habilitar todas as notificações:**
```json
{
  "notificationsConfig": {
    "KYC_APPROVAL_REQUEST": { "enabled": true },
    "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
    "KYC_USER_APPROVED": { "enabled": true },
    "KYC_ADMIN_APPROVED": { "enabled": true },
    "KYC_USER_REJECTED": { "enabled": true },
    "KYC_ADMIN_REJECTED": { "enabled": true }
  }
}
```

**Notificações apenas para o usuário (desabilitar notificações de admin):**
```json
{
  "notificationsConfig": {
    "KYC_APPROVAL_REQUEST": { "enabled": false },
    "KYC_REQUIRE_USER_REVIEW": { "enabled": true },
    "KYC_USER_APPROVED": { "enabled": true },
    "KYC_ADMIN_APPROVED": { "enabled": false },
    "KYC_USER_REJECTED": { "enabled": true },
    "KYC_ADMIN_REJECTED": { "enabled": false }
  }
}
```

---

## Catálogo de Tipos de Input

Referência completa para todos os 19 tipos de input `DataTypesEnum` suportados, com formato de armazenamento, regras de validação, valores de exemplo e dicas de renderização no frontend.

### TEXT

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` |
| **Validação** | Nenhuma |
| **Valor de exemplo** | `"John Doe"` |
| **Frontend** | Input de texto padrão (`<input type="text">`) |

### EMAIL

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` |
| **Validação** | Formato de email RFC 5322 |
| **Valor de exemplo** | `"john@example.com"` |
| **Frontend** | Input de email (`<input type="email">`) |

### PHONE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (número único) ou `complexValue` (com `extraPhones`) |
| **Validação** | Formato de número de telefone |
| **Valor de exemplo (simples)** | `"+5511999999999"` |
| **Valor de exemplo (com extras)** | `{ "phone": "+5511999999999", "extraPhones": ["+5511888888888"] }` |
| **Frontend** | Input de telefone com seletor de código de país; botão opcional "adicionar telefone" para extras |

### URL

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` |
| **Validação** | URL HTTP ou HTTPS válida |
| **Valor de exemplo** | `"https://example.com/profile"` |
| **Frontend** | Input de URL (`<input type="url">`) |

### CPF

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (apenas dígitos, 11 caracteres) |
| **Validação** | Exatamente 11 dígitos; validação de checksum mod-11; único por tenant |
| **Sanitização** | Caracteres não numéricos removidos antes do armazenamento |
| **Valor de exemplo** | `"12345678901"` |
| **Frontend** | Input com máscara no formato CPF (`###.###.###-##`); validação de checksum em tempo real |

### BIRTHDATE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (string de data ISO) |
| **Validação** | Data válida; verificação de idade mínima (padrão: 18 anos) |
| **Valor de exemplo** | `"1990-05-15"` |
| **Frontend** | Seletor de data com data máxima definida como 18 anos atrás |

### DATE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (string de data ISO) |
| **Validação** | Data ISO válida |
| **Valor de exemplo** | `"2026-01-15"` |
| **Frontend** | Seletor de data (`<input type="date">`) |

### USER_NAME

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` |
| **Validação** | Nenhuma |
| **Efeito colateral** | Atualiza automaticamente o campo `name` do usuário em seu perfil |
| **Valor de exemplo** | `"John Doe"` |
| **Frontend** | Input de texto; pode pré-preencher a partir do perfil do usuário |

### CHECKBOX

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (boolean) |
| **Validação** | Valor booleano |
| **Valor de exemplo** | `true` |
| **Frontend** | Input de checkbox; label da definição do input |

### SIMPLE_SELECT

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` (valor da opção selecionada) |
| **Validação** | Deve corresponder a um dos `options[].value`; se `isMultiple`, cada item do array deve corresponder |
| **Valor de exemplo (único)** | `"BR"` |
| **Valor de exemplo (múltiplo)** | `["BR", "US"]` |
| **Frontend** | Dropdown ou botões de rádio; use multi-select se `isMultiple` for true |

**Formato das opções:**
```json
{
  "options": [
    { "label": "Brazil", "value": "BR" },
    { "label": "United States", "value": "US" },
    { "label": "Portugal", "value": "PT" }
  ]
}
```

### DYNAMIC_SELECT

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `simpleValue` |
| **Validação** | Opções buscadas do servidor no momento da renderização |
| **Valor de exemplo** | `"option-uuid-1"` |
| **Frontend** | Dropdown com opções carregadas de um endpoint da API |

### FILE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `assetId` (referência ao upload do Cloudinary) |
| **Validação** | O asset deve existir |
| **Formato de submissão** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | Widget de upload de arquivo; mostrar nome do arquivo após upload; preview para tipos suportados |

### IMAGE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `assetId` (referência ao upload do Cloudinary) |
| **Validação** | O asset deve existir; deve ser um tipo MIME de imagem |
| **Formato de submissão** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | Widget de upload de imagem com preview; suporte a arrastar e soltar |

### IDENTIFICATION_DOCUMENT

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `complexValue` + `assetId` opcional |
| **Validação** | `docType` obrigatório (um de: `passport`, `cpf`, `rg`); número do `document` obrigatório |
| **Formato de submissão** | `{ "inputId": "...", "value": { "docType": "rg", "document": "123456789" }, "assetId": "asset-uuid" }` |
| **Frontend** | Seletor de tipo de documento + input de número do documento + upload de arquivo para foto do documento |

**Formato do complexValue:**
```json
{
  "docType": "rg",
  "document": "123456789"
}
```

Valores de `docType` suportados:
| docType | Descrição |
|---------|-----------|
| `passport` | Passaporte internacional |
| `cpf` | Cartão CPF brasileiro |
| `rg` | RG brasileiro (carteira de identidade) |

### MULTIFACE_SELFIE

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `assetId` (referência ao upload do Cloudinary) |
| **Validação** | O asset deve existir; CPF e data de nascimento devem ter sido enviados antes |
| **Processamento** | Aciona chamada ao serviço de validação biométrica |
| **Formato de submissão** | `{ "inputId": "...", "assetId": "asset-uuid" }` |
| **Frontend** | Widget de captura por câmera; mostrar preview; indicar status do processamento biométrico |

**Pré-requisitos:** O usuário deve ter enviado previamente documentos do tipo `CPF` e `BIRTHDATE` (em qualquer contexto dentro do mesmo tenant) antes que a selfie multiface possa ser processada. O serviço biométrico usa o CPF e a data de nascimento para correspondência de identidade.

### SIMPLE_LOCATION

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `complexValue` |
| **Validação** | `placeId`, `region`, `country` são obrigatórios |
| **Formato de submissão** | `{ "inputId": "...", "value": { "placeId": "...", "region": "SP", "country": "BR", ... } }` |
| **Frontend** | Autocomplete de endereço (Google Places API); fallback manual para região/país |

**Formato do complexValue:**
```json
{
  "placeId": "ChIJrTLr-GyuEmsRBfy61i59si0",
  "region": "SP",
  "country": "BR",
  "city": "Sao Paulo",
  "postal_code": "01000-000",
  "street_address_1": "Av Paulista 1000",
  "street_address_2": "Suite 500",
  "latitude": -23.5505,
  "longitude": -46.6333
}
```

| Campo | Obrigatório | Descrição |
|-------|-------------|-----------|
| `placeId` | Sim | ID do Google Places |
| `region` | Sim | Código do estado/região |
| `country` | Sim | Código do país (ISO 3166-1 alpha-2) |
| `city` | Não | Nome da cidade |
| `postal_code` | Não | CEP/código postal |
| `street_address_1` | Não | Endereço linha 1 |
| `street_address_2` | Não | Endereço linha 2 |
| `latitude` | Não | Coordenada de latitude |
| `longitude` | Não | Coordenada de longitude |

### COMMERCE_PRODUCT

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | `complexValue` |
| **Validação** | `productId` deve referenciar um produto válido no serviço de commerce |
| **Formato de submissão** | `{ "inputId": "...", "value": { "productId": "uuid", "quantity": "1" } }` |
| **Frontend** | Seletor de produto com input de quantidade; pode incluir seleção de variante |

**Formato do complexValue:**
```json
{
  "productId": "product-uuid",
  "quantity": "1",
  "variantIds": ["variant-uuid-1", "variant-uuid-2"]
}
```

### SEPARATOR

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | Não armazenado (ignorado durante o processamento) |
| **Validação** | Nenhuma |
| **Formato de submissão** | Não enviado |
| **Frontend** | Divisor visual ou cabeçalho de seção usando o `label` e `description` do input |

### IFRAME

| Propriedade | Valor |
|-------------|-------|
| **Armazenamento** | Não armazenado (apenas exibição) |
| **Validação** | Nenhuma |
| **Formato de submissão** | Não enviado |
| **Frontend** | Renderizar um elemento `<iframe>`; URL normalmente fornecida no campo `data` ou `description` do input |

---

## Pontos de Integração

### Integração com Commerce

Quando `blockCommerceDeliver: true` está definido em um TenantContext:

1. Quando um usuário faz um pedido de commerce, o sistema verifica se o KYC do usuário está aprovado para o contexto relevante
2. Se o KYC não está aprovado, a entrega do pedido é bloqueada (pedido é criado mas retido)
3. Quando o KYC é subsequentemente aprovado, o serviço de commerce recebe um evento de notificação
4. O serviço de commerce desbloqueia e processa a entrega pendente

### Integração com Webhooks

A plataforma dispara um evento de webhook `USER_KYC_PROPS_UPDATED` quando:
- O status do KYC de um usuário muda (created, approved, denied, required review)
- Documentos são enviados ou atualizados

Este webhook pode ser consumido por sistemas externos para rastreamento de status KYC em tempo real.

### Integração com Email

As notificações por email são configuráveis por tipo de notificação em cada TenantContext. Os emails usam os templates de email do tenant e respeitam a preferência `i18nLocale` do usuário para localização.

### Serviço Biométrico Multiface

O tipo de input multiface selfie integra com um serviço externo de validação biométrica:

1. Usuário envia uma foto de selfie (upload para o Cloudinary)
2. O sistema recupera o CPF e a data de nascimento do usuário a partir de documentos previamente enviados
3. O sistema chama o serviço biométrico com a selfie, CPF e data de nascimento
4. O serviço biométrico realiza reconhecimento facial e correspondência de identidade
5. Se a validação passa, o documento é aceito
6. Se a validação falha, `ItWasNotPossibleRegisterMutiplaceExpection` é lançada

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 400 | `TenantContextDisabledException` | Contexto está inativo | Ative o contexto antes que os usuários possam enviar |
| 403 | `ForbiddenException` | Permissões insuficientes para visualizar/modificar configuração | Requer role Admin |
| 404 | `NotFoundException` | Tenant context não encontrado | Verifique `tenantId` e `contextId` |

---

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|---------|
| 1 | Aprovação automática habilitada mas deseja revisão manual | Defina `autoApprove: false` no TenantContext. Submissões já aprovadas automaticamente não são afetadas |
| 2 | KycApprover não consegue ver submissões | Certifique-se de que o aprovador é membro de uma das whitelists em `approverWhitelistIds`, ou remova a restrição de whitelist |
| 3 | Notificações não estão sendo enviadas | Verifique `notificationsConfig` para o tipo específico de notificação. Cada tipo deve ter `{ "enabled": true }` |
| 4 | Pedidos de commerce não estão sendo bloqueados | Defina `blockCommerceDeliver: true` no TenantContext. Afeta apenas pedidos feitos após a alteração da configuração |
| 5 | Validação multiface falhando | Certifique-se de que os usuários enviem documentos de CPF e data de nascimento antes da selfie. O serviço biométrico requer esses campos |
| 6 | Tipo de input não armazena valor | Os tipos `SEPARATOR` e `IFRAME` são apenas para exibição e não armazenam valores. Não envie documentos para esses tipos |
| 7 | requireSpecificApprover mas nenhum aprovador atribuído | Quando `requireSpecificApprover: true`, o endpoint de submissão requer `approverUserId` no corpo da requisição |

---

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Submissão KYC | Como usuários enviam documentos | [FLOW_KYC_SUBMISSION](./FLOW_KYC_SUBMISSION.md) |
| Aprovação KYC | Como admins revisam submissões | [FLOW_KYC_APPROVAL](./FLOW_KYC_APPROVAL.md) |
| Ciclo de Vida do Contexto | Criação e gerenciamento de contextos | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Gerenciamento de Inputs | Criação e gerenciamento de campos de formulário | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](../configurations/FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Gerenciamento de Whitelist | Gerenciamento de whitelists de aprovadores | [WHITELIST_SKILL_INDEX](../whitelist/WHITELIST_SKILL_INDEX.md) |
| Referência da API | Detalhes completos de endpoints e DTOs | [KYC_API_REFERENCE](./KYC_API_REFERENCE.md) |
