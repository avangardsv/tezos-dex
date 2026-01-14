# Tezos DEX - Technical Architecture

## Overview

A decentralized exchange (DEX) on Tezos implementing automated market maker (AMM) functionality with FA2 token support.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  Swap UI     │  │  Pool UI     │  │  Analytics   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
│                   ┌────────▼────────┐                        │
│                   │  Taquito Client │                        │
│                   └────────┬────────┘                        │
└────────────────────────────┼──────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Temple Wallet  │
                    └────────┬────────┘
                             │
┌────────────────────────────┼──────────────────────────────────┐
│                   Tezos Blockchain                            │
│                            │                                  │
│         ┌──────────────────┴──────────────────┐              │
│         │         Local Baker (Ghostnet)      │              │
│         │         RPC: localhost:8732         │              │
│         └──────────────────┬──────────────────┘              │
│                            │                                  │
│    ┌───────────────────────┼───────────────────────┐         │
│    │                       │                       │         │
│ ┌──▼──────┐      ┌─────────▼────────┐      ┌──────▼─────┐  │
│ │  Pool   │      │  Router Contract │      │  Factory   │  │
│ │Contract │◄─────┤  (Multi-hop)     │──────► Contract   │  │
│ └────┬────┘      └──────────────────┘      └────────────┘  │
│      │                                                       │
│ ┌────▼──────┐                                               │
│ │FA2 Tokens │                                               │
│ └───────────┘                                               │
└───────────────────────────────────────────────────────────────┘
```

---

## Smart Contract Layer

### **Contract Types**

#### 1. **Pool Contract** (Core)
```
Purpose: Liquidity pool for token pair
Storage:
  - tokenA: FA2 contract address
  - tokenB: FA2 contract address
  - reserveA: nat
  - reserveB: nat
  - totalLiquidity: nat
  - providers: map(address, nat)

Entrypoints:
  - swap(tokenIn, amountIn, minAmountOut)
  - addLiquidity(amountA, amountB, minLiquidity)
  - removeLiquidity(liquidity, minA, minB)
  - getPrice()

Math: x * y = k (constant product formula)
```

#### 2. **Router Contract**
```
Purpose: Route swaps across multiple pools
Storage:
  - pools: map((address, address), address)
  - factory: address

Entrypoints:
  - swapExactTokensForTokens(path, amountIn, minOut)
  - swapTokensForExactTokens(path, amountOut, maxIn)
  - addLiquidityOptimal(tokenA, tokenB, amounts)

Features: Multi-hop swaps, optimal routing
```

#### 3. **Factory Contract**
```
Purpose: Create new trading pairs
Storage:
  - pairs: map((address, address), address)
  - allPairs: list(address)
  - feeTo: address

Entrypoints:
  - createPair(tokenA, tokenB)
  - getPair(tokenA, tokenB)
  - setFeeTo(address)

Security: Prevents duplicate pairs
```

#### 4. **FA2 Token Contracts**
```
Purpose: Standard token implementation
Standard: TZIP-12 (FA2)

Entrypoints:
  - transfer(from, to, amount)
  - balance_of(owner)
  - update_operators(add/remove)

Used for: Trading tokens, LP tokens
```

---

## Frontend Layer

### **Technology Stack**

```typescript
Framework:     Next.js 15+
Language:      TypeScript
UI Library:    Chakra UI
Blockchain:    @taquito/taquito
Wallet:        @temple-wallet/dapp
State:         React Context + constate
Styling:       Emotion + CSS
```

### **Page Structure**

```
/                    → Landing/Swap page
/swap                → Token swap interface
/pool                → Liquidity pool management
  /pool/add          → Add liquidity
  /pool/remove       → Remove liquidity
/pools               → Browse all pools
/tokens              → Token list
/account             → User dashboard
/analytics           → Charts and stats
```

### **Component Hierarchy**

```
App
├── Layout
│   ├── Header (wallet connection)
│   ├── Navigation
│   └── Footer
├── Pages
│   ├── SwapPage
│   │   ├── TokenInput
│   │   ├── SwapButton
│   │   └── PriceInfo
│   ├── PoolPage
│   │   ├── PoolList
│   │   ├── PoolStats
│   │   └── LiquidityControls
│   └── AnalyticsPage
│       ├── Charts (volume, TVL)
│       └── PoolMetrics
└── Providers
    ├── WalletProvider (Temple)
    ├── TaquitoProvider
    └── ThemeProvider (Chakra)
```

---

## Data Flow

### **Swap Transaction Flow**

```
1. User Input
   └─► TokenInput component
        └─► Calculate output amount (local)
             └─► Display price impact

2. User Confirms
   └─► SwapButton click
        └─► Call wallet.sign()
             └─► Temple Wallet popup

3. Transaction Submission
   └─► Taquito.contract.methods.swap()
        └─► Submit to blockchain
             └─► Wait for confirmation

4. Transaction Confirmed
   └─► Update UI balances
        └─► Show success message
             └─► Record in history
```

### **Add Liquidity Flow**

```
1. User Inputs Amounts
   └─► Calculate optimal ratio
        └─► Show LP tokens to receive

2. Approve Tokens (if needed)
   └─► FA2 update_operators
        └─► Approve pool contract

3. Add Liquidity
   └─► Pool.addLiquidity()
        └─► Transfer tokens to pool
             └─► Mint LP tokens

4. Update State
   └─► Refresh balances
        └─► Update pool stats
```

---

## API Integration

### **Taquito Configuration**

```typescript
import { TezosToolkit } from '@taquito/taquito';
import { TempleWallet } from '@temple-wallet/dapp';

