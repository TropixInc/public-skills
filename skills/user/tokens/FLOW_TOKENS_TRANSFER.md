---
id: FLOW_TOKENS_TRANSFER
title: "Tokens - Transferencia"
module: tokens
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - tokens
  - transfer
depends_on:
  - TOKENS_API_REFERENCE
---

# Flow: Transferencia de Tokens

Fluxo para transferir NFTs e tokens ERC-20 entre usuarios, incluindo transferencia por endereco de wallet, por email e consulta de status da transferencia.

---

## Visao Geral

O modulo oferece 4 operacoes de transferencia:

| Operacao | Endpoint | Metodo | Descricao |
|----------|----------|--------|-----------|
| Transferir NFT por address | `{companyId}/token-editions/{id}/transfer-token` | PATCH | Envia NFT para endereco de wallet |
| Transferir NFT por email | `{companyId}/token-editions/{id}/transfer-token/email` | PATCH | Envia NFT para usuario por email |
| Consultar ultima transferencia | `{companyId}/token-editions/{id}/get-last/transfer` | GET | Status da ultima transferencia |
| Transferir ERC-20 | `/{companyId}/erc20-tokens/{id}/transfer/user` | PATCH | Transfere tokens fungiveis |

**API:** Key API (`https://api.w3block.io`)
**Auth:** Todos requerem Bearer token

---

## Fluxo A: Transferir NFT por Address

Transfere um NFT para outro endereco de wallet na blockchain.

### Pre-requisitos

1. Usuario autenticado
2. `companyId` do tenant
3. `editionId` do token (UUID da edicao, nao o tokenId da blockchain)
4. `toAddress` da wallet de destino

### Hook SDK

```typescript
import useTransferToken from '@w3block/w3block-ui-sdk';

const TransferirNFT = () => {
  const { mutate, mutateAsync, isLoading, error } = useTransferToken();

  const handleTransfer = async () => {
    try {
      await mutateAsync({
        editionId: 'uuid-da-edicao',
        toAddress: '0xdestinatario...',
      });
      console.log('Transferencia iniciada com sucesso!');
    } catch (err) {
      console.error('Erro na transferencia:', err);
    }
  };

  return (
    <button onClick={handleTransfer} disabled={isLoading}>
      {isLoading ? 'Transferindo...' : 'Transferir NFT'}
    </button>
  );
};
```

### Exemplo API-first

```typescript
const transferTokenByAddress = async (
  companyId: string,
  editionId: string,
  toAddress: string,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/token-editions/${editionId}/transfer-token`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ toAddress }),
    }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  return response.json();
};

// Uso:
await transferTokenByAddress(
  'uuid-empresa',
  'uuid-edicao',
  '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  accessToken
);
```

---

## Fluxo B: Transferir NFT por Email

Transfere um NFT para um usuario identificado por email. O sistema resolve o email para o endereco da wallet do destinatario.

### Pre-requisitos

1. Usuario autenticado
2. `companyId` do tenant
3. `editionId` do token
4. `email` do destinatario (deve ser um usuario cadastrado na plataforma)

### Hook SDK

```typescript
import useTransferTokenWithEmail from '@w3block/w3block-ui-sdk';

