# Skills Transfer Guide: Tezos → Ethereum

## Overview

This guide maps skills and concepts learned from the **tezos-baker** project to their Ethereum equivalents. Use this as a reference when transitioning from Tezos infrastructure to Ethereum smart contract development.

---

## Concept Mapping

### Blockchain Fundamentals

| Concept | Tezos | Ethereum | Notes |
|---------|-------|----------|-------|
| **Consensus** | Proof of Stake (Tenderbake) | Proof of Stake (Casper) | Similar concepts, different implementations |
| **Block time** | ~15 seconds | ~12 seconds | Ethereum slightly faster |
| **Finality** | 2 blocks | 2 epochs (~13 min) | Tezos faster finality |
| **Account model** | Implicit (tz1) + Originated (KT1) | EOA + Contract | Similar dual model |
| **Gas** | Gas + storage fees | Gas (EIP-1559) | Both have gas, different pricing |

### Smart Contract Languages

| Tezos | Ethereum | Comparison |
|-------|----------|------------|
| **Michelson** | **EVM Bytecode** | Low-level, stack-based |
| **SmartPy** | **Vyper** | Python-like syntax |
| **LIGO** | **Solidity** | More formal, typed |
| **Archetype** | **Fe** | Domain-specific |

**Recommendation:** If you learned SmartPy, Solidity will feel familiar in structure but uses different syntax.

### Token Standards

| Tezos | Ethereum | Purpose |
|-------|----------|---------|
| FA1.2 | ERC-20 | Fungible tokens |
| FA2 | ERC-20/721/1155 | Multi-asset standard |
| — | ERC-721 | NFTs |
| — | ERC-1155 | Multi-token |

**Key difference:** Tezos FA2 is a unified standard; Ethereum has separate standards for each use case.

---

## Infrastructure Skills (Direct Transfer)

### Docker & Containers ✅

**What you learned:**
```bash
# Tezos Baker
docker run -d --name tezos-node tezos/tezos:octez-v23.1
docker logs -f tezos-node
docker restart tezos-baker
```

**Ethereum equivalent:**
```bash
# Ethereum Node (Geth)
docker run -d --name geth ethereum/client-go:latest
docker logs -f geth
docker restart geth
```

**Transfer:** 100% - Same Docker patterns work

---

### Monitoring Stack ✅

**What you built (tezos-baker):**
- Prometheus for metrics
- Grafana for dashboards
- Loki for logs
- Promtail for log shipping

**Ethereum equivalent:** Identical! Same tools, different metrics endpoints.

```yaml
# prometheus.yml - just change the target
scrape_configs:
  - job_name: 'ethereum'
    static_configs:
      - targets: ['geth:6060']  # Geth metrics endpoint
```

**Transfer:** 100% - Reuse entire monitoring stack

---

### Node Operations ✅

| Operation | Tezos Command | Ethereum Equivalent |
|-----------|--------------|---------------------|
| Start node | `npm run node:start` | `geth --syncmode snap` |
| Check sync | `npm run node:sync-status` | `geth attach --exec eth.syncing` |
| View logs | `npm run node:logs` | `docker logs geth` |
| RPC call | `curl localhost:8732/chains/main/blocks/head` | `curl -X POST localhost:8545 -d '{"method":"eth_blockNumber"}'` |

**Transfer:** 90% - Same patterns, different commands

---

### Bash Scripting ✅

**Pattern you used (tezos-baker):**
```bash
#!/usr/bin/env bash
source "$(dirname "${BASH_SOURCE[0]}")/lib/common.sh"

main() {
    log_info "Starting operation..."
    check_container_running "tezos-node"
    execute_command
    log_success "Complete!"
}

main "$@"
```

**Ethereum usage:** Identical! Same scripting patterns apply.

**Transfer:** 100% - Scripts are blockchain-agnostic

---

## Wallet Integration (Moderate Transfer)

### Tezos (What you used)

