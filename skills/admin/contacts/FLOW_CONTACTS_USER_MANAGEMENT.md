---
id: FLOW_CONTACTS_USER_MANAGEMENT
title: "Contatos - Gerenciamento de Usuários"
module: offpix/contacts
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-04-01"
authors:
  - rafaelmhp
tags:
  - contacts
  - users
  - invite
  - roles
  - royalty
depends_on:
  - AUTH_API_REFERENCE
  - CONTACTS_API_REFERENCE
---

# Gerenciamento de Usuários (Contatos)

## Visão Geral

Este fluxo cobre o gerenciamento de usuários dentro de um tenant: listar contatos, convidar novos usuários, editar perfis, gerenciar roles e alternar elegibilidade para royalties. No frontend, esta é a seção **Contacts** (`/dash/contacts/`) com subseções para Clientes, Equipe e Parceiros.

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de auth / configuração do ambiente |

## Entidades

```
User
├── name, email, phone
├── roles (UserRoleEnum[])
├── verified (boolean, computado a partir de verifiedEmailAt)
├── mainWallet (endereço vault ou metamask)
├── wallets[] (carteiras adicionais)
└── RoyaltyEligible (vínculo opcional)
```

**Tipos de contato** no frontend são determinados pela role:
- **Clientes** (`/dash/contacts/clients`): Usuários com role `user`
- **Equipe** (`/dash/contacts/team`): Usuários com roles `admin`, `operator`, `loyaltyOperator`, `commerceOrderReceiver`
- **Parceiros** (`/dash/contacts/partiners`): Associados/parceiros

---

## Fluxo A: Listar Contatos

### Etapa 1: Buscar Usuários

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{tenantId}/users` | Bearer (Admin) |

**Requisição Mínima (clientes):**
```
GET /{tenantId}/users?role=user&sortBy=createdAt&orderBy=DESC&limit=10&page=1
```

**Membros da equipe:**
```
GET /{tenantId}/users?role=admin&role=operator&role=loyaltyOperator&role=commerceOrderReceiver&sortBy=createdAt&orderBy=DESC
```

**Resposta (200):**
```json
{
  "items": [
    {
      "id": "user-uuid",
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+5511999999999",
      "roles": ["user"],
      "verified": true,
      "mainWallet": {
        "id": "wallet-uuid",
        "address": "0x1234...abcd",
        "type": "vault"
      },
      "createdAt": "2026-01-01T00:00:00Z"
    }
  ],
  "meta": {
    "totalItems": 100,
    "totalPages": 10,
    "currentPage": 1,
    "itemsPerPage": 10
  }
}
```

**Mapeamento de exibição no frontend:**

| Cabeçalho da Coluna | Campo da API | Observações |
|---------------------|-------------|-------------|
| Email | `email` | Identificador principal exibido |
| Endereço da Carteira | `mainWallet.address` | Truncado na interface |
| Data de Criação | `createdAt` | Data formatada |
| Status | Computado | `verified` + presença de `mainWallet` determina: ativo, verificado, inativo, pendente |
| Status KYC | `kycStatus` | Exibido apenas se KYC está habilitado para o tenant |

---

## Fluxo B: Convidar um Novo Contato

### Etapa 1: Convidar Usuário

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/users/invite` | Bearer (Admin) | application/json |

**Requisição Mínima:**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "tenantId": "tenant-uuid",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

**Requisição Completa (com elegibilidade para royalties):**
```json
{
  "name": "Jane Doe",
  "email": "jane@example.com",
  "role": "user",
  "royaltyEligible": true,
  "tenantId": "tenant-uuid",
  "i18nLocale": "pt-br",
  "generateRandomPassword": true,
  "sendEmail": true
}
```

**Resposta (201):**
```json
{
  "id": "new-user-uuid",
  "email": "jane@example.com",
  "name": "Jane Doe",
  "roles": ["user"],
  "tenantId": "tenant-uuid"
}
```

**Referência de Campos:**

| Campo | Tipo | Obrigatório | Padrão | Descrição |
|-------|------|-------------|--------|-----------|
| `name` | string | Sim | — | Nome completo (frontend: "Full Name") |
| `email` | string | Sim | — | Endereço de e-mail (deve ser único por tenant) |
| `role` | UserRoleEnum | Sim | — | Role a ser atribuída (frontend: "Type") |
| `tenantId` | UUID | Sim | — | Tenant de destino |
| `i18nLocale` | string | Não | — | Idioma para o e-mail de convite (`pt-br`, `en`) |
| `generateRandomPassword` | boolean | Sim | — | Sempre `true` |
| `sendEmail` | boolean | Sim | — | Sempre `true` |

**Observações:**
- O usuário convidado recebe um e-mail com uma senha temporária
- Se `royaltyEligible` for true no frontend, uma chamada subsequente é feita para criar o registro de royalty (Etapa 2)
- O e-mail deve ser único por tenant — retorna 409 se já registrado

### Etapa 2 (Opcional): Criar Registro de Elegibilidade para Royalties

