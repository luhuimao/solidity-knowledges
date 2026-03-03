# Solidity Fuzz 测试完整指南

> 使用 Foundry 对智能合约进行模糊测试，涵盖原理、三种模式、不变量测试实战、调试与 Checklist。

---

## 目录

1. [什么是 Fuzz 测试](#1-什么是-fuzz-测试)
2. [三种 Fuzz 模式](#2-三种-fuzz-模式)
3. [vm.assume vs bound](#3-vmassume-vs-bound)
4. [不变量测试（Invariant Test）完整实战](#4-不变量测试invariant-test完整实战)
5. [失败用例复现与调试](#5-失败用例复现与调试)
6. [Fuzz vs 单元测试 vs 形式化验证](#6-fuzz-vs-单元测试-vs-形式化验证)
7. [Fuzz 测试 Checklist](#7-fuzz-测试-checklist)

---

## 1. 什么是 Fuzz 测试

**Fuzz 测试（模糊测试）** 的核心思想：

| | 输入来源 | 发现能力 |
|---|----------|----------|
| 普通单元测试 | 手动写固定用例 | 只能发现你想到的问题 |
| Fuzz 测试 | 框架自动生成**大量随机输入** | 自动发现你没想到的边界 |

```
普通测试：stake(100 ether) → 检查结果（1 种输入）
Fuzz 测试：stake(随机值 × 10000 次) → 自动发现边界
```

Foundry 使用 **libFuzzer**（LLVM 的工业级 Fuzzer），带有**覆盖率引导**机制：

- 记录哪些代码路径被触发
- 优先生成能覆盖**新路径**的输入，而不是纯随机撒网
- 自动对失败用例做**最小化处理**，找到触发 Bug 的最简输入

---

## 2. 三种 Fuzz 模式

### 模式 1：Property-Based Fuzz（属性测试）

函数参数带类型，Foundry 自动生成随机值：

```solidity
// test/StakingFuzz.t.sol
contract StakingFuzzTest is Test {
    Staking staking;
    MockERC20 token;

    function setUp() public {
        token = new MockERC20("Token", "TKN");
        staking = new Staking(address(token), 1e18);
    }

    // 函数参数有类型 → Foundry 自动 Fuzz
    // 生成各种 amount：0, 1, type(uint256).max, 边界值, 随机中间值...
    function testFuzz_Stake(uint256 amount, address user) public {
        amount = bound(amount, 1, 1e30);
        vm.assume(user != address(0) && user != address(staking));

        token.mint(user, amount);
        vm.startPrank(user);
        token.approve(address(staking), amount);
        staking.stake(amount);
        vm.stopPrank();

        // 属性断言：质押后余额一定等于 amount
        assertEq(staking.balanceOf(user), amount);
        assertEq(staking.totalSupply(), amount);
    }
}
```

运行命令：

```bash
forge test --match-test testFuzz_Stake -v
forge test --fuzz-runs 10000 --match-test testFuzz_Stake  # 增加轮数
```

---

### 模式 2：Invariant Test（不变量测试）

Foundry **随机调用合约的任意函数任意次数**，然后检查全局不变量是否仍然成立。

```
随机序列示例：
stake(500) → unstake(200) → claimReward() → stake(1000) → unstake(300) → ...
每次序列结束后检查：totalStaked 是否 == 所有用户 balanceOf 之和？
```

这是目前**发现复杂状态漏洞**最有效的方法，详见第 4 章。

---

### 模式 3：Stateful Fuzz（有状态序列测试）

通过 Handler 模式控制合法操作序列，是不变量测试的精确实现方式，在第 4 章详细展开。

---

## 3. vm.assume vs bound

这是 Fuzz 测试最常见的误区，直接影响测试效率：

```solidity
// ❌ 用 vm.assume 过滤范围太大 → 大量输入被丢弃，浪费运行次数
function testFuzz_Stake(uint256 amount) public {
    vm.assume(amount > 1e18 && amount < 1e24); // 丢弃率 > 99.9%
    // ...
}

// ✅ 用 bound 将随机值"夹"到合法范围 → 每次输入都有效
function testFuzz_Stake(uint256 amount) public {
    // bound(x, min, max) 等价于：min + (x % (max - min + 1))
    amount = bound(amount, 1e18, 1e24); // 100% 输入有效
    // ...
}
```

| 方法 | 行为 | 适用场景 |
|------|------|----------|
| `vm.assume(cond)` | 条件不满足就**丢弃**这次输入，重新生成 | 过滤地址类型（`assume(addr != address(0))`） |
| `bound(x, min, max)` | 将 x **映射**到 [min, max] 区间内 | 数值范围，**优先使用** |

**经验准则**：`vm.assume` 的拒绝率应 < 50%，否则改用 `bound`。

---

## 4. 不变量测试（Invariant Test）完整实战

不变量测试由两部分组成：

```
Handler 合约         → 封装合法操作，供 Foundry 随机调用
Invariant 合约       → 定义全局不变量，每次序列结束后验证
```

### Step 1：Handler 合约

```solidity
// test/handlers/StakingHandler.sol
contract StakingHandler is Test {
    Staking public staking;
    MockERC20 public token;

    // 参与测试的用户集合
    address[] public actors;
    address internal currentActor;

    // 幽灵变量（Ghost Variables）：记录所有操作的累计统计
    // 用于不变量对比，必须与合约内部状态保持同步
    uint256 public ghost_totalStaked;
    uint256 public ghost_totalWithdrawn;
    uint256 public ghost_totalRewardClaimed;

    constructor(Staking _staking, MockERC20 _token) {
        staking = _staking;
        token = _token;
    }

    // Modifier：随机选择一个用户作为 msg.sender
    modifier useActor(uint256 actorSeed) {
        currentActor = _getOrCreateActor(actorSeed);
        vm.startPrank(currentActor);
        _;
        vm.stopPrank();
    }

    // ─── 合法操作（Foundry 随机调用这些函数）──────────────────────

    function stake(uint256 actorSeed, uint256 amount)
        external useActor(actorSeed)
    {
        amount = bound(amount, 1, 1e30);

        token.mint(currentActor, amount);
        token.approve(address(staking), amount);
        staking.stake(amount);

        ghost_totalStaked += amount; // 同步幽灵变量
    }

    function unstake(uint256 actorSeed, uint256 amount)
        external useActor(actorSeed)
    {
        uint256 balance = staking.balanceOf(currentActor);
        if (balance == 0) return; // 无余额则跳过，不算一次有效调用

        amount = bound(amount, 1, balance);
        staking.unstake(amount);

        ghost_totalWithdrawn += amount;
    }

    function claimReward(uint256 actorSeed)
        external useActor(actorSeed)
    {
        uint256 earned = staking.earned(currentActor);
        staking.claimReward();
        ghost_totalRewardClaimed += earned;
    }

    // ─── 辅助函数 ─────────────────────────────────────────────────

    function _getOrCreateActor(uint256 seed) internal returns (address) {
        // 20% 概率创建新用户，增加多用户交互场景覆盖
        if (actors.length == 0 || seed % 5 == 0) {
            address newActor = address(
                uint160(uint256(keccak256(abi.encode(seed, actors.length))))
            );
            vm.label(newActor, string.concat("Actor", vm.toString(actors.length)));
            actors.push(newActor);
        }
        return actors[seed % actors.length];
    }

    function actorsLength() external view returns (uint256) {
        return actors.length;
    }
}
```

### Step 2：Invariant 合约

```solidity
// test/StakingInvariant.t.sol
contract StakingInvariantTest is Test {
    Staking staking;
    MockERC20 token;
    StakingHandler handler;

    function setUp() public {
        token = new MockERC20("Token", "TKN");
        staking = new Staking(address(token), 1e18);
        handler = new StakingHandler(staking, token);

        // 告诉 Foundry：只随机调用 handler 的函数
        targetContract(address(handler));

        // 可选：指定参与 Fuzz 的具体函数（排除辅助函数）
        bytes4[] memory selectors = new bytes4[](3);
        selectors[0] = StakingHandler.stake.selector;
        selectors[1] = StakingHandler.unstake.selector;
        selectors[2] = StakingHandler.claimReward.selector;
        targetSelector(FuzzSelector(address(handler), selectors));
    }

    // ─── 不变量 1：合约持有代币 >= totalSupply（不能凭空亏空）────
    function invariant_Solvency() public {
        assertGe(
            token.balanceOf(address(staking)),
            staking.totalSupply(),
            "Staking contract is insolvent!"
        );
    }

    // ─── 不变量 2：totalSupply == 所有用户余额之和 ───────────────
    function invariant_TotalSupplyMatchesSum() public {
        address[] memory actors = handler.actors();
        uint256 sumOfBalances;
        for (uint256 i = 0; i < actors.length; i++) {
            sumOfBalances += staking.balanceOf(actors[i]);
        }
        assertEq(
            staking.totalSupply(),
            sumOfBalances,
            "totalSupply != sum of user balances"
        );
    }

    // ─── 不变量 3：幽灵变量守恒（staked - withdrawn == totalSupply）
    function invariant_GhostVariableConsistency() public {
        assertEq(
            handler.ghost_totalStaked() - handler.ghost_totalWithdrawn(),
            staking.totalSupply(),
            "Ghost variable inconsistency"
        );
    }

    // ─── 不变量 4：奖励总量不超出预算 ───────────────────────────
    function invariant_RewardBudgetNotExceeded() public {
        assertLe(
            handler.ghost_totalRewardClaimed(),
            staking.rewardBudget(),
            "Rewards exceeded budget!"
        );
    }

    // ─── 打印统计（不是真正的不变量，用于调试）──────────────────
    function invariant_CallSummary() public view {
        console.log("== Invariant Call Summary ==");
        console.log("Actors:", handler.actorsLength());
        console.log("Total staked:", handler.ghost_totalStaked());
        console.log("Total withdrawn:", handler.ghost_totalWithdrawn());
        console.log("Total reward claimed:", handler.ghost_totalRewardClaimed());
        console.log("Contract totalSupply:", staking.totalSupply());
    }
}
```

运行不变量测试：

```bash
# 基础运行（默认 256 runs × depth 15）
forge test --match-contract InvariantTest -v

# 生产级配置（发现深层漏洞）
forge test --match-contract InvariantTest \
    --fuzz-runs 1000 \    # 1000 个随机序列
    --depth 200 \         # 每个序列最多调用 200 次函数
    -vv

# 持续运行（CI 环境）
forge test --match-contract InvariantTest --fuzz-runs 5000 --depth 500
```

---

## 5. 失败用例复现与调试

Fuzz 测试发现问题后，Foundry 会：
1. 自动**最小化**失败输入（删除不必要的步骤）
2. 输出可重现的**反例**

```bash
# 失败输出示例
[FAIL. Counterexample:
  sender=0x1234..., args=[500000000000000000000, 0xabcd...]
]
Encountered 1 failing test in test/Staking.t.sol

# 用固定种子复现（确保 CI 可重复）
forge test --match-test testFuzz_Stake --fuzz-seed 12345

# 查看详细调用栈
forge test --match-test testFuzz_Stake -vvvv
```

Foundry 会把失败种子保存到 `~/.foundry/fuzz/` 目录，之后每次运行都会先尝试已知的失败用例（回归测试）。

**手动固定种子进行回归测试**（加入 `foundry.toml`）：

```toml
# foundry.toml
[fuzz]
runs = 256
seed = "0xdeadbeef"   # 固定种子，保证结果可重现

[invariant]
runs = 256
depth = 15
fail_on_revert = false  # 是否把 revert 计为失败（默认 false）
```

---

## 6. Fuzz vs 单元测试 vs 形式化验证

| 维度 | 单元测试 | Fuzz 测试 | 形式化验证（Halmos/Certora） |
|------|:--------:|:---------:|:---------------------------:|
| 输入来源 | 手动固定 | 随机生成 | 数学穷举 |
| 覆盖边界 | 低（测试者经验限制） | 中（概率性覆盖） | 高（数学完备） |
| 运行速度 | 极快（毫秒） | 中（秒~分钟） | 慢（分钟~小时） |
| 发现边界漏洞 | ❌ 很难 | ✅ 擅长 | ✅ 完备 |
| 发现状态序列漏洞 | ❌ 极难 | ✅ Invariant 模式 | ✅ 完备 |
| 学习成本 | 低 | 中 | 高 |
| 推荐用途 | 基础回归 | 核心逻辑验证 | 高价值金融合约 |

**最佳实践组合**：

```
单元测试（快速回归）
    +
Property Fuzz（单函数边界）
    +
Invariant Fuzz（状态机不变量）
    +
形式化验证（关键金融逻辑，如 AMM 定价、清算公式）
```

---

## 7. Fuzz 测试 Checklist

### 代码覆盖目标

```bash
# 生成覆盖率报告
forge coverage --report lcov
genhtml lcov.info -o coverage_report
open coverage_report/index.html
```

### 测试质量清单

```
基础 Fuzz
  ✅ 每个 public/external 写函数都有对应的 testFuzz_ 测试
  ✅ 使用 bound() 代替大量 vm.assume()（拒绝率 < 50%）
  ✅ 覆盖正常路径、边界值（0、max）、超出范围

不变量测试
  ✅ 定义 Ghost Variables 追踪操作累计量
  ✅ 不变量覆盖三类：资产守恒 / 权限边界 / 状态机完整性
  ✅ Handler 中的操作不使用 vm.assume（用 if return 代替）
  ✅ --depth >= 50（暴露跨函数依赖漏洞）

CI 配置
  ✅ 本地：--fuzz-runs 256（快速反馈）
  ✅ CI：--fuzz-runs 5000（充分覆盖）
  ✅ 保存已知失败种子，加入回归测试

调试
  ✅ 失败时使用 -vvvv 查看完整调用栈
  ✅ 用 --fuzz-seed 固定种子确保可重现
  ✅ 查看 Counterexample 并手动写固定测试固化回归
```

### 常见不变量模板

```solidity
// 资产守恒
assertGe(token.balanceOf(address(protocol)), protocol.totalLiabilities());

// 权限边界
// → 所有 onlyOwner 函数均有对应的 testFuzz_OnlyOwner_* 测试

// 单调性
// → 累计奖励只增不减
assertGe(currentEarned, previousEarned);

// 价格边界
assertGe(pool.getPrice(), MIN_PRICE);
assertLe(pool.getPrice(), MAX_PRICE);

// 零和守恒
assertEq(totalMinted, totalBurned + totalCirculating);
```
