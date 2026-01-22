

# 振幅  | SWING

在股票**日 K 线分析**中，**SWING（振幅）是一个基础但非常重要的波动性指标**，

用于衡量**当日价格的相对波动幅度**。其计算方法在各交易软件中基本一致。

------

## 一、SWING（振幅）的标准计算公式

**最常用、也是默认的定义为：**



$\huge \text{SWING（振幅）} = \frac{\text{当日最高价} - \text{当日最低价}}{\text{前一交易日收盘价}} \times 100%
$



------

## 二、公式逐项解释

| 项目             | 含义                       |
| ---------------- | -------------------------- |
| 当日最高价       | 当天成交过程中出现的最高价 |
| 当日最低价       | 当天成交过程中出现的最低价 |
| 前一交易日收盘价 | 昨天的收盘价，用于标准化   |
| ×100%            | 转换为百分比形式           |

------

## 三、为什么用“前收盘价”做分母？

这是一个**非常关键的设计点**：

1. **可比性**
   不同价格区间的股票可以横向比较波动强度
   - 10 元股振幅 5%
   - 100 元股振幅 5%
     → 波动意义相同
2. **与涨跌幅口径一致**
   - 涨跌幅 = (收盘价 − 前收) / 前收
   - 振幅同样以“前收”为基准
3. **符合交易所与行情系统习惯**
   - 国内主流行情系统（通达信、同花顺、东方财富等）统一采用该口径

------

## 四、一个简单数值例子

假设：

- 前收盘价 = 10.00
- 当日最高价 = 10.80
- 当日最低价 = 9.60

则：



$\huge \text{SWING} = \frac{10.80 - 9.60}{10.00} \times 100%
= 12%
$



说明：**当天股价在前收价的基础上，上下波动了 12% 的区间宽度。**

------

## 五、常见的几种“振幅”变体（你可能会遇到）

### 1️⃣ 以当日开盘价为分母（较少见）

$\huge \frac{H - L}{O}$

- 主要用于部分短线或盘口分析，不是主流口径。

------

### 2️⃣ 绝对振幅（不标准化）

$\huge H - L$



- 只反映价格“点数”波动，不具备跨股票比较能力。

------

### 3️⃣ 多日平均振幅（用于量化）

$\huge \text{AvgSwing}*N = \frac{1}{N} \sum*{i=1}^{N} \text{Swing}_i
$

常用于：

- 波动率因子
- 策略过滤条件（低波动 / 高波动）

------

## 六、SWING 在实战中的典型用途

| 用途       | 说明                              |
| ---------- | --------------------------------- |
| 判断活跃度 | 高振幅 ≈ 多空博弈激烈             |
| 辅助选股   | 低位放量 + 高振幅，常见于启动阶段 |
| 风控参数   | 高振幅股票通常需要更宽止损        |
| 量化因子   | 与收益、换手率、成交量组合        |

------

## 七、与 ATR、波动率的区别（很重要）

| 指标        | 核心差异                   |
| ----------- | -------------------------- |
| SWING       | 单日**区间波动**，简单直观 |
| ATR         | 多日平均真实波幅，考虑跳空 |
| 波动率（σ） | 收益率统计意义上的离散度   |

> 在**日 K 初筛、因子库设计**中，SWING 是一个非常合适的基础因子。

------

## 八、量化因子库

如果你计划将 SWING 纳入**量化因子库**，建议：

- **存储字段**
  - `high`
  - `low`
  - `pre_close`
  - `swing_pct`
- **可派生因子**
  - `swing_ma_5`
  - `swing_rank_industry`
  - `swing_zscore`





## 九、Pandas库和MySQL(8.0版本)数据库如何计算出振幅指标

下面分别给出**Pandas** 与 **MySQL（8.0）** 中计算 **SWING（振幅）** 的**标准、可直接使用**示例代码，口径统一为：



$\huge \text{SWING} = \frac{High - Low}{PreClose} \times 100
$



------

### 一、利用Pandas库计算 SWING（振幅）

#### 1️⃣ 基础示例（单股票或已排序数据）

假设你的 DataFrame 至少包含以下字段：

- `trade_date`
- `high`
- `low`
- `close`

```python
import pandas as pd

# 示例数据
df = pd.DataFrame({
    "trade_date": ["2025-01-02", "2025-01-03", "2025-01-06"],
    "high": [10.80, 11.20, 11.00],
    "low": [9.60, 10.50, 10.20],
    "close": [10.50, 10.90, 10.40]
})

# 确保按交易日排序
df = df.sort_values("trade_date")

# 前一交易日收盘价
df["pre_close"] = df["close"].shift(1)

# 计算振幅（百分比）
df["swing_pct"] = (df["high"] - df["low"]) / df["pre_close"] * 100

print(df)
```

------

#### 2️⃣ 多股票（推荐量化用法）

```python
df = df.sort_values(["symbol", "trade_date"])

df["pre_close"] = (
    df.groupby("symbol")["close"]
      .shift(1)
)

df["swing_pct"] = (
    (df["high"] - df["low"]) / df["pre_close"] * 100
)
```

------

#### 3️⃣ 常见注意事项（量化环境）

- 首日 `pre_close` 为 `NaN` → **正常**
- 可在计算后处理：

```python
df = df.dropna(subset=["swing_pct"])  # 可以填充为NULL，不建议删除。
```

------



### 二、MySQL 8.0 数据库中如何计算 SWING（振幅）

假设表结构如下（示例）：

```sql
CREATE TABLE stock_daily (
    symbol VARCHAR(20),
    trade_date DATE,
    high DECIMAL(10,2),
    low DECIMAL(10,2),
    close DECIMAL(10,2),
    PRIMARY KEY (symbol, trade_date)
);
```

------



#### 1️⃣ 查询时动态计算（使用窗口函数，推荐）

```sql
SELECT
    symbol,
    trade_date,
    high,
    low,
    close,
    LAG(close) OVER (
        PARTITION BY symbol
        ORDER BY trade_date
    ) AS pre_close,
    (high - low) / LAG(close) OVER (
        PARTITION BY symbol
        ORDER BY trade_date
    ) * 100 AS swing_pct
FROM stock_daily;
```

------



#### 2️⃣ 写入结果字段（生成因子列）

假设你已经新增字段：

```sql
ALTER TABLE stock_daily
ADD COLUMN swing_pct DECIMAL(8,4);
```

更新计算：

```sql
UPDATE stock_daily t
JOIN (
    SELECT
        symbol,
        trade_date,
        (high - low) / LAG(close) OVER (
            PARTITION BY symbol
            ORDER BY trade_date
        ) * 100 AS swing_pct
    FROM stock_daily
) s
ON t.symbol = s.symbol
AND t.trade_date = s.trade_date
SET t.swing_pct = s.swing_pct;
```

------



#### 3️⃣ 过滤首日无前收数据（可选）

```sql
WHERE LAG(close) OVER (
    PARTITION BY symbol
    ORDER BY trade_date
) IS NOT NULL
```

------



### 三、Pandas 与 MySQL 口径一致性建议（很重要）

| 项目 | 建议                                 |
| ---- | ------------------------------------ |
| 排序 | 必须按 `trade_date`                  |
| 分组 | 股票用 `symbol`，期货可用 `contract` |
| 精度 | MySQL 用 `DECIMAL`，避免 FLOAT       |
| 存储 | 建议存 `swing_pct`（百分比）         |

------

### 四、量化因子延伸（可选）

你后续可以很自然地派生：

- `swing_ma_5`
- `swing_std_20`
- `swing_rank_industry`
- `high_swing_flag`



























