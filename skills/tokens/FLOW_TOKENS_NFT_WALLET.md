---
id: FLOW_TOKENS_NFT_WALLET
title: "Tokens - NFT Wallet"
module: tokens
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - tokens
  - nft
  - wallet
depends_on:
  - TOKENS_API_REFERENCE
---

# Flow: NFTs por Wallet

Fluxo para listar todos os NFTs de uma wallet do usuario autenticado, filtrados por blockchain e mintados pelo W3block.

---

## Visao Geral

O usuario autenticado pode visualizar seus NFTs a partir do endereco da sua wallet principal (`mainWallet.address` do profile). A listagem e paginada e filtrada por `chainId`.

**Endpoint:** `GET metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true`
**API:** Key API (`https://api.w3block.io`)
**Hook SDK:** `useGetNFTSByWallet(chainId)`

---

## Pre-requisitos

1. Usuario autenticado com Bearer token valido
2. Profile do usuario carregado (contem `mainWallet.address`)
3. `chainId` definido (ex: 137 para Polygon, 1 para Ethereum)

---

## Fluxo Passo a Passo

### Step 1: Obter endereco da wallet

O hook `useGetNFTSByWallet` obtem automaticamente o endereco da wallet principal do profile do usuario:

```typescript
const { data: profile } = useProfile();
const address = profile?.data?.mainWallet?.address;
```

> **Nota:** A query so e disparada quando `address` e `chainId` estao definidos.

### Step 2: Buscar NFTs

```typescript
import { useGetNFTSByWallet } from '@w3block/w3block-ui-sdk';

// chainId: numero da blockchain (137 = Polygon, 1 = Ethereum)
const { data, isLoading, error } = useGetNFTSByWallet(137);
```

O hook:
1. Usa `useAxios(W3blockAPI.KEY)` para criar instancia autenticada
2. Faz GET em `metadata/nfts/{address}/{chainId}?onlyMintedByWeblock=true`
3. Retorna query paginada via `usePaginatedQuery<NFTByWalletDTO>`

### Step 3: Exibir resultados

Cada item do resultado e um `NFTByWalletDTO`:

```typescript
// Exemplo de uso dos dados
data?.items?.map((nft: NFTByWalletDTO) => ({
  titulo: nft.title,
  descricao: nft.description,
  imagem: nft.media?.[0]?.gateway || nft.metadata?.image,
  contrato: nft.contract.address,
  tokenId: nft.id.tokenId,
  tipoToken: nft.id.tokenMetadata.tokenType,
  colecao: nft.metadata.collectionData.name,
  edicao: nft.metadata.tokenEditionData.editionNumber,
  editionId: nft.metadata.tokenEditionData.id,
  rfid: nft.metadata.tokenEditionData.rfid,
  ehPass: nft.metadata.collectionData.pass,
}));
```

---

## Exemplo Completo (React)

```typescript
import { useGetNFTSByWallet, NFTByWalletDTO, ChainScan } from '@w3block/w3block-ui-sdk';

const MeusNFTs = () => {
  const chainId = ChainScan.POLYGON; // 137
  const { data, isLoading, error } = useGetNFTSByWallet(chainId);

  if (isLoading) return <div>Carregando NFTs...</div>;
  if (error) return <div>Erro ao carregar NFTs</div>;

  return (
    <div>
      <h1>Meus NFTs</h1>
      {data?.items?.map((nft: NFTByWalletDTO) => (
        <div key={`${nft.contract.address}-${nft.id.tokenId}`}>
          <img
            src={nft.media?.[0]?.thumbnail || nft.metadata?.image}
            alt={nft.title}
          />
          <h3>{nft.title}</h3>
          <p>{nft.description}</p>
          <span>Colecao: {nft.metadata.collectionData.name}</span>
          <span>Edicao #{nft.metadata.tokenEditionData.editionNumber}</span>
          <span>Tipo: {nft.id.tokenMetadata.tokenType}</span>
          {nft.metadata.collectionData.pass && (
            <span>Este NFT possui Pass vinculado</span>
          )}
        </div>
      ))}
    </div>
  );
};
```

---

## Exemplo API-first (sem SDK)

```typescript
const fetchNFTsByWallet = async (
  address: string,
  chainId: number,
  token: string
) => {
  const response = await fetch(
    `https://api.w3block.io/metadata/nfts/${address}/${chainId}?onlyMintedByWeblock=true`,
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

  const data = await response.json();
  return data; // { items: NFTByWalletDTO[], meta: { totalItems, currentPage, ... } }
};

// Uso:
const nfts = await fetchNFTsByWallet(
  '0xd3304183ec1fa687e380b67419875f97f1db05f5',
  137,  // Polygon
  accessToken
);
```

---

## Configuracoes do Hook

O hook `useGetNFTSByWallet` possui as seguintes configuracoes do React Query:

| Config | Valor | Descricao |
|--------|-------|-----------|
| `enabled` | `address != undefined && chainId != undefined` | So dispara quando ambos existem |
| `refetchOnMount` | `false` | Nao refaz ao montar componente |
| `refetchOnWindowFocus` | `false` | Nao refaz ao focar janela |

**Query Key:** `[PixwayAPIRoutes.NFTS_BY_WALLET, address, chainId]`

---

## Armadilhas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | Query nao dispara | Verificar se profile foi carregado e `mainWallet.address` existe |
| 2 | Retorna NFTs de terceiros | Sempre incluir `?onlyMintedByWeblock=true` na URL |
| 3 | `chainId` undefined causa erro | Passar `chainId` como `number \| undefined`, o hook trata o caso undefined |
| 4 | Imagem nao aparece | Priorizar `media[0].gateway` ou `media[0].thumbnail`, fallback para `metadata.image` |
| 5 | Dados do W3block faltando | Campos `collectionData` e `tokenEditionData` so existem para NFTs mintados pelo W3block |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Obter `editionId` do NFT | Transferir o token | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Obter `rfid` do NFT | Buscar dados publicos completos | [FLOW_TOKENS_METADATA](./FLOW_TOKENS_METADATA.md) |
| Verificar `collectionData.pass` | Consultar beneficios do pass | Skills de Pass |
| Obter `contract.address` + `tokenId` | Buscar metadados por contrato | [FLOW_TOKENS_METADATA](./FLOW_TOKENS_METADATA.md) |