const TransferirPorEmail = () => {
  const { mutate, mutateAsync, isLoading, error } = useTransferTokenWithEmail();

  const handleTransfer = async () => {
    try {
      await mutateAsync({
        editionId: 'uuid-da-edicao',
        email: 'destinatario@exemplo.com',
      });
      console.log('Transferencia por email iniciada!');
    } catch (err) {
      console.error('Erro na transferencia:', err);
    }
  };

  return (
    <button onClick={handleTransfer} disabled={isLoading}>
      {isLoading ? 'Transferindo...' : 'Transferir por Email'}
    </button>
  );
};
```

### Exemplo API-first

```typescript
const transferTokenByEmail = async (
  companyId: string,
  editionId: string,
  email: string,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/token-editions/${editionId}/transfer-token/email`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ email }),
    }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  return response.json();
};

// Uso:
await transferTokenByEmail(
  'uuid-empresa',
  'uuid-edicao',
  'destinatario@exemplo.com',
  accessToken
);
```

---

## Fluxo C: Consultar Ultima Transferencia

Consulta o status da ultima transferencia de uma edicao de token. Util para exibir progresso e confirmacao de transferencias.

### Hook SDK

```typescript
import { useGetTransferToken, GetLastTransferAPIResponse } from '@w3block/w3block-ui-sdk';

const StatusTransferencia = ({ editionId }: { editionId: string }) => {
  // O segundo parametro (enabled) controla quando a query dispara
  const { data, isLoading, error } = useGetTransferToken(editionId, true);

  if (isLoading) return <div>Consultando status...</div>;
  if (error) return <div>Erro ao consultar transferencia</div>;

  const transfer: GetLastTransferAPIResponse | undefined = data?.data;
  if (!transfer) return <div>Nenhuma transferencia encontrada</div>;

  return (
    <div>
      <h3>Ultima Transferencia</h3>
      <p>De: {transfer.sender}</p>
      <p>Para: {transfer.toAddress}</p>
      <p>Status: {transfer.status}</p>
      <p>Chain: {transfer.chainId}</p>
      {transfer.txHash && (
        <a
          href={`https://polygonscan.com/tx/${transfer.txHash}`}
          target="_blank"
          rel="noopener noreferrer"
        >
          Ver na blockchain: {transfer.txHash}
        </a>
      )}
    </div>
  );
};
```

### Exemplo API-first

```typescript
const getLastTransfer = async (
  companyId: string,
  editionId: string,
  token: string
): Promise<GetLastTransferAPIResponse> => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/token-editions/${editionId}/get-last/transfer`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
    }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  return response.json();
};

// Uso:
const transfer = await getLastTransfer('uuid-empresa', 'uuid-edicao', accessToken);
console.log(transfer.status);    // "completed" | "pending" | "failed"
console.log(transfer.txHash);    // "0xhash..."
```

---

## Fluxo D: Transferir ERC-20

Transfere tokens ERC-20 (moedas, pontos de fidelidade) entre usuarios.

### Pre-requisitos

1. Usuario autenticado
2. `companyId` do tenant
3. `id` do token ERC-20 (loyaltyId)
4. Enderecos de origem e destino
5. Quantidade a transferir

### Hook SDK

```typescript
import { useTransfer } from '@w3block/w3block-ui-sdk';

const TransferirERC20 = () => {
  const { mutate, mutateAsync, isLoading } = useTransfer();

  const handleTransfer = async () => {
    try {
      const result = await mutateAsync({
        id: 'uuid-erc20-token',
        to: '0xdestinatario...',
        from: '0xremetente...',
        amount: '100',
        description: 'Transferencia de pontos de fidelidade',
      });
      console.log('Transferencia ERC-20 realizada:', result);
    } catch (err) {
      console.error('Erro na transferencia ERC-20:', err);
    }
  };

  return (
    <button onClick={handleTransfer} disabled={isLoading}>
      {isLoading ? 'Transferindo...' : 'Transferir Tokens'}
    </button>
  );
};
```

### Exemplo API-first

```typescript
const transferERC20 = async (
  companyId: string,
  erc20TokenId: string,
  to: string,
  from: string,
  amount: string,
  description: string,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/${companyId}/erc20-tokens/${erc20TokenId}/transfer/user`,
    {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        to,
        from,
        amount,
        metadata: { description },
        sendEmail: true,
      }),
    }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  return response.json();
};

// Uso:
await transferERC20(
  'uuid-empresa',
  'uuid-erc20-token',
  '0xdestinatario...',
  '0xremetente...',
  '50.00',
  'Pagamento de servico',
  accessToken
);
```

---

## Exemplo Completo: Tela de Transferencia com Opcoes

```typescript
import { useState } from 'react';
import { useTransferToken, useTransferTokenWithEmail, useGetTransferToken } from '@w3block/w3block-ui-sdk';

type TransferMode = 'address' | 'email';

