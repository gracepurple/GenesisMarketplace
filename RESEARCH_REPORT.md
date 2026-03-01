# Research Report: GenesisMarketplace — A Decentralized NFT Marketplace on Polygon

## Abstract

This report presents a comprehensive analysis of the GenesisMarketplace, a decentralized Non-Fungible Token (NFT) marketplace built on the Polygon blockchain. The system enables users to mint, list, browse, and purchase NFTs using an ERC-721 smart contract integrated with a React-based frontend and IPFS-based decentralized storage via Pinata. This report examines the technical architecture, smart contract design, frontend implementation, decentralized storage strategy, security considerations, and potential areas for future development.

---

## 1. Introduction

### 1.1 Background

Non-Fungible Tokens (NFTs) are unique digital assets that represent ownership of items such as art, music, collectibles, and virtual real estate on a blockchain. Unlike fungible tokens (e.g., ETH or MATIC), each NFT carries a distinct identifier that makes it non-interchangeable. NFT marketplaces serve as platforms where users can create, buy, and sell these digital assets.

### 1.2 Motivation

The GenesisMarketplace project demonstrates a full-stack, end-to-end NFT marketplace implementation. By deploying on the Polygon network, the marketplace leverages lower transaction fees and faster confirmation times compared to Ethereum mainnet, making it more accessible for users.

### 1.3 Scope

This report covers:

- The overall system architecture and technology stack
- The Solidity smart contract design and its ERC-721 implementation
- The React frontend and its interaction with the blockchain
- IPFS integration for decentralized metadata and asset storage
- Security analysis and best practices
- Limitations and recommendations for future work

---

## 2. System Architecture

### 2.1 High-Level Overview

The GenesisMarketplace follows a Web3 full-stack architecture comprising four main layers:

```
┌──────────────────────────────────────────────────────┐
│                   User (Browser)                     │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │           React Frontend (UI Layer)           │    │
│  │  Marketplace │ SellNFT │ NFTPage │ Profile    │    │
│  └──────────────┬───────────────────────────────┘    │
│                  │                                    │
│  ┌───────────────▼──────────────────────────────┐    │
│  │         Ethers.js (Web3 Interface)            │    │
│  └───────────────┬──────────────────────────────┘    │
│                  │                                    │
│  ┌───────────────▼──────────────────────────────┐    │
│  │         MetaMask (Wallet / Signer)            │    │
│  └───────────────┬──────────────────────────────┘    │
└──────────────────┼───────────────────────────────────┘
                   │
     ┌─────────────▼─────────────┐   ┌─────────────────┐
     │  Polygon Blockchain       │   │  IPFS (Pinata)   │
     │  NFTMarketplace Contract  │   │  Metadata & Images│
     │  (ERC-721)                │   │                   │
     └───────────────────────────┘   └─────────────────┘
```

### 2.2 Technology Stack

| Layer              | Technology                                        |
| ------------------ | ------------------------------------------------- |
| **Frontend**       | React 18, React Router v6, Tailwind CSS           |
| **Web3 Library**   | Ethers.js v5.6.8                                  |
| **Wallet**         | MetaMask (via `window.ethereum` provider)         |
| **Blockchain**     | Polygon (Mainnet chain ID 137)                    |
| **Smart Contract** | Solidity 0.8.x, OpenZeppelin ERC721URIStorage     |
| **Development**    | Hardhat v2.9.7                                    |
| **Storage**        | IPFS via Pinata API                               |
| **Build Tools**    | react-app-rewired, Webpack (with polyfills)       |

### 2.3 Deployment Pipeline

The deployment process uses Hardhat to compile and deploy the `NFTMarketplace` smart contract. The deploy script (`deploy.js`) performs the following:

1. Retrieves the deployer signer from the Hardhat environment
2. Deploys the `NFTMarketplace` contract to the configured network
3. Writes the contract ABI and deployed address to `Marketplace.json`
4. The frontend then imports `Marketplace.json` to instantiate contract interactions

---

## 3. Smart Contract Analysis

### 3.1 Contract Overview

The `NFTMarketplace` contract (`NFTMarketplace.sol`) extends OpenZeppelin's `ERC721URIStorage`, providing full ERC-721 compliance with on-chain metadata URI storage. It uses `Counters` for safely incrementing token IDs and tracking items sold.