// Initialize Tezos client
const Tezos = new TezosToolkit('http://localhost:8732');

// Connect Temple Wallet
const wallet = new TempleWallet('TezosDeX');
Tezos.setWalletProvider(wallet);

// Contract interaction
const contract = await Tezos.wallet.at(poolAddress);
const operation = await contract.methods
  .swap(tokenIn, amountIn, minOut)
  .send();
await operation.confirmation();
```

### **Contract Addresses** (Ghostnet)

```typescript
// To be deployed
export const CONTRACTS = {
  factory: 'KT1...',     // Factory contract
  router: 'KT1...',      // Router contract
  pools: {
    'XTZ-tzBTC': 'KT1...',
    'XTZ-USDT': 'KT1...',
  },
  tokens: {
    tzBTC: 'KT1...',
    USDT: 'KT1...',
  }
};
```

---

## State Management

### **Global State (React Context)**

```typescript
interface AppState {
  // Wallet
  wallet: {
    connected: boolean;
    address: string;
    balance: number;
  };

  // Tokens
  tokens: {
    [address: string]: {
      symbol: string;
      balance: number;
      price: number;
    };
  };

  // Pools
  pools: {
    [pairId: string]: {
      reserveA: number;
      reserveB: number;
      tvl: number;
      volume24h: number;
    };
  };

  // User positions
  positions: {
    [poolAddress: string]: {
      liquidity: number;
      sharePercent: number;
    };
  };
}
```

### **Local State (Component)**

```typescript
// Swap component state
const [tokenIn, setTokenIn] = useState<Token>();
const [tokenOut, setTokenOut] = useState<Token>();
const [amountIn, setAmountIn] = useState<string>('');
const [amountOut, setAmountOut] = useState<string>('');
const [priceImpact, setPriceImpact] = useState<number>(0);
const [loading, setLoading] = useState<boolean>(false);
```

---

## Security Considerations

### **Smart Contract Security**

```
✓ Reentrancy guards
✓ Integer overflow protection (Michelson safe)
✓ Access control (admin functions)
✓ Slippage protection (minAmountOut)
✓ Deadline checks (expiry)
✓ Pausable (emergency stop)
```

### **Frontend Security**

```
✓ Wallet signature required
✓ No private key handling
✓ Transaction preview
✓ Input validation
✓ Rate limiting
✓ XSS protection (React default)
```

### **Operational Security**

```
✓ Contract audit before mainnet
✓ Testnet deployment first
✓ Gradual TVL ramp-up
✓ Bug bounty program
✓ Monitoring & alerts
```

---

## Development Infrastructure

### **Local Development**

```bash
# Local baker (already running)
RPC: http://localhost:8732
Network: Ghostnet
Status: ✅ Synced and operational

# Frontend dev server
Port: 3000
Hot reload: Enabled

# Contract development
SmartPy IDE: https://smartpy.io/ide
Local testing: smartpy test contract.py
```

### **Deployment Pipeline**

```
1. Development
   └─► Local baker (localhost:8732)
        └─► Contract testing

2. Staging
   └─► Ghostnet (public RPC)
        └─► Integration testing

3. Production (if viable)
   └─► Mainnet
        └─► Gradual rollout
```

---

## Performance Optimization

### **Frontend**

```
- Code splitting (Next.js dynamic imports)
- Image optimization (next/image)
- API caching (SWR or React Query)
- Lazy loading components
- Memoization (useMemo, useCallback)
```

### **Smart Contracts**

```
- Gas optimization (minimal storage)
- Batch operations where possible
- Efficient data structures
- Avoid unnecessary computations
```

---

## Monitoring & Analytics

### **Contract Events**

```michelson
# Emit events for indexing
- Swap(user, tokenIn, amountIn, tokenOut, amountOut)
- AddLiquidity(user, amountA, amountB, liquidity)
- RemoveLiquidity(user, liquidity, amountA, amountB)
```

### **Metrics to Track**

```
- Total Value Locked (TVL)
- 24h Trading Volume
- Number of trades
- Unique users
- Fee revenue
- Pool liquidity distribution
- Price impact distribution
```

### **Tools**

```
- Better Call Dev: Contract explorer
- TzKT: Blockchain data
- TzStats: Analytics
- Custom dashboard: Grafana (reuse existing)
```

---

## Technical Decisions

### **Why SmartPy over LIGO?**
- ✅ Python-like syntax (easier learning curve)
- ✅ Strong testing framework
- ✅ Good documentation
- ✅ Active community

**Alternative:** LIGO for production (more mature tooling)

### **Why Taquito over beacon-sdk?**
- ✅ Higher-level abstraction
- ✅ Better TypeScript support
- ✅ Good documentation
- ✅ Temple Wallet compatibility

### **Why Local Baker?**
- ✅ Free testnet access
- ✅ Fast iteration
- ✅ Full control
- ✅ No rate limits
- ✅ Already running and operational

---

## Future Enhancements

**V2 Features:**
- [ ] Concentrated liquidity (Uniswap V3 style)
- [ ] Limit orders
- [ ] Farming/staking rewards
- [ ] Governance token
- [ ] Cross-chain bridges
- [ ] Advanced charting
- [ ] Mobile app

**Infrastructure:**
- [ ] Subgraph indexing
- [ ] API for third-party integrations
- [ ] SDK for developers
- [ ] Advanced analytics dashboard

---

**Version:** 1.0
**Last Updated:** 2026-01-14
**Next Review:** After Phase 1 contracts deployed
