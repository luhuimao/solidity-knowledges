````markdown
# Solidity Storage Slot（存储插槽）完整解析

本文从 **EVM 存储模型 → Solidity Slot 分配规则 → 复杂类型 Slot 计算 → 工程实践与安全风险** 四个层次，系统梳理 Solidity 中的 **Storage Slot（存储插槽）** 知识，适用于 **高级智能合约工程师 / 面试 / 安全审计 / Proxy 设计**。

---

## 一、EVM Storage 基础

### 1. Storage 的本质

- EVM Storage 是一个 **持久化的 Key-Value 存储**
- 形式为：

```text
mapping(uint256 => bytes32)
````

| 项目    | 说明            |
| ----- | ------------- |
| Key   | uint256（slot） |
| Value | 32 bytes      |
| 范围    | 0 ~ 2²⁵⁶-1    |
| 生命周期  | 永久上链          |

> 每一个 `slot` 本质上就是一个 32 字节的存储单元。

---

### 2. Slot 的基本特性

| 特性   | 说明                      |
| ---- | ----------------------- |
| 固定大小 | 每个 slot = 32 bytes      |
| 稀疏   | 未写入的 slot 默认为 0         |
| 成本高  | `SSTORE` 是最昂贵的 EVM 操作之一 |
| 合约隔离 | 不同合约 storage 完全隔离       |

---

## 二、Solidity 状态变量 Slot 分配规则

### 总体规则

> **Solidity 按变量声明顺序，从 slot 0 开始顺序分配**

```solidity
contract A {
    uint256 a; // slot 0
    uint256 b; // slot 1
    uint256 c; // slot 2
}
```

---

## 三、值类型（Value Types）的 Slot 规则

### 1. 独占 Slot 的类型

以下类型通常 **独占一个 slot**：

* uint256 / int256
* bytes32
* struct（展开后）
* mapping（仅占位）

---

### 2. Slot Packing（紧凑存储）

小于 32 bytes 的类型会 **尽量合并存储在同一个 slot**。

#### 示例 1

```solidity
contract A {
    uint128 a; // slot 0 (0-15)
    uint128 b; // slot 0 (16-31)
}
```

#### 示例 2

```solidity
contract B {
    uint8  a;  // slot 0 byte 0
    bool   b;  // slot 0 byte 1
    uint16 c;  // slot 0 byte 2-3
    uint256 d; // slot 1
}
```

#### Packing 规则总结

* 仅对 **连续声明的变量** 生效
* 从低位（右侧）开始填充
* 一旦变量无法放入当前 slot，则开启新 slot

---

## 四、复杂类型的 Slot 计算规则（重点）

---

## 1️⃣ 定长数组（Fixed-size Array）

```solidity
uint256[3] arr;
```

| 内容     | Slot     |
| ------ | -------- |
| arr[0] | slot N   |
| arr[1] | slot N+1 |
| arr[2] | slot N+2 |

> 定长数组 **元素连续存储**

---

## 2️⃣ 动态数组（Dynamic Array）

```solidity
uint256[] arr;
```

### Storage 布局

| 内容         | 存储位置             |
| ---------- | ---------------- |
| arr.length | slot N           |
| arr[0]     | keccak256(N)     |
| arr[1]     | keccak256(N) + 1 |

#### 示例

```solidity
contract A {
    uint256[] arr; // slot 0
}
```

```text
arr.length -> slot 0
arr[0]     -> keccak256(0)
```

---

## 3️⃣ Mapping（映射）【核心】

```solidity
mapping(address => uint256) balances;
```

### Slot 规则

* mapping 本身 **只占一个 slot（N）**
* 不存储任何实际数据
* 实际数据 slot 计算公式：

```text
slot = keccak256(abi.encode(key, N))
```

#### 示例

```solidity
balances[user] =
keccak256(abi.encode(user, slot_of_balances))
```

### Mapping 特点

| 特性       | 说明       |
| -------- | -------- |
| 不可遍历     | 无法枚举 key |
| slot 可预测 | 可离线计算    |
| 安全敏感     | 常见攻击切入点  |

---

## 4️⃣ Struct（结构体）

### 4.1 Struct 作为状态变量

```solidity
struct User {
    uint256 id;
    address addr;
}

User user;
```

```text
user.id   -> slot N
user.addr -> slot N+1
```

---

### 4.2 Struct + Mapping

```solidity
mapping(address => User) users;
```

Slot 计算：

```text
base = keccak256(abi.encode(key, slot))
users[key].id   -> base
users[key].addr -> base + 1
```

---

## 五、Storage Slot 的工程实践

---

## 1️⃣ Proxy 合约与 Storage Collision

### 问题本质

* `delegatecall` **共享 storage**
* Proxy 与 Implementation slot 必须 **严格一致**

```text
Slot 对不上 = 数据错位 = 资产损失
```

---

### EIP-1967 标准 Slot

```solidity
bytes32 constant IMPLEMENTATION_SLOT =
keccak256("eip1967.proxy.implementation") - 1;
```

#### 优点

* 避开 slot 0
* 防止普通变量覆盖
* 成为行业标准

---

## 2️⃣ 手动指定 Slot（Assembly）

```solidity
bytes32 internal constant SLOT = keccak256("my.unique.slot");

function get() external view returns (uint256 v) {
    assembly {
        v := sload(SLOT)
    }
}
```

#### 常见用途

* Proxy
* Diamond Storage（EIP-2535）
* Upgrade-safe 变量管理

---

## 六、常见 Storage Slot 陷阱

| 错误                | 后果           |
| ----------------- | ------------ |
| 升级合约中插入变量         | storage 直接错位 |
| struct 字段顺序调整     | 历史数据破坏       |
| packing 误判        | 变量被覆盖        |
| mapping slot 计算错误 | 读写异常         |

---

## 七、Slot 心算示例（面试高频）

```solidity
contract A {
    uint128 a;
    uint64  b;
    uint64  c;
    mapping(address => uint256) m;
}
```

### 分析结果

```text
slot 0:
  a (16 bytes)
  b (8 bytes)
  c (8 bytes)

slot 1:
  m (mapping 占位)
```

```text
m[key] = keccak256(abi.encode(key, 1))
```

---

## 八、高级工程师能力标准

你应该能够：

* 手算任意合约的 storage layout
* 准确推导 mapping / array slot 公式
* 设计 upgrade-safe storage
* 从 slot 角度分析安全漏洞
* 理解 Proxy / Diamond 的存储隔离机制

---

## 九、可延伸学习方向

* Foundry / Hardhat 输出 storage layout
* EIP-1967 / EIP-1822 / EIP-2535 实战
* Storage Collision 真实事故复盘
* Slot 级别的攻击与防御设计

---

```