Se o toggle "Have Royalties" estiver habilitado no frontend:

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/contracts/royalty-eligible/create` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "active": true,
  "displayName": "Jane Doe",
  "userId": "new-user-uuid"
}
```

**Observação:** Este endpoint está no **backend Registry** (`api.w3block.io`), não no PixwayID.

---

## Fluxo C: Editar um Contato

### Etapa 1: Buscar Perfil Atual

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/{tenantId}/users/{userId}` | Bearer (Admin) |

### Etapa 2: Atualizar Perfil

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/{tenantId}/users/{userId}` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "name": "Updated Name",
  "phone": "+5511888888888",
  "roles": ["user", "operator"]
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `name` | string | Não | Nome completo atualizado |
| `phone` | string | Não | Número de telefone atualizado |
| `roles` | UserRoleEnum[] | Não | Lista de roles atualizada (frontend: multi-select "Type of User") |

**Campos editáveis no frontend:**

| Rótulo no Frontend | Campo da API | Editável | Observações |
|-------------------|-------------|----------|-------------|
| Full Name | `name` | Sim | Campo de texto |
| Phone | `phone` | Sim | Campo de telefone (pode vir de documentos KYC) |
| Type of User | `roles` | Sim | Dropdown multi-select |
| Email | `email` | **Não** | Somente leitura |
| Wallet Address | `mainWallet.address` | **Não** | Somente leitura |
| Have Royalties | — | Sim | Endpoint separado (royalty eligible) |

### Etapa 3 (Opcional): Alternar Elegibilidade para Royalties

**Para habilitar:** POST `/{companyId}/contracts/royalty-eligible/create`
**Para atualizar:** PATCH `/{companyId}/contracts/royalty-eligible` com `{ "active": true/false, "userId": "..." }`

### Etapa 4 (Opcional): Editar Referência

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| PATCH | `/users/{companyId}/{userId}/referrer` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "referrerCode": "ABC123",
  "referrerUserId": "referrer-uuid"
}
```

---

## Fluxo D: Buscar Usuário por CPF ou Carteira

### Por CPF

```
GET /{tenantId}/users/documents/find-user-by-any?cpf=12345678901
```

O CPF é buscado nos documentos KYC enviados. Deve ter 11 dígitos com checksum válido.

### Por Endereço de Carteira

```
GET /{tenantId}/users/wallets/by-address/0x1234...abcd
```

---

## Fluxo E: Contatos Externos (Somente Carteira)

Contatos externos são endereços de carteira sem uma conta de usuário. Usados para destinatários de royalties.

### Criar Contato Externo

**Endpoint:**

| Método | Caminho | Auth | Content-Type |
|--------|---------|------|-------------|
| POST | `/{companyId}/external-contacts/import` | Bearer (Admin) | application/json |

**Requisição:**
```json
{
  "name": "External Partner",
  "walletAddress": "0xabcd...1234",
  "email": "partner@example.com",
  "phone": "+5511999999999",
  "royaltyEligible": true,
  "description": "Royalty partner for collection X"
}
```

**Observação:** Este endpoint está no **backend Registry**. Contatos externos não possuem contas de usuário e não podem fazer login — são rastreados apenas para fins de distribuição de royalties/tokens.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 409 | Conflict | E-mail já registrado para este tenant | Use o usuário existente ou um e-mail diferente |
| 404 | NotFoundException | Usuário não encontrado | Verifique o userId |
| 403 | ForbiddenException | Permissões insuficientes | Requer role Admin |
| 400 | InvalidCPFException | Checksum do CPF inválido | Verifique o número do CPF |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Convite retorna 409 | E-mail já está registrado no tenant. Busque o usuário existente em vez disso |
| 2 | Número de telefone do KYC vs perfil | O telefone do usuário pode existir tanto no perfil (`user.phone`) quanto nos documentos KYC. O frontend lê dos documentos KYC (tipo de input `phone`) |
| 3 | Roles devem ser um array | Mesmo para uma única role, envie como `["user"]` e não `"user"` ao editar |
| 4 | Royalty eligible é separado do convite | Convidar um usuário NÃO cria automaticamente elegibilidade para royalties. Requer uma segunda chamada de API ao backend Registry |
| 5 | Sem endpoint de exclusão de usuário | Usuários não podem ser excluídos — só podem ser desativados ou ter roles alteradas |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Auth | Necessário antes de qualquer operação | [FLOW_AUTH_SIGNIN](../auth/FLOW_AUTH_SIGNIN.md) |
| Submissão KYC | Usuário envia documentos KYC | [FLOW_CONTACTS_KYC_SUBMISSION](./FLOW_CONTACTS_KYC_SUBMISSION.md) |
| Aprovação KYC | Admin revisa submissões KYC | [FLOW_CONTACTS_KYC_APPROVAL](./FLOW_CONTACTS_KYC_APPROVAL.md) |
| Configurações | Formulários KYC são definidos aqui | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](../configurations/FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
