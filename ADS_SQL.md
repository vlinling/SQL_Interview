以下是对你的问题的详细解答和重写。我们将逐步拆解 SQL 查询，并解释每一步的作用，确保逻辑清晰且易于理解。

---

### **Q1: 计算 2020 年非垃圾用户（`spam_indicator = 0`）在每个国家每天的广告展示占比（Ad Load）**

#### **需求分析**
我们需要计算广告展示占比（`ad_load`），即广告展示次数占总展示次数的比例。具体要求如下：
1. 按国家 (`country`) 和日期 (`dt`) 分组。
2. 只考虑非垃圾用户（`spam_indicator = 0`）。
3. 时间范围限制为 2020 年。

#### **重写后的SQL代码**
```sql
WITH AdImpressions AS (
    -- Step 1: 获取所有符合条件的展示记录
    SELECT 
        ud.country,
        pi.dt AS calendar_date,
        CASE WHEN pp.advertiser_id IS NOT NULL THEN 1 ELSE 0 END AS is_ad
    FROM pin_impressions pi
    LEFT JOIN promoted_pins pp 
        ON pi.pin_id = pp.pin_id AND pi.dt = pp.dt
    JOIN user_dimension ud 
        ON pi.user_id = ud.user_id
    WHERE ud.spam_indicator = 0 
      AND TO_DATE(pi.dt) BETWEEN TO_DATE('2020-01-01') AND TO_DATE('2020-12-31')
),
AdLoadByCountryDay AS (
    -- Step 2: 计算每个国家每天的广告展示占比
    SELECT 
        country,
        calendar_date,
        SUM(is_ad) AS ad_count,          -- 广告展示次数
        COUNT(*) AS total_count,         -- 总展示次数
        SUM(is_ad) / COUNT(*) AS ad_load -- 广告展示占比
    FROM AdImpressions
    GROUP BY country, calendar_date
)
-- Step 3: 输出最终结果
SELECT 
    country,
    calendar_date,
    ad_load
FROM AdLoadByCountryDay
ORDER BY country, calendar_date;
```

#### **逐步解释**
1. **AdImpressions CTE**:
   - 将 `pin_impressions` 表与 `promoted_pins` 表通过 `LEFT JOIN` 连接，判断每次展示是否为广告（`CASE WHEN` 判断 `advertiser_id` 是否为空）。
   - 同时与 `user_dimension` 表连接，过滤掉垃圾用户（`spam_indicator = 0`）。
   - 时间范围限制为 2020 年。

2. **AdLoadByCountryDay CTE**:
   - 按国家和日期分组，计算广告展示次数（`SUM(is_ad)`）和总展示次数（`COUNT(*)`）。
   - 广告展示占比公式：`SUM(is_ad) / COUNT(*)`。

3. **最终查询**:
   - 输出每个国家每天的广告展示占比，并按国家和日期排序。

---

### **Q2: 找出今天广告展示占比高于昨天的国家**

#### **需求分析**
基于 Q1 的结果，我们需要比较今天的广告展示占比与昨天的广告展示占比，找出占比更高的国家。

#### **重写后的SQL代码**
```sql
WITH AdLoad AS (
    -- Step 1: 复用 Q1 的结果
    SELECT 
        ud.country,
        pi.dt AS calendar_date,
        SUM(CASE WHEN pp.advertiser_id IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*) AS ad_load
    FROM pin_impressions pi
    LEFT JOIN promoted_pins pp 
        ON pi.pin_id = pp.pin_id AND pi.dt = pp.dt
    JOIN user_dimension ud 
        ON pi.user_id = ud.user_id
    WHERE ud.spam_indicator = 0 
      AND TO_DATE(pi.dt) BETWEEN TO_DATE('2020-01-01') AND TO_DATE('2020-12-31')
    GROUP BY ud.country, pi.dt
),
TodayVsYesterday AS (
    -- Step 2: 比较今天的广告展示占比与昨天的广告展示占比
    SELECT 
        t.country,
        t.calendar_date AS today_date,
        y.calendar_date AS yesterday_date,
        t.ad_load AS today_ad_load,
        y.ad_load AS yesterday_ad_load
    FROM AdLoad t
    JOIN AdLoad y 
        ON t.country = y.country 
       AND DATEDIFF(t.calendar_date, y.calendar_date) = 1
    WHERE t.ad_load > y.ad_load
)
-- Step 3: 输出最终结果
SELECT 
    country,
    today_date,
    today_ad_load,
    yesterday_ad_load
FROM TodayVsYesterday
WHERE today_date = CURRENT_DATE;
```

#### **逐步解释**
1. **AdLoad CTE**:
   - 直接复用 Q1 的结果，计算每个国家每天的广告展示占比。

2. **TodayVsYesterday CTE**:
   - 使用自连接（`JOIN`）将今天的广告展示占比与昨天的广告展示占比进行比较。
   - 条件是：`DATEDIFF(t.calendar_date, y.calendar_date) = 1`，即今天的日期比昨天的日期多一天。
   - 筛选出今天广告展示占比高于昨天的记录。

3. **最终查询**:
   - 输出符合条件的国家、今天的日期、今天的广告展示占比以及昨天的广告展示占比。
   - 限制条件为今天的日期等于当前日期（`CURRENT_DATE`）。

---

### **Transaction 表的 Dense Rank 示例**

#### **需求分析**
给定一个交易表（`Transaction`），需要对每个客户的交易记录按日期排序，并分配一个排名（`dense_rank`）。日期格式为 `MM/DD/YYYY`。

#### **重写后的SQL代码**
```sql
WITH RankedTransactions AS (
    -- Step 1: 转换日期格式并分配排名
    SELECT 
        CustomerID,
        Date,
        STR_TO_DATE(Date, '%m/%d/%Y') AS formatted_date, -- 将日期转换为标准格式
        DENSE_RANK() OVER (PARTITION BY CustomerID ORDER BY STR_TO_DATE(Date, '%m/%d/%Y')) AS rank
    FROM Transaction
)
-- Step 2: 输出结果
SELECT 
    CustomerID,
    Date,
    formatted_date,
    rank
FROM RankedTransactions
ORDER BY CustomerID, rank;
```

#### **逐步解释**
1. **RankedTransactions CTE**:
   - 使用 `STR_TO_DATE` 函数将日期从字符串格式（`MM/DD/YYYY`）转换为标准日期格式。
   - 使用 `DENSE_RANK()` 函数对每个客户的交易记录按日期排序，并分配排名。
   - `PARTITION BY CustomerID` 表示按客户分组，`ORDER BY STR_TO_DATE(Date, '%m/%d/%Y')` 表示按日期排序。

2. **最终查询**:
   - 输出每个客户的交易记录及其对应的排名。
   - 按客户 ID 和排名排序，确保结果有序。

---

### **总结**
- **CTE 的作用**：CTE（Common Table Expressions）可以将复杂的查询分解为多个小步骤，提高代码的可读性和维护性。
- **逐步拆解**：每个 CTE 都完成一个特定的任务，例如过滤数据、计算指标或分配排名。
- **逻辑清晰**：通过分步处理，避免了嵌套查询的复杂性，便于调试和扩展。

你可以将上述内容直接复制到 Markdown 文件中，并上传到 GitHub。希望这些解释对你有帮助！如果有其他问题，请随时提问。 😊
