# 使用 Dune

## 一、ETH 转账分析

### 1. 统计地址之间的 ETH 转账总金额

- 表：`ethereum.transactions`
- 字段：`from`, `to`, `value`
- 操作：聚合按 `from` 或 `to` 分组
- 目标：找出最活跃的发送者或接收者（按总金额排序）
- 可筛选时间范围，例如最近 30 天

```sql
## 7 天内发出 ETH 的地址的数量排名
SELECT
  "from" AS address,
  SUM("value") / 1e18 AS total_sent_eth,
  COUNT(*) AS tx_count
FROM
  ethereum.transactions
WHERE
  block_time >= CURRENT_DATE - INTERVAL '7' DAY
  AND value > 0
GROUP BY
  "from"
ORDER BY
  total_sent_eth DESC
LIMIT 100;
```

------

### 2. 识别某个地址的所有转账

- 表：`ethereum.transactions`
- 条件：`from = '0x...' OR to_address = '0x...'`
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
  - 收入 = 所有 `to = 该地址` 的 value 之和
  - 支出 = 所有 `from = 该地址` 的 value 之和
  - 净值 = 收入 - 支出
  - 可筛选时间范围，例如最近 30 天

```sql
select sum(transfer_in) as sum_transfer_in, sum(transfer_out) sum_transfer_out, sum(transfer_in) - sum(transfer_out) as delta
from (select *, 
      (case when "to" = 0x974CaA59e49682CdA0AD2bbe82983419A2ECC400 then "value" / 1e18
            else 0 end
      ) as transfer_in, 
      (case when "from" = 0x974CaA59e49682CdA0AD2bbe82983419A2ECC400 then "value" / 1e18
            else 0 end
      ) as transfer_out
      from ethereum.transactions
      where block_time >= CURRENT_DATE - INTERVAL '30' DAY
        and ( "from" = 0x974CaA59e49682CdA0AD2bbe82983419A2ECC400
              or "to" = 0x974CaA59e49682CdA0AD2bbe82983419A2ECC400)
        and "value" > 0
     )
```

------

### 5. 统计每个地址的交易次数

- 表：`ethereum.transactions`
- 字段：`from`, `to`
- 操作：对每个地址出现的次数求和（无论是发送者还是接收者）
- 目标：找出最活跃的地址（按交易次数排序）
- 可筛选时间范围，例如最近 30 天

```sql
select
    active_address
    , count(1) as times
    , sum("value")/1e18 as total_value
from (
        select
            "to" as active_address,"value"
        from ethereum.transactions
        where block_time >= CURRENT_DATE - INTERVAL '30' DAY
        union
        select
            "from" as active_address,"value"
        from ethereum.transactions
        where block_time >= CURRENT_DATE - INTERVAL '30' DAY
)
group by
    active_address
order by count(1) desc
```

------

### 6. 找出最频繁的转账对

- 表：`ethereum.transactions`
- 字段：`from`, `to`, `value`
- 操作：按 `(from, to)` 分组并求和
- 目标：找出最频繁的转账对（按总金额或笔数排序）
- 识别地址对之间的累计转账总额（地址对 = from + to）
- 可筛选时间范围，例如最近 30 天

------

### 7. 分析某地址在高峰期的转账活动

- 表：`ethereum.transactions`
- 条件：`from = '0x...' OR to = '0x...'`
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

### 1. 查询某个代币的前 n 个发送地址

- 事件：ERC20 `Transfer`
- 维度：`from` 地址、value
- 条件：特定 token 合约地址
- 按转账金额降序

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

### 2. 查询某个地址收到最多转账的某个 ERC20 代币

- 汇总该地址为 `to` 的所有转账事件
- Group by 合约地址
- 按金额排序

```sql
select * from (
  select
    "topic2"
    , sum(bytearray_to_uint256 ("data")) / 1e6 as total_amount
  from ethereum.logs
  where block_time >= CURRENT_DATE - INTERVAL '60' DAY
  and "contract_address" = 0xdAC17F958D2ee523a2206206994597C13D831ec7
  and "topic0" = 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef
  and "topic3" is null
  group by "topic2"
)order by total_amount desc
```

### 3. 追踪某个 NFT 合约中交易最活跃的地址

- 事件：ERC721 `Transfer`
- 按交易次数分析买家/卖家地址出现频率

### 4. 分析某个 NFT 合约的 mint 活动

- 识别 mint 时间、mint 地址、mint 数量等
- from 为 0x0 的 Transfer 事件

## 三、Uniswap 相关事件分析

1. 查询某个地址在 Uniswap 上的 swap 总金额
   - 事件：`Swap`
   - 合约地址过滤为特定的池子或路由器
2. 统计每天的 Uniswap 总交易数量或总交易额
   - 按时间聚合 `Swap` 事件
3. 查找 Uniswap 上最频繁被 swap 的 token（按交易次数）
   - 分析 tokenIn/tokenOut 的合约地址频率

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
