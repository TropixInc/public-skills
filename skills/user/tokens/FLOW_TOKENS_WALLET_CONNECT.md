---
id: FLOW_TOKENS_WALLET_CONNECT
title: "Tokens - Wallet Connect"
module: tokens
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - tokens
  - wallet-connect
depends_on:
  - TOKENS_API_REFERENCE
---

# Flow: WalletConnect

Fluxo para conectar uma wallet externa via protocolo WalletConnect, permitindo que o usuario interaja com a plataforma usando wallets como MetaMask, Trust Wallet, etc.

---

## Visao Geral

O WalletConnect permite ao usuario conectar uma wallet externa (MetaMask, Trust Wallet, etc.) a plataforma W3block. A sessao e gerenciada pela Identity API.

**Endpoint:** `POST /blockchain/request-session-wallet-connect`
**API:** Identity API (`https://id.w3block.io`) -- **NAO** a Key API
**Hook SDK:** `useRequestWalletConnect()`

> **IMPORTANTE:** Este e o unico fluxo do modulo Tokens que usa a Identity API (W3blockAPI.ID) em vez da Key API (W3blockAPI.KEY).

---

## Pre-requisitos

1. Usuario autenticado com Bearer token valido
2. `chainId` da blockchain desejada
3. `address` da wallet do usuario na plataforma
4. `uri` do WalletConnect (opcional, formato `wc:...`)

---

## Fluxo Passo a Passo

### Step 1: Obter URI do WalletConnect

O URI do WalletConnect e gerado pela wallet externa (MetaMask, Trust Wallet, etc.) quando o usuario inicia o processo de conexao. Formato tipico:

```
wc:a1b2c3d4-e5f6-7890-abcd-ef1234567890@1?bridge=https://bridge.walletconnect.org&key=abc123def456
```

### Step 2: Solicitar sessao

```typescript
import { useRequestWalletConnect, RequestWalletConnectDTO } from '@w3block/w3block-ui-sdk';

const ConectarWallet = () => {
  const { mutate, mutateAsync, isLoading, error } = useRequestWalletConnect();

  const handleConnect = async () => {
    try {
      const payload: RequestWalletConnectDTO = {
        chainId: 137,                                    // Polygon
        address: '0xd3304183ec1fa687e380b67419875f97f1db05f5', // Wallet do usuario
        uri: 'wc:a1b2c3@1?bridge=https://bridge.walletconnect.org&key=abc123',
      };

      const result = await mutateAsync(payload);
      console.log('Sessao WalletConnect iniciada:', result);
    } catch (err) {
      console.error('Erro ao conectar wallet:', err);
    }
  };

  return (
    <button onClick={handleConnect} disabled={isLoading}>
      {isLoading ? 'Conectando...' : 'Conectar Wallet'}
    </button>
  );
};
```

### Step 3: Confirmar conexao

Apos a sessao ser iniciada, a wallet externa recebe a solicitacao e o usuario deve aprovar a conexao no app da wallet.

---

## Interface do Payload

```typescript
export interface RequestWalletConnectDTO {
  chainId: number;   // ID da blockchain (137 = Polygon, 1 = Ethereum)
  address: string;   // Endereco da wallet do usuario na plataforma
  uri?: string;      // URI do WalletConnect (formato wc:...)
}
```

| Campo | Tipo | Obrigatorio | Descricao |
|-------|------|-------------|-----------|
| `chainId` | `number` | Sim | ID da blockchain para a sessao |
| `address` | `string` | Sim | Endereco da wallet do usuario |
| `uri` | `string` | Nao | URI do WalletConnect gerado pela wallet externa |

---

## Exemplo API-first

```typescript
const requestWalletConnect = async (
  chainId: number,
  address: string,
  token: string,
  uri?: string
) => {
  const response = await fetch(
    'https://id.w3block.io/blockchain/request-session-wallet-connect',
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        chainId,
        address,
        ...(uri ? { uri } : {}),
      }),
    }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  return response.json();
};

// Uso:
const session = await requestWalletConnect(
  137,
  '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  accessToken,
  'wc:a1b2c3@1?bridge=https://bridge.walletconnect.org&key=abc123'
);
```

---

## Desconectar Sessao

Para desconectar uma sessao WalletConnect ativa:

**Endpoint:** `POST /blockchain/disconnect-session-wallet-connect`
**API:** Identity API (`https://id.w3block.io`)

```typescript
const disconnectWalletConnect = async (token: string) => {
  const response = await fetch(
    'https://id.w3block.io/blockchain/disconnect-session-wallet-connect',
    {
      method: 'POST',
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
```

