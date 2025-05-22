# 使用 Dune

## 一、ethereum.transactions

### 1. **统计地址之间的 ETH 转账总金额**

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

### 2. **识别某个地址在一段时间内的所有转账（收款和付款）**



### 3. **找出 ETH 转账金额最大的一笔交易**



### 4. **分析某个地址的净转账额（收款 - 支出）**

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

### 5. **识别地址之间的双向转账关系（可能是交易对手或 CEX 热钱包）**

- 条件：A 转给 B，同时 B 又转给 A