```typescript
// Temple Wallet + Taquito
import { TezosToolkit } from '@taquito/taquito';
import { TempleWallet } from '@temple-wallet/dapp';

const Tezos = new TezosToolkit('http://localhost:8732');
await Tezos.setWalletProvider(await TempleWallet.isAvailable());

// Send transaction
const contract = await Tezos.wallet.at('KT1...');
await contract.methods.transfer(params).send();
```

### Ethereum (What you'll use)

```typescript
// MetaMask + Ethers.js
import { ethers } from 'ethers';

const provider = new ethers.BrowserProvider(window.ethereum);
const signer = await provider.getSigner();

// Send transaction
const contract = new ethers.Contract(address, abi, signer);
await contract.transfer(params);
```

**OR with Wagmi (recommended):**

```typescript
// Wagmi + RainbowKit
import { useContractWrite } from 'wagmi';

const { write } = useContractWrite({
  address: '0x...',
  abi: contractABI,
  functionName: 'transfer',
});

// Send transaction
write({ args: [recipient, amount] });
```

**Transfer:** 70% - Similar patterns, different APIs

### Comparison Table

| Feature | Tezos (Taquito) | Ethereum (Ethers/Wagmi) |
|---------|-----------------|-------------------------|
| Provider setup | `TezosToolkit(rpc)` | `ethers.BrowserProvider(ethereum)` |
| Wallet connect | `setWalletProvider(temple)` | `provider.getSigner()` |
| Contract call | `contract.methods.fn().send()` | `contract.fn()` |
| View call | `contract.storage()` | `contract.fn()` (view) |
| Events | `contract.events()` | `contract.on('Event', cb)` |

---

## Smart Contract Development (New Learning)

### Tezos SmartPy → Ethereum Solidity

**SmartPy (what you would have written):**
```python
import smartpy as sp

class Counter(sp.Contract):
    def __init__(self):
        self.init(count = 0)

    @sp.entry_point
    def increment(self):
        self.data.count += 1

    @sp.entry_point
    def decrement(self):
        self.data.count -= 1
```