### 3.2 Key Data Structures

**ListedToken struct:**

```solidity
struct ListedToken {
    uint256 tokenId;
    address payable owner;
    address payable seller;
    uint256 price;
    bool currentlyListed;
}
```

This struct stores the on-chain listing information for each NFT, including the marketplace contract as the owner (since tokens are transferred to the contract upon listing), the original seller address, the price, and the listing status.

**State Variables:**

- `_tokenIds` — Counter tracking the most recently minted token ID
- `_itemsSold` — Counter tracking the number of completed sales
- `owner` — The contract deployer's address (receives listing fees)
- `listPrice` — The listing fee (set to 0.01 MATIC)
- `idToListedToken` — Mapping from token ID to `ListedToken` details

### 3.3 Core Functions

| Function            | Visibility | Description                                                                 |
| ------------------- | ---------- | --------------------------------------------------------------------------- |
| `createToken`       | public     | Mints a new NFT, sets its metadata URI, and lists it on the marketplace     |
| `createListedToken` | private    | Registers listing data and transfers the token to the contract              |
| `getAllNFTs`         | view       | Returns an array of all listed tokens                                       |
| `getMyNFTs`         | view       | Returns tokens where the caller is the owner or seller                      |
| `executeSale`       | public     | Executes a purchase: transfers token, pays seller, and pays listing fee     |
| `updateListPrice`   | public     | Allows the contract owner to update the listing fee                         |
| `getListPrice`      | view       | Returns the current listing fee                                             |
| `getListedTokenForId` | view     | Returns listing details for a specific token                                |
| `getCurrentToken`   | view       | Returns the current (latest) token ID                                       |

### 3.4 Marketplace Workflow (On-Chain)

**Minting & Listing:**

1. User calls `createToken(tokenURI, price)` with a payment of `listPrice`
2. A new token ID is incremented and assigned
3. The NFT is minted to the caller via `_safeMint`
4. The metadata URI is set via `_setTokenURI`
5. `createListedToken` stores listing data and transfers the token from the seller to the contract address
6. A `TokenListedSuccess` event is emitted

**Purchasing:**

1. Buyer calls `executeSale(tokenId)` with `msg.value` equal to the listed price
2. The token is transferred from the contract to the buyer
3. The listing fee is transferred to the contract owner
4. The sale proceeds are transferred to the seller
5. `_itemsSold` counter is incremented

### 3.5 Events

```solidity
event TokenListedSuccess(
    uint256 indexed tokenId,
    address owner,
    address seller,
    uint256 price,
    bool currentlyListed
);
```

This event provides an auditable on-chain log of each successful listing and can be monitored by off-chain systems.

---

## 4. Frontend Implementation

### 4.1 Component Architecture

The React frontend is organized into the following page components, connected via React Router v6:

| Component       | Route         | Purpose                                               |
| --------------- | ------------- | ----------------------------------------------------- |
| `Marketplace`   | `/`           | Homepage displaying all listed NFTs                   |
| `SellNFT`       | `/sellNFT`    | Form for creating and listing new NFTs                |
| `NFTPage`       | `/nftPage/:tokenId` | Detailed view for a single NFT with purchase option |
| `Profile`       | `/profile`    | User dashboard showing owned NFTs and portfolio value |
| `Navbar`        | (all pages)   | Navigation bar with wallet connection                 |
| `NFTTile`       | (component)   | Reusable card for displaying NFT summary              |

### 4.2 Wallet Integration

The application integrates with MetaMask through the `window.ethereum` provider:

1. **Connection**: Users click "Connect Wallet" which triggers `eth_requestAccounts`
2. **Network Switching**: The app requests switching to Polygon (chain ID 137), or adds the network configuration if not present
3. **Signer Retrieval**: All contract interactions use `provider.getSigner()` for transaction signing
4. **Account Change Detection**: The `accountsChanged` event listener refreshes the page on account switch

### 4.3 Contract Interaction Pattern

Each component that interacts with the smart contract follows a consistent pattern:

