
# Solidity 智能合约 Gas 优化指南（工程实战版）

本文从 **编译层、语言层、存储与数据结构、控制流、EVM 机制、工程化与架构层** 六个维度，系统性总结 Solidity 智能合约的 Gas 优化方法，偏重 **实战可落地** 与 **可量化收益**。

---

## 一、编译与工具链层优化（最低成本，收益立竿见影）

### 1. 启用 Solidity Optimizer

```json
{
  "optimizer": {
    "enabled": true,
    "runs": 200
  }
}
````

**建议**

* 高频调用合约：`runs = 10_000`
* 一次性部署或低频调用：`runs = 200`

**优化效果**

* 函数内联
* 公共子表达式消除
* 常量折叠

---

### 2. 使用最新稳定 Solidity 编译器（0.8.x）

优势：

* 内置 overflow / underflow 检查（无需 SafeMath）
* IR-based optimizer（0.8.13+）生成更优字节码

---

## 二、Storage 优化（最重要，80% Gas 消耗来源）

### 1. 减少 SSTORE / SLOAD 次数

**反例**

```solidity
count += 1;
count += 1;
```

**优化**

```solidity
uint256 c = count;
c += 2;
count = c;
```

---

### 2. Storage Packing（插槽压缩）

**反例（2 个 slot）**

```solidity
uint256 a;
uint256 b;
```

**优化（1 个 slot）**

```solidity
uint128 a;
uint128 b;
```

**规则**

* 同一 slot 不超过 32 bytes
* 变量按 **从小到大** 类型顺序声明

---

### 3. 使用 `immutable` / `constant` 替代 storage

```solidity
address public immutable owner;
uint256 public constant FEE = 30;
```

Gas 行为：

* `immutable`：部署时写入，运行期从 code 读取
* `constant`：直接内联进字节码

---

### 4. 使用 `mapping` 代替数组搜索

**反例**

```solidity
for (uint i; i < users.length; i++) {
    if (users[i] == msg.sender) ...
}
```

**优化**

```solidity
mapping(address => bool) isUser;
```

---

## 三、函数与控制流优化

### 1. `external` 优于 `public`

```solidity
function foo(uint256 x) external;
```

原因：

* 避免 calldata → memory 的隐式拷贝

---

### 2. 使用 `calldata` 替代 `memory`

```solidity
function set(bytes calldata data) external;
```

---

### 3. `unchecked` 块（数学安全可证明时）

```solidity
unchecked {
    ++i;
}
```

收益：

* 每次算术操作节省约 30 gas

---

### 4. Fail Fast（尽早 revert）

```solidity
require(amount != 0);
```

避免后续无效执行消耗 Gas。

---

## 四、循环与批处理优化

### 1. 缓存数组长度

```solidity
uint256 len = arr.length;
for (uint256 i; i < len; ++i) {}
```

---

### 2. 避免不受限循环（尤其是 storage 操作）

**反例**

```solidity
for (...) {
    balances[user] += x;
}
```

**优化策略**

* Off-chain 计算 + Merkle Proof
* 分批处理（batch）并设置上限

---

## 五、EVM / Opcode 级优化（进阶）

### 1. 位运算替代除法 / 乘法

```solidity
x >> 1; // x / 2
x << 1; // x * 2
```

---

### 2. 合理利用短路逻辑

```solidity
if (cheapCheck && expensiveCheck) {}
```

---

### 3. Inline Assembly（谨慎使用）

```solidity
assembly {
    let x := sload(slot)
}
```

适用场景：

* 高频热点路径
* Proxy / storage slot 操作
* hash / memcpy 等底层逻辑

---

## 六、架构级 Gas 优化（协议设计层）

### 1. 使用事件代替存储

```solidity
emit Transfer(from, to, amount);
```

事件成本远低于 storage 写入。

---

### 2. Pull over Push 模式

* 用户主动 claim
* 避免批量分发导致的高 Gas 与失败风险

---

### 3. 使用 EIP-1167 Minimal Proxy

* 部署成本：~1,000,000 gas → ~3,000 gas
* 适用于工厂模式、Vault、Strategy

---

### 4. UUPS / Diamond 架构

* 逻辑复用
* Storage 稳定
* 支持协议升级

---

## 七、常见 Gas 反模式（务必避免）

| 反模式               | 影响 |
| ----------------- | -- |
| 在循环中写 storage     | 极高 |
| 动态 string 拼接      | 高  |
| 重复 keccak256      | 中  |
| public view 被链上调用 | 中  |
| 事件滥用              | 中  |

---

## 八、Gas 优化 Checklist（工程落地）

* [ ] Solidity Optimizer 已启用
* [ ] Storage 是否合理 pack
* [ ] 是否使用 immutable / constant
* [ ] 函数是否使用 external + calldata
* [ ] 是否可以使用 unchecked
* [ ] 循环是否存在上限
* [ ] 是否可 off-chain 计算
* [ ] 是否适合 proxy / minimal clone

---

## 九、推荐工具（工程必备）

* `forge test --gas-report`
* `hardhat-gas-reporter`
* `evm.codes`
* `solc --ir-optimized`
* `tenderly profiler`

---

---

## 十、最新编译器特性与 Gas 优化（Solidity 0.8.20 ~ 0.8.26）

> 以下特性均需设置对应的 `evmVersion` 方可生效。

---

### 1. `PUSH0` 操作码（0.8.20 / Shanghai，EIP-3855）

`PUSH0` 专用于将 `0` 推入栈顶，Gas 为 **2**（替代 `PUSH1 0x00` 的 3 Gas）。  
编译器自动将所有字面量 `0` 替换为 `PUSH0`，无需修改代码。

```toml
# foundry.toml
evm_version = "shanghai"   # 或 "cancun"
```

验证：`forge inspect MyContract bytecode | grep 5f`（`5f` 为 PUSH0 操作码）

---

### 2. EIP-1153 瞬态存储 `tstore` / `tload`（0.8.24 / Cancun）

**瞬态存储**：仅在当前交易期间存在，交易结束后 EVM 自动清零。

| 操作 | Gas |
|---|:---:|
| `SSTORE`（冷，0→非0） | 20,000 |
| `SSTORE`（暖，非0→非0） | 5,000 |
| **`TSTORE`** | **100** |
| `SLOAD`（冷） | 2,100 |
| **`TLOAD`** | **100** |

```solidity
pragma solidity ^0.8.24;