**Solidity (what you'll write):**
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Counter {
    uint256 public count;

    function increment() public {
        count += 1;
    }

    function decrement() public {
        count -= 1;
    }
}
```

### Key Syntax Differences

| Concept | SmartPy | Solidity |
|---------|---------|----------|
| Contract definition | `class Counter(sp.Contract):` | `contract Counter { }` |
| State variable | `self.data.count` | `uint256 public count;` |
| Function | `@sp.entry_point` | `function name() public` |
| Modifier | `@sp.private` | `modifier onlyOwner` |
| Constructor | `def __init__(self):` | `constructor() { }` |
| Types | Dynamic (Python) | Static (explicit types) |
| Storage | `self.data` | State variables |

### Testing Comparison

**SmartPy:**
```python
@sp.add_test(name = "CounterTest")
def test():
    scenario = sp.test_scenario()
    c = Counter()
    scenario += c
    scenario += c.increment()
    scenario.verify(c.data.count == 1)
```

**Hardhat (JavaScript):**
```javascript
const { expect } = require("chai");

describe("Counter", function () {
  it("Should increment", async function () {
    const Counter = await ethers.getContractFactory("Counter");
    const counter = await Counter.deploy();

    await counter.increment();
    expect(await counter.count()).to.equal(1);
  });
});
```

---

## DeFi Concepts (Direct Transfer)

### AMM Math

**Learned from tezos-dex planning:**
- Constant product formula: `x * y = k`
- Price impact calculation
- Slippage protection
- Liquidity pool mechanics

**Ethereum application:** Identical math! Uniswap uses same formulas.

```solidity
// Same formula in Solidity
function getAmountOut(
    uint256 amountIn,
    uint256 reserveIn,
    uint256 reserveOut
) public pure returns (uint256) {
    uint256 amountInWithFee = amountIn * 997;
    uint256 numerator = amountInWithFee * reserveOut;
    uint256 denominator = (reserveIn * 1000) + amountInWithFee;
    return numerator / denominator;
}
```

**Transfer:** 100% - Math is blockchain-agnostic

### Token Standards

**FA2 patterns you planned:**
- Transfer
- Balance lookup
- Operator approval

**ERC-20 equivalent:**
- `transfer(to, amount)`
- `balanceOf(owner)`
- `approve(spender, amount)`

**Transfer:** 90% - Same concepts, different function names

---

## Development Tools Comparison

| Tool Type | Tezos | Ethereum | Notes |
|-----------|-------|----------|-------|
| **IDE** | SmartPy IDE | Remix | Web-based |
| **Framework** | SmartPy CLI | Hardhat/Foundry | Local development |
| **Testing** | SmartPy tests | Mocha/Forge | Similar patterns |
| **Deployment** | Taquito | Hardhat/Ethers | Script-based |
| **Explorer** | TzKT, Better Call Dev | Etherscan | Contract verification |
| **Wallet** | Temple | MetaMask | Browser extension |
| **Faucet** | Ghostnet Faucet | Sepolia Faucets | Testnet tokens |

---

## Quick Reference: Tezos → Ethereum Cheat Sheet

### Addresses
```
Tezos:    tz1... (implicit), KT1... (contract)
Ethereum: 0x... (all addresses, 20 bytes)
```

### RPC Calls
```bash
# Get latest block
Tezos:    curl localhost:8732/chains/main/blocks/head
Ethereum: curl -X POST localhost:8545 -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'

# Get balance
Tezos:    curl localhost:8732/chains/main/blocks/head/context/contracts/tz1.../balance
Ethereum: curl -X POST localhost:8545 -d '{"method":"eth_getBalance","params":["0x...","latest"]}'
```

### Transaction Structure
```
Tezos: {
  source: "tz1...",
  destination: "KT1...",
  amount: 1000000,  // mutez
  parameters: { entrypoint: "transfer", value: {...} }
}

Ethereum: {
  from: "0x...",
  to: "0x...",
  value: "0x...",  // wei (hex)
  data: "0x..."    // encoded function call
}
```

### Units
```
Tezos:    1 XTZ = 1,000,000 mutez
Ethereum: 1 ETH = 1,000,000,000,000,000,000 wei (10^18)
```

---

## Learning Path Summary

### Week 1: Environment
- [ ] Install Hardhat: `npm install --save-dev hardhat`
- [ ] Set up project: `npx hardhat init`
- [ ] Configure Sepolia network
- [ ] Get test ETH from faucet

### Week 2: Basic Contracts
- [ ] Write HelloWorld.sol
- [ ] Write Counter.sol
- [ ] Deploy to local Hardhat network
- [ ] Write tests

### Week 3: Token Development
- [ ] Study ERC-20 standard
- [ ] Implement simple token
- [ ] Deploy to Sepolia
- [ ] Verify on Etherscan

### Week 4: Integration
- [ ] Update frontend with Wagmi
- [ ] Connect MetaMask
- [ ] Interact with deployed contract
- [ ] Build mint/transfer UI

---

## Resources for Transition

### Solidity Learning
- [Solidity by Example](https://solidity-by-example.org/) - Similar to SmartPy examples
- [CryptoZombies](https://cryptozombies.io/) - Interactive tutorial
- [Ethereum Speedrun](https://speedrunethereum.com/) - Project-based learning

### Tooling
- [Hardhat Docs](https://hardhat.org/docs) - Development framework
- [OpenZeppelin](https://docs.openzeppelin.com/) - Secure contract library
- [Wagmi Docs](https://wagmi.sh/) - React hooks for Ethereum

### Reference Code
- [Uniswap V2](https://github.com/Uniswap/v2-core) - AMM reference (study this!)
- [OpenZeppelin Contracts](https://github.com/OpenZeppelin/openzeppelin-contracts)

---

**Version:** 1.0
**Last Updated:** 2026-01-14
**Purpose:** Bridge Tezos knowledge to Ethereum development
