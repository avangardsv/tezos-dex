# Getting Started - Tezos DEX Development

## Prerequisites

### **Required**
- ✅ Node.js 18+ and Yarn
- ✅ Git
- ✅ Temple Wallet browser extension
- ✅ Local Tezos baker running on Ghostnet (you have this!)

### **Recommended**
- VSCode with SmartPy/LIGO extensions
- Docker (for contract testing)
- Postman/Insomnia (API testing)

---

## Environment Setup

### **1. Clone and Install**

```bash
# Clone the repository
git clone https://github.com/avangardsv/tezos-dex.git
cd tezos-dex

# Install dependencies
yarn install

# Create environment file
cp .env.example .env
```

### **2. Configure Environment**

Edit `.env`:

```bash
# Your local baker RPC
NEXT_PUBLIC_RPC_URL=http://localhost:8732

# Network
NEXT_PUBLIC_NETWORK=ghostnet

# Contract addresses (deploy these first)
NEXT_PUBLIC_FACTORY_CONTRACT=KT1...
NEXT_PUBLIC_ROUTER_CONTRACT=KT1...

# Optional: Analytics
NEXT_PUBLIC_ENABLE_ANALYTICS=false
```

### **3. Verify Local Baker**

```bash
# Check your baker is running
curl http://localhost:8732/chains/main/blocks/head

# Should return JSON with block info
# If error: Start your baker
cd ~/tezos-baker
npm run node:start
```

---

## Development Workflow

### **Frontend Development**

```bash
# Start dev server
yarn dev

# Open browser
open http://localhost:3000

# Hot reload enabled - make changes and see them live
```

### **Contract Development**

#### **Option A: SmartPy (Recommended for beginners)**

```bash
# Install SmartPy CLI
wget https://smartpy.io/cli/install.sh -O /tmp/smartpy-install.sh
bash /tmp/smartpy-install.sh

# Create contract directory
mkdir -p contracts/smartpy
cd contracts/smartpy

# Create your first contract
cat > hello.py <<'EOF'
import smartpy as sp

class HelloWorld(sp.Contract):
    def __init__(self):
        self.init(message = "Hello World")

    @sp.entry_point
    def update_message(self, new_message):
        self.data.message = new_message

@sp.add_test(name = "HelloWorld")
def test():
    c = HelloWorld()
    scenario = sp.test_scenario()
    scenario += c
    scenario += c.update_message("Hello Tezos")
EOF

# Test contract
~/smartpy-cli/SmartPy.sh test hello.py output/

# Deploy to your local baker
~/smartpy-cli/SmartPy.sh originate-contract \
  --code output/hello/step_000_cont_0_contract.tz \
  --storage output/hello/step_000_cont_0_storage.tz \
  --rpc http://localhost:8732
```

#### **Option B: LIGO**

```bash
# Install LIGO
curl -L https://ligolang.org/install.sh | bash

# Create contract
mkdir -p contracts/ligo
cd contracts/ligo

cat > hello.mligo <<'EOF'
type storage = string

type return = operation list * storage

let main (action, store : string * storage) : return =
  [], action
EOF

# Compile
ligo compile contract hello.mligo

# Deploy (manual via Tezos client)
```

---

## First Steps

### **Week 1: Environment Familiarization**

#### **Day 1-2: Frontend Exploration**

```bash
# Explore the codebase
code .

# Key files to review:
components/       # React components
pages/           # Next.js pages
utils/           # Helper functions
hooks/           # Custom React hooks

# Make a small UI change to understand hot reload
# Example: Edit pages/index.tsx
```

#### **Day 3-4: Wallet Connection**

```bash
# Install Temple Wallet
# Visit: https://templewallet.com/

# Configure for Ghostnet
# 1. Open Temple
# 2. Settings → Network → Ghostnet
# 3. Get testnet XTZ from faucet

# Test wallet connection on frontend
yarn dev
# Click "Connect Wallet" button
```

#### **Day 5-7: First Contract**

```bash
# Write, test, and deploy the HelloWorld contract (see above)

# Once deployed, get contract address from output
# Example: KT1ABC...XYZ

# Interact via Taquito
# Create utils/contracts.ts and add interaction code
```

---

### **Week 2-3: FA2 Token**

#### **Step 1: Learn FA2 Standard**

```bash
# Read the standard
open https://gitlab.com/tezos/tzip/-/blob/master/proposals/tzip-12/tzip-12.md

# Clone FA2 template
git clone https://github.com/oxheadalpha/smart-contracts.git fa2-reference
cd fa2-reference/multi_asset
```

#### **Step 2: Deploy Test Token**

```bash
# Use SmartPy FA2 template
# Visit: https://smartpy.io/ide?template=fa2_lib.py

# Customize and deploy via IDE or CLI
# Name it: DEX_TOKEN
# Symbol: DEX
# Decimals: 6
```

#### **Step 3: Integrate with Frontend**