> **Nota:** A rota `DISCONNECT_WALLET_CONNECT` esta registrada em `PixwayAPIRoutes` como `/blockchain/disconnect-session-wallet-connect`.

---

## Exemplo Completo (React)

```typescript
import { useState } from 'react';
import { useRequestWalletConnect, RequestWalletConnectDTO, useProfile, ChainScan } from '@w3block/w3block-ui-sdk';

const WalletConnectScreen = () => {
  const [uri, setUri] = useState('');
  const [connected, setConnected] = useState(false);
  const [selectedChain, setSelectedChain] = useState<ChainScan>(ChainScan.POLYGON);

  const { data: profile } = useProfile();
  const address = profile?.data?.mainWallet?.address;
  const { mutateAsync, isLoading, error } = useRequestWalletConnect();

  const chains = [
    { id: ChainScan.POLYGON, name: 'Polygon', currency: 'MATIC' },
    { id: ChainScan.ETHEREUM, name: 'Ethereum', currency: 'ETH' },
    { id: ChainScan.MUMBAI, name: 'Mumbai (Testnet)', currency: 'MATIC' },
  ];

  const handleConnect = async () => {
    if (!address) {
      alert('Wallet nao encontrada no profile');
      return;
    }

    try {
      const payload: RequestWalletConnectDTO = {
        chainId: selectedChain,
        address,
        uri: uri || undefined,
      };

      await mutateAsync(payload);
      setConnected(true);
    } catch (err) {
      console.error('Erro ao conectar:', err);
    }
  };

  return (
    <div>
      <h2>WalletConnect</h2>

      {/* Selecao de blockchain */}
      <div>
        <label>Blockchain:</label>
        <select
          value={selectedChain}
          onChange={(e) => setSelectedChain(Number(e.target.value))}
        >
          {chains.map((chain) => (
            <option key={chain.id} value={chain.id}>
              {chain.name} ({chain.currency})
            </option>
          ))}
        </select>
      </div>

      {/* URI do WalletConnect */}
      <div>
        <label>WalletConnect URI (opcional):</label>
        <input
          type="text"
          placeholder="wc:..."
          value={uri}
          onChange={(e) => setUri(e.target.value)}
        />
      </div>

      {/* Endereco da wallet */}
      <div>
        <label>Sua wallet:</label>
        <span>{address || 'Nao disponivel'}</span>
      </div>

      {/* Botao de conexao */}
      <button onClick={handleConnect} disabled={isLoading || !address}>
        {isLoading ? 'Conectando...' : 'Conectar via WalletConnect'}
      </button>

      {/* Status */}
      {connected && <div>Wallet conectada com sucesso!</div>}
      {error && <div>Erro: {String(error)}</div>}
    </div>
  );
};
```

---

## Detalhes Tecnicos do Hook

O hook `useRequestWalletConnect` internamente:

1. Usa `useAxios(W3blockAPI.ID)` -- Identity API, nao Key API
2. Faz POST em `PixwayAPIRoutes.WALLET_CONNECT` (`/blockchain/request-session-wallet-connect`)
3. Envia `{ chainId, address, uri }` no body
4. Retorna mutation do React Query (`useMutation`)

```typescript
export const useRequestWalletConnect = () => {
  const axios = useAxios(W3blockAPI.ID); // <-- Identity API

  const _requestWalletConnect = ({ chainId, address, uri }: RequestWalletConnectDTO) => {
    return axios.post(PixwayAPIRoutes.WALLET_CONNECT, {
      chainId,
      address,
      uri,
    });
  };

  const requestWalletConnect = useMutation(_requestWalletConnect);
  return requestWalletConnect;
};
```

---

## Armadilhas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | Usar Key API em vez de Identity API | WalletConnect usa `W3blockAPI.ID` (https://id.w3block.io), nao KEY |
| 2 | URI expirado | URIs do WalletConnect expiram. Gere um novo se a conexao falhar |
| 3 | `uri` e opcional | A sessao pode ser iniciada sem URI; nesse caso, o backend pode gerar |
| 4 | `chainId` deve ser numero | Nao passe string; use `Number(chainId)` ou enum `ChainScan` |
| 5 | Wallet nao tem address | Verifique `profile?.data?.mainWallet?.address` antes de chamar |
| 6 | Sessao anterior ativa | Desconecte a sessao anterior antes de iniciar uma nova |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Wallet conectada | Consultar saldo | [FLOW_TOKENS_BALANCE](./FLOW_TOKENS_BALANCE.md) |
| Wallet conectada | Listar NFTs | [FLOW_TOKENS_NFT_WALLET](./FLOW_TOKENS_NFT_WALLET.md) |
| Wallet conectada | Transferir tokens | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Obter address do profile | Iniciar WalletConnect | Este fluxo |
