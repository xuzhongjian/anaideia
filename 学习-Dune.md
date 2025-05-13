# 学习-Dune

### **Day 1：认识 Dune + 基础链上 SQL 查询（交易、转账）**

**目标**：了解 Dune 平台的原理，能完成基础的链上数据查询。

**内容：**

- 注册并熟悉 Dune：https://dune.com
- 阅读官方文档：https://dune.com/docs/
- 认识核心数据表：
  - `ethereum.transactions`：交易记录
  - `ethereum.logs`：事件（事件监听）
  - `ethereum.traces`：内部调用
- 编写基本 SQL：
  - 查询地址的转账历史
  - 筛选合约地址的调用频次
  - 按日聚合交易数量/金额

**任务：**

- 查询 USDT 的前 10 个收款地址（转账金额降序）
- 查询某钱包 30 天交易次数和交易金额

------

### **Day 2：事件监听和合约交互分析（logs + topics）**

**目标**：掌握如何监听链上事件，解析标准合约如 ERC20 转账记录。

**内容：**

- 理解合约事件结构：topics[0] = keccak(function signature)
- 解码标准事件（如 `Transfer`, `Approval`）
- 使用 `ethereum.logs` 查询事件
- 字段处理技巧：`decode_hex`, `lower`, `substr`, `bytea_to_text`

**任务：**

- 提取某 ERC20 Token 的转账事件：`from`, `to`, `value`
- 分析近一周该 token 的转账活跃地址数（日活）

------

### **Day 3：trace 分析与合约调用链理解**

**目标**：分析合约内部调用与失败交易，掌握调用路径。

**内容**：

- 理解 `ethereum.traces` 表结构：
  - 字段：`call_type`, `input`, `success`, `value`
- 识别 Router 合约的外部与内部调用
- 结合 `method_id` 分析合约函数使用情况

**任务**：

- 分析 Uniswap Router 调用最多的方法（按 `method_id` 统计）
- 查询某地址的所有合约调用及成功率

------

### **Day 4：DApp 行为分析：DEX、NFT、Staking**

**目标**：能用 SQL 编写 DeFi / NFT 分析逻辑。

**内容：**

- 使用标准表：
  - `dex.trades`：swap 数据
  - `nft.trades` / `nft.transfers`：NFT 活动
  - `staking.events`（取决于协议）或 `logs` + decode
- 分析交易量、用户数、活跃钱包、mint/claim 行为等

**任务：**

- 分析某 DEX 过去 7 天的交易量变化
- 分析一个 NFT 项目 mint 次数 + 时间分布

------

### **Day 5：Dune 可视化与仪表盘构建**

**目标**：掌握图表构建，完成仪表盘设计。

**内容：**

- 图表类型：折线图、条形图、饼图、表格
- 时间轴、地址筛选器的使用
- 页面布局与可读性优化
- 仪表盘描述撰写与图表注释

**任务：**

- 创建一个仪表盘，分析某 token 的转账行为
  - 日交易量
  - Top 钱包
  - Gas 成本
- 添加至少 3 个图表 + 1 个注释 + 总体描述

### 附件-常见的表

在 **Dune** 上，许多分析师和开发者依赖于社区维护和平台官方提供的标准化 **链上数据库表（tables）**，这些表通过 SQL 查询链上数据，主要来自 **EVM 系公链（如 Ethereum、Arbitrum、Polygon 等）**。

------

##### 1. **交易与区块相关表**

| 表名                    | 说明                                                       |
| ----------------------- | ---------------------------------------------------------- |
| `ethereum.transactions` | 所有交易记录（`hash`, `from`, `to`, `value`, `input` 等）  |
| `ethereum.blocks`       | 区块信息（`number`, `hash`, `timestamp`, `miner` 等）      |
| `ethereum.traces`       | 合约内部调用（trace 类型，如 call/delegatecall/create 等） |
| `ethereum.logs`         | 事件日志（可以用来解析合约事件）                           |

------

##### 2. **智能合约与调用数据**

| 表名                           | 说明                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| `ethereum.contracts`           | 合约地址与元数据（如创建者、创建时间）                       |
| `ethereum.decoded_logs`        | 解码后的 logs，包含 event 名称、参数等（Dune 解码支持）      |
| `ethereum.decoded_traces`      | 解码后的合约调用 trace                                       |
| `ethereum.function_signatures` | 函数选择器与函数名的对应关系（`0xa9059cbb => transfer(address,uint256)`） |

------

##### 3. **ERC20/Token 相关表**

| 表名                        | 说明                                           |
| --------------------------- | ---------------------------------------------- |
| `ethereum.erc20`            | ERC20 token 的元数据（地址、symbol、decimals） |
| `ethereum.erc20_transfers`  | 所有 ERC20 的 `Transfer` 事件                  |
| `ethereum.erc721_transfers` | NFT（ERC721）转账事件                          |
| `ethereum.token_prices`     | Token 的历史价格数据（由社区或 Dune 维护）     |

------

##### 4. **协议/DeFi 相关表（部分由社区提供）**

| 表名                                | 示例协议   | 说明                           |
| ----------------------------------- | ---------- | ------------------------------ |
| `uniswap_v2.ethereum.swaps`         | Uniswap V2 | 交易对信息，币种、数量、价格等 |
| `uniswap_v3.ethereum.swaps`         | Uniswap V3 | 更复杂，含 tick、fee 等        |
| `aave_v2.ethereum.borrow`           | Aave       | 借款记录                       |
| `compound_v2.ethereum.liquidations` | Compound   | 清算数据                       |
| `curvefi.ethereum.trades`           | Curve      | Curve 交易记录                 |

------

##### 5. **跨链桥数据（社区项目）**

| 表名                              | 示例         | 说明             |
| --------------------------------- | ------------ | ---------------- |
| `hop_protocol.ethereum.transfers` | Hop Protocol | 记录跨链行为     |
| `layerzero.ethereum.messages`     | LayerZero    | 消息传输记录     |
| `wormhole_ethereum.transfers`     | Wormhole     | 资产跨链转移记录 |

------

##### 使用方式

可以在 Dune 上使用：

```sql
SELECT *
FROM ethereum.transactions
WHERE to = LOWER('0xUniswapRouter地址')
AND block_time > NOW() - INTERVAL '7 days'
```

也可以查看表结构和字段定义（Data Explorer 界面）来辅助查询。

### 附件-参考链接

1. 整体上，这个教程讲的已经非常详细了：https://tutorial.sixdegree.xyz/zh。
2. 在完成前面的 SQL 的学习后，就不需要学习上面的教程的第一大部分了。