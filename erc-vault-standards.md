# ERC Vault Standards 详解

> **涵盖标准**：ERC-4626 · ERC-7540 · ERC-7575  
> **参考资料**：官方 EIP 原文（eips.ethereum.org）  
> **适用读者**：希望实现或集成 DeFi Vault 的 Solidity 开发者

---

## 目录

1. [背景与设计哲学](#背景与设计哲学)
2. [ERC-4626：代币化 Vault 标准](#erc-4626代币化-vault-标准)
3. [ERC-7540：异步 ERC-4626 Vault](#erc-7540异步-erc-4626-vault)
4. [ERC-7575：多资产 ERC-4626 Vault](#erc-7575多资产-erc-4626-vault)
5. [三者关系与选型指南](#三者关系与选型指南)
6. [安全注意事项](#安全注意事项)
7. [实战：完整骨架代码](#实战完整骨架代码)

---

## 背景与设计哲学

DeFi 中的 **Vault**（金库）是一种智能合约，允许用户将资产存入，换取代表其份额的 **share token**（份额代币）。Vault 自动管理底层资产，通过各种策略（借贷、做市、质押等）产生收益。

在 ERC-4626 出现之前，各协议（Yearn、Compound、Aave 封装层）各自定义存取接口，导致集成复杂。ERC-4626 首次统一了这一接口；ERC-7540 和 ERC-7575 则在此基础上扩展，以应对异步操作和多资产场景。

```
ERC-4626  ──┬──→  ERC-7540（异步 Request/Claim 流程）
            └──→  ERC-7575（多资产 / Share 与 Vault 分离）
```

---

## ERC-4626：代币化 Vault 标准

- **状态**：Final（最终稿）
- **EIP 地址**：https://eips.ethereum.org/EIPS/eip-4626
- **核心思想**：Vault 本身是一个 ERC-20，其 token 代表对底层资产的份额权益。

### 关键定义

| 术语 | 说明 |
|---|---|
| `asset` | 底层 ERC-20 代币（如 USDC、WETH） |
| `shares` | Vault 自身铸造的 ERC-20 份额代币 |
| `totalAssets` | Vault 管理的底层资产总量（含收益） |

### 完整接口

#### 1. 资产与份额换算

```solidity
// 返回底层资产合约地址
function asset() external view returns (address assetTokenAddress);

// Vault 管理的底层资产总量（含复利）
function totalAssets() external view returns (uint256 totalManagedAssets);

// assets → shares 的理论兑换比（不含 fees，向下取整）
function convertToShares(uint256 assets) external view returns (uint256 shares);

// shares → assets 的理论兑换比（不含 fees，向下取整）
function convertToAssets(uint256 shares) external view returns (uint256 assets);
```

#### 2. 存入（Deposit / Mint）

```solidity
// 查询 receiver 最多可存入多少底层资产
function maxDeposit(address receiver) external view returns (uint256 maxAssets);

// 模拟 deposit 操作，返回将获得的 shares（含 deposit fee）
function previewDeposit(uint256 assets) external view returns (uint256 shares);

// 存入 assets，向 receiver 铸造对应 shares
// MUST emit Deposit event
function deposit(uint256 assets, address receiver) external returns (uint256 shares);

// 查询最大可铸造 shares
function maxMint(address receiver) external view returns (uint256 maxShares);

// 模拟 mint 操作，返回需支付的 assets（含 mint fee）
function previewMint(uint256 shares) external view returns (uint256 assets);

// 铸造精确数量的 shares，资产从 msg.sender 划转
// MUST emit Deposit event
function mint(uint256 shares, address receiver) external returns (uint256 assets);
```

#### 3. 提取（Withdraw / Redeem）

```solidity
// 查询 owner 最多可提取多少底层资产
function maxWithdraw(address owner) external view returns (uint256 maxAssets);

// 模拟 withdraw 操作，返回需燃烧的 shares
function previewWithdraw(uint256 assets) external view returns (uint256 shares);

// 提取精确 assets，从 owner 燃烧对应 shares，资产转至 receiver
// MUST emit Withdraw event
function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);

// 查询 owner 最多可赎回多少 shares
function maxRedeem(address owner) external view returns (uint256 maxShares);

// 模拟 redeem 操作，返回将获得的 assets
function previewRedeem(uint256 shares) external view returns (uint256 assets);

// 赎回精确 shares，从 owner 燃烧，assets 转至 receiver
// MUST emit Withdraw event
function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assets);
```

#### 4. 事件

```solidity
// 存入时触发（deposit 和 mint 都需发出）
event Deposit(address indexed sender, address indexed owner, uint256 assets, uint256 shares);

// 提取时触发（withdraw 和 redeem 都需发出）
event Withdraw(
    address indexed sender,
    address indexed receiver,
    address indexed owner,
    uint256 assets,
    uint256 shares
);
```

### deposit vs mint / withdraw vs redeem

| 操作 | 指定数量 | 特点 |
|---|---|---|
| `deposit(assets, ...)` | 指定资产量 | 获得的 shares 由价格决定 |
| `mint(shares, ...)` | 指定份额量 | 支付的 assets 由价格决定 |
| `withdraw(assets, ...)` | 指定提取资产量 | 燃烧的 shares 由价格决定 |
| `redeem(shares, ...)` | 指定赎回份额量 | 获得的 assets 由价格决定 |

### 舍入规则（重要！）

- `convertToShares`：**向下取整**（对用户不利，防止套利）
- `convertToAssets`：**向下取整**
- `previewWithdraw` / `previewMint`：**向上取整**（因为是 Vault 发行方视角）

### 通胀攻击与防护

ERC-4626 存在著名的**首次存款通胀攻击**（Inflation Attack）：

1. 攻击者首先存入 1 wei，获得 1 share。
2. 直接向 Vault 转入大量 assets（不经过 `deposit`），使 `totalAssets` 膨胀。
3. 受害者存入后，`convertToShares` 向下取整，获得 0 shares。

**防护方案**：
- **虚拟份额法**（OpenZeppelin 推荐）：初始化时预存一些"死亡份额"（`_decimalsOffset` 偏移）。
- **最小存款量限制**：要求首次存款不低于某个阈值。
- **锁定初始流动性**：合约部署时 owner 先存入正常量的资产。

```solidity
// OpenZeppelin ERC4626 防通胀攻击（decimals offset）
function _convertToShares(uint256 assets, Math.Rounding rounding)
    internal view virtual returns (uint256)
{
    return assets.mulDiv(
        totalSupply() + 10 ** _decimalsOffset(),
        totalAssets() + 1,
        rounding
    );
}
```

### 最小可行实现（OpenZeppelin）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";

contract SimpleYieldVault is ERC4626 {
    constructor(IERC20 _asset)
        ERC20("Simple Yield Vault Share", "svSHARE")
        ERC4626(_asset)
    {}

    // 收益策略：override _deposit / _withdraw 调用外部协议
    // 这里仅做最简演示
}
```

---

## ERC-7540：异步 ERC-4626 Vault

- **状态**：Draft（草案）
- **EIP 地址**：https://eips.ethereum.org/EIPS/eip-7540
- **核心思想**：扩展 ERC-4626，增加**异步 Request/Claim 两阶段流程**，适用于需要延迟处理的资产（RWA、跨链资产、流动性不足时的赎回等）。

### 背景动机

ERC-4626 要求存取操作**原子完成**——在单笔交易内完成资产转移和份额铸造/燃烧。这对以下场景不适用：

- **RWA（现实世界资产）协议**：底层资产需要 T+1 或 T+2 结算。
- **欠抵押借贷**：赎回需要等待借款人还款。
- **跨链 Vault**：资金需要跨链桥接，有延迟。
- **保险模块**：赎回需要 challenge period。
- **流动性质押**：unstake 需要解绑期（如 28 天）。

### 关键定义

| 术语 | 说明 |
|---|---|
| **Request** | 进入（`requestDeposit`）或退出（`requestRedeem`）Vault 的请求 |
| **Pending** | 已提交 Request 但尚未可被 Claim 的状态 |
| **Claimable** | Request 被 Vault 处理后，用户可以执行 Claim 的状态 |
| **Claimed** | 用户执行 Claim 函数后完成最终状态 |
| **controller** | Request 的拥有者，可管理该 Request |
| **operator** | 被授权代表 controller 管理 Request 的第三方地址 |

### Request 生命周期

以存款为例：

```
requestDeposit(assets, controller, owner)
       ↓ asset.transferFrom(owner, vault, assets)
       ↓ pendingDepositRequest[controller] += assets
    [Pending 状态]
       ↓ （Vault 内部处理，可能跨区块甚至跨天）
       ↓ pendingDepositRequest[controller] -= assets
       ↓ claimableDepositRequest[controller] += assets
    [Claimable 状态]
       ↓ deposit(assets, receiver, controller)  ← 由 controller/operator 调用
       ↓ claimableDepositRequest[controller] -= assets
       ↓ vault.balanceOf[receiver] += shares
    [Claimed 状态]
```

> **注意**：Request **不能** 跳过 Claimable 阶段直接到 Claimed。Vault 不能主动 "push" token 给用户，用户必须主动调用 Claim 函数 "pull"。

### 接口规范

#### 异步存款接口

```solidity
// 提交异步存款请求，资产从 owner 划入 Vault，进入 Pending 状态
// owner 必须是 msg.sender 或已授权 msg.sender 为 operator
// MUST emit DepositRequest event
function requestDeposit(
    uint256 assets,
    address controller,
    address owner
) external returns (uint256 requestId);

// 查询 controller 在 requestId 下处于 Pending 状态的资产量
function pendingDepositRequest(
    uint256 requestId,
    address controller
) external view returns (uint256 assets);

// 查询 controller 在 requestId 下处于 Claimable 状态的资产量
function claimableDepositRequest(
    uint256 requestId,
    address controller
) external view returns (uint256 assets);
```

#### 异步赎回接口

```solidity
// 提交异步赎回请求，shares 从 owner 划入 Vault，进入 Pending 状态
// MUST emit RedeemRequest event
function requestRedeem(
    uint256 shares,
    address controller,
    address owner
) external returns (uint256 requestId);

// 查询 controller 在 requestId 下处于 Pending 状态的 shares 量
function pendingRedeemRequest(
    uint256 requestId,
    address controller
) external view returns (uint256 shares);

// 查询 controller 在 requestId 下处于 Claimable 状态的 assets 量
function claimableRedeemRequest(
    uint256 requestId,
    address controller
) external view returns (uint256 assets);
```

#### Operator 授权

```solidity
// 授权/撤销 operator 管理 msg.sender 的 Requests
function setOperator(address operator, bool approved) external returns (bool);

// 查询 operator 是否被授权管理 controller 的 Requests
function isOperator(address controller, address operator) external view returns (bool);
```

#### 事件

```solidity
// 存款请求事件
event DepositRequest(
    address indexed controller,
    address indexed owner,
    uint256 indexed requestId,
    address sender,
    uint256 assets
);

// 赎回请求事件
event RedeemRequest(
    address indexed controller,
    address indexed owner,
    uint256 indexed requestId,
    address sender,
    uint256 shares
);

// Operator 授权事件
event OperatorSet(address indexed controller, address indexed operator, bool approved);
```

### ERC-4626 方法的 Override 规则

**异步存款 Vault 必须**：
1. `deposit` / `mint` 不再从用户转移资产（资产已在 `requestDeposit` 时转入）。
2. `previewDeposit` 和 `previewMint` **必须 revert**（无法预测价格）。

**异步赎回 Vault 必须**：
1. `redeem` / `withdraw` 不再从用户燃烧 shares（shares 已在 `requestRedeem` 时转入）。
2. `owner` 参数改为 `controller`，调用者须是 `controller` 或其授权的 `operator`。
3. `previewRedeem` 和 `previewWithdraw` **必须 revert**。

### Request ID 设计

`requestId` 用于在 controller 级别区分不同的 Request。常见实现：

- **简单模式**：`requestId = 0`（所有 Request 合并为一个）。
- **批次模式**：`requestId` 对应批次 epoch（用于 RWA 类场景的周期充值）。
- **唯一模式**：每次请求生成唯一 ID（适合需精细追踪的场景）。

### 与 ERC-7575 的集成

```solidity
// ERC-7540 要求同时支持 ERC-7575（用 supportsInterface 声明）
function supportsInterface(bytes4 interfaceId) public view returns (bool) {
    return
        interfaceId == type(IERC7540Deposit).interfaceId ||
        interfaceId == type(IERC7540Redeem).interfaceId ||
        interfaceId == type(IERC7575).interfaceId ||
        interfaceId == type(IERC165).interfaceId;
}
```

### 典型用例：RWA Vault（现实世界资产）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract RWAVault is ERC4626, IERC7540 {
    mapping(address => uint256) public pendingDepositRequest;
    mapping(address => uint256) public claimableDepositRequest;
    mapping(address => mapping(address => bool)) public isOperator;

    address public manager; // 链下 RWA 管理者，负责定期处理请求

    modifier onlyManager() { require(msg.sender == manager); _; }
    modifier onlyControllerOrOperator(address controller) {
        require(msg.sender == controller || isOperator[controller][msg.sender]);
        _;
    }

    // 用户提交存款请求
    function requestDeposit(uint256 assets, address controller, address owner)
        external returns (uint256 requestId)
    {
        require(msg.sender == owner || isOperator[owner][msg.sender]);
        IERC20(asset()).transferFrom(owner, address(this), assets);
        pendingDepositRequest[controller] += assets;
        emit DepositRequest(controller, owner, 0, msg.sender, assets);
        return 0;
    }

    // 链下管理者处理请求（如 RWA 结算后），标记为 Claimable
    function fulfillDepositRequest(address controller, uint256 assets)
        external onlyManager
    {
        pendingDepositRequest[controller] -= assets;
        claimableDepositRequest[controller] += assets;
    }

    // 用户 Claim：将 Claimable 资产转换为 shares
    function deposit(uint256 assets, address receiver, address controller)
        public onlyControllerOrOperator(controller) returns (uint256 shares)
    {
        require(claimableDepositRequest[controller] >= assets);
        claimableDepositRequest[controller] -= assets;
        shares = previewDeposit(assets); // 基于当前汇率计算
        _mint(receiver, shares);
        emit Deposit(controller, receiver, assets, shares);
    }

    function setOperator(address operator, bool approved) external returns (bool) {
        isOperator[msg.sender][operator] = approved;
        emit OperatorSet(msg.sender, operator, approved);
        return true;
    }
}
```

---

## ERC-7575：多资产 ERC-4626 Vault

- **状态**：Draft（草案）
- **EIP 地址**：https://eips.ethereum.org/EIPS/eip-7575
- **核心思想**：扩展 ERC-4626 以支持**多种资产入口**共享同一 share token，并将 ERC-20 依赖从 Vault 合约中分离。

### 背景动机

ERC-4626 要求 Vault 合约本身就是 ERC-20（share token = Vault 合约）。这带来了限制：

1. **单一资产入口**：Vault 只能有一个 `asset()`，无法支持多种抵押品（如同时接受 USDC 和 USDT）。
2. **LP Token Vault**：AMM LP token 对应的 Vault，底层价值是两种资产的组合，不符合 ERC-4626 的单资产模型。
3. **Share 与逻辑合约分离**：很多协议希望 share token 是独立的 ERC-20 合约，便于升级 Vault 逻辑而不改变 share 地址。
4. **Pipe（单向转换器）**：如 wstETH → stETH 的包装合约，本质就是 Vault 但不需要自己是 ERC-20。

### 新增接口

#### `share()` 方法

```solidity
// 返回存款获得的 share token 地址
// share() MAY return address(this)（兼容标准 ERC-4626）
// 若 share() != address(this)，Vault 负责调用 share.mint/burn
function share() external view returns (address shareTokenAddress);
```

#### Share Token 的 `vault()` 查询

```solidity
// 在 ERC-20 share token 合约上实现，返回某资产对应的 Vault 地址
function vault(address asset) external view returns (address vault);

// 当关联的 Vault 地址变更时触发
event VaultUpdate(address indexed asset, address vault);
```

### 三种模式

#### 模式一：标准 ERC-4626（向后兼容）

```
share() == address(this)  →  Vault 本身就是 ERC-20
```

这与原始 ERC-4626 完全一致，`share()` 返回 Vault 自身地址。

#### 模式二：Multi-Asset Vault（多资产入口）

```
shareToken（独立 ERC-20）
   ↑
   ├── VaultA (asset = USDC)  ──  deposit USDC, get shares
   ├── VaultB (asset = USDT)  ──  deposit USDT, get shares
   └── VaultC (asset = DAI)   ──  deposit DAI, get shares
```

- 每个 VaultX 实现完整的 ERC-4626 接口（除 ERC-20 部分）。
- 每个 VaultX 的 `share()` 返回同一个独立 share token 地址。
- share token 实现 `vault(asset)` 查询对应的 Vault 地址。
- 入口合约（VaultX）**不应**自己是 ERC-20。

```solidity
// share token 合约
contract MultiAssetShareToken is ERC20 {
    mapping(address asset => address vault) public vaultByAsset;
    
    function vault(address asset) external view returns (address) {
        return vaultByAsset[asset];
    }
    
    function updateVault(address asset, address _vault) external onlyOwner {
        vaultByAsset[asset] = _vault;
        emit VaultUpdate(asset, _vault);
    }
    
    // Vault 调用：存款时 mint shares 给 receiver
    function mint(address to, uint256 amount) external onlyVault {
        _mint(to, amount);
    }
    
    // Vault 调用：赎回时 burn shares from owner
    function burn(address from, uint256 amount) external onlyVault {
        _burn(from, amount);
    }
}

// USDC 入口 Vault
contract USDCVaultEntry {
    address public immutable shareToken;
    IERC20 public immutable usdc;
    
    function share() external view returns (address) {
        return shareToken;
    }
    
    function asset() external view returns (address) {
        return address(usdc);
    }
    
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        usdc.transferFrom(msg.sender, address(this), assets);
        shares = convertToShares(assets);
        IShareToken(shareToken).mint(receiver, shares);
        emit Deposit(msg.sender, receiver, assets, shares);
    }
    
    // ... 其他 ERC-4626 方法
}
```

#### 模式三：Pipe（单向或双向转换器）

```
asset（ERC-20） ──deposit──→ share（ERC-20）
                ←─redeem──
```

- **单向 Pipe**：只实现 `deposit`/`mint`，不实现 `redeem`/`withdraw`。
- **双向 Pipe**：同时实现存取两侧。
- 典型场景：wstETH ↔ stETH 包装、xToken ↔ Token 线性释放。

```solidity
// 单向 Pipe：将 stETH 包装为 wstETH
contract STETHPipe {
    IERC20 public immutable stETH;
    IWstETH public immutable wstETH;

    function share() external view returns (address) {
        return address(wstETH);
    }

    function asset() external view returns (address) {
        return address(stETH);
    }

    // 单向：存入 stETH，获得 wstETH shares
    function deposit(uint256 assets, address receiver) external returns (uint256 shares) {
        stETH.transferFrom(msg.sender, address(this), assets);
        stETH.approve(address(wstETH), assets);
        shares = wstETH.wrap(assets);
        wstETH.transfer(receiver, shares);
        emit Deposit(msg.sender, receiver, assets, shares);
    }

    // 单向 Pipe 不实现 redeem/withdraw
}
```

### ERC-165 支持

ERC-7575 强制要求实现 ERC-165 的 `supportsInterface`：

```solidity
bytes4 constant ERC7575_INTERFACE_ID = 0x2f0a18c5;

function supportsInterface(bytes4 interfaceId) public view returns (bool) {
    return interfaceId == ERC7575_INTERFACE_ID
        || interfaceId == type(IERC165).interfaceId;
}
```

---

## 三者关系与选型指南

### 继承关系图

```
        ERC-4626（基础）
            │
     ┌──────┴──────┐
     │             │
  ERC-7540      ERC-7575
（异步流程）   （多资产/Share分离）
     │             │
     └──────┬──────┘
            │
    可以同时实现
   （ERC-7540 推荐配合 ERC-7575）
```

### 选型对比

| 特性 | ERC-4626 | ERC-7540 | ERC-7575 |
|---|:---:|:---:|:---:|
| 存取原子完成 | ✅ | ❌（异步） | ✅ |
| 多资产入口 | ❌ | ❌ | ✅ |
| Share 与 Vault 分离 | ❌ | ❌ | ✅ |
| 异步 Request/Claim | ❌ | ✅ | ❌ |
| 需要 ERC-165 | ❌ | ✅ | ✅ |
| 适合 RWA / 跨链 | ❌ | ✅ | △ |
| 适合 AMM LP Vault | ❌ | ❌ | ✅ |
| 向后兼容 ERC-4626 | — | ✅（部分 override） | ✅（`share()` 可返回 self） |

### 选型建议

- **选 ERC-4626**：收益聚合器、简单质押 Vault，存取即时完成，底层是单一 ERC-20。
- **选 ERC-7540**：需要结算延迟的 Vault（RWA、跨链、欠抵押借贷赎回、unstake 等待期）。
- **选 ERC-7575**：多种抵押品 Vault（AMM LP、稳定币篮子），或需要将 Vault 逻辑与 share token 合约分离以便升级。
- **同时实现 ERC-7540 + ERC-7575**：完整的生产级 Vault，如 Centrifuge 的 RWA 协议。

---

## 安全注意事项

### ERC-4626 安全

| 风险 | 说明 | 防护措施 |
|---|---|---|
| 通胀攻击 | 操纵 `totalAssets` 导致 share 价格畸高 | 虚拟份额 / decimals offset（OZ 默认实现） |
| 重入攻击 | `deposit` / `withdraw` 回调恶意合约 | CEI 模式：先更新状态，再执行外部调用 |
| 精度损失 | 整数除法向下取整积累误差 | 使用高精度比率或 virtual offset |
| 价格操纵 | 闪贷操纵 `totalAssets` 进行套利 | 使用 TWA（时间加权平均）或 oracle 价格 |

### ERC-7540 安全

| 风险 | 说明 | 防护措施 |
|---|---|---|
| Operator 滥用 | operator 可代表 controller 提交/取消请求 | 严格限制 setOperator 的使用，建议 multisig |
| 价格波动风险 | Pending 到 Claimable 期间资产价格变化 | 用户需了解无固定汇率保证 |
| 拒绝服务 | 管理者故意不处理 Pending 请求 | 设置超时机制，超时后允许用户取回资产 |
| 请求 ID 冲突 | 不同 requestId 实现可能引起前端混淆 | 统一文档说明 requestId 语义 |

### ERC-7575 安全

| 风险 | 说明 | 防护措施 |
|---|---|---|
| Share Token 权限 | Vault 需要 mint/burn share 的权限 | 严格审计 share token 的 minter/burner 访问控制 |
| Vault 注册欺骗 | 伪造 `vault(asset)` 返回恶意地址 | share token 的 `updateVault` 函数加强权限控制 |
| 多入口汇率不一致 | 多资产 Vault 各入口价格计算逻辑不同 | 统一汇率计算模块，多入口共享同一 oracle |

---

## 实战：完整骨架代码

### 生产级 ERC-4626（含通胀防护）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title ProductionVault
 * @notice 生产级 ERC-4626 Vault，OpenZeppelin 实现已内置虚拟份额防通胀攻击
 */
contract ProductionVault is ERC4626, Ownable {
    address public strategy;

    event StrategyUpdated(address indexed oldStrategy, address indexed newStrategy);

    constructor(IERC20 _asset, address _initialOwner)
        ERC20("Production Vault Share", "pvSHARE")
        ERC4626(_asset)
        Ownable(_initialOwner)
    {}

    // 更新收益策略
    function setStrategy(address _strategy) external onlyOwner {
        emit StrategyUpdated(strategy, _strategy);
        strategy = _strategy;
    }

    // 存款后将资产部署到策略（Hook）
    function _deposit(
        address caller,
        address receiver,
        uint256 assets,
        uint256 shares
    ) internal virtual override {
        super._deposit(caller, receiver, assets, shares);
        if (strategy != address(0)) {
            IERC20(asset()).approve(strategy, assets);
            IStrategy(strategy).deposit(assets);
        }
    }

    // 提款前从策略撤回资产（Hook）
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal virtual override {
        if (strategy != address(0)) {
            IStrategy(strategy).withdraw(assets);
        }
        super._withdraw(caller, receiver, owner, assets, shares);
    }

    // 总资产 = 策略中的资产 + 合约余额
    function totalAssets() public view virtual override returns (uint256) {
        if (strategy != address(0)) {
            return IStrategy(strategy).totalAssets();
        }
        return IERC20(asset()).balanceOf(address(this));
    }
}

interface IStrategy {
    function deposit(uint256 assets) external;
    function withdraw(uint256 assets) external;
    function totalAssets() external view returns (uint256);
}
```

### 简化 ERC-7540 + ERC-7575 联合实现

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/interfaces/IERC165.sol";

/**
 * @title AsyncMultiVault
 * @notice ERC-7540（异步）+ ERC-7575（多资产/share 分离）联合骨架
 */
contract AsyncMultiVault is IERC165 {
    address public immutable shareToken;
    address public immutable assetToken;
    address public manager;

    mapping(address controller => uint256 assets) public pendingDepositRequest;
    mapping(address controller => uint256 assets) public claimableDepositRequest;
    mapping(address controller => uint256 shares) public pendingRedeemRequest;
    mapping(address controller => uint256 assets) public claimableRedeemRequest;
    mapping(address controller => mapping(address operator => bool)) public isOperator;

    // === ERC-7575 ===
    function share() external view returns (address) { return shareToken; }
    function asset() external view returns (address) { return assetToken; }

    // === ERC-7540：异步存款 ===
    function requestDeposit(uint256 assets, address controller, address owner)
        external returns (uint256 requestId)
    {
        require(msg.sender == owner || isOperator[owner][msg.sender], "Unauthorized");
        IERC20(assetToken).transferFrom(owner, address(this), assets);
        pendingDepositRequest[controller] += assets;
        emit DepositRequest(controller, owner, 0, msg.sender, assets);
        return 0;
    }

    // Manager 标记为 Claimable（链下结算后调用）
    function settleDeposit(address controller, uint256 assets) external {
        require(msg.sender == manager, "Only manager");
        pendingDepositRequest[controller] -= assets;
        claimableDepositRequest[controller] += assets;
    }

    // 用户 Claim shares
    function deposit(uint256 assets, address receiver, address controller)
        external returns (uint256 shares)
    {
        require(msg.sender == controller || isOperator[controller][msg.sender], "Unauthorized");
        require(claimableDepositRequest[controller] >= assets, "Not enough claimable");
        claimableDepositRequest[controller] -= assets;
        shares = _convertToShares(assets);
        IShareToken(shareToken).mint(receiver, shares);
        emit Deposit(controller, receiver, assets, shares);
    }

    // === ERC-7540：异步赎回 ===
    function requestRedeem(uint256 shares, address controller, address owner)
        external returns (uint256 requestId)
    {
        require(msg.sender == owner || isOperator[owner][msg.sender], "Unauthorized");
        IShareToken(shareToken).burn(owner, shares);
        pendingRedeemRequest[controller] += shares;
        emit RedeemRequest(controller, owner, 0, msg.sender, shares);
        return 0;
    }

    function settleRedeem(address controller, uint256 shares, uint256 assets) external {
        require(msg.sender == manager, "Only manager");
        pendingRedeemRequest[controller] -= shares;
        claimableRedeemRequest[controller] += assets;
    }

    function redeem(uint256 shares, address receiver, address controller)
        external returns (uint256 assets)
    {
        require(msg.sender == controller || isOperator[controller][msg.sender], "Unauthorized");
        assets = claimableRedeemRequest[controller];
        require(assets > 0, "Nothing claimable");
        claimableRedeemRequest[controller] = 0;
        IERC20(assetToken).transfer(receiver, assets);
        emit Withdraw(controller, receiver, controller, assets, shares);
    }

    // === Operator ===
    function setOperator(address operator, bool approved) external returns (bool) {
        isOperator[msg.sender][operator] = approved;
        emit OperatorSet(msg.sender, operator, approved);
        return true;
    }

    // === ERC-165 ===
    function supportsInterface(bytes4 interfaceId) public pure override returns (bool) {
        return
            interfaceId == 0x2f0a18c5 || // ERC-7575
            interfaceId == type(IERC165).interfaceId;
    }

    // === 内部工具 ===
    function _convertToShares(uint256 assets) internal view returns (uint256) {
        uint256 supply = IShareToken(shareToken).totalSupply();
        uint256 total = IERC20(assetToken).balanceOf(address(this));
        if (supply == 0 || total == 0) return assets;
        return (assets * supply) / total;
    }

    // === 事件 ===
    event DepositRequest(address indexed controller, address indexed owner, uint256 indexed requestId, address sender, uint256 assets);
    event RedeemRequest(address indexed controller, address indexed owner, uint256 indexed requestId, address sender, uint256 shares);
    event OperatorSet(address indexed controller, address indexed operator, bool approved);
    event Deposit(address indexed sender, address indexed receiver, uint256 assets, uint256 shares);
    event Withdraw(address indexed sender, address indexed receiver, address indexed owner, uint256 assets, uint256 shares);
}

interface IShareToken {
    function mint(address to, uint256 amount) external;
    function burn(address from, uint256 amount) external;
    function totalSupply() external view returns (uint256);
}

interface IERC20 {
    function transferFrom(address, address, uint256) external returns (bool);
    function transfer(address, uint256) external returns (bool);
    function balanceOf(address) external view returns (uint256);
}
```

---

## 参考资料

| 标准 | 官方 EIP | 状态 |
|---|---|---|
| ERC-4626 | https://eips.ethereum.org/EIPS/eip-4626 | **Final** |
| ERC-7540 | https://eips.ethereum.org/EIPS/eip-7540 | Draft |
| ERC-7575 | https://eips.ethereum.org/EIPS/eip-7575 | Draft |
| OpenZeppelin ERC4626 | https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol | — |
| Centrifuge ERC-7540 参考实现 | https://github.com/centrifuge/liquidity-pools | — |

---

*文档最后更新：2026-03-04*
