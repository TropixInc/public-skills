---
id: FLOW_CONFIGURATIONS_SIGNUP_ACTIVATION
title: "Configurações - Ativação / Desativação de Cadastro"
module: offpix/configurations
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-31"
authors:
  - rafaelmhp
tags:
  - configurations
  - signup
  - activation
  - kyc
depends_on:
  - CONFIGURATIONS_API_REFERENCE
  - FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE
---

# Ativação / Desativação de Cadastro

## Visão Geral

Este fluxo cobre a habilitação e desabilitação do formulário de registro de cadastro para um tenant. Quando ativado, novos usuários podem se registrar através de um formulário de cadastro habilitado com KYC. Quando desativado, o formulário de cadastro fica oculto. No frontend, isso é um interruptor toggle na página de **Configuração KYC** (`/dash/kyc`), com o rótulo "Registration".

## Pré-requisitos

| Requisito | Descrição | Como obter |
|-----------|-----------|------------|
| Bearer token | JWT com role Admin | [Fluxo de Sign-In](../auth/FLOW_AUTH_SIGNIN.md) |
| `tenantId` | UUID do Tenant | Fluxo de auth / configuração do ambiente |

## Fluxo: Verificar Estado Atual

### Etapa 1: Buscar Tenant Contexts

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenants/{tenantId}/tenant-context` | Bearer (Admin) |

**Query (opcional):** `?type=user_properties`

Verifique na resposta se existe um context com `context.slug === 'signup'` (o frontend faz a correspondência por `context.description.toUpperCase() === 'SIGNUP'`).

- Se encontrado com `active: true` → Cadastro está LIGADO
- Se encontrado com `active: false` → Cadastro está DESLIGADO
- Se não encontrado → Cadastro nunca foi configurado

---

## Fluxo: Ativar Cadastro

### Etapa 1: Chamar Endpoint de Ativação

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenant-contexts/activate-signup/{tenantId}` | Bearer (Admin) |

Nenhum corpo de requisição é necessário. Esta é uma requisição GET que dispara a ativação.

**Resposta (200):** Status HTTP 200 indica sucesso.

**Observações:**
- Este endpoint cria o context de cadastro se ele não existir, ou o ativa se foi desativado anteriormente
- O frontend trata HTTP 200 como sucesso e atualiza a interface do toggle adequadamente

---

## Fluxo: Desativar Cadastro

### Etapa 1: Chamar Endpoint de Desativação

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/tenant-contexts/deactivate-signup/{tenantId}` | Bearer (Admin) |

Nenhum corpo de requisição é necessário.

**Resposta (200):** Status HTTP 200 indica sucesso.

**Observações:**
- Desativar o cadastro oculta o formulário de registro para usuários finais
- Usuários já registrados não são afetados
- O context de cadastro e seus inputs são preservados — reativar restaura a configuração anterior

---

## Fluxo: Configurar o Conteúdo do Formulário de Cadastro

Após ativar o cadastro, configure a aparência e os campos do formulário:

### Etapa 1: Criar ou Atualizar o Context de Cadastro

Se o context de cadastro ainda não existir, crie-o:

```
POST /tenants/{tenantId}/admin/tenant-context
```

```json
{
  "slug": "signup",
  "description": "SignUp Form",
  "type": "user_properties",
  "maxSubmissions": 1,
  "isPreOrder": false,
  "active": true,
  "data": {
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": false,
      "position": "center",
      "textContainer": {
        "mainText": "Create Your Account",
        "subtitleText": "Fill in the details below"
      }
    }
  }
}
```

Se já existir, atualize-o:

```
PATCH /tenants/{tenantId}/tenant-context/{tenantContextId}
```

```json
{
  "active": true,
  "data": {
    "screenConfig": {
      "postKycUrl": "https://myapp.com/welcome",
      "skipConfirmation": true,
      "position": "center",
      "textContainer": {
        "mainText": "Welcome!",
        "subtitleText": "Get started in minutes"
      }
    }
  }
}
```

### Etapa 2: Adicionar Campos ao Formulário

Adicione inputs ao context de cadastro. Veja [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md).

Campos típicos do formulário de cadastro:

| Ordem | Tipo | Rótulo | Obrigatório |
|-------|------|--------|-------------|
| 1 | `user_name` | Full Name | Sim |
| 2 | `email` | Email Address | Sim |
| 3 | `cpf` | CPF | Sim (Brasil) |
| 4 | `phone` | Phone Number | Não |
| 5 | `identification_document` | Government ID | Sim |
| 6 | `multiface_selfie` | Selfie | Sim |

---

## Buscar Tipos de Documento Habilitados

A página de configuração KYC também exibe quais tipos de documento estão habilitados para o tenant.

**Endpoint:**

| Método | Caminho | Auth |
|--------|---------|------|
| GET | `/data-types/{tenantId}` | Bearer (Admin) |

**Resposta (200):**
```json
[
  {
    "id": "dtype-uuid",
    "tenantId": "tenant-uuid",
    "label": "ID Document",
    "type": "document",
    "description": "Government ID document",
    "enabled": true,
    "createdAt": "2026-01-01T00:00:00Z",
    "updatedAt": "2026-01-01T00:00:00Z"
  }
]
```

O frontend filtra esta lista para mostrar apenas itens com `enabled: true`, exibindo o campo `label`.

---

## Tratamento de Erros

| Status | Erro | Causa | Resolução |
|--------|------|-------|-----------|
| 401 | Unauthorized | Token inválido ou ausente | Reautenticar |
| 403 | Forbidden | Usuário não possui role Admin | Use uma conta Admin |

## Armadilhas Comuns

| # | Problema | Solução |
|---|---------|----------|
| 1 | Toggle de cadastro não persiste | Verifique se o endpoint de ativar/desativar retorna 200. O frontend só atualiza o estado da interface em caso de sucesso |
| 2 | Formulário de cadastro aparece mas não tem campos | A ativação apenas habilita o formulário — você ainda precisa criar inputs (campos) para ele |
| 3 | Formulário desativado perde configuração | Não — a desativação preserva todos os dados do context e inputs. A reativação restaura tudo |

## Fluxos Relacionados

| Fluxo | Relacionamento | Documento |
|-------|---------------|----------|
| Ciclo de Vida do Context | Gerenciamento completo de context | [FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE](./FLOW_CONFIGURATIONS_CONTEXT_LIFECYCLE.md) |
| Gerenciamento de Inputs | Adicionar campos ao formulário de cadastro | [FLOW_CONFIGURATIONS_INPUT_MANAGEMENT](./FLOW_CONFIGURATIONS_INPUT_MANAGEMENT.md) |
| Auth Signup | O fluxo de cadastro voltado ao usuário | [FLOW_AUTH_SIGNUP](../auth/FLOW_AUTH_SIGNUP.md) |
