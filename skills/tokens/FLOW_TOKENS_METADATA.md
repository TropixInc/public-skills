---
id: FLOW_TOKENS_METADATA
title: "Tokens - Metadados"
module: tokens
version: "1.0.0"
type: flow
status: implemented
last_updated: "2026-03-30"
authors:
  - fernandodevpascoal
tags:
  - tokens
  - metadata
depends_on:
  - TOKENS_API_REFERENCE
---

# Flow: Metadados de Tokens

Fluxo para consultar metadados de tokens/NFTs por diferentes identificadores: RFID, endereco de contrato + tokenId, ou por colecao.

---

## Visao Geral

O modulo de metadados oferece 3 formas de consultar informacoes de um token:

| Metodo | Endpoint | Auth | Caso de Uso |
|--------|----------|------|-------------|
| Por RFID | `GET metadata/rfid/{rfid}` | Publico | Etiqueta fisica / QR code |
| Por Contrato | `GET metadata/address/{contractAddress}/{chainId}/{tokenId}` | Publico | Link direto para token na blockchain |
| Por Colecao | `GET metadata/{companyId}/{collectionId}` | Bearer | Listar todas as edicoes de uma colecao |

**API:** Key API (`https://api.w3block.io`)

---

## Fluxo A: Metadados por RFID

Consulta dados publicos de um token a partir do seu RFID (etiqueta fisica). Nao requer autenticacao.

### Endpoint

```
GET metadata/rfid/{rfid}
```

### Hook SDK

```typescript
import { usePublicTokenData, PublicTokenPageDTO } from '@w3block/w3block-ui-sdk';

const { data, isLoading, error } = usePublicTokenData({
  rfid: 'RFID-001',
  enabled: true,
});

// data?.data retorna PublicTokenPageDTO
const tokenInfo = data?.data;
```

### Exemplo API-first

```typescript
const fetchTokenByRfid = async (rfid: string) => {
  const response = await fetch(
    `https://api.w3block.io/metadata/rfid/${rfid}`,
    { method: 'GET' }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  const data: PublicTokenPageDTO = await response.json();
  return data;
};

// Uso:
const token = await fetchTokenByRfid('RFID-001');
console.log(token.information.title);       // "Meu Token"
console.log(token.edition.currentNumber);   // "1"
console.log(token.isMinted);                // true
```

---

## Fluxo B: Metadados por Contrato + Token ID

Consulta dados publicos de um token pelo endereco do contrato, chainId e tokenId. Nao requer autenticacao.

### Endpoint

```
GET metadata/address/{contractAddress}/{chainId}/{tokenId}
```

### Hook SDK

```typescript
import { usePublicTokenData, PublicTokenPageDTO } from '@w3block/w3block-ui-sdk';

const { data, isLoading, error } = usePublicTokenData({
  contractAddress: '0x1234...abcd',
  chainId: '137',
  tokenId: '1',
  enabled: true,
});

// data?.data retorna PublicTokenPageDTO
const tokenInfo = data?.data;
```

> **IMPORTANTE:** Quando `rfid` e fornecido, ele tem prioridade sobre `contractAddress/chainId/tokenId`. Nao passe ambos simultaneamente.

### Exemplo API-first

```typescript
const fetchTokenByContract = async (
  contractAddress: string,
  chainId: number,
  tokenId: string
) => {
  const response = await fetch(
    `https://api.w3block.io/metadata/address/${contractAddress}/${chainId}/${tokenId}`,
    { method: 'GET' }
  );

  if (!response.ok) {
    throw new Error(`Erro ${response.status}: ${response.statusText}`);
  }

  const data: PublicTokenPageDTO = await response.json();
  return data;
};

