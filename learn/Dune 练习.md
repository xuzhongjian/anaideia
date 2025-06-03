# 使用 Dune

## 一、ETH 转账分析

### 1. 统计地址之间的 ETH 转账总金额

- 表：`ethereum.transactions`
- 字段：`from_address`, `to_address`, `value`
- 操作：聚合按 `from_address` 或 `to_address` 分组
- 目标：找出最活跃的发送者或接收者（按总金额排序）
- 可筛选时间范围，例如最近 30 天

------

### 2. 识别某个地址的所有转账

- 表：`ethereum.transactions`
- 条件：`from_address = '0x...' OR to_address = '0x...'`
- 可筛选时间范围，例如最近 30 天

------

### 3. 找出 ETH 转账金额最大的一笔交易

- 表：`ethereum.transactions`
- 排序：按 `value` 降序
- 结果：显示交易哈希、from、to、value、时间戳
- 时间范围最近 30 天

------

### 4. 分析某个地址的净转账额

- 逻辑：
  - 收入 = 所有 `to_address = 该地址` 的 value 之和
  - 支出 = 所有 `from_address = 该地址` 的 value 之和
  - 净值 = 收入 - 支出
  - 可筛选时间范围，例如最近 30 天

------

### 5. 统计每个地址的交易次数

- 表：`ethereum.transactions`
- 字段：`from_address`, `to_address`
- 操作：对每个地址出现的次数求和（无论是发送者还是接收者）
- 目标：找出最活跃的地址（按交易次数排序）
- 可筛选时间范围，例如最近 30 天

------

### 6. 找出最频繁的转账对

- 表：`ethereum.transactions`
- 字段：`from_address`, `to_address`, `value`
- 操作：按 `(from_address, to_address)` 分组并求和
- 目标：找出最频繁的转账对（按总金额或笔数排序）
- 识别地址对之间的累计转账总额（地址对 = from + to）
- 可筛选时间范围，例如最近 30 天

------

### 7. 分析某地址在高峰期的转账活动

- 表：`ethereum.transactions`
- 条件：`from_address = '0x...' OR to_address = '0x...'`
- 操作：筛选特定时间段内的交易
- 目标：找出高峰期的交易数量和总金额
- 可筛选时间范围，例如最近 7 天

------

### 8. 找出所有“循环转账”路径

- 表：`ethereum.transactions`
- 操作：识别在某时间段内出现的转账闭环 A→B→A
- 目标：找出频繁发生这种操作的地址对
- 可筛选时间范围，例如最近 30 天

------

### 9. 识别大额地址

- 表：`ethereum.transactions`
- 操作：分别统计地址的总收入和支出，计算其净变化
- 目标：筛选出净变化大于阈值的地址（如 1000 ETH）
- 可筛选时间范围，例如最近 30 天

## 二、ERC20 类事件分析

### 事件签名

在 Ethereum 的日志系统中，**`topics[0]` 是事件签名的 Keccak-256 哈希值**，用于标识事件的类型。

#### 转换方式

假设事件签名是：

```
Transfer(address,address,uint256)
```

我们使用 Keccak-256 对该字符串做哈希运算（注意不要有空格），结果就是 `topics[0]` 的值。

#### 示例：常见事件签名的 `topic[0]` 哈希值

| 事件名称        | 签名                                                    | `topics[0]` 哈希值                                           |
| --------------- | ------------------------------------------------------- | ------------------------------------------------------------ |
| ERC20 Transfer  | `Transfer(address,address,uint256)`                     | `0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef` |
| ERC20 Approval  | `Approval(address,address,uint256)`                     | `0x8c5be1e5ebec7d5bd14f714fce0b77a1c53a9c06d0d2cd7f7f6f0a2b93d2fbe0` |
| ERC721 Transfer | `Transfer(address,address,uint256)`                     | 与 ERC20 的相同签名，因此相同 topic[0]                       |
| Uniswap V2 Swap | `Swap(address,uint256,uint256,uint256,uint256,address)` | `0xd78ad95fa46c994b6551d0da85fc275fe613d1a72c56c60d7c6b6f8b9f1e7f86` |

#### 工具：如何自己计算

##### 方法 1：用 Python 代码（Web3.py）

```python
from web3 import Web3

event_signature = "Transfer(address,address,uint256)"
topic0 = Web3.keccak(text=event_signature).hex()
print(topic0)
```

输出：

```
0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
```

##### 方法 2：用在线工具

- 打开 https://emn178.github.io/online-tools/keccak_256.html

- 输入 `Transfer(address,address,uint256)`

- 选择 `Keccak-256`

- 得到哈希值（就是 `topics[0]`）