contract ReentrancyGuardTransient {
    uint256 transient private _status;   // 每次交易自动归零，Gas ~100 vs SSTORE ~20000

    modifier nonReentrant() {
        require(_status == 0, "Reentrant");
        _status = 1;
        _;
        _status = 0;
    }
}
```

**三大应用场景**：
- **重入锁**：节省约 ~19,900 Gas/次（vs `bool private _locked` storage 变量）
- **Flash Accounting**：Uniswap V4 用此在 `unlock` 期间记录全局余额变化，避免多次 SSTORE
- **临时中间值**：跨函数临时计算状态，避免 storage 污染

> 需 `evm_version = "cancun"`（Solidity 0.8.25 起设为默认值）

---

### 3. EIP-5656 `MCOPY` 高效内存拷贝（0.8.25 / Cancun）

`MCOPY` 提供高效的内存区域拷贝，替代原有 `mload/mstore` 循环。

| 拷贝长度 | 旧方式 | `MCOPY` |
|---|:---:|:---:|
| 32 bytes | ~9 Gas | ~6 Gas |
| 128 bytes | ~36 Gas | ~12 Gas |
| 1024 bytes | ~288 Gas | ~60 Gas |

**0.8.25+ 编译器在代码生成层自动使用 `mcopy`，无需修改代码**。受益最大的场景：
- `abi.encode` / `abi.encodePacked` 大型数据
- `bytes memory` / `string memory` 参数传递
- 自定义 ABI 编解码

```solidity
// 手动使用（Yul / Inline Assembly）
assembly {
    mcopy(destOffset, srcOffset, length)
}
```

---

### 4. Custom Error 替代 `require` 字符串（0.8.4+，现已成熟）

```solidity
// ❌ 旧方式：字符串存储在 bytecode，部署和 revert Gas 均高
require(amount > 0, "Amount must be positive");

