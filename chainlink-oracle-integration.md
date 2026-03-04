# 预言机原理与 Chainlink 集成完整指南

> **适用读者**：Solidity 中级及以上开发者  
> **参考资料**：Chainlink 官方文档（docs.chain.link）  
> **文档日期**：2026-03-04

---

## 目录

1. [什么是预言机（Oracle）](#什么是预言机oracle)
2. [预言机价格喂价原理](#预言机价格喂价原理)
3. [Chainlink 去中心化预言机网络架构](#chainlink-去中心化预言机网络架构)
4. [Chainlink Data Feeds 核心接口](#chainlink-data-feeds-核心接口)
5. [基础集成：读取价格喂价](#基础集成读取价格喂价)
6. [进阶集成：生产级安全实践](#进阶集成生产级安全实践)
7. [L2 网络特殊处理（Sequencer 检查）](#l2-网络特殊处理sequencer-检查)
8. [Chainlink VRF：链上可验证随机数](#chainlink-vrf链上可验证随机数)
9. [Chainlink Automation：自动化触发](#chainlink-automation自动化触发)
10. [常用 Feed 地址速查表](#常用-feed-地址速查表)
11. [预言机攻击与防护](#预言机攻击与防护)
12. [完整生产示例：借贷协议价格检查](#完整生产示例借贷协议价格检查)

---

## 什么是预言机（Oracle）

**区块链预言机**是连接链上智能合约与链下真实世界数据的中间件。由于区块链本身是确定性封闭系统，无法主动获取链外信息（如资产价格、天气、体育赛事结果等），因此需要预言机承担"数据搬运工"角色。

```
链外世界（API / 交易所 / 数据源）
          │
     预言机节点
          │ 签名 + 上报
     聚合合约（Aggregator）
          │
   你的智能合约（Consumer）
```

### 核心问题：如何保证数据可信？

| 方案 | 机制 | 风险 |
|---|---|---|
| 中心化预言机 | 单一数据源上报 | 单点故障、人为操纵 |
| 去中心化预言机（DON） | 多节点聚合 + 经济激励 | 成本更高，但安全性大幅提升 |
| TWAP（时间加权均价） | 链上历史价格加权 | 防闪贷，但延迟高 |
| 多源聚合（Chainlink + TWAP） | 链上 + 链下双重校验 | 最强防护，是生产首选 |

---

## 预言机价格喂价原理

### 传统中心化方案（已被淘汰）

```
外部 API → 单一节点 → 合约
```

**问题**：单点故障，节点可以任意篡改价格，无法被链上验证。

### Chainlink 去中心化喂价方案

喂价更新由两个触发条件控制，**满足任意一个即触发本次聚合轮**：

```
┌─────────────────────────────────────────┐
│  触发条件 1：Deviation Threshold（偏差阈值）│
│  链下价格与链上最新价格偏差 > 阈值（如 0.5%）│
│                                          │
│  触发条件 2：Heartbeat Threshold（心跳阈值）│
│  距上次更新已超过最大时间（如 3600 秒）      │
└─────────────────────────────────────────┘
          │（满足其一）
          ↓
  Oracle 节点集群各自上报报价
          ↓
  链上 Aggregator 收集足够数量的报价
          ↓
  计算中位数（Median），写入 latestAnswer
          ↓
  Consumer 合约读取价格
```

### OCR（Offchain Reporting）——现代聚合协议

Chainlink v2 引入 OCR，大幅降低链上 Gas 成本：

```
链下：
  各节点通过 P2P 网络交换彼此的签名报价
  Leader 节点汇总 → 构造一个包含所有节点签名的聚合报告
            │
链上：
  Leader 广播聚合报告（一次 tx 替代 N 次 tx）
  OCR Aggregator 合约验证所有签名
  写入最新聚合价格
```

**Gas 节省**：假设 31 个节点，传统方式需 31 笔 tx，OCR 只需 1 笔，节省约 10 倍 Gas。

---

## Chainlink 去中心化预言机网络架构

### 三层合约结构

```
你的合约（Consumer）
    │  调用 latestRoundData()
    ↓
Proxy 合约（代理层）
    │  转发调用到当前 Aggregator
    ↓
AccessControlledOffchainAggregator（聚合层）
    │  存储各轮次聚合价格
    ↑
Oracle 节点（链下）
    │  OCR 协议聚合后提交
```

#### Consumer（消费者合约）

- 任何调用 Chainlink Data Feed 的合约。
- 实现 `AggregatorV3Interface`，调用 `latestRoundData()` 获取价格。
- **不直接与 Aggregator 交互**，而是通过 Proxy。

#### Proxy（代理合约）

- 指向当前有效的 Aggregator 合约地址。
- **关键价值**：当 Chainlink 升级 Aggregator 合约时，Consumer 的代码不需要变动，Proxy 自动转发到新的 Aggregator。
- 典型实现：`EACAggregatorProxy.sol`

#### Aggregator（聚合合约）

- 接收 Oracle 节点的定期数据更新。
- 存储每一轮（round）的聚合结果。
- 更新触发：Deviation Threshold 或 Heartbeat Threshold。

---

## Chainlink Data Feeds 核心接口

### AggregatorV3Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface AggregatorV3Interface {
    // 喂价精度（如 ETH/USD 为 8，表示返回值需除以 1e8）
    function decimals() external view returns (uint8);

    // 喂价描述（如 "ETH / USD"）
    function description() external view returns (string memory);

    // 接口版本号（当前为 4）
    function version() external view returns (uint256);

    // 获取指定 roundId 的历史数据
    function getRoundData(uint80 _roundId) external view returns (
        uint80 roundId,
        int256 answer,       // 价格（需除以 10^decimals）
        uint256 startedAt,   // 本轮开始时间戳
        uint256 updatedAt,   // 本轮更新时间戳
        uint80 answeredInRound // 完成本次聚合的 roundId
    );

    // 获取最新价格数据（最常用）
    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );
}
```

### 返回字段详解

| 字段 | 类型 | 说明 |
|---|---|---|
| `roundId` | `uint80` | 当前轮次 ID（含 phaseId 前缀） |
| `answer` | `int256` | 聚合价格，**需除以 `10 ** decimals()`** |
| `startedAt` | `uint256` | 本轮开始的 Unix 时间戳 |
| `updatedAt` | `uint256` | 本轮最后更新的 Unix 时间戳（最重要，用于判断数据新鲜度） |
| `answeredInRound` | `uint80` | 完成聚合的 roundId（应 == roundId，否则数据可能过期）|

### decimals 换算

```solidity
// ETH/USD 的 decimals = 8
// latestRoundData 返回 answer = 200000000000 (即 $2000.00000000)
// 实际价格 = 200000000000 / 10**8 = 2000.00 USD

int256 price = answer;     // e.g. 200000000000
uint8 dec = feed.decimals(); // 8
// 使用时：uint256 priceUSD = uint256(price) / (10 ** dec);
```

---

## 基础集成：读取价格喂价

### 安装依赖

```bash
# Foundry
forge install smartcontractkit/chainlink-brownie-contracts

# npm
npm install @chainlink/contracts
```

### remappings.txt（Foundry）

```
@chainlink/contracts/=lib/chainlink-brownie-contracts/contracts/
```

### 最简合约（官方示例）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.7;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract DataConsumerV3 {
    AggregatorV3Interface internal dataFeed;

    // Sepolia 测试网：BTC / USD
    // 地址：0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43
    constructor() {
        dataFeed = AggregatorV3Interface(
            0x1b44F3514812d835EB1BDB0acB33d3fA3351Ee43
        );
    }

    function getLatestPrice() public view returns (int256) {
        (
            /* uint80 roundId */,
            int256 answer,
            /* uint256 startedAt */,
            /* uint256 updatedAt */,
            /* uint80 answeredInRound */
        ) = dataFeed.latestRoundData();
        return answer;
    }
}
```

---

## 进阶集成：生产级安全实践

### 必须校验的三个条件

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract SafePriceFeed {
    AggregatorV3Interface public immutable priceFeed;
    
    // 心跳阈值：超过此时间未更新视为过期
    // 不同 feed 的心跳不同，ETH/USD 通常为 3600s，稳定币对可为 86400s
    uint256 public constant HEARTBEAT = 3600; // 1 小时

    constructor(address _feed) {
        priceFeed = AggregatorV3Interface(_feed);
    }

    /**
     * @notice 生产级价格读取，包含三重安全校验
     */
    function getSafePrice() public view returns (uint256 price, uint8 decimals) {
        (
            uint80 roundId,
            int256 answer,
            /* uint256 startedAt */,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        // ✅ 校验 1：price 不能为负数（防止聚合异常）
        require(answer > 0, "Chainlink: invalid negative price");

        // ✅ 校验 2：数据不能过期（Heartbeat 检查）
        require(
            block.timestamp - updatedAt <= HEARTBEAT,
            "Chainlink: stale price data"
        );

        // ✅ 校验 3：round 完整性（answeredInRound 应 >= roundId）
        require(
            answeredInRound >= roundId,
            "Chainlink: round not complete"
        );

        price = uint256(answer);
        decimals = priceFeed.decimals();
    }

    /**
     * @notice 将 Chainlink 价格统一换算为 18 位精度（WAD）
     */
    function getPriceWAD() public view returns (uint256) {
        (uint256 rawPrice, uint8 dec) = getSafePrice();
        // 统一放大到 1e18
        if (dec < 18) {
            return rawPrice * (10 ** (18 - dec));
        } else if (dec > 18) {
            return rawPrice / (10 ** (dec - 18));
        }
        return rawPrice;
    }
}
```

### 多喂价源组合（容灾设计）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

/**
 * @notice 使用两个独立 feed（如 ETH/USD + ETH/BTC × BTC/USD）交叉验证
 * 当两个价格相差过大时，拒绝使用（防止单一 feed 被攻击）
 */
contract CrossValidatedPriceFeed {
    AggregatorV3Interface public immutable primaryFeed;   // 主 feed：ETH/USD
    AggregatorV3Interface public immutable secondaryFeed; // 备份 feed：STETH/USD 或 ETH/USD 另一源

    uint256 public constant HEARTBEAT = 3600;
    uint256 public constant MAX_DEVIATION_BPS = 200; // 2% 最大偏差

    constructor(address _primary, address _secondary) {
        primaryFeed = AggregatorV3Interface(_primary);
        secondaryFeed = AggregatorV3Interface(_secondary);
    }

    function getCrossValidatedPrice() external view returns (uint256) {
        uint256 price1 = _getPrice(primaryFeed);
        uint256 price2 = _getPrice(secondaryFeed);

        // 计算两个价格的偏差（基点）
        uint256 diff = price1 > price2 ? price1 - price2 : price2 - price1;
        uint256 avg = (price1 + price2) / 2;
        uint256 deviationBps = (diff * 10000) / avg;

        require(deviationBps <= MAX_DEVIATION_BPS, "Price feeds diverged too much");

        // 返回两者均值
        return avg;
    }

    function _getPrice(AggregatorV3Interface feed) internal view returns (uint256) {
        (, int256 answer, , uint256 updatedAt, ) = feed.latestRoundData();
        require(answer > 0, "Invalid price");
        require(block.timestamp - updatedAt <= HEARTBEAT, "Stale price");
        // 统一为 18 位精度
        uint8 dec = feed.decimals();
        uint256 raw = uint256(answer);
        return dec < 18 ? raw * 10 ** (18 - dec) : raw / 10 ** (dec - 18);
    }
}
```

### 可配置 Feed 地址（支持升级）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract UpgradeablePriceFeed is Ownable {
    mapping(address token => AggregatorV3Interface feed) public priceFeeds;
    uint256 public heartbeat = 3600;

    event FeedUpdated(address indexed token, address indexed feed);
    event HeartbeatUpdated(uint256 newHeartbeat);

    constructor(address _initialOwner) Ownable(_initialOwner) {}

    function setFeed(address token, address feed) external onlyOwner {
        priceFeeds[token] = AggregatorV3Interface(feed);
        emit FeedUpdated(token, feed);
    }

    function setHeartbeat(uint256 _heartbeat) external onlyOwner {
        heartbeat = _heartbeat;
        emit HeartbeatUpdated(_heartbeat);
    }

    function getPrice(address token) external view returns (uint256 price, uint8 dec) {
        AggregatorV3Interface feed = priceFeeds[token];
        require(address(feed) != address(0), "Feed not configured");

        (, int256 answer, , uint256 updatedAt, ) = feed.latestRoundData();
        require(answer > 0, "Invalid price");
        require(block.timestamp - updatedAt <= heartbeat, "Stale price");

        price = uint256(answer);
        dec = feed.decimals();
    }
}
```

---

## L2 网络特殊处理（Sequencer 检查）

在 Arbitrum、Optimism 等 L2 上，如果排序器（Sequencer）宕机，Chainlink 的喂价可能停止更新，但合约代码无法自动感知。**Chainlink 提供了 Sequencer Uptime Feed** 专门检测这一状态。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";

contract L2PriceFeed {
    AggregatorV3Interface public immutable sequencerFeed;
    AggregatorV3Interface public immutable priceFeed;
    
    // Sequencer 恢复后需等待的宽限期（防止 Sequencer 刚恢复时价格陈旧）
    uint256 private constant GRACE_PERIOD = 3600; // 1 小时
    uint256 private constant HEARTBEAT = 3600;

    // Arbitrum Sequencer Uptime Feed: 0xFdB631F5EE196F0ed6FAa767959853A9F217697D
    // Optimism Sequencer Uptime Feed: 0x371EAD81c9102C9BF4874A9075FFFf170F2Ee389
    constructor(address _sequencerFeed, address _priceFeed) {
        sequencerFeed = AggregatorV3Interface(_sequencerFeed);
        priceFeed = AggregatorV3Interface(_priceFeed);
    }

    function getL2SafePrice() external view returns (uint256) {
        // Step 1：检查 Sequencer 是否在线
        (
            /* uint80 roundId */,
            int256 sequencerAnswer, // 0 = up, 1 = down
            uint256 startedAt,      // Sequencer 状态变更时间
            /* uint256 updatedAt */,
            /* uint80 answeredInRound */
        ) = sequencerFeed.latestRoundData();

        bool isSequencerUp = sequencerAnswer == 0;
        require(isSequencerUp, "Sequencer is down");

        // Step 2：等待宽限期（Sequencer 刚恢复时数据可能不准）
        uint256 timeSinceUp = block.timestamp - startedAt;
        require(timeSinceUp > GRACE_PERIOD, "Sequencer grace period not met");

        // Step 3：读取价格（正常三重校验）
        (
            uint80 roundId,
            int256 answer,
            /* uint256 pStartedAt */,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();

        require(answer > 0, "Invalid price");
        require(block.timestamp - updatedAt <= HEARTBEAT, "Stale price");
        require(answeredInRound >= roundId, "Round not complete");

        return uint256(answer);
    }
}
```

---

## Chainlink VRF：链上可验证随机数

VRF（Verifiable Random Function）提供可验证的链上随机数，任何人可以密码学验证结果未被篡改。

### VRF v2.5 集成（新版本）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {VRFConsumerBaseV2Plus} from "@chainlink/contracts/src/v0.8/vrf/dev/VRFConsumerBaseV2Plus.sol";
import {VRFV2PlusClient} from "@chainlink/contracts/src/v0.8/vrf/dev/libraries/VRFV2PlusClient.sol";

contract VRFExample is VRFConsumerBaseV2Plus {
    // Sepolia 测试网配置
    address constant VRF_COORDINATOR = 0x9DdfaCa8183c41ad55329BdeeD9F6A8d53168B1b;
    bytes32 constant KEY_HASH = 0x787d74caea10b2b357790d5b5247c2f63d1d91572a9846f780606e4d953677ae;
    uint256 public subscriptionId;

    uint32 constant CALLBACK_GAS_LIMIT = 100000;
    uint16 constant REQUEST_CONFIRMATIONS = 3;
    uint32 constant NUM_WORDS = 1;

    mapping(uint256 requestId => uint256[] randomWords) public results;
    mapping(uint256 requestId => address requester) public requesters;

    event RandomWordsRequested(uint256 indexed requestId, address indexed requester);
    event RandomWordsFulfilled(uint256 indexed requestId, uint256[] randomWords);

    constructor(uint256 _subscriptionId) VRFConsumerBaseV2Plus(VRF_COORDINATOR) {
        subscriptionId = _subscriptionId;
    }

    /**
     * @notice 发起随机数请求
     * @return requestId 用于追踪本次请求
     */
    function requestRandomness() external returns (uint256 requestId) {
        requestId = s_vrfCoordinator.requestRandomWords(
            VRFV2PlusClient.RandomWordsRequest({
                keyHash: KEY_HASH,
                subId: subscriptionId,
                requestConfirmations: REQUEST_CONFIRMATIONS,
                callbackGasLimit: CALLBACK_GAS_LIMIT,
                numWords: NUM_WORDS,
                extraArgs: VRFV2PlusClient._argsToBytes(
                    VRFV2PlusClient.ExtraArgsV1({nativePayment: false})
                )
            })
        );
        requesters[requestId] = msg.sender;
        emit RandomWordsRequested(requestId, msg.sender);
    }

    /**
     * @notice Chainlink 节点回调此函数，传入随机数结果
     */
    function fulfillRandomWords(uint256 requestId, uint256[] calldata randomWords)
        internal override
    {
        results[requestId] = randomWords;
        emit RandomWordsFulfilled(requestId, randomWords);

        // 应用随机数（示例：从 0 到 99 的随机数）
        uint256 randomNum = randomWords[0] % 100;
        // ... 你的业务逻辑
    }
}
```

### VRF 使用要点

| 项目 | 说明 |
|---|---|
| 订阅管理 | 在 vrf.chain.link 创建 Subscription 并充值 LINK |
| `keyHash` | 代表不同 Gas 价格档位，不同网络不同值 |
| `callbackGasLimit` | 需足够覆盖 `fulfillRandomWords` 的 Gas |
| `requestConfirmations` | 确认数越多越安全，等待时间越长（建议 ≥ 3） |
| 安全性 | 随机数结果由 Chainlink 节点用私钥签名，不可预测也不可操纵 |

---

## Chainlink Automation：自动化触发

Automation（原 Keepers）允许合约在满足条件时自动执行，无需链下 Bot。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AutomationCompatibleInterface} from "@chainlink/contracts/src/v0.8/automation/AutomationCompatible.sol";

contract AutomatedVault is AutomationCompatibleInterface {
    uint256 public lastHarvestTime;
    uint256 public constant HARVEST_INTERVAL = 1 days;
    uint256 public constant MIN_HARVEST_AMOUNT = 100e18;

    address public strategy;

    /**
     * @notice Chainlink Automation 节点定期调用此函数检查是否需要执行
     * @return upkeepNeeded 返回 true 时，Automation 节点会调用 performUpkeep
     */
    function checkUpkeep(bytes calldata)
        external
        view
        override
        returns (bool upkeepNeeded, bytes memory performData)
    {
        bool intervalReached = (block.timestamp - lastHarvestTime) >= HARVEST_INTERVAL;
        bool hasEnoughYield = IStrategy(strategy).pendingYield() >= MIN_HARVEST_AMOUNT;
        upkeepNeeded = intervalReached && hasEnoughYield;
        performData = ""; // 可附带参数传给 performUpkeep
    }

    /**
     * @notice 条件满足时，Automation 节点调用此函数执行实际逻辑
     */
    function performUpkeep(bytes calldata) external override {
        // 再次校验（防止 checkUpkeep 到 performUpkeep 之间状态变化）
        require(
            (block.timestamp - lastHarvestTime) >= HARVEST_INTERVAL,
            "Not yet"
        );
        lastHarvestTime = block.timestamp;
        IStrategy(strategy).harvest();
    }
}

interface IStrategy {
    function pendingYield() external view returns (uint256);
    function harvest() external;
}
```

---

## 常用 Feed 地址速查表

### Ethereum Mainnet

| 交易对 | 合约地址 | decimals | Heartbeat |
|---|---|:---:|:---:|
| ETH / USD | `0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419` | 8 | 3600s |
| BTC / USD | `0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c` | 8 | 3600s |
| USDC / USD | `0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6` | 8 | 86400s |
| LINK / USD | `0x2c1d072e956AFFC0D435Cb7AC38EF18d24d9127c` | 8 | 3600s |
| stETH / USD | `0xCfE54B5cD566aB89272946F602D76Ea879CAb4a8` | 8 | 86400s |

### Arbitrum One

| 交易对 | 合约地址 | decimals |
|---|---|:---:|
| ETH / USD | `0x639Fe6ab55C921f74e7fac1ee960C0B6293ba612` | 8 |
| BTC / USD | `0x6ce185860a4963106506C203335A2910413708e9` | 8 |
| USDC / USD | `0x50834F3163758fcC1Df9973b6e91f0F0F0434aD3` | 8 |

### Optimism

| 交易对 | 合约地址 | decimals |
|---|---|:---:|
| ETH / USD | `0x13e3Ee699D1909E989722E753853AE30b17e08c5` | 8 |
| BTC / USD | `0xD702DD976Fb76Fffc2D3963D037dfDae5b04E593` | 8 |

> 完整地址列表：https://docs.chain.link/data-feeds/price-feeds/addresses

---

## 预言机攻击与防护

### 攻击类型 1：价格操纵（闪贷攻击）

**场景**：DEX 链上价格被闪贷临时拉高/打低，合约读取此价格做决策。

```
攻击者 → 闪贷大量资金 → 操纵 DEX 价格 → 触发合约执行（借贷/清算）→ 还款套利
```

**防护**：
- 使用 Chainlink Price Feed 替代 DEX spot price
- Chainlink 价格来自 CEX 链下 + 多节点聚合，不受单一 DEX 闪贷影响

### 攻击类型 2：陈旧价格（Stale Price）

**场景**：Chainlink 节点故障或心跳时间内未更新，合约读取过期价格。

```solidity
// 必须校验 updatedAt！
require(block.timestamp - updatedAt <= HEARTBEAT, "Stale price");
```

### 攻击类型 3：价格范围外（Circuit Breaker 绕过）

Chainlink 的 Aggregator 有 `minAnswer` 和 `maxAnswer` 电路熔断机制；如果真实价格超出范围，返回的是边界值而非真实值（已在 LUNA 事件中出现）。

```solidity
// 对于高波动资产，检查价格是否在合理范围内
function getPriceWithCircuitBreaker(address token) external view returns (uint256) {
    AggregatorV3Interface feed = priceFeeds[token];
    (, int256 answer, , uint256 updatedAt, ) = feed.latestRoundData();

    // 自定义合理价格范围（可配置）
    require(answer >= int256(minPrices[token]), "Price below minimum");
    require(answer <= int256(maxPrices[token]), "Price above maximum");
    require(block.timestamp - updatedAt <= heartbeats[token], "Stale price");

    return uint256(answer);
}
```

### 攻击类型 4：L2 Sequencer 宕机

见前文 [L2 网络特殊处理](#l2-网络特殊处理sequencer-检查)。

### 安全 Checklist

```
[x] 使用 Chainlink Price Feed 而非 DEX spot 价格
[x] 校验 answer > 0（防止负价格）
[x] 校验 updatedAt（防止陈旧价格）
[x] 校验 answeredInRound >= roundId（防止轮次异常）
[x] L2 上增加 Sequencer 存活检查
[x] 对高波动资产增加合理价格范围校验
[x] 对关键场景使用多源交叉验证
[x] 设置价格异常时的应急暂停机制
```

---

## 完整生产示例：借贷协议价格检查

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {AggregatorV3Interface} from "@chainlink/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Pausable.sol";

/**
 * @title LendingPriceOracle
 * @notice 借贷协议价格预言机模块
 * - 多 token 独立配置 feed 地址和心跳
 * - 三重安全校验
 * - 支持管理员暂停（价格异常时使用）
 * - 价格统一为 18 位精度（WAD）
 */
contract LendingPriceOracle is Ownable, Pausable {
    struct FeedConfig {
        AggregatorV3Interface feed;
        uint256 heartbeat;   // 心跳时间（秒）
        uint256 minPrice;    // 最低有效价格（WAD）
        uint256 maxPrice;    // 最高有效价格（WAD）
    }

    mapping(address token => FeedConfig) public feedConfigs;

    event FeedConfigured(address indexed token, address indexed feed, uint256 heartbeat);
    event PriceEmergency(address indexed token, int256 rawAnswer);

    error InvalidPrice(address token, int256 answer);
    error StalePrice(address token, uint256 updatedAt);
    error RoundNotComplete(address token, uint80 roundId, uint80 answeredInRound);
    error PriceOutOfRange(address token, uint256 price, uint256 min, uint256 max);
    error FeedNotConfigured(address token);

    constructor(address initialOwner) Ownable(initialOwner) {}

    // === 管理函数 ===

    function configureFeed(
        address token,
        address feed,
        uint256 heartbeat,
        uint256 minPriceWAD,
        uint256 maxPriceWAD
    ) external onlyOwner {
        feedConfigs[token] = FeedConfig({
            feed: AggregatorV3Interface(feed),
            heartbeat: heartbeat,
            minPrice: minPriceWAD,
            maxPrice: maxPriceWAD
        });
        emit FeedConfigured(token, feed, heartbeat);
    }

    function pause() external onlyOwner { _pause(); }
    function unpause() external onlyOwner { _unpause(); }

    // === 核心价格读取 ===

    /**
     * @notice 获取 token 的 USD 价格（WAD 精度，即 1e18 = $1）
     */
    function getPrice(address token)
        external
        view
        whenNotPaused
        returns (uint256 priceWAD)
    {
        FeedConfig memory config = feedConfigs[token];
        if (address(config.feed) == address(0)) revert FeedNotConfigured(token);

        (
            uint80 roundId,
            int256 answer,
            /* uint256 startedAt */,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = config.feed.latestRoundData();

        // 校验 1：价格为正数
        if (answer <= 0) revert InvalidPrice(token, answer);

        // 校验 2：数据未过期
        if (block.timestamp - updatedAt > config.heartbeat)
            revert StalePrice(token, updatedAt);

        // 校验 3：round 完整
        if (answeredInRound < roundId)
            revert RoundNotComplete(token, roundId, answeredInRound);

        // 换算为 WAD（18 位精度）
        uint8 dec = config.feed.decimals();
        uint256 raw = uint256(answer);
        priceWAD = dec < 18
            ? raw * 10 ** (18 - dec)
            : raw / 10 ** (dec - 18);

        // 校验 4：价格在合理范围内（可选，高波动资产建议开启）
        if (config.minPrice > 0 && priceWAD < config.minPrice)
            revert PriceOutOfRange(token, priceWAD, config.minPrice, config.maxPrice);
        if (config.maxPrice > 0 && priceWAD > config.maxPrice)
            revert PriceOutOfRange(token, priceWAD, config.minPrice, config.maxPrice);
    }

    /**
     * @notice 计算抵押品价值（USD，WAD 精度）
     * @param token 抵押品代币地址
     * @param amount 抵押品数量（代币精度）
     * @param tokenDecimals 代币精度位数
     */
    function getCollateralValueUSD(
        address token,
        uint256 amount,
        uint8 tokenDecimals
    ) external view returns (uint256 valueUSD) {
        uint256 priceWAD = this.getPrice(token);

        // valueUSD = amount * price / tokenDecimals（统一为 WAD）
        if (tokenDecimals <= 18) {
            valueUSD = amount * 10 ** (18 - tokenDecimals) * priceWAD / 1e18;
        } else {
            valueUSD = amount / 10 ** (tokenDecimals - 18) * priceWAD / 1e18;
        }
    }
}
```

---

## 参考资料

| 资源 | 链接 |
|---|---|
| Chainlink Data Feeds 官方文档 | https://docs.chain.link/data-feeds |
| Data Feeds 地址（所有网络） | https://docs.chain.link/data-feeds/price-feeds/addresses |
| Chainlink 去中心化架构 | https://docs.chain.link/architecture-overview/architecture-decentralized-model |
| Offchain Reporting（OCR） | https://docs.chain.link/architecture-overview/off-chain-reporting |
| VRF v2.5 文档 | https://docs.chain.link/vrf |
| Automation 文档 | https://docs.chain.link/chainlink-automation |
| L2 Sequencer Uptime Feeds | https://docs.chain.link/data-feeds/l2-sequencer-feeds |
| AggregatorV3Interface 源码 | https://github.com/smartcontractkit/chainlink/blob/contracts-v1.3.0/contracts/src/v0.8/shared/interfaces/AggregatorV3Interface.sol |

---

*文档最后更新：2026-03-04*
