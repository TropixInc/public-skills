---
id: FLOW_TOKENS_BALANCE
title: "Tokens - Consulta de Saldo"
module: tokens
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - tokens
  - balance
depends_on:
  - TOKENS_API_REFERENCE
---

# Flow: Saldo Blockchain

Fluxo para consultar o saldo nativo (ETH, MATIC) de uma wallet em uma blockchain especifica, incluindo consulta individual e consulta em lote para multiplas wallets.

---

## Visao Geral

O modulo de balance oferece 2 funcionalidades:

| Funcionalidade | Hook SDK | Descricao |
|----------------|----------|-----------|
| Saldo individual | `useBalance` | Consulta saldo de uma wallet em uma chain |
| Saldo em lote | `useGetBalancesForWallets` | Consulta saldo de multiplas wallets em paralelo |

**Endpoint:** `GET blockchain/balance/{address}/{chainId}`
**API:** Key API (`https://api.w3block.io`)
**Auth:** Bearer token

---

## Pre-requisitos

1. Usuario autenticado com Bearer token valido
2. `address` da wallet (formato 0x...)
3. `chainId` numerico (1 = Ethereum, 137 = Polygon, 80001 = Mumbai)

---

## Fluxo A: Saldo Individual

Consulta o saldo nativo de uma wallet em uma blockchain.

### Hook SDK

```typescript
import { useBalance, ChainScan } from '@w3block/w3block-ui-sdk';

const SaldoWallet = () => {
  const balance = useBalance({
    chainId: ChainScan.POLYGON, // 137
    address: '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  });

  if (!balance) return <div>Endereco ou chain nao definidos</div>;
  if (balance.isLoading) return <div>Consultando saldo...</div>;
  if (balance.error) return <div>Erro ao consultar saldo</div>;

  const data = balance.data?.data;

  return (
    <div>
      <h3>Saldo</h3>
      <p>{data?.balance} {data?.currency}</p>
      {/* Para exibir em formato legivel: */}
      <p>{(parseFloat(data?.balance || '0') / 1e18).toFixed(6)} {data?.currency}</p>
    </div>
  );
};
```

> **IMPORTANTE:** O hook retorna `null` se `address` e undefined, vazia, ou se `chainId` nao esta definido. Verifique antes de acessar `.data`.

### Logica interna do hook

```typescript
export const useBalance = ({ chainId, address }: useBalanceParams) => {
  const axios = useAxios(W3blockAPI.KEY);

  const balance = usePrivateQuery(
    [chainId, address],
    () =>
      axios.get<GetBalanceAPIResponse>(
        PixwayAPIRoutes.BALANCE
          .replace('{address}', address)
          .replace('{chainId}', String(chainId))
      ),
    { enabled: address != undefined && address != '' }
  );

  // Retorna null se chainId ou address invalidos
  return chainId && address && address != '' ? balance : null;
};
```

### Exemplo API-first

```typescript
interface GetBalanceAPIResponse {
  balance: string;
  currency: 'ETH' | 'MATIC';
}

const fetchBalance = async (
  address: string,
  chainId: number,
  token: string
): Promise<GetBalanceAPIResponse> => {
  const response = await fetch(
    `https://api.w3block.io/blockchain/balance/${address}/${chainId}`,
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

// Uso - Saldo em Polygon:
const balancePolygon = await fetchBalance(
  '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  137,
  accessToken
);
console.log(`${balancePolygon.balance} ${balancePolygon.currency}`);
// Ex: "1500000000000000000 MATIC" (1.5 MATIC)

// Uso - Saldo em Ethereum:
const balanceEth = await fetchBalance(
  '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  1,
  accessToken
);
console.log(`${balanceEth.balance} ${balanceEth.currency}`);
// Ex: "500000000000000000 ETH" (0.5 ETH)
```

---

## Fluxo B: Saldo de Multiplas Wallets

Consulta saldo de varias wallets em paralelo. Util para exibir dashboard com todas as wallets do usuario.

### Hook SDK

```typescript
import { useGetBalancesForWallets } from '@w3block/w3block-ui-sdk';

interface WalletSimple {
  address: string;
  chainId: number;
  balance?: string;
  // outros campos da wallet
}