// Uso:
const token = await fetchTokenByContract(
  '0x1234567890abcdef1234567890abcdef12345678',
  137,   // Polygon
  '42'
);
console.log(token.company.name);            // "Minha Empresa"
console.log(token.information.description); // "Descricao do token"
console.log(token.token?.address);          // "0x1234..."
```

---

## Fluxo C: Metadados por Colecao

Lista todas as edicoes de tokens de uma colecao. Requer autenticacao.

### Endpoint

```
GET metadata/{companyId}/{collectionId}?page=1&limit=10&sortBy=createdAt&orderBy=DESC
```

### Hook SDK

```typescript
import { useGetCollectionMetadata } from '@w3block/w3block-ui-sdk';

const { data, isLoading, error } = useGetCollectionMetadata({
  id: 'uuid-da-colecao',
  query: {
    page: 1,
    limit: 10,
    sortBy: 'createdAt',
    orderBy: 'DESC',
    walletAddresses: '0xabc...',  // opcional: filtrar por wallet
  },
  enabled: true,
});

// data?.items retorna MetadataApiInterface<any>[]
data?.items?.map((edition) => ({
  id: edition.id,
  nome: edition.name,
  descricao: edition.description,
  edicao: edition.editionNumber,
  status: edition.status,
  rfid: edition.rfid,
  contrato: edition.contractAddress,
  dono: edition.ownerAddress,
  chainId: edition.chainId,
  tokenId: edition.tokenId,
  mintedAt: edition.mintedAt,
  imagem: edition.mainImage,
}));
```

### Exemplo API-first

```typescript
const fetchCollectionMetadata = async (
  companyId: string,
  collectionId: string,
  token: string,
  page: number = 1,
  limit: number = 10
) => {
  const params = new URLSearchParams({
    page: String(page),
    limit: String(limit),
    sortBy: 'createdAt',
    orderBy: 'DESC',
  });

  const response = await fetch(
    `https://api.w3block.io/metadata/${companyId}/${collectionId}?${params}`,
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
  return data; // { items: MetadataApiInterface[] }
};
```

---

## Configuracoes dos Hooks

### usePublicTokenData

| Config | Valor | Descricao |
|--------|-------|-----------|
| `staleTime` | `Infinity` | Dados nunca ficam stale (token publico nao muda) |
| `enabled` | `contractAddress && tokenId && chainId` (todos definidos e nao-vazios) | So dispara com todos os parametros |
| `refetchOnWindowFocus` | `false` | Nao refaz ao focar janela |
| `refetchOnMount` | `false` | Nao refaz ao montar componente |

**Query Key:** Gerada por `getPublicTokenDataQueryKey({ contractAddress, rfid, tokenId })`

### useGetCollectionMetadata

| Config | Valor | Descricao |
|--------|-------|-----------|
| `enabled` | `id != undefined && id != '' && enabled` | So dispara com ID valido |
| `refetchOnWindowFocus` | `false` | Nao refaz ao focar janela |
| `refetchOnMount` | `false` | Nao refaz ao montar componente |
| `retry` | `0` | Nao faz retry em caso de erro |

**Query Key:** `[PixwayAPIRoutes.METADATA_BY_COLLECTION_ID, companyId, id, walletAddresses]`

---

## Exemplo Completo (React) - Pagina de Detalhes do Token

```typescript
import { usePublicTokenData, PublicTokenPageDTO } from '@w3block/w3block-ui-sdk';

interface TokenDetailProps {
  rfid?: string;
  contractAddress?: string;
  chainId?: string;
  tokenId?: string;
}

const TokenDetail = ({ rfid, contractAddress, chainId, tokenId }: TokenDetailProps) => {
  const { data, isLoading, error } = usePublicTokenData({
    rfid,
    contractAddress,
    chainId,
    tokenId,
    enabled: true,
  });

  if (isLoading) return <div>Carregando detalhes do token...</div>;
  if (error) return <div>Erro ao carregar token</div>;

  const token: PublicTokenPageDTO | undefined = data?.data;
  if (!token) return <div>Token nao encontrado</div>;

  return (
    <div>
      {/* Informacoes basicas */}
      <h1>{token.information.title}</h1>
      {token.information.mainImage && (
        <img src={token.information.mainImage} alt={token.information.title} />
      )}
      <p>{token.information.description}</p>

      {/* Informacoes da empresa */}
      <div>
        <span>Empresa: {token.company.name}</span>
        <span>Categoria: {token.group.categoryName}</span>
        <span>Subcategoria: {token.group.subcategoryName}</span>
      </div>

      {/* Informacoes da edicao */}
      <div>
        <span>Edicao: {token.edition.currentNumber} / {token.edition.total}</span>
        <span>RFID: {token.edition.rfid}</span>
        {token.edition.mintedAt && <span>Mintado em: {token.edition.mintedAt}</span>}
        <span>Status: {token.isMinted ? 'Mintado' : 'Pendente'}</span>
      </div>

      {/* Informacoes da blockchain */}
      {token.token && (
        <div>
          <span>Contrato: {token.token.address}</span>
          <span>Token ID: {token.token.tokenId}</span>
          <span>Chain ID: {token.token.chainId}</span>
        </div>
      )}

      {/* Dados dinamicos */}
      {token.dynamicInformation?.tokenData && (
        <div>
          <h3>Dados Adicionais</h3>
          {Object.entries(token.dynamicInformation.tokenData).map(([key, value]) => (
            <div key={key}>
              <strong>{key}:</strong> <span>{String(value)}</span>
            </div>
          ))}
        </div>
      )}

      {/* Pass vinculado */}
      {token.group.collectionPass && (
        <div>Este token possui um Pass vinculado com beneficios.</div>
      )}
    </div>
  );
};
```

---

## Logica de Prioridade do usePublicTokenData

O hook decide qual endpoint chamar baseado nos parametros:

```
Se rfid existe:
  -> GET metadata/rfid/{rfid}

Senao (contractAddress + chainId + tokenId):
  -> GET metadata/address/{contractAddress}/{chainId}/{tokenId}
```

O codigo fonte:

```typescript
const endpoint = rfid
  ? PixwayAPIRoutes.METADATA_BY_RFID.replace('{rfid}', rfid)
  : PixwayAPIRoutes.METADATA_BY_CHAINADDRESS_AND_TOKENID
      .replace('{contractAddress}', contractAddress ?? '')
      .replace('{tokenId}', tokenId ?? '')
      .replace('{chainId}', chainId ?? '');
```

---

## Armadilhas

| # | Problema | Solucao |
|---|----------|---------|
| 1 | RFID tem prioridade sobre contractAddress | Se passar ambos, so o RFID sera usado. Passe um ou outro |
| 2 | `chainId` e string no hook, nao number | O hook `usePublicTokenData` aceita `chainId` como string (`'137'`), diferente de `useGetNFTSByWallet` que aceita number |
| 3 | `staleTime: Infinity` no usePublicTokenData | Dados nunca sao revalidados. Force invalidacao manual se necessario |
| 4 | `retry: 0` no useGetCollectionMetadata | Se falhar, nao tenta novamente. Implemente retry manual se necessario |
| 5 | `collectionId` vazio causa request desnecessario | O hook verifica `id != ''`, mas certifique-se de nao passar string vazia |

---

## Relacao com Outros Fluxos

| A partir deste fluxo... | Voce pode... | Documento |
|--------------------------|-------------|-----------|
| Obter `editionId` da colecao | Transferir o token | [FLOW_TOKENS_TRANSFER](./FLOW_TOKENS_TRANSFER.md) |
| Obter dados do token | Verificar se tem pass | Skills de Pass |
| Obter `contractAddress` + `chainId` | Consultar saldo | [FLOW_TOKENS_BALANCE](./FLOW_TOKENS_BALANCE.md) |
| Obter `rfid` | Escanear etiqueta fisica | Este fluxo (Fluxo A) |
