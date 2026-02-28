# Solidity 合约升级代理模式详解

> 本文系统梳理 Solidity 智能合约升级的主流代理模式，涵盖数学原理、存储布局、安全边界与实战代码，适合合约架构设计参考。

---

## 目录

1. [为什么需要升级代理](#一为什么需要升级代理)
2. [代理模式的核心原理](#二代理模式的核心原理)
3. [Transparent Proxy（透明代理）](#三transparent-proxy透明代理)
4. [UUPS Proxy（通用可升级代理）](#四uups-proxy通用可升级代理)
5. [Beacon Proxy（信标代理）](#五beacon-proxy信标代理)
6. [Diamond Proxy / EIP-2535](#六diamond-proxy--eip-2535)
7. [各模式横向对比](#七各模式横向对比)
8. [存储冲突与 EIP-1967 标准槽](#八存储冲突与-eip-1967-标准槽)
9. [初始化安全：`initializer` vs `constructor`](#九初始化安全initializer-vs-constructor)
10. [常见漏洞与防御](#十常见漏洞与防御)

---

## 一、为什么需要升级代理

以太坊合约一经部署，字节码**不可更改**。升级代理通过将**存储**（状态数据）与**逻辑**（代码）分离，实现在同一地址上替换合约逻辑：

```
用户/Dapp      Proxy 合约            Implementation 合约
   │             │  storage 在此         │  logic 在此
   └──── call ──►│──── delegatecall ────►│
                 │◄──── return ──────────│
```

- **代理（Proxy）**：持有状态，地址不变，永久存在
- **实现（Implementation）**：持有逻辑，可随时替换

---

## 二、代理模式的核心原理

### `delegatecall` 语义

```solidity
// 在 Proxy 的上下文中执行 impl 的代码
(bool ok, bytes memory ret) = impl.delegatecall(msg.data);
```

| 属性 | 值 |
|---|---|
| `msg.sender` | 原始调用者（不变） |
| `msg.value` | 原始 ETH（不变） |
| 存储读写 | **Proxy 的存储**（不是 impl 的） |
| 执行代码 | impl 的代码 |

### Fallback 转发模式（最小 Proxy）

```solidity
contract Proxy {
    address public implementation;

    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

---

## 三、Transparent Proxy（透明代理）

> OpenZeppelin 实现：`@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol`

### 设计思想

**透明代理**在 `fallback` 中区分调用者身份：

- **管理员（admin）** 调用 → 执行代理自身的管理函数（如 `upgradeTo`）
- **普通用户** 调用 → `delegatecall` 到实现合约

```solidity
modifier ifAdmin() {
    if (msg.sender == _getAdmin()) {
        _;
    } else {
        _fallback();  // 转发给 implementation
    }
}

function upgradeTo(address newImpl) external ifAdmin {
    _setImplementation(newImpl);
}
```

### 存储布局（EIP-1967）

```solidity
// admin 槽：keccak256("eip1967.proxy.admin") - 1
bytes32 private constant _ADMIN_SLOT =
    0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

// implementation 槽：keccak256("eip1967.proxy.implementation") - 1
bytes32 private constant _IMPLEMENTATION_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
```

### 完整示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";

// 1. 实现合约（V1）
contract BoxV1 {
    uint256 public value;
    function store(uint256 _val) external { value = _val; }
    function version() external pure returns (string memory) { return "v1"; }
}

// 2. 实现合约（V2，新增功能）
contract BoxV2 is BoxV1 {
    function increment() external { value += 1; }
    function version() external pure override returns (string memory) { return "v2"; }
}

// 3. 部署脚本（伪代码）
// ProxyAdmin admin = new ProxyAdmin(msg.sender);
// BoxV1 implV1 = new BoxV1();
// TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(
//     address(implV1), address(admin), ""
// );
// admin.upgradeAndCall(ITransparentUpgradeableProxy(proxy), address(new BoxV2()), "");
```

### 优缺点

| 优点 | 缺点 |
|---|---|
| 概念简单，admin 与业务完全隔离 | 每次调用都需要多一次 SLOAD（区分 admin） |
| admin 无法误调用实现合约函数 | 需要独立的 `ProxyAdmin` 合约管理 |
| OpenZeppelin 成熟支持 | Gas 开销略高 |

---

## 四、UUPS Proxy（通用可升级代理）

> EIP-1822 / OpenZeppelin `UUPSUpgradeable`

### 设计思想

**UUPS（Universal Upgradeable Proxy Standard）** 将升级逻辑**放进实现合约**，而非代理合约。代理本身极简（只负责 `delegatecall` 转发），升级函数 `upgradeTo` 由实现合约提供。

```
Transparent Proxy：升级逻辑在 Proxy 合约中
UUPS Proxy：       升级逻辑在 Implementation 合约中
```

### 合约结构

```solidity
// Proxy：极简，只转发 calldata
contract ERC1967Proxy {
    constructor(address impl, bytes memory data) {
        _setImplementation(impl);
        if (data.length > 0) impl.delegatecall(data);
    }

    fallback() external payable virtual {
        _delegate(_getImplementation());
    }
}

// Implementation：继承 UUPSUpgradeable，自带升级函数
contract MyContractV1 is Initializable, UUPSUpgradeable {
    uint256 public value;

    function initialize(uint256 _val) public initializer {
        __UUPSUpgradeable_init();
        value = _val;
    }

    // 必须实现：控制谁能升级
    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}

    function store(uint256 _val) external { value = _val; }
}
```

### 升级流程

```solidity
// 调用实现合约（通过代理）的 upgradeToAndCall
MyContractV1(address(proxy)).upgradeToAndCall(
    address(new MyContractV2()),
    abi.encodeCall(MyContractV2.initializeV2, ())
);
```

### UUPS 安全检查（防砖块）

OpenZeppelin UUPS 在升级时会验证新实现仍然是 UUPS 兼容的（调用 `proxiableUUID()`），防止升级到不支持继续升级的合约导致合约永久锁死：

```solidity
bytes32 public constant PROXIABLE_UUID =
    keccak256("eip1967.proxy.implementation");

// 升级时检查新合约是否实现了 proxiableUUID
function upgradeToAndCall(address newImpl, bytes memory data) public virtual onlyProxy {
    _authorizeUpgrade(newImpl);
    _upgradeToAndCallUUPS(newImpl, data);
}

function _upgradeToAndCallUUPS(address newImpl, bytes memory data) internal {
    // 检查新 impl 返回正确的 PROXIABLE_UUID，否则 revert
    try UUPSUpgradeable(newImpl).proxiableUUID() returns (bytes32 slot) {
        require(slot == PROXIABLE_UUID, "ERC1967: unsupported proxiableUUID");
    } catch {
        revert("ERC1967: new implementation is not UUPS");
    }
    ERC1967Utils.upgradeToAndCall(newImpl, data);
}
```

### 优缺点

| 优点 | 缺点 |
|---|---|
| Proxy 极简，Gas 最低 | 升级逻辑在 impl，若忘记继承 UUPSUpgradeable 则升级后合约被锁 |
| 无需独立的 ProxyAdmin | 需要开发者严格遵循继承规范 |
| 实现合约可自定义升级权限逻辑 | 比 Transparent 稍复杂 |

---

## 五、Beacon Proxy（信标代理）

> OpenZeppelin `BeaconProxy` + `UpgradeableBeacon`

### 设计思想

多个代理实例共享同一个**信标（Beacon）**，信标存储 `implementation` 地址。升级时只需更新信标一次，所有代理同步生效——适合**工厂模式**（大量同逻辑合约实例）。

```
Beacon
  └── implementation = ImplV1

ProxyA  ──── 读取 Beacon ────►  ImplV1
ProxyB  ──── 读取 Beacon ────►  ImplV1
ProxyC  ──── 读取 Beacon ────►  ImplV1

升级：Beacon.upgradeTo(ImplV2)  → A/B/C 同时切换
```

### 合约结构

```solidity
// 信标合约
contract UpgradeableBeacon {
    address private _implementation;
    address public owner;

    function upgradeTo(address newImpl) external {
        require(msg.sender == owner, "Not owner");
        _implementation = newImpl;
    }

    function implementation() external view returns (address) {
        return _implementation;
    }
}

// 信标代理合约
contract BeaconProxy {
    address private immutable _beacon;

    constructor(address beacon, bytes memory data) {
        _beacon = beacon;
        if (data.length > 0) {
            IBeacon(beacon).implementation().delegatecall(data);
        }
    }

    fallback() external payable {
        address impl = IBeacon(_beacon).implementation();
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

### 工厂模式部署示例

```solidity
contract VaultFactory {
    UpgradeableBeacon public immutable beacon;

    constructor(address impl) {
        beacon = new UpgradeableBeacon(impl);
    }

    // 每次创建一个新的 Vault 实例（共享同一实现）
    function createVault(address owner) external returns (address) {
        BeaconProxy proxy = new BeaconProxy(
            address(beacon),
            abi.encodeCall(Vault.initialize, (owner))
        );
        return address(proxy);
    }

    // 一次升级所有 Vault 实例
    function upgradeAll(address newImpl) external onlyAdmin {
        beacon.upgradeTo(newImpl);
    }
}
```

### 优缺点

| 优点 | 缺点 |
|---|---|
| 批量升级：一次操作升级所有实例 | 每次调用需额外一次 SLOAD 读取 Beacon |
| 天然支持工厂模式 | Beacon 成为单点故障（需严格权限管理） |
| 各实例可保留独立状态 | 所有实例必须同版本（无法按实例差异升级） |

---

## 六、Diamond Proxy / EIP-2535

> EIP-2535：Diamonds, Multi-Facet Proxy

### 设计思想

Diamond 模式允许一个代理合约拥有**多个实现合约（Facets）**，每个 `function selector` 映射到不同的 Facet。克服了单一实现合约的 **24KB 大小限制**，同时支持细粒度升级。

```
Diamond Proxy
    ├── storage（所有状态）
    └── selector → Facet 映射表
           ├── 0xaabbccdd → FacetA（Token 功能）
           ├── 0x11223344 → FacetB（治理功能）
           └── 0x55667788 → FacetC（流动性功能）
```

### 核心数据结构

```solidity
// EIP-2535 标准存储布局
library LibDiamond {
    // Diamond 存储槽，避免冲突
    bytes32 constant DIAMOND_STORAGE_POSITION =
        keccak256("diamond.standard.diamond.storage");

    struct FacetAddressAndPosition {
        address facetAddress;
        uint96  functionSelectorPosition; // 在 facetFunctionSelectors 中的位置
    }

    struct FacetFunctionSelectors {
        bytes4[] functionSelectors;
        uint256  facetAddressPosition;    // 在 facetAddresses 中的位置
    }

    struct DiamondStorage {
        // selector => facet 地址 + 位置
        mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
        // facet 地址 => 其所有 selectors
        mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
        // 所有 facet 地址列表
        address[] facetAddresses;
        // 合约是否支持某接口
        mapping(bytes4 => bool) supportedInterfaces;
        // 合约拥有者
        address contractOwner;
    }

    function diamondStorage() internal pure returns (DiamondStorage storage ds) {
        bytes32 position = DIAMOND_STORAGE_POSITION;
        assembly { ds.slot := position }
    }
}
```

### `diamondCut`：升级操作

```solidity
enum FacetCutAction { Add, Replace, Remove }

struct FacetCut {
    address facetAddress;       // 新 Facet 地址（Remove 时为 address(0)）
    FacetCutAction action;
    bytes4[] functionSelectors; // 要操作的 selectors
}

interface IDiamondCut {
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,           // 初始化合约地址（可选）
        bytes calldata _calldata // 初始化调用数据（可选）
    ) external;
}
```

### `fallback` 路由

```solidity
fallback() external payable {
    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
    address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
    require(facet != address(0), "Diamond: function not found");
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

### Diamond 存储模式（AppStorage vs LibStorage）

```solidity
// 方法一：AppStorage（统一结构体）
struct AppStorage {
    uint256 totalSupply;
    mapping(address => uint256) balances;
    address owner;
    // 所有 Facet 共享同一个 AppStorage
}
// 每个 Facet 通过同一个 slot 访问
AppStorage internal s;  // 默认 slot 0 开始

// 方法二：LibStorage（每个 Facet 独立存储槽）
library LibToken {
    bytes32 constant STORAGE_SLOT = keccak256("app.token.storage");
    struct Storage {
        uint256 totalSupply;
        mapping(address => uint256) balances;
    }
    function getStorage() internal pure returns (Storage storage s) {
        bytes32 slot = STORAGE_SLOT;
        assembly { s.slot := slot }
    }
}
```

> **推荐**：LibStorage 更安全，各 Facet 存储隔离，避免升级时意外覆盖。

### 优缺点

| 优点 | 缺点 |
|---|---|
| 突破 24KB 合约大小限制 | 架构复杂，理解和审计难度高 |
| 细粒度升级（单 selector 级别） | Facet 间需要严格管理共享存储 |
| 单一地址暴露所有功能 | `diamondCut` 本身是高危操作 |
| 支持无限扩展 | 工具链支持相对较少 |

---

## 七、各模式横向对比

| 维度 | Transparent | UUPS | Beacon | Diamond |
|---|:---:|:---:|:---:|:---:|
| **Gas（部署）** | 中 | 低（Proxy极简） | 中 | 高 |
| **Gas（调用）** | 中（admin检查） | 低 | 中（读Beacon） | 中（查映射） |
| **升级权限位置** | Proxy 中 | Implementation 中 | Beacon 中 | Diamond 中 |
| **批量升级** | ✗ | ✗ | ✓ | ✗ |
| **突破24KB限制** | ✗ | ✗ | ✗ | ✓ |
| **细粒度升级** | ✗ | ✗ | ✗ | ✓（selector级） |
| **实现复杂度** | 低 | 中 | 中 | 高 |
| **OZ官方支持** | ✓ | ✓ | ✓ | 社区 |
| **适用场景** | 通用 | Gas敏感 | 工厂/批量 | 大型/模块化 |

---

## 八、存储冲突与 EIP-1967 标准槽

### 问题：存储槽冲突

代理合约和实现合约共享存储，若 Proxy 的变量（如 `implementation`、`admin`）占用了 slot 0、slot 1，而实现合约的变量也从 slot 0 开始，就会发生**冲突**。

```
Proxy slot 0: implementation address  ← 与 ImplV1.value 冲突！
Proxy slot 1: admin address           ← 与 ImplV1.owner 冲突！

ImplV1 slot 0: value (uint256)
ImplV1 slot 1: owner (address)
```

### 解决方案：EIP-1967 随机槽

通过 `keccak256(string) - 1` 计算一个极难碰撞的存储槽：

```solidity
// EIP-1967 标准槽
// Implementation: keccak256("eip1967.proxy.implementation") - 1
bytes32 constant IMPLEMENTATION_SLOT =
    0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

// Admin: keccak256("eip1967.proxy.admin") - 1
bytes32 constant ADMIN_SLOT =
    0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103;

// Beacon: keccak256("eip1967.proxy.beacon") - 1
bytes32 constant BEACON_SLOT =
    0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50;

// 读写操作
function _getImplementation() internal view returns (address impl) {
    assembly { impl := sload(IMPLEMENTATION_SLOT) }
}

function _setImplementation(address impl) internal {
    assembly { sstore(IMPLEMENTATION_SLOT, impl) }
}
```

> **`- 1` 的作用**：防止已知输入的 `keccak256` 值被恶意构造的 `mapping` 键碰撞到（哈希抗碰撞保证）。

---

## 九、初始化安全：`initializer` vs `constructor`

### 为什么不能用 `constructor`

实现合约的 `constructor` 在其自身的存储上执行，**不会**在代理的存储上运行。代理的存储初始化必须通过普通函数（`initialize`）配合 `delegatecall` 完成。

```solidity
// ❌ 错误：constructor 中的状态仅在实现合约存储生效
contract MyContractV1 {
    address public owner;
    constructor(address _owner) {
        owner = _owner;  // 这个 owner 写在 impl 合约的存储，不在 Proxy 存储
    }
}

// ✓ 正确：通过 initializer 初始化
contract MyContractV1 is Initializable {
    address public owner;

    function initialize(address _owner) public initializer {
        owner = _owner;  // 通过 proxy.delegatecall → 写在 Proxy 存储
    }
}
```

### `Initializable` 的防重入保护

```solidity
contract Initializable {
    // 存储槽：keccak256("openzeppelin.storage.Initializable")
    uint8 private _initialized;   // 当前版本号
    bool  private _initializing;  // 是否正在初始化（防递归）

    modifier initializer() {
        bool isTopLevelCall = !_initializing;
        require(
            (isTopLevelCall && _initialized < 1) ||
            (!AddressUpgradeable.isContract(address(this)) && _initialized == 1),
            "Already initialized"
        );
        _initialized = 1;
        if (isTopLevelCall) _initializing = true;
        _;
        if (isTopLevelCall) _initializing = false;
    }

    // 用于 V2+ 升级时的再初始化
    modifier reinitializer(uint8 version) {
        require(!_initializing && _initialized < version, "Already initialized");
        _initialized = version;
        _initializing = true;
        _;
        _initializing = false;
    }
}
```

### 升级时初始化链（`__X_init` 规范）

```solidity
// OZ 约定：每个父合约提供 __ContractName_init 和 __ContractName_init_unchained
contract MyContractV1 is Initializable, OwnableUpgradeable, ERC20Upgradeable {
    function initialize(string memory name, string memory symbol) public initializer {
        __Ownable_init(msg.sender);       // 负责调用父链 init
        __ERC20_init(name, symbol);       // 同上
        // __MyContractV1_init_unchained() 仅初始化本合约变量
    }
}
```

---

## 十、常见漏洞与防御

### 1. 未经授权的 `initialize`

```solidity
// ❌ 漏洞：实现合约本身未被初始化，攻击者可调用 initialize 控制 impl
contract VulnerableImpl is Initializable, UUPSUpgradeable {
    function initialize() public initializer { /* ... */ }
    function _authorizeUpgrade(address) internal override {}  // 没有权限检查！
}

// ✓ 修复：在 impl 的 constructor 中锁定
constructor() { _disableInitializers(); }
```

### 2. 存储布局破坏

```solidity
// ❌ 错误：V2 在 V1 变量之间插入新变量，破坏存储布局
contract V1 { uint256 public a; uint256 public b; }
contract V2 { uint256 public a; uint256 public NEW; uint256 public b; }
//                                       ^^^^ 插在中间，b 的 slot 错位！

// ✓ 正确：只在末尾追加新变量
contract V2 { uint256 public a; uint256 public b; uint256 public NEW; }

// ✓ 或使用 storage gap（预留槽）
contract V1Base {
    uint256 public a;
    uint256[49] private __gap;  // 预留 49 个 slot，共 50 个
}
contract V2Base is V1Base {
    uint256 public NEW;         // 占用 __gap[0]，原有变量不移位
    uint256[48] private __gap; // 更新 gap
}
```

### 3. `selfdestruct` 销毁实现合约

```solidity
// ❌ 漏洞：攻击者直接调用 impl 的 selfdestruct（非通过代理）
function kill() external { selfdestruct(payable(msg.sender)); }

// ✓ 修复：在实现合约中禁止直接调用（仅允许通过代理调用）
modifier onlyProxy() {
    require(address(this) != _IMPL_ADDRESS, "Direct call not allowed");
    _;
}
```

### 4. UUPS 升级到非 UUPS 合约（合约砖块）

```solidity
// ❌ 漏洞：升级到一个没有 upgradeToAndCall 的普通合约，永久无法再升级
proxy.upgradeToAndCall(address(new NonUUPSContract()), "");

// ✓ OZ 内置防御：upgradeTo 时调用 proxiableUUID() 验证
// 新合约必须继承 UUPSUpgradeable 并返回正确的 PROXIABLE_UUID
```

### 5. 函数选择器冲突（Diamond）

```solidity
// ❌ 漏洞：两个 Facet 实现同名函数，后注册的覆盖前者
// FacetA: function transfer(address, uint256) → selector 0xa9059cbb
// FacetB: function transfer(address, uint256) → selector 0xa9059cbb  ← 冲突

// ✓ 修复：diamondCut 前必须检查 selector 冲突
function _addFunctions(address facet, bytes4[] memory selectors) internal {
    for (uint i; i < selectors.length; i++) {
        require(
            ds.selectorToFacetAndPosition[selectors[i]].facetAddress == address(0),
            "Selector already exists"
        );
    }
}
```

---

## 附录：推荐工具与参考

| 工具/资源 | 说明 |
|---|---|
| [OpenZeppelin Contracts Upgradeable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable) | 升级代理官方实现 |
| [OpenZeppelin Upgrades Plugin](https://docs.openzeppelin.com/upgrades-plugins/) | Hardhat/Foundry 升级插件，自动检查存储布局 |
| [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) | 标准代理存储槽规范 |
| [EIP-1822（UUPS）](https://eips.ethereum.org/EIPS/eip-1822) | UUPS 标准规范 |
| [EIP-2535（Diamond）](https://eips.ethereum.org/EIPS/eip-2535) | Diamond 标准规范 |
| [Nick Mudge Diamond 实现](https://github.com/mudgen/diamond-3-hardhat) | 参考实现 |
| `forge inspect <Contract> storage` | Foundry 查看合约存储布局 |
| `oz upgrades validate` | 自动验证升级安全性 |