const DashboardWallets = () => {
  const getWalletsBalances = useGetBalancesForWallets();
  const [wallets, setWallets] = useState<WalletSimple[]>([]);

  const loadBalances = async () => {
    const walletsInput: WalletSimple[] = [
      { address: '0xabc...', chainId: 137 },
      { address: '0xdef...', chainId: 1 },
      { address: '0x123...', chainId: 80001 },
    ];

    // Consulta todas em paralelo (Promise.all)
    const walletsWithBalance = await getWalletsBalances(walletsInput);
    setWallets(walletsWithBalance);
  };

  return (
    <div>
      <h2>Minhas Wallets</h2>
      <button onClick={loadBalances}>Atualizar Saldos</button>
      {wallets.map((wallet) => (
        <div key={`${wallet.address}-${wallet.chainId}`}>
          <span>{wallet.address}</span>
          <span>Chain: {wallet.chainId}</span>
          <span>Saldo: {wallet.balance || 'N/A'}</span>
        </div>
      ))}
    </div>
  );
};
```

### Logica interna do hook

O `useGetBalancesForWallets` retorna uma funcao que:
1. Recebe array de `WalletSimple[]`
2. Faz `Promise.all` com GET para cada wallet
3. Enriquece cada wallet com o campo `balance`
4. Se uma consulta falha, retorna a wallet original (sem balance) - nao interrompe as demais

```typescript
export const useGetBalancesForWallets = () => {
  const axios = useAxios(W3blockAPI.KEY);

  const getWalletsBalances = async (wallets: WalletSimple[]): Promise<WalletSimple[]> => {
    return Promise.all(
      wallets.map((wallet) => {
        return axios
          .get<GetBalanceAPIResponse>(
            PixwayAPIRoutes.BALANCE
              .replace('{address}', wallet.address)
              .replace('{chainId}', String(wallet.chainId))
          )
          .then((response) => ({
            ...wallet,
            balance: response.data.balance,
          }))
          .catch(() => wallet); // Falha silenciosa: retorna wallet sem balance
      })
    );
  };

  return getWalletsBalances;
};
```

---

## Conversao de Saldo

O `balance` e retornado como string em wei (a menor unidade da moeda nativa). Para converter para a unidade legivel:

```typescript
// Wei para ETH/MATIC (18 decimais)
const balanceInWei = '1500000000000000000';
const balanceReadable = parseFloat(balanceInWei) / 1e18; // 1.5

// Formatacao com precisao
const formatted = (parseFloat(balanceInWei) / 1e18).toFixed(6); // "1.500000"

// Utilitario reutilizavel
const weiToEther = (wei: string, decimals: number = 6): string => {
  return (parseFloat(wei) / 1e18).toFixed(decimals);
};
```

### Tabela de Currency por ChainId

| ChainId | Rede | Currency | Exploradora |
|---------|------|----------|-------------|
| `1` | Ethereum Mainnet | `ETH` | etherscan.io |
| `3` | Ropsten | `ETH` | ropsten.etherscan.io |
| `4` | Rinkeby | `ETH` | rinkeby.etherscan.io |
| `42` | Kovan | `ETH` | kovan.etherscan.io |
| `137` | Polygon | `MATIC` | polygonscan.com |
| `80001` | Mumbai | `MATIC` | mumbai.polygonscan.com |

---

## Exemplo Completo (React) - Card de Saldo

```typescript
import { useBalance, useProfile, ChainScan } from '@w3block/w3block-ui-sdk';

const BalanceCard = ({ chainId }: { chainId: ChainScan }) => {
  const { data: profile } = useProfile();
  const address = profile?.data?.mainWallet?.address;

  const balance = useBalance({
    chainId,
    address: address || '',
  });

  if (!balance || !address) {
    return <div>Wallet nao disponivel</div>;
  }

  if (balance.isLoading) {
    return <div>Carregando saldo...</div>;
  }

  const data = balance.data?.data;
  const balanceFormatted = data
    ? (parseFloat(data.balance) / 1e18).toFixed(6)
    : '0.000000';

  const chainName = chainId === ChainScan.POLYGON ? 'Polygon' : 'Ethereum';

  return (
    <div>
      <h3>{chainName}</h3>
      <p>Endereco: {address}</p>
      <p>
        Saldo: {balanceFormatted} {data?.currency || 'N/A'}
      </p>
    </div>
  );
};

// Uso:
const App = () => (
  <div>
    <BalanceCard chainId={ChainScan.POLYGON} />
    <BalanceCard chainId={ChainScan.ETHEREUM} />
  </div>
);
```

---

## Armadilhas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | `balance` e string, nao number | Sempre usar `parseFloat(balance)` para calculos |
| 2 | Balance em wei, nao em ETH/MATIC | Dividir por `1e18` para obter valor legivel |
| 3 | `useBalance` retorna `null` | Verificar se `address` e nao-vazio e `chainId` esta definido |
| 4 | Falha em `useGetBalancesForWallets` nao propaga | Falhas individuais sao silenciosas - wallet sem `balance` indica erro |
| 5 | Query key usa `[chainId, address]` | Mudanca de chain ou address dispara nova consulta |
| 6 | `enabled` verifica `address != ''` | String vazia e tratada como address invalido |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Verificar saldo suficiente | Transferir ERC-20 | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Obter address do profile | Listar NFTs | [FLOW_TOKENS_NFT_WALLET](./FLOW_TOKENS_NFT_WALLET.md) |
| Verificar saldo para gas | Realizar transferencia NFT | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Consultar multiplas wallets | Exibir dashboard | Este fluxo (Fluxo B) |
