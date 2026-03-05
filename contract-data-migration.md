# 智能合约数据迁移方案大全

> **关键词**：合约升级、数据迁移、Storage Layout、代理模式、Merkle Proof  
> **适用场景**：合约漏洞修复、业务逻辑重构、存储结构调整、跨链迁移  
> **文档日期**：2026-03-05

---

## 目录

1. [为什么需要数据迁移？](#1-为什么需要数据迁移)
2. [迁移方案全景对比](#2-迁移方案全景对比)
3. [方案一：代理升级模式（零迁移）](#3-方案一代理升级模式零迁移)
4. [方案二：链上逐步迁移（Lazy Migration）](#4-方案二链上逐步迁移lazy-migration)
5. [方案三：链下快照 + Merkle Proof 迁移](#5-方案三链下快照--merkle-proof-迁移)
6. [方案四：全量数据复制（Owner Migration）](#6-方案四全量数据复制owner-migration)
7. [方案五：双合约并行过渡（Staging）](#7-方案五双合约并行过渡staging)
8. [存储布局兼容性验证](#8-存储布局兼容性验证)
9. [迁移安全设计](#9-迁移安全设计)
10. [完整迁移 Checklist](#10-完整迁移-checklist)

---

## 1. 为什么需要数据迁移？

区块链数据是**不可变**的——已部署的合约代码和存储数据无法直接修改。需要迁移的常见场景：

| 场景 | 说明 |
|---|---|
| 发现安全漏洞 | 必须部署新合约修复，旧合约中的用户资产/状态需转移 |
| 业务逻辑重构 | 添加新功能要求改变存储结构 |
| 存储冲突/槽碰撞 | 代理模式下存储布局被破坏 |
| 性能优化 | 数据结构重新设计，降低 Gas |
| 跨链部署 | 将 L1 状态复制到 L2 |
| ERC 标准升级 | 从非标准接口迁移到 ERC-4626、ERC-7540 等 |

---

## 2. 迁移方案全景对比

| 方案 | 链上改动 | Gas 成本 | 用户感知 | 适用规模 | 核心原理 |
|---|:---:|:---:|:---:|:---:|---|
| 代理升级（UUPS/Transparent） | 极小 | 极低 | 无感 | 全规模 | 存储不变，换逻辑合约 |
| 懒惰迁移（Lazy Migration） | 中 | 低（按需） | 首次操作略慢 | 大规模 | 读时迁移，摊销成本 |
| 快照 + Merkle Proof | 中 | 低（仅认领时） | 主动认领 | 大规模 | 链下证明，链上验证 |
| 全量数据复制 | 高 | 极高 | 可暂停服务 | 小规模 | 脚本逐条写入新合约 |
| 双合约并行过渡 | 中 | 中 | 有切换期 | 中规模 | 新旧并存，逐步引流 |

> **核心原则**：能用代理升级解决的，绝不做数据迁移；真的需要迁移时，优先使用 Merkle Proof 或懒惰迁移降低链上成本。

---

## 3. 方案一：代理升级模式（零迁移）

**最推荐的方案**。不迁移数据——存储原地保留，只更换执行逻辑。

### 前提条件

- 旧合约必须是可升级代理（UUPS / Transparent / Beacon）
- 新旧逻辑合约存储变量顺序**严格向后兼容**

### UUPS 升级流程

```solidity
// 旧逻辑合约 V1
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract TokenVaultV1 is UUPSUpgradeable, OwnableUpgradeable {
    // ── Storage Layout（必须保持不变）──
    mapping(address => uint256) public balances; // slot 0 (after OZ gaps)
    uint256 public totalDeposited;               // slot 1
    // ────────────────────────────────────

    function initialize(address _owner) external initializer {
        __Ownable_init(_owner);
        __UUPSUpgradeable_init();
    }

    function deposit(uint256 amount) external {
        balances[msg.sender] += amount;
        totalDeposited += amount;
    }

    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}
}
```

```solidity
// 新逻辑合约 V2：增加 pausable 功能，不改变已有 storage
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/utils/PausableUpgradeable.sol";

contract TokenVaultV2 is UUPSUpgradeable, OwnableUpgradeable, PausableUpgradeable {
    // ── Storage Layout（原有变量位置严格不变）──
    mapping(address => uint256) public balances; // slot 0 ← 不变！
    uint256 public totalDeposited;               // slot 1 ← 不变！
    // 新增变量只能追加到末尾
    uint256 public feeRate;                      // slot 2 ← 新增
    // ────────────────────────────────────────────

    /// @notice 升级后初始化新变量（只能调用一次）
    function initializeV2(uint256 _feeRate) external reinitializer(2) {
        feeRate = _feeRate;
        __Pausable_init();
    }

    function deposit(uint256 amount) external whenNotPaused {
        uint256 fee = (amount * feeRate) / 10000;
        balances[msg.sender] += amount - fee;
        totalDeposited += amount;
    }

    function _authorizeUpgrade(address newImpl) internal override onlyOwner {}
}
```

```typescript
// scripts/upgrade.ts —— Hardhat + OpenZeppelin Upgrades Plugin
import { ethers, upgrades } from "hardhat";

async function main() {
    const PROXY_ADDRESS = "0x...";  // 代理合约地址（固定不变）

    console.log("Deploying V2 implementation...");
    const TokenVaultV2 = await ethers.getContractFactory("TokenVaultV2");

    // 升级：自动验证存储兼容性 + 部署新实现合约 + 更新代理指针
    const vault = await upgrades.upgradeProxy(PROXY_ADDRESS, TokenVaultV2, {
        call: {
            fn: "initializeV2",
            args: [50], // feeRate = 0.5%
        }
    });

    await vault.waitForDeployment();
    console.log("Upgrade complete. Proxy still at:", PROXY_ADDRESS);
}

main().catch(console.error);
```

### 存储兼容性自动检测

```bash
# OpenZeppelin Upgrades Plugin 在部署时自动检测不兼容存储变更
# 如果触发以下情况会阻止升级：
# - 变量顺序改变
# - 变量类型改变（uint256 → uint128）
# - 删除已有变量
# - 在中间位置插入新变量

# 使用 storage-layout 命令手动对比
forge inspect TokenVaultV1 storage-layout
forge inspect TokenVaultV2 storage-layout
```

---

## 4. 方案二：链上逐步迁移（Lazy Migration）

**原理**：新合约部署后，不立即迁移所有数据；用户首次与新合约交互时，自动从旧合约读取其数据并在新合约中写入。

**适用场景**：
- 无法代理升级（旧合约未部署代理）
- 数据量庞大，不适合一次性全量迁移
- 存储结构需要调整（如 struct 字段重排）

### 实现示例

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IOldVault {
    function balances(address user) external view returns (uint256);
    function lastActivity(address user) external view returns (uint256);
}

contract NewVault {
    IOldVault public immutable oldVault;

    struct UserInfo {
        uint256 balance;
        uint256 lastActivity;
        bool    migrated;    // 标记是否已迁移
    }

    mapping(address => UserInfo) public users;

    // 迁移状态事件
    event UserMigrated(address indexed user, uint256 balance);

    constructor(address _oldVault) {
        oldVault = IOldVault(_oldVault);
    }

    // ─── 内部：懒加载迁移 ───────────────────────────────────
    modifier withMigration(address user) {
        _migrateIfNeeded(user);
        _;
    }

    function _migrateIfNeeded(address user) internal {
        if (users[user].migrated) return;

        // 首次访问：从旧合约读取数据
        uint256 oldBalance = oldVault.balances(user);
        uint256 oldActivity = oldVault.lastActivity(user);

        // 写入新合约存储（旧合约可以为 0，允许新用户直接注册）
        users[user] = UserInfo({
            balance:      oldBalance,
            lastActivity: oldActivity > 0 ? oldActivity : block.timestamp,
            migrated:     true
        });

        if (oldBalance > 0) {
            emit UserMigrated(user, oldBalance);
        }
    }
    // ────────────────────────────────────────────────────────

    function deposit(uint256 amount) external withMigration(msg.sender) {
        users[msg.sender].balance += amount;
        users[msg.sender].lastActivity = block.timestamp;
    }

    function withdraw(uint256 amount) external withMigration(msg.sender) {
        require(users[msg.sender].balance >= amount, "Insufficient balance");
        users[msg.sender].balance -= amount;
    }

    function getBalance(address user) external withMigration(user) returns (uint256) {
        return users[user].balance;
    }

    // 批量预迁移（owner 可主动推送，降低用户首次操作 Gas）
    function batchMigrate(address[] calldata usersToMigrate) external {
        for (uint256 i = 0; i < usersToMigrate.length; i++) {
            _migrateIfNeeded(usersToMigrate[i]);
        }
    }
}
```

### 注意事项

```
⚠️ 旧合约数据必须只读（部署新合约后立即暂停旧合约的写入操作）
⚠️ 旧合约地址必须是 immutable（防止管理员篡改数据源）
⚠️ 首次操作 Gas 成本略高（包含迁移写入 SSTORE × 字段数）
✅ 迁移完成后 Gas 与普通操作无差别
```

---

## 5. 方案三：链下快照 + Merkle Proof 迁移

**原理**：在特定区块高度对旧合约状态做快照，链下计算 Merkle 树，在新合约中存储 Merkle Root。用户通过提交 Merkle Proof 认领属于自己的资产，无需管理员逐个操作。

**适用场景**：
- 用户体量极大（数万~数百万地址）
- 希望将迁移 Gas 成本转嫁给用户
- Token/NFT 空投、余额快照转移

### 流程

```
1. 暂停旧合约（停止状态变化）
2. 链下：读取快照区块的所有用户余额（eth_call / subgraph）
3. 链下：构建 Merkle 树，计算 Merkle Root
4. 部署新合约，传入 Merkle Root
5. 用户调用新合约 claim()，提交 Merkle Proof 认领余额
```

### 链下：构建 Merkle 树（TypeScript）

```typescript
import { MerkleTree } from "merkletreejs";
import { ethers } from "ethers";
import { keccak256 } from "ethers/lib/utils";

interface Snapshot {
    address: string;
    balance: bigint;
}

// 从链上读取快照（可用 subgraph 或遍历 Transfer 事件）
async function fetchSnapshot(contractAddr: string, blockNumber: number): Promise<Snapshot[]> {
    // 省略：遍历 Transfer 事件计算每个地址余额
    return [];
}

function buildMerkleTree(snapshot: Snapshot[]) {
    // 叶节点 = keccak256(abi.encodePacked(address, uint256))
    // 注意：对叶节点再做一次 hash 防止二次原像攻击
    const leaves = snapshot.map(({ address, balance }) =>
        keccak256(
            ethers.utils.solidityPack(["address", "uint256"], [address, balance])
        )
    );

    const tree = new MerkleTree(leaves, keccak256, { sortPairs: true });
    const root = tree.getHexRoot();

    console.log("Merkle Root:", root);

    // 为每个地址生成 proof
    const proofs: Record<string, string[]> = {};
    snapshot.forEach(({ address, balance }, i) => {
        proofs[address] = tree.getHexProof(leaves[i]);
    });

    return { root, proofs };
}

// 导出 proof 给前端使用
import fs from "fs";
const snapshot = await fetchSnapshot("0x...", 21000000);
const { root, proofs } = buildMerkleTree(snapshot);
fs.writeFileSync("snapshot-proofs.json", JSON.stringify({ root, proofs }, null, 2));
```

### 链上：新合约验证 Merkle Proof

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MigrationVault is Ownable {
    bytes32 public immutable merkleRoot;
    IERC20  public immutable token;

    // 防止重复认领
    mapping(address => bool) public claimed;

    // 迁移截止时间（过期后 owner 可取回未认领资产）
    uint256 public immutable deadline;

    event Claimed(address indexed user, uint256 amount);

    constructor(
        bytes32 _merkleRoot,
        address _token,
        uint256 _deadline,
        address _owner
    ) Ownable(_owner) {
        merkleRoot = _merkleRoot;
        token = IERC20(_token);
        deadline = _deadline;
    }

    /**
     * @notice 用户凭 Merkle Proof 认领旧合约中属于自己的余额
     * @param amount    快照中记录的余额（必须与构建 Merkle 树时一致）
     * @param proof     由链下工具生成的 Merkle Proof 数组
     */
    function claim(uint256 amount, bytes32[] calldata proof) external {
        require(block.timestamp <= deadline, "Claim period ended");
        require(!claimed[msg.sender], "Already claimed");

        // 重建叶节点：keccak256(keccak256(abi.encodePacked(address, uint256)))
        // 双重 hash 防止二次原像攻击（与 OpenZeppelin MerkleProof 规范一致）
        bytes32 leaf = keccak256(
            bytes.concat(
                keccak256(abi.encodePacked(msg.sender, amount))
            )
        );

        // 验证 proof
        require(
            MerkleProof.verify(proof, merkleRoot, leaf),
            "Invalid proof"
        );

        claimed[msg.sender] = true;
        token.transfer(msg.sender, amount);
        emit Claimed(msg.sender, amount);
    }

    /**
     * @notice 迁移期结束后，owner 取回未认领资产
     */
    function recoverUnclaimed() external onlyOwner {
        require(block.timestamp > deadline, "Claim period not ended");
        token.transfer(owner(), token.balanceOf(address(this)));
    }

    /**
     * @notice 批量验证（只读，接口供前端检查 proof 是否有效）
     */
    function verifyProof(
        address user,
        uint256 amount,
        bytes32[] calldata proof
    ) external view returns (bool) {
        bytes32 leaf = keccak256(
            bytes.concat(keccak256(abi.encodePacked(user, amount)))
        );
        return MerkleProof.verify(proof, merkleRoot, leaf);
    }
}
```

### 前端：查询并提交 Proof

```typescript
import proofData from "./snapshot-proofs.json";

async function claimMigration(signer: ethers.Signer, vaultAddress: string) {
    const userAddress = await signer.getAddress();
    const proof = proofData.proofs[userAddress];

    if (!proof) {
        console.log("No balance in snapshot for this address");
        return;
    }

    // 从 snapshot 中读取该用户的余额（与构建树时一致）
    const amount = proofData.balances[userAddress];

    const vault = new ethers.Contract(vaultAddress, MigrationVaultABI, signer);
    const tx = await vault.claim(amount, proof);
    await tx.wait();
    console.log("Claimed successfully!");
}
```

---

## 6. 方案四：全量数据复制（Owner Migration）

**原理**：由 Owner/多签 通过脚本将旧合约数据逐条写入新合约，适合数据量较小的场景。

**适用场景**：
- 用户量小（< 1000 地址）
- 有充足资金支付迁移 Gas
- 可接受短暂停服

```solidity
// 新合约提供批量写入接口（仅迁移期间开放）
contract NewVaultWithMigration {
    mapping(address => uint256) public balances;
    bool public migrationComplete;
    address public immutable migrationAdmin;

    event BatchMigrated(uint256 count);
    event MigrationFinalized();

    modifier onlyDuringMigration() {
        require(!migrationComplete, "Migration already complete");
        require(msg.sender == migrationAdmin, "Not migration admin");
        _;
    }

    constructor(address _admin) {
        migrationAdmin = _admin;
    }

    /**
     * @notice 批量初始化用户余额（仅迁移期间可调用）
     * @dev 每批建议 ≤ 200 条（防止超出 gas limit）
     */
    function batchSetBalances(
        address[] calldata users,
        uint256[] calldata amounts
    ) external onlyDuringMigration {
        require(users.length == amounts.length, "Length mismatch");
        require(users.length <= 200, "Batch too large");

        for (uint256 i = 0; i < users.length; i++) {
            require(balances[users[i]] == 0, "Already migrated");
            balances[users[i]] = amounts[i];
        }

        emit BatchMigrated(users.length);
    }

    /**
     * @notice 迁移结束：禁用批量写入，开放正常业务
     */
    function finalizeMigration() external onlyDuringMigration {
        migrationComplete = true;
        emit MigrationFinalized();
    }

    // 正式保留之后的业务逻辑...
}
```

```typescript
// scripts/migrate-all.ts —— 链下批量迁移脚本
import { ethers } from "ethers";
import { chunk } from "lodash";

async function migrateAll() {
    const provider = new ethers.JsonRpcProvider(process.env.RPC_URL);
    const wallet = new ethers.Wallet(process.env.PRIVATE_KEY!, provider);

    const oldVault = new ethers.Contract(OLD_VAULT, OldVaultABI, provider);
    const newVault = new ethers.Contract(NEW_VAULT, NewVaultABI, wallet);

    // 1. 从链上事件或 subgraph 获取所有用户地址
    const depositEvents = await oldVault.queryFilter(oldVault.filters.Deposit());
    const allUsers = [...new Set(depositEvents.map(e => e.args!.user))];

    console.log(`Found ${allUsers.length} users`);

    // 2. 读取每个用户余额（可并发）
    const balances = await Promise.all(
        allUsers.map(async (user) => ({
            user,
            balance: await oldVault.balances(user)
        }))
    );

    // 过滤余额为 0 的用户
    const nonZero = balances.filter(b => b.balance > 0n);
    console.log(`Users with balance: ${nonZero.length}`);

    // 3. 分批写入（每批 200 条）
    const batches = chunk(nonZero, 200);
    for (let i = 0; i < batches.length; i++) {
        const batch = batches[i];
        const users = batch.map(b => b.user);
        const amounts = batch.map(b => b.balance);

        console.log(`Migrating batch ${i + 1}/${batches.length}...`);
        const tx = await newVault.batchSetBalances(users, amounts, {
            gasLimit: 5_000_000
        });
        await tx.wait();
        console.log(`Batch ${i + 1} done. tx: ${tx.hash}`);
    }

    // 4. 校验：抽查几个用户
    for (const { user, balance: oldBalance } of nonZero.slice(0, 5)) {
        const newBalance = await newVault.balances(user);
        console.assert(newBalance === oldBalance, `Mismatch for ${user}!`);
    }

    // 5. 完成迁移，锁定批量写入接口
    await newVault.finalizeMigration();
    console.log("Migration complete!");
}

migrateAll().catch(console.error);
```

---

## 7. 方案五：双合约并行过渡（Staging）

**原理**：新合约部署后，新旧两个合约同时接受操作，Router 合约负责分流；逐步将用户引流到新合约，待旧合约用户量降到零时下线。

**适用场景**：
- 不能容许任何服务中断
- 需要灰度验证新合约功能
- ERC-20 Transfer 场景（Token 地址不能变）可用 wrapper 过渡

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IVault {
    function deposit(address user, uint256 amount) external;
    function withdraw(address user, uint256 amount) external;
    function balances(address user) external view returns (uint256);
}

/**
 * @title VaultRouter
 * @notice 双合约路由：新用户/操作路由到新合约，旧用户保留在旧合约
 */
contract VaultRouter {
    IVault public immutable oldVault;
    IVault public immutable newVault;

    // 记录每个用户已被分配到哪个合约
    mapping(address => bool) public onNewVault;

    // 迁移进度控制：true = 新注册用户进新合约
    bool public routeNewUsersToNew = true;

    address public owner;

    event UserRouted(address indexed user, bool isNewVault);

    constructor(address _old, address _new) {
        oldVault = IVault(_old);
        newVault = IVault(_new);
        owner = msg.sender;
    }

    function deposit(uint256 amount) external {
        IVault target = _getOrAssignVault(msg.sender);
        target.deposit(msg.sender, amount);
    }

    function withdraw(uint256 amount) external {
        IVault target = _getAssignedVault(msg.sender);
        target.withdraw(msg.sender, amount);
    }

    function balanceOf(address user) external view returns (uint256) {
        if (onNewVault[user]) {
            return newVault.balances(user);
        }
        return oldVault.balances(user);
    }

    // 将旧合约用户主动迁移到新合约（可批量操作）
    function migrateUser(address user) external {
        require(msg.sender == owner, "Not owner");
        require(!onNewVault[user], "Already on new vault");

        uint256 balance = oldVault.balances(user);
        if (balance > 0) {
            oldVault.withdraw(user, balance);
            newVault.deposit(user, balance);
        }
        onNewVault[user] = true;
    }

    function _getOrAssignVault(address user) internal returns (IVault) {
        if (!onNewVault[user] && oldVault.balances(user) == 0 && routeNewUsersToNew) {
            // 新用户自动分配到新合约
            onNewVault[user] = true;
            emit UserRouted(user, true);
        }
        return onNewVault[user] ? newVault : oldVault;
    }

    function _getAssignedVault(address user) internal view returns (IVault) {
        return onNewVault[user] ? newVault : oldVault;
    }
}
```

---

## 8. 存储布局兼容性验证

代理升级时，**存储布局不兼容是最危险的问题**，会导致数据静默损坏。

### Foundry：导出并对比 Storage Layout

```bash
# 导出两个版本的存储布局
forge inspect TokenVaultV1 storage-layout --json > layout_v1.json
forge inspect TokenVaultV2 storage-layout --json > layout_v2.json

# 对比差异（macOS 使用 diff）
diff <(jq '.storage[] | "\(.slot) \(.label) \(.type)"' layout_v1.json) \
     <(jq '.storage[] | "\(.slot) \(.label) \(.type)"' layout_v2.json)
```

### Hardhat：自动存储检查

```typescript
// test/storage-layout.test.ts
import { upgrades } from "hardhat";

it("V2 storage is compatible with V1", async () => {
    const V1 = await ethers.getContractFactory("TokenVaultV1");
    const V2 = await ethers.getContractFactory("TokenVaultV2");

    // 部署 V1 代理
    const proxy = await upgrades.deployProxy(V1, [owner.address]);

    // 验证存储兼容性（不兼容时自动抛错）
    await upgrades.validateUpgrade(proxy.target, V2);
});
```

### 手动检查规则

```
✅ 允许的变更:
  - 在末尾追加新变量
  - 将末尾变量替换为同等大小的新变量

❌ 禁止的变更:
  - 改变已有变量的顺序
  - 改变已有变量的类型（即使大小相同，如 address → uint160）
  - 在中间位置插入新变量
  - 删除已有变量
  - 缩小变量大小（uint256 → uint128）
```

### 用 `__gap` 预留升级空间

```solidity
contract TokenVaultV1 is UUPSUpgradeable, OwnableUpgradeable {
    mapping(address => uint256) public balances;
    uint256 public totalDeposited;

    // 预留 48 个 slot 供未来升级使用
    // 每次添加新变量，对应减少 gap 大小
    uint256[48] private __gap;

    // 版本 2 添加 feeRate 时：
    // uint256 public feeRate;       ← 新增
    // uint256[47] private __gap;    ← 从 48 减为 47
}
```

---

## 9. 迁移安全设计

### 关键安全措施

#### 9.1 旧合约暂停

迁移期间必须先暂停旧合约，防止用户继续写入导致快照不一致：

```solidity
// 旧合约必须有 pause 功能（Pausable）
function startMigration() external onlyOwner {
    _pause(); // 停止所有状态变化
    emit MigrationStarted(block.number);
}
```

#### 9.2 迁移数据校验

```solidity
// 迁移结束后，对关键指标做一致性检查
function verifyMigration(address[] calldata sampleUsers) external view {
    for (uint256 i = 0; i < sampleUsers.length; i++) {
        address user = sampleUsers[i];
        uint256 oldBal = IOldVault(oldVault).balances(user);
        uint256 newBal = balances[user];
        require(oldBal == newBal, string.concat("Mismatch: ", Strings.toHexString(user)));
    }
}
```

#### 9.3 防止双重认领（Merkle Proof 方案）

```solidity
mapping(address => bool) public claimed;

function claim(...) external {
    require(!claimed[msg.sender], "Already claimed"); // ← 必须
    claimed[msg.sender] = true;                       // ← 先标记，再转账（CEI 模式）
    token.transfer(msg.sender, amount);
}
```

#### 9.4 时间锁保护

重要迁移操作通过 Timelock 延迟执行，给社区审查时间：

```solidity
// 使用 OpenZeppelin TimelockController
// 设置 2-7 天的 minDelay，重要操作（如 finalizeMigration）需提前提案
```

#### 9.5 多签审批

```
合约升级/迁移 → 多签钱包（Gnosis Safe）3/5 签名 → 执行
```

### 应急回滚方案

```solidity
// 新合约预留 24-48 小时回滚窗口
bool public paused = true; // 初始暂停，验证后再开放

// 如果发现问题，回滚步骤：
// 1. 新合约 pause（已暂停则无操作）
// 2. 旧合约 unpause（恢复旧合约服务）
// 3. 退还已迁移用户的 Token 到旧合约
// 4. 分析问题，修复后重新迁移
```

---

## 10. 完整迁移 Checklist

### 迁移前准备

```
[ ] 确定迁移方案（代理升级 / 懒惰迁移 / Merkle / 全量复制）
[ ] 导出并对比 Storage Layout，确认无冲突
[ ] 新合约代码通过 audit 或内部 review
[ ] 迁移脚本在 Fork 测试网上验证（mainnet fork）
[ ] 准备回滚方案（旧合约 unpause 流程）
[ ] 准备应急联系人列表（多签持有者、技术负责人）
[ ] 通知用户：迁移时间、对用户的影响、如何认领
[ ] 多签钱包配置正确（3/5 方案，持有人分散保管）
```

### 迁移执行

```
[ ] 暂停旧合约写入（pause）
[ ] 记录快照区块高度（snapshot block）
[ ] 部署新合约（使用 hardhat-deploy 或 CREATE2 确保地址可复现）
[ ] 执行数据迁移（批量写入 / 等待 Merkle Proof 认领）
[ ] 对比关键指标：totalSupply / totalDeposited / 抽样用户余额
[ ] 新合约 unpause，开放业务
[ ] 更新前端 dApp 指向新合约
[ ] 更新 subgraph 索引新合约
[ ] 通知 CEX / 钱包等集成方更新合约地址
```

### 迁移后验证

```
[ ] 监控新合约 24-48 小时：事件、余额、交易量
[ ] 对比新旧合约关键 Storage 变量
[ ] 确认未认领余额有 recoverUnclaimed 兜底
[ ] 在 Etherscan 上验证（verify）新合约源码
[ ] 更新项目文档：新合约地址、ABI、部署区块
[ ] 旧合约标记为 deprecated（在代码注释和文档中）
```

---

## 参考示例：知名项目迁移案例

| 项目 | 迁移类型 | 方案 |
|---|---|---|
| Uniswap V2 → V3 | 流动性迁移 | 双合约并行 + Router |
| Compound V2 → V3 | 借贷状态迁移 | 新部署，用户手动提取再充值 |
| Aave V2 → V3 | 存款迁移 | aToken 迁移工具合约 |
| Synthetix | 多次存储升级 | 独立 State 合约（EternalStorage 模式）|
| ENS | 注册记录迁移 | Merkle 证明 + 自助认领 |

---

*文档最后更新：2026-03-05*