const TransferScreen = ({ editionId }: { editionId: string }) => {
  const [mode, setMode] = useState<TransferMode>('address');
  const [toAddress, setToAddress] = useState('');
  const [email, setEmail] = useState('');
  const [showStatus, setShowStatus] = useState(false);

  const transferByAddress = useTransferToken();
  const transferByEmail = useTransferTokenWithEmail();
  const lastTransfer = useGetTransferToken(editionId, showStatus);

  const handleTransfer = async () => {
    try {
      if (mode === 'address') {
        await transferByAddress.mutateAsync({ editionId, toAddress });
      } else {
        await transferByEmail.mutateAsync({ editionId, email });
      }
      setShowStatus(true);
    } catch (err) {
      console.error('Erro na transferencia:', err);
    }
  };

  const isLoading = transferByAddress.isLoading || transferByEmail.isLoading;

  return (
    <div>
      <h2>Transferir Token</h2>

      {/* Selecao de modo */}
      <div>
        <button onClick={() => setMode('address')}>Por Endereco</button>
        <button onClick={() => setMode('email')}>Por Email</button>
      </div>

      {/* Formulario */}
      {mode === 'address' ? (
        <input
          type="text"
          placeholder="0x..."
          value={toAddress}
          onChange={(e) => setToAddress(e.target.value)}
        />
      ) : (
        <input
          type="email"
          placeholder="email@exemplo.com"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
        />
      )}

      <button onClick={handleTransfer} disabled={isLoading}>
        {isLoading ? 'Processando...' : 'Confirmar Transferencia'}
      </button>

      {/* Status da transferencia */}
      {showStatus && lastTransfer.data?.data && (
        <div>
          <h3>Status: {lastTransfer.data.data.status}</h3>
          <p>Para: {lastTransfer.data.data.toAddress}</p>
          {lastTransfer.data.data.txHash && (
            <p>TX: {lastTransfer.data.data.txHash}</p>
          )}
        </div>
      )}
    </div>
  );
};
```

---

## Payloads dos Hooks

### useTransferToken

```typescript
interface Payload {
  toAddress: string;   // Endereco da wallet de destino
  editionId: string;   // UUID da edicao do token
}
```

**Mutation Key:** `[PixwayAPIRoutes.TRANSFER_TOKEN]`

### useTransferTokenWithEmail

```typescript
interface Payload {
  email: string;       // Email do destinatario
  editionId: string;   // UUID da edicao do token
}
```

**Mutation Key:** `[PixwayAPIRoutes.TRANSFER_TOKEN_EMAIL]`

### useTransfer (ERC-20)

```typescript
interface Payload {
  to: string;          // Endereco da wallet de destino
  from: string;        // Endereco da wallet de origem
  amount: string;      // Quantidade a transferir
  id: string;          // UUID do token ERC-20
  description: string; // Descricao da transferencia
}
```

**Mutation Key:** `[PixwayAPIRoutes.TRANSFER_COIN, companyId]`

### useGetTransferToken

```typescript
// Parametros do hook:
editionId: string;  // UUID da edicao do token
enabled: boolean;   // Se a query deve ser disparada
```

**Query Key:** `[route, companyId, editionId]` (onde route e o path resolvido)

---

## Armadilhas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | Todos os endpoints de transfer usam PATCH, nao POST | Metodo HTTP e PATCH para transferencias |
| 2 | `editionId` != `tokenId` | `editionId` e o UUID interno do W3block, `tokenId` e o ID na blockchain. Use `editionId` (campo `metadata.tokenEditionData.id` do NFTByWalletDTO) |
| 3 | Transferencia por email requer usuario cadastrado | O email deve pertencer a um usuario existente na plataforma |
| 4 | Status `pending` pode demorar | Transferencias blockchain podem levar segundos a minutos. Implemente polling do status |
| 5 | `useTransfer` (ERC-20) envia `sendEmail: true` | O backend envia email de notificacao automaticamente |
| 6 | `companyId` vem do `useCompanyConfig()` | Os hooks `useTransferToken` e `useTransferTokenWithEmail` obtem `companyId` automaticamente |
| 7 | Transferencia ja em andamento | Se uma transferencia esta pendente, uma nova pode retornar 409 Conflict |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Obter `editionId` da listagem | Transferir o token | Este fluxo |
| Verificar status da transferencia | Exibir no explorer | Use `txHash` + ChainScan para montar URL |
| Confirmar transferencia | Atualizar listagem de NFTs | [FLOW_TOKENS_NFT_WALLET](./FLOW_TOKENS_NFT_WALLET.md) |
| Transferir ERC-20 | Verificar saldo antes | [FLOW_TOKENS_BALANCE](./FLOW_TOKENS_BALANCE.md) |