// ✓ 新方式：仅 4 bytes selector，节省 ~50% revert Gas
error InvalidAmount(uint256 provided);
if (amount == 0) revert InvalidAmount(amount);
```

| 对比项 | `require(bool, string)` | Custom Error |
|---|:---:|:---:|
| 部署 Gas | 高（存储字符串字节码） | 低（仅 4B selector） |
| Revert Gas | ~500+ | ~100+ |
| 携带上下文参数 | ✗ | ✓ |
| 工具自动解析 | ✗ | ✓（ABI 友好） |

> Uniswap V4 全库使用 Custom Error，如 `error HookAddressNotValid(address hooks)` 同时节省 Gas 并携带调试信息。

---

### 5. IR 优化管线 `--via-ir`（0.8.13+）

通过 Yul IR 中间层进行优化，比旧版 Opcode-level 优化更深：

```toml
# foundry.toml（生产 profile）
[profile.production]
via_ir = true
optimizer = true
optimizer_runs = 200
```

**额外效果**（相比默认编译）：
- 更激进的函数内联与常量折叠
- 跨函数公共子表达式消除（CSE）
- 未使用内部函数死代码消除
- 通常减少 **5~15% Gas**

> **权衡**：编译时间增加 3~10 倍，建议仅在生产构建时开启。

---

### 6. `assembly ("memory-safe")` 标注（0.8.13+）

```solidity
// 无标注：编译器保守处理，禁止跨块优化
assembly { let x := mload(ptr) }

// 有标注：允许编译器跨 assembly 块做更激进优化
assembly ("memory-safe") {
    let x := mload(ptr)  // 只操作 free-pointer 之上的 allocated 内存
}
```

使用前提：assembly 块内仅访问 0x00–0x3f（scratch space）或 `mload(0x40)` 之后的自由内存。  
> Uniswap V4 的 `Hooks.sol`、`CustomRevert.sol` 均大量使用此标注。

---

### 7. `evmVersion` 版本选择速查

| EVM Version | 最低 Solidity | 关键新 Opcode |
|---|:---:|---|
| `london` | 0.8.7 | — |
| `paris` | 0.8.18 | `PREVRANDAO` |
| `shanghai` | 0.8.20 | `PUSH0`（EIP-3855）|
| `cancun` | 0.8.24 | `TSTORE/TLOAD`（EIP-1153）、`MCOPY`（EIP-5656）|

```toml
# 推荐配置
[profile.default]
solc = "0.8.26"
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
```

---

## 十一、完整 Gas 优化 Checklist（2025 版）

**编译器层**
- [ ] Solidity ≥ 0.8.24，`evm_version = "cancun"`
- [ ] `optimizer = true`，高频合约 `runs = 10000`，低频 `runs = 200`
- [ ] 生产构建考虑 `via_ir = true`

**最新语言特性层**
- [ ] `require(bool, string)` 全量替换为 `custom error`
- [ ] 重入锁改用 `uint256 transient`（0.8.24+，节省 ~19,900 Gas/次）
- [ ] bytes/string 拷贝升级至 0.8.25+ 自动享用 `mcopy`
- [ ] 安全的 assembly 块添加 `("memory-safe")` 标注

**Storage 层**
- [ ] 合理 Packing，减少 slot 占用
- [ ] `immutable` / `constant` 替代频繁 SLOAD
- [ ] 跨函数临时状态改用 `transient`

**函数与控制流层**
- [ ] 对外接口用 `external` + `calldata`
- [ ] 数学安全可证处用 `unchecked`
- [ ] Fail Fast：最廉价的 check 放最前

**架构层**
- [ ] 事件替代高频查询的 storage 写入
- [ ] 批处理 + 分页，避免无限 loop
- [ ] 工厂场景用 EIP-1167 Minimal Clone

---

## 十二、推荐工具（2025 版）

| 工具 | 用途 |
|---|---|
| `forge test --gas-report` | 函数级 Gas 报告 |
| `forge snapshot` | Gas 快照 + CI 回归 |
| `forge inspect <C> gasEstimates` | 函数 Gas 静态估算 |
| `hardhat-gas-reporter` | Hardhat Gas 报告 |
| `evm.codes` | Opcode Gas 成本速查（含 Cancun）|
| `solc --ir-optimized` | 查看 via-ir 优化后 Yul |
| `tenderly profiler` | 指令级 trace 热点分析 |
| `sol2uml` | 存储布局可视化 |

---