### 1. 查询某个代币的前 n 个发送地址

  ```
  0x4206931337dc273a630d328dA6441786BfaD668f
  ```

  - 事件：ERC20 `Transfer`
  - 维度：`from` 地址、value
  - 条件：特定 token 合约地址
  - 按转账金额降序

  ```sql
  select sum(bytearray_to_uint256("data")) as total_amount, topic1
  from ethereum.logs
  where block_time >= CURRENT_DATE - INTERVAL '7' DAY
  	and "contract_address" = 0x4206931337dc273a630d328dA6441786BfaD668f
  	and topic0 = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
  group by topic1 
  order by sum(bytearray_to_uint256("data")) desc
  limit 10
  ```

  ```sql
  select * from (
      select
          "topic1"
          , sum(bytearray_to_uint256("data")) as total_amount
      from ethereum.logs
      where block_time >= CURRENT_DATE - INTERVAL '30' DAY
         and "contract_address" = 0xdac17f958d2ee523a2206206994597c13d831ec7
         and "topic0" = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
      group by
          "topic1"
  ) order by total_amount desc
  ```

  ### 2. 查询最近 7 天内，收到最多 USDT 转账的用户地址

  ```
  USDT: 0xdAC17F958D2ee523a2206206994597C13D831ec7
  ```

  ```sql
  select topic2, sum(bytearray_to_uint256("data"))
  from ethereum.logs
  where block_time >= CURRENT_DATE - INTERVAL '7' DAY
         and "contract_address" = 0xdAC17F958D2ee523a2206206994597C13D831ec7
         and "topic0" = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
  group by topic2
  order by sum(bytearray_to_uint256("data")) desc
  limit 10
  ```

### 3. 查询某个永固地址发出的所有 ERC20 转账

- 事件：ERC20 `Transfer`
- 条件：`from` 为目标地址
- 输出：合约地址（代币）、接收地址、转账金额、时间戳

------

### 4. 统计 USDT 的活跃地址数（发送或接收）

```
USDT: 0xdAC17F958D2ee523a2206206994597C13D831ec7
```

- 事件：ERC20 `Transfer`
- 条件：合约地址 = 目标 token
- 提取唯一的 `from` 和 `to` 地址
- 统计活跃地址数（`distinct`）

------

### 5. 查询最近 24 小时内 USDT 的总交易量（以转账金额衡量）

```
USDT: 0xdAC17F958D2ee523a2206206994597C13D831ec7
```

- 事件：ERC20 `Transfer`
- 条件：合约地址 + 时间戳 > 24 小时前
- 汇总 `value`

------

### 6. 查询 USDT 最频繁交易的地址（发起 + 接收总次数）

- 事件：ERC20 `Transfer`
- 条件：合约地址 = 目标 token
- group by 地址（from 和 to 分别统计 + 合并）
- 排序：总次数降序

------

### 7. USDT 在不同时间段的交易活跃度趋势

- 事件：ERC20 `Transfer`
- 条件：合约地址
- group by 时间窗口（按小时/天）
- 输出：每个时间段的交易次数 or 总交易金额

------

### 8. 统计 USDT 的“鲸鱼地址”（持有或发送金额 > 阈值）

- 事件：ERC20 `Transfer`
- 条件：合约地址
- 汇总每个地址的发送 + 接收金额
- 过滤金额 > 某个高阈值（如 1M tokens）

------

### 9. 查询首次收到 USDT 的地址列表及时间

- 事件：ERC20 `Transfer`
- 条件：合约地址
- group by `to` 地址，取最早时间戳的一次

------

### 10. 统计某个用户地址参与最多的代币（发送+接收次数）

- 事件：ERC20 `Transfer`
- 条件：`from = address` 或 `to = address`
- group by 合约地址
- 统计次数（或金额）并排序

------

### 11. 查询 USDT 的大额转账记录（如超过 1M）

- 事件：ERC20 `Transfer`
- 条件：合约地址 + `value > 阈值`
- 输出：from、to、value、block time

------

### 12. USDT 的“合约与非合约地址”交互分析

- 事件：ERC20 `Transfer`
- 条件：合约地址
- 判断 `from` 或 `to` 是否为合约（需要结合 `code size`）
- 分类统计：合约地址 vs 普通地址 的交互数量、金额

## 三、Uniswap 相关事件分析

### 实现提示

```sql
-- 示例：查询 USDC/WETH 交易对的交易记录
SELECT 
    tx_hash,
    block_timestamp,
    "from" as trader,
    amount0In, amount0Out,
    amount1In, amount1Out
FROM ethereum_logs 
WHERE contract_address = '0xB4e16d0168e52d35CacdBe35Da4b227fa2bC9F0c'
  AND topic0 = '0xd78ad95fa46c994b6551d0da85fc275fe613ce37657fb8d5e3d130840159d822' -- Swap事件
  AND block_timestamp >= '2024-01-01'
ORDER BY block_timestamp DESC
```

