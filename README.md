# Solidity 智能合约知识库

> 系统性梳理 Solidity / EVM 核心原理，面向高级合约工程师与安全审计方向。内容偏工程实战，每篇文档均含代码示例、对比表与 Checklist。

---

### [foundry-fuzz-testing.md](./foundry-fuzz-testing.md)
**Solidity Fuzz 测试完整指南**

使用 Foundry 对智能合约进行模糊测试，包括：
- **三种 Fuzz 模式**：Property-Based 属性测试、Invariant 不变量测试、Stateful 有状态序列
- **`vm.assume` vs `bound`**：输入过滤的正确姿势，防止测试效率下降
- **Handler + Ghost Variables 完整实现**：封装合法操作、幼灵变量追踪、不变量断言
- **失败用例复现与调试**：固定种子、最小化反例、覆盖率报告
- **三种测试方式对比**：单元测试 / Fuzz / 形式化验证选型指南
- **Fuzz 测试 Checklist**：代码覆盖目标、CI 配置、常见不变量模板

---

## 🗂️ 文档结构

### [proxy-upgrade-patterns.md](./proxy-upgrade-patterns.md)
**Solidity 合约升级代理模式详解**

系统梳理四种主流升级代理架构，包括：
- **Transparent Proxy**：`ifAdmin` 调用者区分、EIP-1967 标准槽、ProxyAdmin 管理
- **UUPS Proxy**：极简代理、`_authorizeUpgrade`、`proxiableUUID` 防砖块验证
- **Beacon Proxy**：信标批量升级、工厂模式部署示例
- **Diamond / EIP-2535**：`FacetCut` 操作、selector 路由、AppStorage vs LibStorage
- 各模式横向对比（Gas、升级权限、批量升级、24KB 限制等）
- EIP-1967 存储槽设计原理（`keccak256 - 1` 防碰撞）
- 初始化安全（`_disableInitializers`、`__gap` 预留槽）
- 常见漏洞与防御（未授权 initialize、存储布局破坏、UUPS 砖块、selector 冲突）

---

### [storage-slot.md](./storage-slot.md)
**Solidity Storage Slot（存储插槽）完整解析**

从 EVM 存储模型到复杂类型计算规则，包括：
- EVM Storage 基础（持久化 KV 存储、Slot 特性）
- 值类型 Slot 分配与 Slot Packing（紧凑存储规则）
- 复杂类型 Slot 计算：定长数组、动态数组、Mapping、Struct
- Proxy 合约 Storage Collision 问题与 EIP-1967 解决方案
- `assembly` 手动指定 Slot（Diamond / Proxy 场景）
- 常见存储陷阱（升级插变量、Struct 字段调整、packing 误判）
- 高频面试 Slot 心算示例

---

### [gas-optimize.md](./gas-optimize.md)
**Solidity 智能合约 Gas 优化指南（工程实战版）**

从六个维度系统总结 Gas 优化方法，包括：
- **编译层**：Optimizer 配置、IR-based 优化器
- **Storage 层**：减少 SSTORE/SLOAD、插槽压缩、`immutable` / `constant`
- **函数与控制流**：`external` + `calldata`、`unchecked`、Fail Fast
- **循环与批处理**：缓存数组长度、off-chain 计算 + Merkle Proof
- **EVM/Opcode 层**：位运算、短路逻辑、Inline Assembly
- **架构层**：事件代替存储、Pull over Push、EIP-1167 Minimal Proxy
- Gas 反模式表 & 工程落地 Checklist

---

### [opcodes.md](./opcodes.md)
**Solidity 操作码（EVM Opcodes）系统性知识指南**