```typescript
// utils/tokens.ts
import { TezosToolkit } from '@taquito/taquito';

export const getTokenBalance = async (
  tezos: TezosToolkit,
  tokenAddress: string,
  owner: string
) => {
  const contract = await tezos.wallet.at(tokenAddress);
  const storage: any = await contract.storage();

  // FA2 balance lookup
  const balance = await storage.ledger.get({
    0: owner,
    1: 0 // token_id
  });

  return balance?.toNumber() || 0;
};

// Test in browser console
```

---

## Testing Guide

### **Smart Contract Testing**

```bash
# SmartPy tests
~/smartpy-cli/SmartPy.sh test contracts/smartpy/pool.py output/

# Check test output
cat output/log.txt

# Expected: All tests pass ✓
```

### **Frontend Testing**

```bash
# Unit tests (to be added)
yarn test

# E2E tests (to be added)
yarn test:e2e

# Manual testing
yarn dev
# Click through all features
```

---

## Deployment Guide

### **Deploy to Ghostnet (via your local baker)**

#### **1. Prepare Wallet**

```bash
# Using Tezos client (installed with your baker)
cd ~/tezos-baker

# Import your Temple wallet (optional - can use Temple directly)
npm run account:show
# Note your tz1... address
```

#### **2. Deploy Contracts**

```bash
# Example: Deploy pool contract
~/smartpy-cli/SmartPy.sh originate-contract \
  --code output/pool/contract.tz \
  --storage output/pool/storage.tz \
  --rpc http://localhost:8732 \
  --payer YOUR_TZ1_ADDRESS

# Save contract address from output
# Example: KT1PoolContractXYZ...

# Update .env
echo "NEXT_PUBLIC_POOL_CONTRACT=KT1PoolContractXYZ..." >> .env
```

#### **3. Verify Deployment**

```bash
# Check contract on explorer
open https://ghostnet.tzkt.io/KT1PoolContractXYZ...

# Or via Better Call Dev
open https://better-call.dev/ghostnet/KT1PoolContractXYZ...
```

#### **4. Deploy Frontend**

```bash
# Push to GitHub
git add .
git commit -m "Deploy pool contract"
git push

# Vercel auto-deploys from GitHub
# Or manual deploy:
vercel --prod
```

---

## Common Tasks

### **Add a New Token**

```typescript
// utils/tokens.ts
export const TOKENS = {
  DEX: {
    address: 'KT1...',
    symbol: 'DEX',
    decimals: 6,
    icon: '/tokens/dex.png'
  },
  tzBTC: {
    address: 'KT1...',
    symbol: 'tzBTC',
    decimals: 8,
    icon: '/tokens/tzbtc.png'
  },
  // Add your token here
};
```

### **Create a New Pool**

```bash
# 1. Deploy pool contract with token pair
# 2. Add initial liquidity
# 3. Register in factory
# 4. Update frontend token list
```

### **Update Dependencies**

```bash
# Check outdated packages
yarn outdated

# Update all (carefully)
yarn upgrade

# Test after updating
yarn dev
```

---

## Troubleshooting

### **Frontend Issues**

**Problem:** `yarn dev` fails
```bash
# Solution 1: Clear cache
rm -rf .next node_modules
yarn install
yarn dev

# Solution 2: Check Node version
node -v  # Should be 18+
```

**Problem:** Wallet won't connect
```bash
# 1. Check Temple is on Ghostnet
# 2. Check RPC URL in .env matches baker
# 3. Check baker is running:
curl http://localhost:8732/chains/main/blocks/head
```

### **Contract Issues**

**Problem:** Contract deployment fails
```bash
# Check baker sync status
cd ~/tezos-baker
npm run node:status

# Should show: "Sync: Up to date"
```

**Problem:** Can't find contract after deployment
```bash
# Check transaction on explorer
# Baker may be behind - wait for sync
npm run node:sync-status
```

---

## Next Steps

Once you're comfortable with the basics:

1. **Read ROADMAP.md** - Understand the development phases
2. **Read ARCHITECTURE.md** - Understand the system design
3. **Start Phase 1** - Build your first real contract
4. **Join Community** - Tezos Discord, Stack Exchange

---

## Resources

**Official Docs:**
- [Tezos Developer Docs](https://tezos.com/developers/)
- [SmartPy Docs](https://smartpy.io/docs/)
- [Taquito Docs](https://tezostaquito.io/)
- [FA2 Standard](https://gitlab.com/tezos/tzip/-/blob/master/proposals/tzip-12/)

**Tools:**
- [Better Call Dev](https://better-call.dev/) - Contract explorer
- [TzKT](https://tzkt.io/) - Block explorer
- [Temple Wallet](https://templewallet.com/) - Browser wallet
- Your baker: http://localhost:8732

**Community:**
- [Tezos Discord](https://discord.gg/tezos)
- [Tezos Stack Exchange](https://tezos.stackexchange.com/)
- [Reddit r/tezos](https://reddit.com/r/tezos)

---

**Questions?** Check the troubleshooting section or ask in Tezos Discord!