```javascript
// 1. Initialize provider and signer
const provider = new ethers.providers.Web3Provider(window.ethereum);
const signer = provider.getSigner();

// 2. Instantiate contract
let contract = new ethers.Contract(
    MarketplaceJSON.address,
    MarketplaceJSON.abi,
    signer
);

// 3. Call contract function
let transaction = await contract.getAllNFTs();
```

### 4.4 Data Flow

```
Smart Contract (on-chain data)
        │
        ▼
    Ethers.js (fetch token IDs, prices, owners)
        │
        ▼
    Token URI (IPFS URL stored on-chain)
        │
        ▼
    Axios GET to IPFS (fetch metadata JSON)
        │
        ▼
    Parse image URL, name, description
        │
        ▼
    React State → Render UI Components
```

The frontend fetches on-chain data (token IDs, prices, owners) and off-chain metadata (names, descriptions, images from IPFS) in parallel using `Promise.all`, creating a merged data object for rendering.

---

## 5. Decentralized Storage (IPFS via Pinata)

### 5.1 Storage Architecture

NFT assets are stored on IPFS through the Pinata pinning service. Two types of data are uploaded:

1. **Image files**: The actual NFT artwork/image uploaded as binary data
2. **Metadata JSON**: A JSON document containing the NFT's name, description, price, and IPFS image URL

### 5.2 Upload Flow

```
User selects image file
        │
        ▼
uploadFileToIPFS() → Pinata API → Returns IPFS hash
        │
        ▼
Construct metadata JSON { name, description, price, image: ipfsURL }
        │
        ▼
uploadJSONToIPFS() → Pinata API → Returns metadata IPFS hash
        │
        ▼
Metadata URL passed to createToken() as tokenURI
```

### 5.3 Pinata Configuration

- **API Authentication**: Uses API key and secret stored in environment variables (`REACT_APP_PINATA_KEY`, `REACT_APP_PINATA_SECRET`)
- **Replication**: File pinning is configured for replication across two Pinata regions (FRA1 and NYC1) for redundancy
- **Gateway**: Content is served via `https://gateway.pinata.cloud/ipfs/{hash}`

---

## 6. Security Considerations

### 6.1 Smart Contract Security

| Aspect                          | Status       | Notes                                                                 |
| ------------------------------- | ------------ | --------------------------------------------------------------------- |
| **Access Control**              | Partial      | `updateListPrice` restricted to owner; other functions are open       |
| **Reentrancy Protection**       | Not present  | `executeSale` transfers MATIC after state updates (follows CEI pattern partially) |
| **Integer Overflow**            | Mitigated    | Solidity 0.8.x has built-in overflow checks                          |
| **Price Validation**            | Present      | Requires `price > 0` and exact payment matching                      |
| **Listing Fee Enforcement**     | Present      | `createListedToken` requires `msg.value == listPrice`                 |

### 6.2 Potential Vulnerabilities

1. **No reentrancy guard**: The `executeSale` function performs external calls (MATIC transfers) after state changes. While the Checks-Effects-Interactions pattern is partially followed, adding OpenZeppelin's `ReentrancyGuard` would provide stronger protection.

2. **No token de-listing mechanism**: Once listed, tokens cannot be removed from the marketplace by the seller without a sale occurring.

3. **No resale support**: The contract notes that resale functionality may be added in the future; currently NFTs are listed by default upon minting.

4. **Front-end API key exposure**: Pinata API keys are stored as environment variables prefixed with `REACT_APP_`, which means they are bundled into the client-side JavaScript. This exposes the keys in the browser.

### 6.3 Recommendations

- Add `ReentrancyGuard` from OpenZeppelin to `executeSale`
- Implement a token de-listing function for sellers
- Move Pinata API calls to a backend server to protect API credentials
- Add event indexing for efficient off-chain querying
- Implement role-based access control for admin functions

---

## 7. Comparative Analysis

### 7.1 Comparison with Existing Marketplaces