从 EVM 执行模型到实战安全，适合进阶开发与审计：
- EVM 三大组件（Stack / Memory / Storage）与执行模型
- Opcode 分类总览（算术、比较、位运算、存储、调用等）
- 关键 Opcode 详解：`SLOAD/SSTORE` Gas 成本、`CALL/DELEGATECALL` 语义、`KECCAK256` 优化
- Solidity 语法 → Opcode 映射（加法、Storage 读写、`require`）
- Memory vs Storage Opcode 对比
- `unchecked`、`external`、Storage Packing 的 Opcode 本质
- Inline Assembly 优势与风险
- 安全审计重点：重入（CALL）、DELEGATECALL 注入、SELFDESTRUCT
- 调试工具：`evm.codes`、`forge inspect`、`tenderly`

---

### [智能合约攻击原理与防御方案.md](./智能合约攻击原理与防御方案.md)
**智能合约攻击原理与防御方案（工程实践版）**

涵盖 DeFi 生态中 15 类主流攻击向量，每类均含原理 + 代码 + 解决方案：

| # | 攻击类型 | 核心防御 |
|:---:|---|---|
| 1 | 重入攻击 | CEI 模式、ReentrancyGuard |
| 2 | 整数溢出 / 精度 | Solidity ≥0.8、WAD/RAY 精度 |
| 3 | 权限控制缺陷 | 分层权限、多签 + Timelock |
| 4 | Delegatecall 存储冲突 | EIP-1967 固定槽、namespace storage |
| 5 | 未初始化合约 | `_disableInitializers`、`initializer` |
| 6 | 闪电贷攻击 | TWAP、delay、风控上限 |
| 7 | 预言机攻击 | 多源 + Chainlink + TWAP 窗口 |
| 8 | AMM 价格操纵 | minOut、滑点限制、Batch Auction |
| 9 | 清算 / 利率攻击 | 平滑曲线、清算激励上限 |
| 10 | 治理攻击 | Timelock、投票冷却期 |
| 11 | MEV / 前置交易 | Commit-Reveal、私有交易池 |
| 12 | DoS 攻击 | 分页 + try/catch |
| 13 | 时间 / 区块依赖 | VRF、避免随机性依赖 |
| 14 | 签名重放 | EIP-712、per-user nonce |
| 15 | 跨链桥攻击 | 多层验证、延迟提款 |

---

### [solidity-advanced-production.md](./solidity-advanced-production.md)
**Solidity 生产级实战问题解析**

针对 5 个高阶工程场景，每篇均含完整代码实现：
- **Staking 合约优化**：Synthetix 全局奖励指数（O(1) 分发）、Storage Packing、解质押批量队列
- **Foundry Fuzzing**：基础 Fuzz、闪电贷攻击模拟、Invariant 不变量测试、权限滥用测试
- **UUPS 升级实战**：升级前暂停、Storage Layout 兼容验证、时间锁防权限泄露
- **ERC-6551 跨链迁移**：状态快照与哈希验证、两阶段提交、交易回滚恢复
- **安全漏洞应急响应**：多签紧急暂停、资金隔离、PoC 复现、修复后上线 Checklist

---

```
solidity-knowledges/
├── README.md                              # 本文件
├── proxy-upgrade-patterns.md             # 代理升级模式（Transparent / UUPS / Beacon / Diamond）
├── storage-slot.md                       # Storage Slot 完整解析
├── gas-optimize.md                       # Gas 优化工程实战
├── opcodes.md                            # EVM Opcodes 系统指南
├── 智能合约攻击原理与防御方案.md            # 15 类攻击向量与防御
├── solidity-advanced-production.md       # 生产级实战：Staking / Fuzzing / UUPS / ERC-6551 / 应急响应
└── foundry-fuzz-testing.md               # Foundry Fuzz 测试完整指南
```

---

## 🔗 延伸阅读

| 资源 | 说明 |
|---|---|
| [evm.codes](https://evm.codes) | 所有 Opcode 说明与 Gas 成本 |
| [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) | 标准代理存储槽规范 |
| [EIP-2535](https://eips.ethereum.org/EIPS/eip-2535) | Diamond 标准规范 |
| [OpenZeppelin Upgrades](https://docs.openzeppelin.com/upgrades-plugins/) | 升级插件文档 |
| [Foundry](https://book.getfoundry.sh) | `forge test --gas-report`、`forge inspect storage` |
