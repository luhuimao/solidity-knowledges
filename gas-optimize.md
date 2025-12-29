
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