具体可以参考 分析文档 中的代码模版。

### 1. 查询某个交易对的所有交易记录

- **事件**: `Swap`
- **条件**: 指定 pair 合约地址
- **输出**: 交易哈希、发起地址、token0/token1 数量变化、时间戳
- **示例地址**: USDC/WETH pair: `0xB4e16d0168e52d35CacdBe35Da4b227fa2bC9F0c`

### 2. 统计某个交易对的日交易量
- **事件**: `Swap`
- **条件**: 指定 pair 和时间范围
- **计算**: 按日期聚合 amount0In/Out 和 amount1In/Out
- **输出**: 日期、token0 交易量、token1 交易量、交易次数

### 3. 分析流动性提供者行为
- **事件**: `Mint` (添加流动性) 和 `Burn` (移除流动性)
- **条件**: 特定交易对或特定地址（在 eth scan 上找一个活跃的交易对）
- **输出**: LP 持有时长分布，找出 LP 添加流动性 和 移除流动性 的事件，并计算其中间隔。按照 LP 地址 group
- **进阶**: 在上面的基础上，找出添加/移除流动性的金额分布。

### 4. 分析交易对池子中的币种的数量的变化情况
- **事件**: `Mint` (添加流动性) 、 `Burn` (移除流动性)、交易事件
- **条件**: 特定交易对或特定地址（在 eth scan 上找一个活跃的交易对）
- **输出**: 字段如下：

1. 时间
2. 币种（是两个币种的哪一个）
3. 池子中币种的变化值，如果是变多，为正，变少为负数

- **进阶**：

1. 时间范围，时间点1 到 时间点2
2. 币种1
3. 币种2
4. 币种1在池子中的数量
5. 币种2在池子中的数量

### 5. 套利机器人识别与分析 [⭐️]
- **事件**: `Swap`
- **识别条件**:
  - 同一交易中多次 swap
  - 短时间内反向交易
  - MEV 特征（sandwich attack）
- **输出**: 疑似套利地址、套利频次、估算收益

### 6. 价格影响分析 [⭐️]
- **事件**: `Swap` + `Sync`
- **计算**: 
  - 大额交易对价格的冲击
  - 滑点分析
  - 价格恢复时间
- **输出**: 交易金额 vs 价格影响关系图

### 7. Uniswap V2 TVL 变化追踪 [⭐️⭐️]
- **事件**: `Mint`, `Burn`, `Swap`
- **数据源**: 多个主要交易对
- **计算**:
  - 实时 TVL 计算
  - 按交易对分解 TVL 占比
  - TVL 与交易量相关性分析

### 8. 代币价格发现分析 [⭐️⭐️]
- **目标**: 新代币首次在 Uniswap 上市后的价格走势
- **事件**: `PairCreated`, `Mint`, `Swap`
- **分析**:
  - 初始流动性规模
  - 早期交易者行为
  - 价格波动率变化
  - 与 CEX 价格对比（如适用）

### 9. 无常损失实证分析  [⭐️⭐️]
- **数据**: LP 的完整生命周期（Mint 到 Burn）
- **计算**:
  - 实际 LP 收益 vs Hold 策略收益
  - 不同价格波动幅度下的无常损失
  - 手续费收入补偿效果

------

## 四、合约交互行为分析

1. 分析特定合约的调用次数最多的地址
   - 根据日志里的 `address`（合约地址）筛选
   - 统计 topics[1]（可能是调用者）频率
2. 追踪某个项目或协议上线后的用户增长情况（unique address）
   - 事件：任意可识别交互行为的事件
   - 分析 logs 出现的唯一地址数量

------

## 五、安全与反洗钱分析

1. 识别与 Tornado Cash 合约交互过的地址
   - 合约地址过滤
   - topics 反推匿名存取款地址
2. 查找可疑批量转账行为（同一地址短时间多次转出）
   - 时间窗口内统计 logs 中相同 `from` 地址频率
3. 监控高价值转账事件（大于某阈值）
   - ERC20 `Transfer` 的 `value` 字段筛选

------

## 六、DeFi 协议行为监控

1. 分析 Aave、Compound 等借贷平台的借款事件
   - 事件如 `Borrow(address,address,uint256,uint256,uint256)`
   - 统计借款总量、借款地址等
2. 识别 MakerDAO 清算行为（Liquidation 事件）
   - 特定合约地址 + Liquidation 事件 topic

------

## 七、其他合约行为追踪

1. 识别 Gnosis Safe 的创建事件（Proxy 合约创建）
   - 特定工厂合约的事件，如 `ProxyCreation(address)`
2. 统计某个时间范围内新创建的 Uniswap V3 池子数量
   - 分析 `PoolCreated` 事件