| Feature                    | GenesisMarketplace | OpenSea     | Rarible     |
| -------------------------- | ------------------ | ----------- | ----------- |
| **Blockchain**             | Polygon            | Multi-chain | Multi-chain |
| **Token Standard**         | ERC-721            | ERC-721/1155| ERC-721/1155|
| **Auction Support**        | No                 | Yes         | Yes         |
| **Royalty Support**        | No                 | Yes (EIP-2981) | Yes      |
| **Lazy Minting**           | No                 | Yes         | Yes         |
| **Collection Support**     | Single contract    | Multiple    | Multiple    |
| **Listing Fee**            | 0.01 MATIC fixed   | 2.5% sale   | 2.5% sale   |
| **Decentralized Storage**  | IPFS (Pinata)      | IPFS/Arweave| IPFS        |

### 7.2 Strengths

- **Simplicity**: Clean, minimal implementation suitable for learning and extension
- **Low Fees**: Polygon deployment provides significantly lower gas costs
- **IPFS Storage**: Decentralized metadata ensures asset permanence
- **Open Source**: Full transparency of marketplace logic

### 7.3 Limitations

- No auction or bidding mechanism
- No royalty distribution for secondary sales (EIP-2981)
- Single collection model (all NFTs under one contract)
- No off-chain order book or meta-transactions

---

## 8. Technical Deep Dive

### 8.1 Webpack Polyfills

Since the project uses `react-app-rewired` with custom Webpack configuration, several Node.js modules are polyfilled for browser compatibility:

- `crypto-browserify` — Cryptographic operations
- `stream-browserify` — Stream handling
- `https-browserify` — HTTPS module
- `os-browserify` — OS utilities
- `buffer` — Buffer handling
- `path-browserify` — Path utilities

This is necessary because Ethers.js and related Web3 libraries depend on Node.js built-in modules that are unavailable in the browser environment.

### 8.2 Hardhat Configuration

The Hardhat development environment is configured with:

- **Solidity 0.8.4** compiler
- **Polygon Mumbai** testnet (via Alchemy RPC)
- **Polygon Mainnet** configuration
- **@nomiclabs/hardhat-waffle** for testing
- **@nomiclabs/hardhat-ethers** for Ethers.js integration

### 8.3 Gas Cost Analysis

On Polygon, typical gas costs for GenesisMarketplace operations:

| Operation     | Estimated Gas Units | Approximate Cost (at 30 Gwei) |
| ------------- | ------------------- | ----------------------------- |
| `createToken` | ~250,000            | ~0.0075 MATIC                 |
| `executeSale` | ~100,000            | ~0.003 MATIC                  |
| `getAllNFTs`   | View (no gas)       | Free                          |
| `getMyNFTs`   | View (no gas)       | Free                          |

---

## 9. Conclusion

The GenesisMarketplace is a functional, end-to-end NFT marketplace that demonstrates the core concepts of decentralized application development. It successfully integrates blockchain smart contracts, decentralized storage, wallet authentication, and a modern React frontend into a cohesive Web3 application.

**Key Findings:**

1. The ERC-721 smart contract provides basic but functional marketplace operations (minting, listing, purchasing)
2. IPFS integration via Pinata ensures decentralized and persistent storage of NFT metadata and assets
3. The React frontend provides an intuitive interface for interacting with the blockchain
4. Deployment on Polygon offers a cost-effective alternative to Ethereum mainnet

**Areas for Enhancement:**

- Auction and bidding mechanisms
- EIP-2981 royalty standard implementation
- Multi-collection support
- Backend API for secure Pinata credential handling
- Enhanced search and filtering capabilities
- Comprehensive testing suite

The project serves as a strong foundation for understanding NFT marketplace architecture and can be extended with additional features to approach production readiness.

---

## 10. References

1. Ethereum Foundation. "ERC-721: Non-Fungible Token Standard." https://eips.ethereum.org/EIPS/eip-721
2. OpenZeppelin. "ERC721 Documentation." https://docs.openzeppelin.com/contracts/4.x/erc721
3. Polygon Technology. "Polygon Documentation." https://wiki.polygon.technology/
4. Pinata. "IPFS Pinning Service API." https://docs.pinata.cloud/
5. Hardhat. "Hardhat Documentation." https://hardhat.org/docs
6. Ethers.js. "Ethers.js Documentation v5." https://docs.ethers.io/v5/

---

*Report generated for the GenesisMarketplace project — a decentralized NFT marketplace implementation on the Polygon blockchain.*
