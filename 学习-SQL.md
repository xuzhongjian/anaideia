# 学习 SQL

```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    signup_date DATE
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    category VARCHAR(50),
    price DECIMAL(10, 2)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    product_id INT,
    quantity INT,
    order_date DATE,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);

-- 示例：插入 10000 个用户
INSERT INTO users (id, name, signup_date)
SELECT x, CONCAT('User_', x), DATEADD('DAY', -x % 365, CURRENT_DATE())
FROM system_range(1, 10000);

-- 插入 100 个商品
INSERT INTO products (id, name, category, price)
SELECT 1000 + x, CONCAT('Product_', x), 
       CASE WHEN x % 2 = 0 THEN 'Electronics' ELSE 'Furniture' END,
       ROUND(RAND() * 1000, 2)
FROM system_range(1, 100);

-- 插入 100000 个订单（用户随机、商品随机、数量 1-5）
INSERT INTO orders (id, user_id, product_id, quantity, order_date)
SELECT x, 
       FLOOR(RAND() * 10000) + 1,
       FLOOR(RAND() * 100) + 1001,
       FLOOR(RAND() * 5) + 1,
       DATEADD('DAY', -FLOOR(RAND() * 365), CURRENT_DATE())
FROM system_range(1, 100000);
```

## 学习目标：

- 掌握 `SELECT`, `WHERE`, `GROUP BY`, `JOIN`, 子查询、窗口函数等核心语法
- 能够独立完成中等难度的业务数据分析 SQL 题目
- 适应大厂数据分析岗/链上数据分析中的 SQL 使用场景

------

## Day 1：SQL 基础语法入门 + 数据筛选

### 目标：

- 熟悉数据库和表结构概念
- 学会使用 `SELECT`, `WHERE`, `ORDER BY`, `LIMIT` 等基础语句
- 掌握基本查询语法、筛选条件、排序语句

### 学习内容：

- [牛客 SQL 入门教程（精选题）](https://www.nowcoder.com/exam/oj?tab=SQL篇&topicId=199)
- 学习内容覆盖：
  - `SELECT column FROM table`
  - `WHERE` 条件判断（=, <>, >, IN, BETWEEN, LIKE）
  - `ORDER BY` + `LIMIT`

### 练习任务：

1. 查询所有用户的 `id`、`name` 和注册时间，按注册时间降序排列。
2. 查询价格超过 500 元的商品名称和价格。
3. 查询注册时间在 2023 年之后的用户。
4. 查询用户 `id` 为 1 的所有订单。
5. 查询商品类别为 `Electronics` 且价格在 200 到 500 元之间的商品。

------

## Day 2：多表查询 + 数据关联分析

### 目标：

- 熟悉连接多个表进行联合查询
- 理解 `JOIN` 的不同类型（`INNER`, `LEFT`, `RIGHT`）

### 学习内容：

- 联合查询的本质（连接两个数据集）
- [牛客 SQL JOIN 部分题库](https://www.nowcoder.com/exam/oj?tab=SQL篇&topicId=199)

### 重点讲解：

- `INNER JOIN`：交集，最常用
- `LEFT JOIN`：包含左表所有记录，右表不匹配时为 NULL
- `ON` 和 `USING` 的区别

### 练习任务：

1. 查询所有订单中每个用户的总下单数量（按用户分组）。
2. 查询每个商品的总销量（按 `product_id` 分组），只显示销量超过 100 的商品。
3. 查询每个用户的订单总金额（注意用 `quantity * price` 计算）。
4. 查询 2025 年的订单中，每天的下单总数量。
5. 查询下单数量最多的 10 个用户（显示用户 id 和总数量）。

------

## Day 3：聚合函数 + 分组统计

### 目标：

- 使用 `GROUP BY` 做汇总分析
- 掌握 `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`

### 学习内容：

- 聚合函数的使用场景
- `GROUP BY` + `HAVING` 用法
- 排名：`ORDER BY count(*) DESC LIMIT 10`

### 练习任务：

- 查询下单总金额排名前 5 的用户（用窗口函数 `RANK()`）。
- 查询每个用户最近的一次下单时间。
- 查询总金额大于平均订单金额的所有订单（用子查询）。
- 查询下单次数在用户中排名前 10% 的用户。
- 查询每个商品最近 3 次的销售记录（用 `ROW_NUMBER()`）。

------

## Day 4：子查询 + 窗口函数（Dune/业务分析常用）

### 目标：

- 使用子查询实现复杂筛选
- 掌握窗口函数（如 `ROW_NUMBER`, `RANK`, `SUM() OVER()`）

### 学习内容：

- 子查询：放在 `FROM`, `WHERE`, `SELECT` 中的情况
- 窗口函数：
  - `ROW_NUMBER() OVER(PARTITION BY ... ORDER BY ...)`
  - `SUM(...) OVER(...)`：累计值分析
- [Leetcode Pandas/SQL 题](https://leetcode.com/studyplan/30-days-of-sql/)

### 练习任务：

1. 查询每个用户的等级（下单总金额 > 5000 为 “高”，1000-5000 为 “中”，否则为 “低”）。
2. 查询商品名称包含 “Product_1” 的所有商品。
3. 查询下单日期在周末（星期六、日）的订单（`DAYOFWEEK()`）。
4. 查询每个月的订单总数（按月份聚合）。
5. 查询所有用户的注册年份，并统计每个年份注册了多少人。

------

## Day 5-6：实战模拟 + 项目小练习

### 目标：

- 运用所学完成一个完整业务分析 SQL 模拟任务

### 学习内容 & 练习项目：

在 Dune 上完成下面的查询：

- 表：`transactions`, `logs`, `erc20_transfers`
- 要求：
  - 每天的 swap 总笔数
  - 每个地址的首次交易时间
  - 哪些地址频繁调用 `approve`

------

## 补充资源

| 类型             | 推荐                                                         |
| ---------------- | ------------------------------------------------------------ |
| SQL 在线测试平台 | [Leetcode SQL](https://leetcode.com/problemset/database/)、[Mode SQL Tutorial](https://mode.com/sql-tutorial/) |
| 可视化练习平台   | [Dune SQL Editor](https://dune.com/queries)（可练链上数据）  |
| 中文题库         | [牛客 SQL 大厂题库](https://www.nowcoder.com/exam/oj?tab=SQL篇) |