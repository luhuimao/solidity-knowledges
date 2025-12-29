
# Solidity 操作码（EVM Opcodes）系统性知识指南

本文从 **EVM 执行模型 → 操作码分类 → 关键 Opcode 详解 → Gas 成本 → Solidity 映射关系 → 实战与安全** 六个层面，系统讲清 Solidity 背后的 **EVM 操作码机制**。内容偏工程与审计视角，适合进阶合约开发者。

---

## 一、EVM 与操作码基础模型

### 1. EVM 是什么
- **Ethereum Virtual Machine**
- 基于 **栈（Stack）** 的虚拟机
- 每条指令 = 1 个 **Opcode**
- 所有 Solidity / Vyper / Huff 最终都会编译为 Opcode

---

### 2. EVM 的三大核心组件

| 组件 | 说明 |
|----|----|
| Stack | 1024 深度，32 bytes/slot |
| Memory | 临时内存，按 word 扩容 |
| Storage | 持久化 KV（slot → value） |

---

### 3. Opcode 执行模型

```text
PUSH → STACK
OP    → STACK
SSTORE → STORAGE
RETURN / REVERT
````

* 栈：计算
* Memory：中间态
* Storage：最贵的持久状态

---

## 二、Opcode 分类总览

| 分类         | 示例                   |
| ---------- | -------------------- |
| Arithmetic | ADD, SUB, MUL, DIV   |
| Comparison | LT, GT, EQ, ISZERO   |
| Bitwise    | AND, OR, XOR, SHL    |
| Stack      | PUSH, POP, DUP, SWAP |
| Memory     | MLOAD, MSTORE        |
| Storage    | SLOAD, SSTORE        |
| Flow       | JUMP, JUMPI          |
| Context    | CALLER, CALLVALUE    |
| Hash       | KECCAK256            |
| Call       | CALL, DELEGATECALL   |
| Return     | RETURN, REVERT       |

---

## 三、最重要的 Opcode（Gas & 安全核心）

### 1. `SLOAD` / `SSTORE`（最昂贵）

#### SLOAD

```text
读取 storage slot
Gas ≈ 2100
```

#### SSTORE（分情况）

| 场景        | Gas             |
| --------- | --------------- |
| 0 → 非 0   | 20,000          |
| 非 0 → 非 0 | 5,000           |
| 非 0 → 0   | 5,000（+ refund） |

**结论**

> 减少 storage 写 = 最大 Gas 优化点

---

### 2. `CALL` / `DELEGATECALL`

```text
CALL          → 切换上下文
DELEGATECALL  → 保持 storage / msg.sender
```

**代理合约核心**

* Transparent / UUPS / Diamond 都依赖 `DELEGATECALL`

---

### 3. `KECCAK256`

```text
keccak256(memory[start : start+size])
```

Gas：

* 固定成本 + memory 扩展成本

**优化点**

* 避免 loop 中重复 hash
* storage slot 计算应缓存

---

### 4. `JUMP` / `JUMPI`

* EVM 没有 if / for
* Solidity 所有控制流 → jump

```text
JUMPI(condition, destination)
```

---

## 四、Solidity 代码 → Opcode 映射示例

### 示例 1：简单加法

```solidity
function add(uint a, uint b) pure returns (uint) {
    return a + b;
}
```

核心 Opcode：

```text
PUSH
ADD
RETURN
```

---

### 示例 2：Storage 读写

```solidity
uint x;

function set(uint v) external {
    x = v;
}
```

Opcode 关键路径：

```text
PUSH slot
PUSH value
SSTORE
```

---

### 示例 3：require

```solidity
require(x > 0);
```

等价逻辑：

```text
GT
ISZERO
JUMPI
REVERT
```

---

## 五、Memory vs Storage Opcode 对比

| 操作            | Opcode | Gas    |
| ------------- | ------ | ------ |
| Memory Read   | MLOAD  | 3      |
| Memory Write  | MSTORE | 3      |
| Storage Read  | SLOAD  | 2100   |
| Storage Write | SSTORE | 5k–20k |

**结论**

> 能用 memory 绝不用 storage

---

## 六、常见 Solidity 语法对应 Opcode

| Solidity          | Opcode       |
| ----------------- | ------------ |
| `msg.sender`      | CALLER       |
| `msg.value`       | CALLVALUE    |
| `block.timestamp` | TIMESTAMP    |
| `address(this)`   | ADDRESS      |
| `balance`         | BALANCE      |
| `selfdestruct`    | SELFDESTRUCT |

---

## 七、Gas 优化与 Opcode 的直接关系

### 1. unchecked 的本质

```solidity
unchecked { a + b; }
```

减少：

```text
ISZERO + JUMPI + REVERT
```

---

### 2. external vs public

* `public`：calldata → memory → stack
* `external`：calldata → stack

**减少 MLOAD / MSTORE**

---

### 3. Storage Packing

* 多变量 → 同一 slot
* SLOAD / SSTORE 次数直接下降

---

## 八、Inline Assembly 与 Opcode

```solidity
assembly {
    let v := sload(slot)
    sstore(slot, add(v, 1))
}
```

优势：

* 减少冗余指令
* 精确控制 gas

风险：

* 无类型检查
* 易引入 slot collision

---

## 九、Opcode 相关安全风险（审计重点）

### 1. 重入（CALL）

```text
CALL → 控制权外泄
```

* 必须 **Checks-Effects-Interactions**

---

### 2. DELEGATECALL 注入

* Storage 污染
* Proxy 实现合约被接管

---

### 3. SELFDESTRUCT

* 强制转账
* 合约逻辑终止

---

### 4. CALLCODE（已废弃）

* 历史漏洞来源
* 审计需识别

---

## 十、Opcode 学习与调试工具

| 工具             | 用途              |
| -------------- | --------------- |
| evm.codes      | Opcode 说明 & Gas |
| forge inspect  | 查看字节码           |
| solc --opcodes | 反汇编             |
| tenderly       | 指令级 trace       |
| hevm / dapp    | 低层调试            |

---

## 十一、工程级理解总结

> **Solidity 是语法糖，Opcode 才是执行真相**

* Gas 优化 = 减少昂贵 Opcode
* 安全漏洞 = Opcode 组合不当
* 架构设计 = 对 CALL / SSTORE 的控制

---

## 十二、进阶方向（建议顺序）

1. 熟记高频 Opcode（SLOAD / SSTORE / CALL）
2. 能手写关键 Opcode 执行路径
3. 阅读 Proxy / ERC20 的 opcode trace
4. 学习 Huff / Yul
5. 从 opcode 角度做审计

---

