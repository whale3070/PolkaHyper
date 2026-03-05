# PolkaHyper — 技术设计文档

**Polkadot Hub ↔ HyperLiquid 双资金池协同交易平台**

> Version 1.1.0 · Draft · 2026-03
> All code, comments, variable names, and commit messages MUST be written in English.

---

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 技术架构总览](#2-技术架构总览)
- [3. 双资金池模型设计](#3-双资金池模型设计)
- [4. 智能合约架构](#4-智能合约架构)
- [5. Relayer 服务设计](#5-relayer-服务设计)
- [6. HyperLiquid 测试网集成](#6-hyperliquid-测试网集成)
- [7. 前端设计](#7-前端设计)
- [8. 安全设计](#8-安全设计)
- [9. 测试策略](#9-测试策略)
- [10. 部署流程](#10-部署流程)
- [11. 风险分析](#11-风险分析)
- [12. 开发规范](#12-开发规范)
- [附录](#附录)

---

## 1. 项目概述

### 1.1 定位

**PolkaHyper** 是一个 Demo 级跨链资金协同平台，目标是让用户在同一个前端界面内完成：

- 在 **HyperLiquid Testnet** 进行永续合约交易（Trading）
- 从 **Polkadot Hub TestNet**（Chain ID: `420420417`）向 HyperLiquid Core 充值 USDC
- 从 HyperLiquid Core 提现 USDC 回 Polkadot Hub TestNet

**核心设计原则**：Polkadot Hub TestNet 已完全兼容 EVM/Ethereum 工具链，直接使用 MetaMask + wagmi + viem 连接，**无需 SS58 地址或 Polkadot.js Extension**。不依赖 XCM 跨链消息，不依赖中间跨链桥，在两条链上各建立一个链下资金池（Vault），通过后台 Relayer 服务自动调拨，实现准实时的双向资金流动。

### 1.2 核心目标

| 目标 | 说明 |
|------|------|
| 💱 同屏交易 | 同一 UI 内完成充值、交易、提现，无需跳转多个平台 |
| 🏦 双池模型 | Polkadot Hub Vault + HyperLiquid Vault，Relayer 自动平衡 |
| 🔗 无跨链桥 | 不依赖 XCM 或第三方跨链桥，降低复杂度与风险 |
| 🦊 单一钱包 | MetaMask / Brave 一个钱包同时操作两条链，体验统一 |
| 🧪 测试网优先 | Polkadot Hub TestNet + HyperLiquid Testnet，安全验证流程 |

### 1.3 与原文方案的差异

| 维度 | 原文方案 | PolkaHyper v1.1 |
|------|---------|----------------|
| Polkadot 钱包 | Polkadot.js Extension + SS58 | MetaMask / Brave（纯 EVM） |
| 跨链机制 | XCM + HoudiniSwap + Relayer | 双资金池 + Relayer 链下调拨 |
| 地址格式 | SS58 + EVM 双套 | 统一 EVM `0x` 地址 |
| 依赖 | XCM、跨链桥合约、Polkadot.js | 仅 RPC + REST API |
| 复杂度 | 高（多跳路径） | 低（直接池对池） |

---

## 2. 技术架构总览

### 2.1 系统架构图

```
┌────────────────────────────────────────────────────────────────┐
│                  PolkaHyper System Architecture                 │
├────────────────────────────────────────────────────────────────┤
│                                                                  │
│   React Frontend (Vite + TypeScript + Tailwind CSS v4)          │
│   [Trade Panel]  [Deposit Panel]  [Withdraw Panel]  [Portfolio] │
│        │  wagmi/viem (single wallet)  │  @nktkas/hyperliquid    │
│        ▼                              ▼                          │
│  ┌─────────────────────┐   ┌────────────────────────────────┐  │
│  │  Polkadot Hub       │   │  HyperLiquid Testnet           │  │
│  │  TestNet            │   │                                │  │
│  │  Chain ID: 420420417│   │  HyperCore ←→ HyperEVM        │  │
│  │  EVM compatible     │   │  Chain ID: 998                 │  │
│  │                     │   │  L1 Perp-DEX Trading Engine    │  │
│  │  VaultDOT.sol       │   │  VaultEVM.sol                  │  │
│  └──────────┬──────────┘   └────────────┬───────────────────┘  │
│             │                            │                       │
│             └────── Relayer (Node.js) ───┘                      │
│                  Monitor events, rebalance pools                 │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 技术栈总览

| 层级 | 选型 | 说明 |
|------|------|------|
| **区块链·Polkadot侧** | Polkadot Hub TestNet | Chain ID `420420417`，EVM 兼容，原生 ETH 工具可直接使用 |
| **区块链·HL侧** | HyperLiquid Testnet | 独立 L1，HyperBFT 共识，HyperCore + HyperEVM (Chain ID: 998) |
| **合约语言** | Solidity 0.8.28 | 两条链均使用 EVM 兼容合约 |
| **合约工具** | Hardhat + @parity/hardhat-polkadot | Polkadot Hub 兼容插件 |
| **前端框架** | React 18 + Vite 5 + TypeScript | SPA，Hash Router |
| **样式** | Tailwind CSS v4 + @tailwindcss/vite | Polkadot 粉 + HL 暗色混合主题 |
| **状态管理** | Zustand + @tanstack/react-query | 全局钱包/交易状态 + 链数据缓存 |
| **钱包连接** | wagmi v2 + viem 2.x + RainbowKit | 单一 EVM 钱包同时操作两链，支持 MetaMask / Brave |
| **HL SDK** | @nktkas/hyperliquid + viem | InfoClient / ExchangeClient / SubscriptionClient |
| **WebSocket** | @nktkas/hyperliquid WebSocketTransport | 实时订单簿、持仓、价格推送 |
| **Relayer服务** | Node.js + viem（双链） | 监听双侧事件，自动调拨资金 |
| **部署** | Netlify（前端）+ Railway/Fly.io（Relayer） | Netlify Functions 代理 HL API |
| **国际化** | i18next + react-i18next | zh-CN / en |

---

## 3. 双资金池模型设计

### 3.1 核心概念

PolkaHyper 的核心是「**双池模型（Dual Pool Model）**」。不依赖原子性跨链消息，在两条链上分别维护一个 Vault（资金池），通过链下 Relayer 服务监听事件并自动在两侧调拨资金。这是一种「异步流动性桥接」模式，以牺牲一定即时性（约 10–30 秒）换取极低的技术复杂度。

由于 Polkadot Hub TestNet 已完全 EVM 兼容，两侧合约均为标准 Solidity，Relayer 使用 viem 统一操作两条链，**架构高度对称**。

### 3.2 充值流程（Polkadot Hub → HyperLiquid）

```
User
 │  1. 输入充值金额，点击 "Deposit to HyperLiquid"
 ▼
Frontend
 │  2. wagmi 调用 VaultDOT.deposit(amount, hlAddress)
 │     用户使用 MetaMask 在 Polkadot Hub TestNet 签名
 ▼
VaultDOT (Polkadot Hub TestNet)
 │  3. 锁定 USDC，emit Deposited(user, hlAddress, amount, txId)
 ▼
Relayer
 │  4. 监听 Deposited 事件
 │  5. 等待 6 个区块确认
 │  6. 调用 VaultEVM.forwardDeposit(hlAddress, amount)
 ▼
VaultEVM (HyperEVM Testnet)
 │  7. approve USDC → HyperCore Bridge
 │  8. 调用官方 Bridge 合约将 USDC 注入 HyperCore
 ▼
HyperCore
    9. 用户账户余额增加，可立即交易（~10-30 秒）
```

### 3.3 提现流程（HyperLiquid → Polkadot Hub）

```
User
 │  1. 输入提现金额，点击 "Withdraw to Polkadot"
 ▼
Frontend
 │  2. 调用 ExchangeClient.withdraw3() 将 USDC 从 Core 提至 EVM
 │  3. wagmi 调用 VaultEVM.requestWithdraw(amount, dotUserEVMAddr)
 ▼
VaultEVM (HyperEVM Testnet)
 │  4. 锁定 USDC，emit WithdrawRequested(requestId, user, amount)
 ▼
Relayer
 │  5. 监听 WithdrawRequested 事件
 │  6. 调用 VaultDOT.releaseWithdraw(userAddr, amount)
 ▼
VaultDOT (Polkadot Hub TestNet)
 │  7. 释放 USDC 给用户在 Polkadot Hub 的 EVM 地址
 ▼
User
    8. 收到 USDC（~10-30 秒）
```

### 3.4 Vault 状态机

| 状态 | 触发条件 | 下一状态 |
|------|---------|---------|
| `PENDING` | 用户提交充值/提现请求 | `PROCESSING` |
| `PROCESSING` | Relayer 确认链上事件 | `CONFIRMING` |
| `CONFIRMING` | 目标链交易已提交 | `CONFIRMED` / `FAILED` |
| `CONFIRMED` | 目标链确认 N 个区块 | 终态 ✅ |
| `FAILED` | 超时或交易失败 | `REFUNDING` |
| `REFUNDING` | Relayer 发起退款 | `REFUNDED` 终态 |

### 3.5 资金池平衡策略（Demo 级）

- **初始预充值**：在两侧 Vault 中预存缓冲 USDC（建议各 500 USDC）
- **监控阈值**：Relayer 监控余额，低于 100 USDC 时发出 Discord 告警
- **最大单笔限额**：Demo 阶段每笔最大 200 USDC，防止池子被单次耗尽
- **人工介入**：Demo 阶段管理员手动补充 Vault 余额

---

## 4. 智能合约架构

### 4.1 合约模块总览

| 合约 | 部署链 | 职责 | 升级性 |
|------|--------|------|--------|
| `VaultDOT` | Polkadot Hub TestNet | 接收用户 USDC，记录映射，发出事件；Relayer 释放提现 | UUPS 可升级 |
| `VaultEVM` | HyperEVM Testnet | Relayer 注资后转发至 HyperCore；接收提现申请 | UUPS 可升级 |
| `AddressRegistry` | Polkadot Hub TestNet | 可选：记录用户在两链的地址绑定（Demo 中地址相同可省略） | UUPS 可升级 |

> **注**：由于两条链均使用 EVM 兼容地址（`0x...`），同一个 MetaMask 账户在两链的地址完全相同。Demo 阶段 `AddressRegistry` 可简化甚至省略。

### 4.2 目录结构

```
polkahyper/
├── contracts/                    # Hardhat 工程（两链合约）
│   ├── contracts/
│   │   ├── polkadot/
│   │   │   ├── VaultDOT.sol
│   │   │   └── interfaces/
│   │   │       └── IVaultDOT.sol
│   │   ├── hyperliquid/
│   │   │   ├── VaultEVM.sol
│   │   │   └── interfaces/
│   │   │       └── IVaultEVM.sol
│   │   └── libraries/
│   │       └── SafeTransferLib.sol
│   ├── scripts/
│   │   ├── deploy/
│   │   │   ├── 01_deploy_vault_dot.ts    # → Polkadot Hub TestNet
│   │   │   └── 02_deploy_vault_evm.ts    # → HyperEVM Testnet
│   │   └── setup/
│   │       └── 03_setup_roles.ts
│   ├── test/
│   │   ├── VaultDOT.test.ts
│   │   ├── VaultEVM.test.ts
│   │   └── integration/
│   │       └── FullFlow.test.ts
│   └── hardhat.config.ts
├── relayer/                      # Node.js Relayer 服务
│   ├── src/
│   │   ├── listeners/
│   │   │   ├── polkadotHubListener.ts
│   │   │   └── hyperEVMListener.ts
│   │   ├── executors/
│   │   │   ├── depositExecutor.ts
│   │   │   └── withdrawExecutor.ts
│   │   ├── store/
│   │   │   └── pendingQueue.ts       # SQLite 持久化
│   │   ├── utils/
│   │   │   └── alert.ts              # Discord Webhook
│   │   └── index.ts
│   └── package.json
└── frontend/                     # React 前端
    └── src/...
```

### 4.3 Hardhat 配置（双链 EVM）

```typescript
// hardhat.config.ts
import { HardhatUserConfig } from 'hardhat/config';
import '@parity/hardhat-polkadot';
import '@nomicfoundation/hardhat-ethers';

const config: HardhatUserConfig = {
  solidity: '0.8.28',
  networks: {
    // Polkadot Hub TestNet — 纯 EVM 兼容，使用官方 ETH RPC
    polkadotHubTestnet: {
      url: 'https://eth-rpc-testnet.polkadot.io/',
      chainId: 420420417,
      accounts: [process.env.DEPLOYER_PRIVATE_KEY!],
    },
    // HyperLiquid HyperEVM Testnet
    hyperEVMTestnet: {
      url: 'https://rpc.hyperliquid-testnet.xyz/evm',
      chainId: 998,
      accounts: [process.env.DEPLOYER_PRIVATE_KEY!],
    },
    // 本地测试
    localhost: {
      url: 'http://127.0.0.1:8545',
      chainId: 31337,
    },
  },
  polkadot: {
    resolc: {
      version: '0.1.0-dev',
      settings: { optimizer: { enabled: true, runs: 200 } }
    }
  }
};

export default config;
```

### 4.4 VaultDOT.sol 核心接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

// Deployed on: Polkadot Hub TestNet (Chain ID: 420420417)
interface IVaultDOT {
    // User deposits USDC into vault, specifying their HL EVM address
    // Since both chains use same EVM address format, hlAddress == msg.sender in most cases
    function deposit(uint256 amount, address hlAddress) external returns (bytes32 txId);

    // Relayer confirms deposit was forwarded to HyperCore
    function confirmDeposit(bytes32 txId) external; // onlyRelayer

    // Relayer releases USDC to user after HL withdrawal confirmed
    function releaseWithdraw(address user, uint256 amount) external; // onlyRelayer

    // ---- Events ----
    event Deposited(
        address indexed user,
        address indexed hlAddress,
        uint256 amount,
        bytes32 indexed txId
    );
    event WithdrawReleased(address indexed user, uint256 amount);

    // ---- Errors ----
    error VaultDOT__InsufficientBalance();
    error VaultDOT__ExceedsMaxAmount(uint256 amount, uint256 max);
    error VaultDOT__Unauthorized();
}
```

### 4.5 VaultEVM.sol 核心接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

// Deployed on: HyperEVM Testnet (Chain ID: 998)
interface IVaultEVM {
    // Relayer calls this to forward USDC from vault → HyperCore Bridge
    function forwardDeposit(address hlAddress, uint256 amount) external; // onlyRelayer

    // User calls this after withdrawing from HyperCore to EVM
    // Locks USDC in vault and emits WithdrawRequested for Relayer to pick up
    function requestWithdraw(uint256 amount, address dotUserAddress) external returns (bytes32 requestId);

    // Relayer confirms Polkadot Hub side has released funds
    function confirmWithdraw(bytes32 requestId) external; // onlyRelayer

    // ---- Events ----
    event DepositForwarded(address indexed hlAddress, uint256 amount);
    event WithdrawRequested(
        bytes32 indexed requestId,
        address indexed user,
        address dotUserAddress,
        uint256 amount
    );
    event WithdrawConfirmed(bytes32 indexed requestId);
}
```

### 4.6 部署脚本示例

```typescript
// scripts/deploy/01_deploy_vault_dot.ts
import { ethers, upgrades } from 'hardhat';

async function main() {
  const USDC_TESTNET = process.env.USDC_POLKADOT_HUB!;
  const RELAYER_ADDRESS = process.env.RELAYER_ADDRESS!;
  const MAX_PER_TX = ethers.parseUnits('200', 6); // 200 USDC max per tx

  const VaultDOT = await ethers.getContractFactory('VaultDOT');
  const vault = await upgrades.deployProxy(VaultDOT, [
    USDC_TESTNET,
    RELAYER_ADDRESS,
    MAX_PER_TX,
  ], { kind: 'uups' });

  await vault.waitForDeployment();
  console.log('VaultDOT deployed to:', await vault.getAddress());
}

main().catch(console.error);
```

---

## 5. Relayer 服务设计

### 5.1 架构概述

Relayer 是 PolkaHyper 的核心后端服务，负责监听两条链的事件并协调资金流动。Demo 阶段为单节点服务。由于两链均为 EVM 兼容，Relayer 统一使用 **viem** 操作，代码高度复用。

### 5.2 技术栈

| 模块 | 技术选型 | 说明 |
|------|---------|------|
| 运行时 | Node.js 20 LTS + TypeScript | 异步事件驱动 |
| 双链操作 | viem 2.x（统一） | 两链均 EVM 兼容，同一套代码 |
| HL SDK | @nktkas/hyperliquid | HyperCore 充提 API |
| 队列持久化 | better-sqlite3 | 防 Relayer 重启丢失任务 |
| 告警 | Discord Webhook + node-cron | 余额不足/异常通知 |
| 部署 | Docker → Railway / Fly.io | 免费层足够 Demo |

### 5.3 viem 双链客户端配置

```typescript
// relayer/src/clients.ts
import { createPublicClient, createWalletClient, http, defineChain } from 'viem';
import { privateKeyToAccount } from 'viem/accounts';

// Polkadot Hub TestNet — EVM compatible, same as any EVM chain
const polkadotHubTestnet = defineChain({
  id: 420420417,
  name: 'Polkadot Hub TestNet',
  nativeCurrency: { name: 'PAS', symbol: 'PAS', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://eth-rpc-testnet.polkadot.io/'] },
  },
  blockExplorers: {
    default: {
      name: 'Blockscout',
      url: 'https://blockscout-testnet.polkadot.io/',
    },
  },
});

const hyperEVMTestnet = defineChain({
  id: 998,
  name: 'HyperEVM Testnet',
  nativeCurrency: { name: 'HYPE', symbol: 'HYPE', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://rpc.hyperliquid-testnet.xyz/evm'] },
  },
});

const account = privateKeyToAccount(process.env.RELAYER_PRIVATE_KEY as `0x${string}`);

// Polkadot Hub clients
export const dotPublicClient = createPublicClient({ chain: polkadotHubTestnet, transport: http() });
export const dotWalletClient = createWalletClient({ account, chain: polkadotHubTestnet, transport: http() });

// HyperEVM clients
export const hlPublicClient = createPublicClient({ chain: hyperEVMTestnet, transport: http() });
export const hlWalletClient = createWalletClient({ account, chain: hyperEVMTestnet, transport: http() });
```

### 5.4 事件监听器

```typescript
// relayer/src/listeners/polkadotHubListener.ts
import { dotPublicClient } from '../clients';
import { VAULT_DOT_ABI, VAULT_DOT_ADDRESS } from '../constants';
import { db } from '../store/pendingQueue';

export async function startPolkadotHubListener() {
  console.log('[Relayer] Watching VaultDOT on Polkadot Hub TestNet...');

  dotPublicClient.watchContractEvent({
    address: VAULT_DOT_ADDRESS,
    abi: VAULT_DOT_ABI,
    eventName: 'Deposited',
    onLogs: (logs) => {
      for (const log of logs) {
        const { user, hlAddress, amount, txId } = log.args;
        console.log(`[Deposit] ${user} → HL:${hlAddress}, amount: ${amount}`);
        db.insertPendingDeposit({ user, hlAddress, amount: amount.toString(), txId, blockNumber: log.blockNumber });
      }
    },
  });
}
```

### 5.5 充值执行器

```typescript
// relayer/src/executors/depositExecutor.ts
import { dotPublicClient, hlWalletClient } from '../clients';
import { VAULT_EVM_ABI, VAULT_EVM_ADDRESS } from '../constants';
import { db } from '../store/pendingQueue';
import { checkVaultBalance, sendAlert } from '../utils/alert';

export async function processDepositQueue() {
  const pending = db.getPendingDeposits();

  for (const tx of pending) {
    try {
      // 1. Wait for 6 block confirmations on Polkadot Hub
      const currentBlock = await dotPublicClient.getBlockNumber();
      if (currentBlock < tx.blockNumber + 6n) continue;

      // 2. Check HyperEVM vault has enough USDC buffer
      await checkVaultBalance(tx.amount);

      // 3. Call VaultEVM.forwardDeposit on HyperEVM
      const hash = await hlWalletClient.writeContract({
        address: VAULT_EVM_ADDRESS,
        abi: VAULT_EVM_ABI,
        functionName: 'forwardDeposit',
        args: [tx.hlAddress, BigInt(tx.amount)],
      });

      db.markDepositConfirmed(tx.id, hash);
      console.log(`[Deposit] Confirmed: ${tx.txId} → EVM tx: ${hash}`);

    } catch (err) {
      db.markDepositFailed(tx.id, (err as Error).message);
      await sendAlert(`Deposit failed: ${tx.txId} — ${(err as Error).message}`);
    }
  }
}
```

### 5.6 提现执行器

```typescript
// relayer/src/executors/withdrawExecutor.ts
import { hlPublicClient, dotWalletClient } from '../clients';
import { VAULT_DOT_ABI, VAULT_DOT_ADDRESS } from '../constants';
import { db } from '../store/pendingQueue';

export async function processWithdrawQueue() {
  const pending = db.getPendingWithdraws();

  for (const req of pending) {
    try {
      // 1. Wait for 3 block confirmations on HyperEVM
      const currentBlock = await hlPublicClient.getBlockNumber();
      if (currentBlock < req.blockNumber + 3n) continue;

      // 2. Release USDC to user on Polkadot Hub TestNet
      const hash = await dotWalletClient.writeContract({
        address: VAULT_DOT_ADDRESS,
        abi: VAULT_DOT_ABI,
        functionName: 'releaseWithdraw',
        args: [req.dotUserAddress as `0x${string}`, BigInt(req.amount)],
      });

      db.markWithdrawConfirmed(req.id, hash);
      console.log(`[Withdraw] Confirmed: ${req.requestId} → DOT tx: ${hash}`);

    } catch (err) {
      db.markWithdrawFailed(req.id, (err as Error).message);
    }
  }
}
```

### 5.7 主入口

```typescript
// relayer/src/index.ts
import { startPolkadotHubListener } from './listeners/polkadotHubListener';
import { startHyperEVMListener } from './listeners/hyperEVMListener';
import { processDepositQueue } from './executors/depositExecutor';
import { processWithdrawQueue } from './executors/withdrawExecutor';

async function main() {
  console.log('[PolkaHyper Relayer] Starting...');
  await startPolkadotHubListener();
  await startHyperEVMListener();

  // Poll queues every 5 seconds
  setInterval(processDepositQueue, 5_000);
  setInterval(processWithdrawQueue, 5_000);

  console.log('[PolkaHyper Relayer] Ready.');
}

main().catch(console.error);
```

---

## 6. HyperLiquid 测试网集成

### 6.1 测试网网络参数

| 参数 | 值 |
|------|-----|
| API Base URL | `https://api.hyperliquid-testnet.xyz` |
| WebSocket URL | `wss://api.hyperliquid-testnet.xyz/ws` |
| HyperEVM RPC | `https://rpc.hyperliquid-testnet.xyz/evm` |
| HyperEVM Chain ID | `998` |
| HyperCore Bridge 合约 | `0x2Df1c51E09aECF9cacB7bc98cB1742757f163dF7` |
| 区块浏览器 | `https://app.hyperliquid-testnet.xyz` |

### 6.2 测试网代币领取

开发和测试前，需要先领取测试网代币：

**Mock USDC（用于交易保证金）：**
- 🔗 官方 Drip：**https://app.hyperliquid-testnet.xyz/drip**
- 每次领取 **1,000 mock USDC**，每 4 小时可领一次

**HYPE（用于 HyperEVM Gas）：**
- 🔗 QuickNode Faucet：**https://faucet.quicknode.com/hyperliquid/testnet**（每 12 小时 1 HYPE，需推文验证）
- 🔗 Chainstack Faucet：**https://faucet.chainstack.com/hyperliquid-testnet-faucet**（每 24 小时 1 HYPE，需登录）
- 🔗 Gas.zip Faucet：**https://www.gas.zip/faucet/hyperevm**（每 12 小时 0.0025 HYPE）

**PAS（Polkadot Hub TestNet Gas）：**
- 🔗 Polkadot 官方水龙头：**https://faucet.polkadot.io/**
- 选择网络 `Polkadot Hub TestNet`，粘贴你的 EVM 地址（`0x...` 格式）即可

### 6.3 SDK 初始化

```typescript
// frontend/src/services/hyperliquid.ts
import {
  HttpTransport, WebSocketTransport,
  InfoClient, ExchangeClient, SubscriptionClient
} from '@nktkas/hyperliquid';
import { privateKeyToAccount } from 'viem/accounts';

const TESTNET_URL = 'https://api.hyperliquid-testnet.xyz';

// Public info client (no auth needed)
const infoTransport = new HttpTransport({ url: TESTNET_URL });
export const infoClient = new InfoClient({ transport: infoTransport });

// Authenticated exchange client — initialized with wallet signer
export function createExchangeClient(walletClient: any) {
  const transport = new HttpTransport({ url: TESTNET_URL });
  return new ExchangeClient({
    transport,
    wallet: walletClient, // viem WalletClient, not private key
    isTestnet: true,
  });
}

// WebSocket for real-time streams
const wsTransport = new WebSocketTransport({ url: TESTNET_URL });
export const subClient = new SubscriptionClient({ transport: wsTransport });
```

### 6.4 HyperCore 充值（通过 HyperEVM Bridge）

```typescript
// Deposit USDC from HyperEVM into HyperCore
import { parseUnits } from 'viem';
import { HYPERCORE_BRIDGE_ABI } from '../constants/abis';

const HYPERCORE_BRIDGE = '0x2Df1c51E09aECF9cacB7bc98cB1742757f163dF7';
const USDC_HYPEREVM = '0x...'; // HyperEVM testnet USDC address

// Step 1: Approve USDC spend
await walletClient.writeContract({
  address: USDC_HYPEREVM,
  abi: ERC20_ABI,
  functionName: 'approve',
  args: [HYPERCORE_BRIDGE, parseUnits(amount, 6)],
});

// Step 2: Bridge to HyperCore
await walletClient.writeContract({
  address: HYPERCORE_BRIDGE,
  abi: HYPERCORE_BRIDGE_ABI,
  functionName: 'sendToCore', // official bridge function
  args: [userAddress, parseUnits(amount, 6)],
});
```

### 6.5 HyperCore 提现至 EVM

```typescript
// Withdraw USDC from HyperCore back to HyperEVM
await exchangeClient.withdraw3({
  destination: evmAddress,   // 0x EVM address
  amount: '100',             // USDC amount as string
  time: Date.now(),
});
```

### 6.6 下单交易

```typescript
// Place a limit order on HyperLiquid Testnet perp market
await exchangeClient.order({
  orders: [{
    a: 0,                          // asset index (0 = BTC-PERP)
    b: true,                       // is_buy
    p: '95000',                    // limit price
    s: '0.01',                     // size
    r: false,                      // reduce_only
    t: { limit: { tif: 'Gtc' } }, // Good-Till-Cancel
  }],
  grouping: 'na',
});

// Query account state
const state = await infoClient.clearinghouseState({ user: hlAddress });

// Query all mids (mark prices)
const mids = await infoClient.getAllMids();
```

### 6.7 WebSocket 实时数据

```typescript
// src/services/websocket.ts
import { wsManager } from './websocket';

// Subscribe to order book
await subClient.l2Book({ coin: 'BTC' }, (data) => {
  marketStore.updateOrderBook(data);
});

// Subscribe to all mid prices
await subClient.allMids((data) => {
  marketStore.updatePrices(data);
});

// Subscribe to user fills
await subClient.userFills({ user: hlAddress }, (data) => {
  positionStore.updateFills(data);
});
```

---

## 7. 前端设计

### 7.1 技术选型

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 18.x | UI 框架 |
| Vite | 5.x | 构建工具，HMR |
| TypeScript | 5.x | 类型安全 |
| Tailwind CSS | v4 + @tailwindcss/vite | 样式 |
| wagmi | v2 | EVM 钱包连接（支持 MetaMask, Brave, WalletConnect） |
| viem | 2.x | 合约交互、链操作 |
| RainbowKit | latest | 钱包连接 UI 组件 |
| Zustand | 4.x | 全局状态 |
| @tanstack/react-query | 5.x | 服务端状态、轮询 |
| @nktkas/hyperliquid | latest | HL SDK |
| Radix UI | latest | 无障碍组件 |
| lucide-react | 0.263.1 | 图标 |
| i18next | latest | 国际化 zh-CN / en |

### 7.2 wagmi 配置（双链）

```typescript
// src/lib/wagmi.ts
import { createConfig, http } from 'wagmi';
import { defineChain } from 'viem';
import { metaMask, walletConnect } from 'wagmi/connectors';

// Define Polkadot Hub TestNet as a standard EVM chain
const polkadotHubTestnet = defineChain({
  id: 420420417,
  name: 'Polkadot Hub TestNet',
  nativeCurrency: { name: 'PAS', symbol: 'PAS', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://eth-rpc-testnet.polkadot.io/'] },
  },
  blockExplorers: {
    default: {
      name: 'Blockscout',
      url: 'https://blockscout-testnet.polkadot.io/',
    },
  },
  testnet: true,
});

const hyperEVMTestnet = defineChain({
  id: 998,
  name: 'HyperEVM Testnet',
  nativeCurrency: { name: 'HYPE', symbol: 'HYPE', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://rpc.hyperliquid-testnet.xyz/evm'] },
  },
  testnet: true,
});

export const wagmiConfig = createConfig({
  chains: [polkadotHubTestnet, hyperEVMTestnet],
  connectors: [
    metaMask(),
    walletConnect({ projectId: process.env.VITE_WC_PROJECT_ID! }),
  ],
  transports: {
    [polkadotHubTestnet.id]: http(),
    [hyperEVMTestnet.id]: http(),
  },
});

export { polkadotHubTestnet, hyperEVMTestnet };
```

> **说明**：由于两链均为 EVM 兼容，同一个 MetaMask 钱包在两链的地址相同（`0x...`）。用户无需切换钱包，只需在 MetaMask 中切换网络。

### 7.3 前端目录结构

```
frontend/
├── src/
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.tsx            # 顶部导航 + 钱包状态 + 网络切换
│   │   │   ├── Sidebar.tsx           # 左侧市场列表
│   │   │   └── StatusBar.tsx         # 底部 Vault 余额状态
│   │   ├── trading/
│   │   │   ├── TradingPanel.tsx      # 主交易面板
│   │   │   ├── OrderBook.tsx         # 实时订单簿
│   │   │   ├── OrderForm.tsx         # 下单表单（市价/限价）
│   │   │   ├── Positions.tsx         # 持仓列表
│   │   │   └── TradeHistory.tsx      # 成交历史
│   │   ├── vault/
│   │   │   ├── DepositPanel.tsx      # 充值面板（Polkadot Hub → HL）
│   │   │   ├── WithdrawPanel.tsx     # 提现面板（HL → Polkadot Hub）
│   │   │   ├── VaultStatus.tsx       # 双侧 Vault 余额展示
│   │   │   └── TxHistory.tsx         # 充提记录
│   │   └── ui/                       # Button, Input, Toast, Dialog...
│   ├── hooks/
│   │   ├── usePolkadotVault.ts       # VaultDOT 合约交互（wagmi useWriteContract）
│   │   ├── useHyperVault.ts          # VaultEVM 合约交互
│   │   ├── useHLMarket.ts            # 市场行情 + WebSocket
│   │   ├── useHLPositions.ts         # 持仓 + 账户状态
│   │   └── useNetworkSwitch.ts       # 自动/手动切换 MetaMask 网络
│   ├── services/
│   │   ├── hyperliquid.ts            # HL SDK 封装
│   │   └── websocket.ts              # WS 管理器
│   ├── store/
│   │   ├── walletStore.ts            # 钱包状态（address, chainId）
│   │   ├── marketStore.ts            # 行情状态
│   │   └── vaultStore.ts             # 充提状态 + 余额
│   ├── lib/
│   │   ├── wagmi.ts                  # wagmi 配置
│   │   └── contracts/
│   │       └── abis/                 # VaultDOT.json, VaultEVM.json
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   └── format.ts
│   └── App.tsx
├── netlify/
│   └── functions/
│       └── hl-proxy.ts               # HL API CORS 代理
├── vite.config.ts
├── tailwind.config.js
└── package.json
```

### 7.4 网络切换 Hook

```typescript
// src/hooks/useNetworkSwitch.ts
import { useSwitchChain } from 'wagmi';
import { polkadotHubTestnet, hyperEVMTestnet } from '../lib/wagmi';

export function useNetworkSwitch() {
  const { switchChainAsync } = useSwitchChain();

  const switchToPolkadot = () => switchChainAsync({ chainId: polkadotHubTestnet.id });
  const switchToHyperEVM = () => switchChainAsync({ chainId: hyperEVMTestnet.id });

  return { switchToPolkadot, switchToHyperEVM };
}
```

### 7.5 充值 Hook

```typescript
// src/hooks/usePolkadotVault.ts
import { useWriteContract, useWaitForTransactionReceipt, useSwitchChain } from 'wagmi';
import { parseUnits } from 'viem';
import { VAULT_DOT_ADDRESS, VAULT_DOT_ABI } from '../lib/contracts/abis';
import { polkadotHubTestnet } from '../lib/wagmi';

export function useDeposit() {
  const { writeContractAsync } = useWriteContract();
  const { switchChainAsync } = useSwitchChain();

  const deposit = async (amount: string, userAddress: `0x${string}`) => {
    // Auto-switch to Polkadot Hub TestNet
    await switchChainAsync({ chainId: polkadotHubTestnet.id });

    // First approve USDC
    await writeContractAsync({
      address: USDC_POLKADOT_HUB,
      abi: ERC20_ABI,
      functionName: 'approve',
      args: [VAULT_DOT_ADDRESS, parseUnits(amount, 6)],
    });

    // Then deposit — hlAddress == userAddress (same EVM address on both chains)
    const hash = await writeContractAsync({
      address: VAULT_DOT_ADDRESS,
      abi: VAULT_DOT_ABI,
      functionName: 'deposit',
      args: [parseUnits(amount, 6), userAddress],
    });

    return hash;
  };

  return { deposit };
}
```

### 7.6 主界面布局

```
┌──────────────────────────────────────────────────────────────┐
│  PolkaHyper  [⬡ Polkadot Hub ↔ HyperLiquid]  [0xABCD...] 🦊│  ← Header
├────────┬─────────────────────────┬────────────────────────────┤
│Markets │     OrderBook           │  Deposit / Withdraw        │
│BTC-PERP│  ASK   Price   Size     │  ┌─────────────────────┐  │
│ETH-PERP│  ─── 95,200  0.5       │  │ [Deposit] [Withdraw]│  │
│SOL-PERP│      95,100  1.2       │  │ Amount: [__________]│  │
│        │  ─── 95,000 ───        │  │ From: Polkadot Hub  │  │
│        │      94,900  0.8       │  │ To:   HyperLiquid   │  │
│        │      94,800  2.1       │  │ [Confirm Deposit]   │  │
│        │                        │  └─────────────────────┘  │
│        │   Order Form           │                            │
│        │  [Buy Long][Sell Short]│  Vault Status:             │
│        │  Price:  [_________]   │  DOT Hub: 432 USDC ✅      │
│        │  Size:   [_________]   │  HL Vault: 218 USDC ✅     │
│        │  [Place Order]         │                            │
├────────┴─────────────────────────┴────────────────────────────┤
│  Positions | Open Orders | Trade History | Funding Rate        │
└──────────────────────────────────────────────────────────────┘
```

### 7.7 Vite 配置

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    proxy: {
      // Dev proxy for HL API
      '/api/hl': {
        target: 'https://api.hyperliquid-testnet.xyz',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api\/hl/, ''),
      },
    },
  },
});
```

### 7.8 Netlify 配置

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "dist"
  functions = "netlify/functions"

[[redirects]]
  from = "/api/hl/*"
  to = "https://api.hyperliquid-testnet.xyz/:splat"
  status = 200
  force = true

[dev]
  command = "npm run dev"
  port = 5173
```

### 7.9 依赖安装

```bash
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install

# Core Web3
npm install wagmi viem @rainbow-me/rainbowkit @nktkas/hyperliquid

# State management
npm install zustand @tanstack/react-query

# UI
npm install tailwindcss @tailwindcss/vite
npm install @radix-ui/react-dialog @radix-ui/react-tabs @radix-ui/react-toast
npm install clsx tailwind-merge lucide-react class-variance-authority

# i18n
npm install i18next react-i18next
```

---

## 8. 安全设计

### 8.1 合约安全

| 风险 | 等级 | 缓解措施 |
|------|------|---------|
| 重入攻击 | 🔴 高 | ReentrancyGuard + CEI 模式（Check-Effects-Interactions） |
| 权限控制 | 🔴 高 | OpenZeppelin AccessControl，RELAYER_ROLE 单独管理 |
| 资金溢出 | 🟡 中 | SafeERC20 + Solidity 0.8+ 内置溢出检查 |
| 单笔超限 | 🟡 中 | 每笔最大 200 USDC，防止池子被单次耗尽 |
| 双花攻击 | 🔴 高 | txId 幂等校验，已处理的 txId 不可重复执行 |

### 8.2 前端安全

- 私钥永远不接触前端，所有操作通过 MetaMask 签名
- 充值前弹窗二次确认金额与目标地址
- 通过 Netlify Functions 代理 HL API，不暴露原始端点
- 网络切换前检查 Chain ID，防止用户在错误网络操作

### 8.3 Relayer 安全

- 私钥通过环境变量注入，严禁写入代码
- SQLite 记录所有操作日志，完整审计链
- 每笔操作前检查 txId 是否已处理（幂等保护）
- Relayer 账户仅持有 `RELAYER_ROLE`，不持有其他权限

---

## 9. 测试策略

### 9.1 合约测试

| 类型 | 工具 | 覆盖目标 |
|------|------|---------|
| 单元测试 | Hardhat + Chai + viem | 每个合约函数 ≥ 90% |
| 集成测试 | Hardhat local fork | 充提完整流程端到端 |
| 测试网验证 | Polkadot Hub TestNet + HyperEVM Testnet | 真实网络验证 |

### 9.2 关键测试用例

1. **充值正常流程**：用户存入 100 USDC → `Deposited` 事件 → Relayer 转发 → HyperCore 余额增加
2. **提现正常流程**：申请提现 → `WithdrawRequested` → Relayer 释放 → 用户收到 USDC
3. **充值幂等**：相同 txId 不可重复执行 `forwardDeposit`
4. **余额不足**：VaultEVM 余额不足时 Relayer 暂停执行并告警
5. **并发安全**：多笔并发请求按序处理，不双花

### 9.3 测试网部署命令

```bash
# Deploy VaultDOT to Polkadot Hub TestNet
npx hardhat run scripts/deploy/01_deploy_vault_dot.ts \
  --network polkadotHubTestnet

# Deploy VaultEVM to HyperEVM Testnet
npx hardhat run scripts/deploy/02_deploy_vault_evm.ts \
  --network hyperEVMTestnet

# Setup roles on both chains
npx hardhat run scripts/setup/03_setup_roles.ts \
  --network polkadotHubTestnet
npx hardhat run scripts/setup/03_setup_roles.ts \
  --network hyperEVMTestnet
```

---

## 10. 部署流程

### 10.1 环境变量

```bash
# .env.example

# Deployer / Relayer shared key (same EVM address on both chains)
DEPLOYER_PRIVATE_KEY=0x...
RELAYER_PRIVATE_KEY=0x...

# Polkadot Hub TestNet
POLKADOT_HUB_RPC=https://eth-rpc-testnet.polkadot.io/
POLKADOT_HUB_CHAIN_ID=420420417
VAULT_DOT_ADDRESS=0x...
USDC_POLKADOT_HUB=0x...        # USDC contract on Polkadot Hub TestNet

# HyperLiquid Testnet
HYPER_EVM_RPC=https://rpc.hyperliquid-testnet.xyz/evm
HYPER_EVM_CHAIN_ID=998
VAULT_EVM_ADDRESS=0x...
USDC_HYPEREVM=0x...            # USDC contract on HyperEVM
HL_TESTNET_API=https://api.hyperliquid-testnet.xyz
HYPERCORE_BRIDGE=0x2Df1c51E09aECF9cacB7bc98cB1742757f163dF7

# Relayer config
MIN_VAULT_BALANCE=100          # USDC, alert threshold
MAX_TX_AMOUNT=200              # USDC, per-tx limit
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/...

# Frontend
VITE_WC_PROJECT_ID=...         # WalletConnect project ID
```

### 10.2 一键启动（开发）

```bash
# 1. Clone & install
git clone https://github.com/yourname/polkahyper
cd polkahyper

# 2. Install all dependencies
cd contracts && npm install && cd ..
cd relayer  && npm install && cd ..
cd frontend && npm install && cd ..

# 3. Copy and fill in .env
cp .env.example .env

# 4. (Optional) Run local Hardhat node for unit tests
cd contracts && npx hardhat node &

# 5. Start Relayer
cd relayer && npm run dev

# 6. Start Frontend (http://localhost:5173)
cd frontend && npm run dev
```

### 10.3 项目路线图

| 阶段 | 时间 | 里程碑 | 关键交付物 |
|------|------|--------|-----------|
| **Phase 0** 准备期 | Week 1 | 环境搭建 | 双链 Hardhat 配置；测试网账户充值（USDC + HYPE + PAS）；合约接口设计定稿 |
| **Phase 1** 合约 | Week 2 | 合约部署 | VaultDOT + VaultEVM 部署测试网；单元测试通过 ≥ 90% |
| **Phase 2** Relayer | Week 3 | Relayer 上线 | 充提完整流程端到端验证；Discord 告警机制验证 |
| **Phase 3** 前端 | Week 4–5 | 前端 MVP | 交易面板 + 充提面板 + MetaMask 双链切换；WebSocket 实时数据 |
| **Phase 4** Demo | Week 6 | Demo 发布 | Netlify 部署；演示视频录制；文档完善 |

---

## 11. 风险分析

| 风险类型 | 描述 | 等级 | 缓解措施 |
|---------|------|------|---------|
| 流动性风险 | Vault 一侧耗尽无法处理请求 | 🔴 高 | 预充值缓冲 + 余额监控告警 + 单笔 200 USDC 上限 |
| Relayer 宕机 | 服务中断导致资金卡住 | 🟡 中 | SQLite 持久化队列；重启自动恢复；超时自动退款 |
| HyperEVM 不稳定 | 测试网 RPC 不稳定 | 🟡 中 | 指数退避重试；备用 RPC 切换；状态监控 |
| 网络切换错误 | 用户在错误网络操作 | 🟡 中 | 前端自动检测并提示切换；操作前 Chain ID 验证 |
| 私钥泄露 | Relayer 私钥暴露 | 🔴 高 | 环境变量隔离；最小权限；定期轮换 |
| Polkadot Hub 升级 | 测试网升级导致 API 不兼容 | 🟢 低 | 关注官方公告；版本锁定；CI 持续验证 |

---

## 12. 开发规范

### 12.1 通用规范

> ⚠️ **All code, comments, variable names, function names, and commit messages MUST be in English.**

| 规范项 | 要求 |
|--------|------|
| 代码语言 | 全英文（合约 + Relayer + 前端） |
| 开发测试网 | Polkadot Hub TestNet (420420417) + HyperLiquid Testnet (998) |
| 合约语言 | Solidity 0.8.28，EVM 兼容 |
| 前端语言 | TypeScript strict mode |
| Linting | ESLint + Prettier，提交前自动格式化 |
| Commit 规范 | Conventional Commits: `feat/fix/docs/test/chore` |
| 测试覆盖率 | 合约核心逻辑 ≥ 90% |

### 12.2 工具版本锁定

| 工具 | 版本 | 用途 |
|------|------|------|
| Node.js | 20 LTS | Relayer + 前端 |
| Hardhat | ^2.22.x | 合约开发 |
| @parity/hardhat-polkadot | latest stable | Polkadot Hub 兼容 |
| Solidity | 0.8.28 | 合约编译器 |
| React | 18.x | 前端框架 |
| wagmi | v2 | EVM 钱包 |
| viem | 2.x | 链操作 |
| @nktkas/hyperliquid | latest | HL SDK |

---

## 附录

### 附录 A：网络参数速查

**Polkadot Hub TestNet**

| 参数 | 值 |
|------|-----|
| Network Name | `Polkadot Hub TestNet` |
| Chain ID | `420420417` |
| Currency Symbol | `PAS` |
| RPC URL (Parity) | `https://eth-rpc-testnet.polkadot.io/` |
| RPC URL (OpsLayer) | `https://services.polkadothub-rpc.com/testnet/` |
| Block Explorer | `https://blockscout-testnet.polkadot.io/` |
| Faucet | `https://faucet.polkadot.io/` |

**HyperLiquid Testnet**

| 参数 | 值 |
|------|-----|
| Network Name | `HyperEVM Testnet` |
| Chain ID | `998` |
| Currency Symbol | `HYPE` |
| RPC URL | `https://rpc.hyperliquid-testnet.xyz/evm` |
| API URL | `https://api.hyperliquid-testnet.xyz` |
| Block Explorer | `https://app.hyperliquid-testnet.xyz` |
| USDC Faucet | `https://app.hyperliquid-testnet.xyz/drip`（1000 USDC / 4h）|
| HYPE Faucet | `https://faucet.quicknode.com/hyperliquid/testnet` |

### 附录 B：关键合约 ABI 片段（VaultDOT）

```json
[
  {
    "type": "function",
    "name": "deposit",
    "inputs": [
      { "name": "amount", "type": "uint256" },
      { "name": "hlAddress", "type": "address" }
    ],
    "outputs": [{ "name": "txId", "type": "bytes32" }],
    "stateMutability": "nonpayable"
  },
  {
    "type": "event",
    "name": "Deposited",
    "inputs": [
      { "name": "user", "type": "address", "indexed": true },
      { "name": "hlAddress", "type": "address", "indexed": true },
      { "name": "amount", "type": "uint256", "indexed": false },
      { "name": "txId", "type": "bytes32", "indexed": true }
    ]
  }
]
```

### 附录 C：参考链接

**Polkadot 开发文档**
- 连接指南：https://docs.polkadot.com/smart-contracts/connect/
- 钱包集成：https://docs.polkadot.com/smart-contracts/integrations/wallets/
- Hardhat 环境：https://docs.polkadot.com/smart-contracts/dev-environments/hardhat/
- viem 集成：https://docs.polkadot.com/smart-contracts/libraries/viem/
- Blockscout 浏览器：https://blockscout-testnet.polkadot.io/

**HyperLiquid 文档**
- 官方文档：https://hyperliquid.gitbook.io/hyperliquid-docs/
- 测试网 App：https://app.hyperliquid-testnet.xyz
- @nktkas/hyperliquid SDK：https://github.com/nktkas/hyperliquid
- HyperEVM Bridge 合约：`0x2Df1c51E09aECF9cacB7bc98cB1742757f163dF7`

---

*PolkaHyper Design Document v1.1 · 2026-03*
*Bridging Polkadot and HyperLiquid — One Wallet, One Interface, Two Chains.*
