# MySQL 数据库索引设计指南

MySQL数据库索引是提高查询性能的关键，但设计不当会导致性能下降和维护困难。

下面从基本概念、速查表和最佳工程实践三个方面进行阐述。

## 一、索引基本概念

### 1.1 什么是索引？
**索引**是数据库中的一种数据结构，用于快速查找和访问表中的数据。相当于书籍的目录。

索引是数据库表中一列或多列的值进行排序的一种数据结构，使用索引可以快速访问数据库表中的特定信息。



### 1.2 索引的数据结构
| 结构类型 | 适用场景 | MySQL存储引擎支持 |
|---------|---------|------------------|
| **B-Tree** | 范围查询、排序查询、精确查找 | InnoDB, MyISAM |
| **Hash** | 精确匹配、等值查询 | Memory, NDB |
| **R-Tree** | 空间数据、地理数据 | MyISAM |
| **Full-Text** | 全文搜索 | InnoDB(5.6+), MyISAM |



### 1.3 MySQL索引类型

- **主键索引（PRIMARY KEY）**：唯一且非空，一张表只能有一个。
- **唯一索引（UNIQUE）**：列值必须唯一，允许有空值。
- **普通索引（INDEX）**：最基本的索引，无唯一性限制。
- **组合索引（复合索引）**：多个列上创建的索引，遵循最左前缀原则。
- **全文索引（FULLTEXT）**：用于全文搜索，仅适用于MyISAM和InnoDB（MySQL5.6+）。
- **空间索引（SPATIAL）**：用于地理数据，适用于MyISAM。

```sql
-- 1. 主键索引 (Primary Key)
CREATE TABLE t (
    id INT PRIMARY KEY,  -- 自动创建主键索引
    name VARCHAR(100)
);

-- 2. 唯一索引 (Unique Index)
CREATE UNIQUE INDEX idx_email ON users(email);

-- 3. 普通索引 (Normal Index)
CREATE INDEX idx_name ON users(name);

-- 4. 全文索引 (Fulltext Index)
CREATE FULLTEXT INDEX idx_content ON articles(content);

-- 5. 空间索引 (Spatial Index)
CREATE SPATIAL INDEX idx_location ON places(location);
```



### 1.4 索引的物理实现
- **聚簇索引**：索引和数据存储在一起（InnoDB主键）
- **非聚簇索引**：索引和数据分开存储（二级索引）



## 二、索引设计速查表

### 2.1 索引创建语法速查
```sql
-- 建表时创建索引
CREATE TABLE employees (
    id INT PRIMARY KEY,
    emp_no VARCHAR(20),
    name VARCHAR(100),
    dept_id INT,
    hire_date DATE,
    salary DECIMAL(10,2),
    -- 主键索引
    PRIMARY KEY (id),
    -- 唯一索引
    UNIQUE KEY uk_emp_no (emp_no),
    -- 复合索引
    INDEX idx_dept_name (dept_id, name),
    -- 前缀索引
    INDEX idx_name_prefix (name(10)),
    -- 覆盖索引
    INDEX idx_cover (dept_id, hire_date, salary)
);

-- 修改表添加索引
ALTER TABLE employees ADD INDEX idx_hire_date (hire_date);
CREATE INDEX idx_salary ON employees(salary DESC);

-- 删除索引
DROP INDEX idx_name ON employees;
ALTER TABLE employees DROP INDEX idx_dept_name;
```



### 2.2 索引选择策略矩阵
| 查询类型 | 推荐索引类型 | 示例 |
|---------|-------------|------|
| **等值查询** | 唯一索引/普通索引 | `WHERE id = 100` |
| **范围查询** | B-Tree索引 | `WHERE age > 18 AND age < 30` |
| **排序查询** | 复合索引（排序字段在最后） | `ORDER BY create_time DESC` |
| **分组查询** | 复合索引（分组字段在前） | `GROUP BY dept_id, status` |
| **多列查询** | 复合索引（按查询频率排序） | `WHERE dept_id=1 AND status='active'` |
| **模糊查询** | 前缀索引 | `WHERE name LIKE '张%'` |
| **全文搜索** | 全文索引 | `MATCH(content) AGAINST('keyword')` |



### 2.3 索引设计检查清单

​	✅1.为WHERE子句中的列添加索引
□ 2. 为JOIN连接条件的列添加索引
□ 3. 为ORDER BY/GROUP BY的列添加索引
□ 4. 复合索引列顺序：等值列 → 范围列 → 排序列
□ 5. 索引列长度尽量短（使用合适的数据类型）
□ 6. 避免对频繁更新的列创建索引
□ 7. 为NULL值较少的列创建索引
□ 8. 考虑创建覆盖索引减少回表
□ 9. 定期分析索引使用情况
□ 10. 删除未使用或重复的索引

```

```



## 三、最佳工程实践



### 3.1 复合索引设计法则

#### 法则1：最左前缀原则
```sql
-- 索引: idx_a_b_c (a, b, c)

-- 有效使用索引的查询
WHERE a = 1 AND b = 2 AND c = 3      -- ✓ 全部使用
WHERE a = 1 AND b = 2                 -- ✓ 使用a,b
WHERE a = 1                           -- ✓ 使用a
WHERE a = 1 AND c = 3                 -- ✓ 使用a（c无法使用）
WHERE b = 2 AND c = 3                 -- ✗ 无法使用索引（缺少a）
WHERE a = 1 ORDER BY b, c            -- ✓ 使用a，并利用索引排序
```

#### 法则2：索引列顺序原则
```sql
-- 推荐的列顺序
CREATE INDEX idx_good_order ON table (
    equality_columns,      -- 等值条件（=, IN）
    range_columns,         -- 范围条件（>, <, BETWEEN）
    order_by_columns,      -- 排序字段
    include_columns        -- 覆盖索引包含列（非必选）
);

-- 示例：查询 WHERE dept_id=10 AND salary>5000 ORDER BY hire_date
CREATE INDEX idx_example ON employees(dept_id, salary, hire_date);
```

#### 法则3：三星索引原则
- ⭐ 一星：WHERE条件匹配索引前缀
- ⭐⭐ 二星：ORDER BY/GROUP BY匹配索引顺序
- ⭐⭐⭐ 三星：SELECT列被索引覆盖



### 3.2 索引优化实战技巧

#### 技巧1：覆盖索引优化
```sql
-- 需要回表的查询
SELECT id, name, dept_id, salary 
FROM employees 
WHERE dept_id = 10;

-- 创建覆盖索引避免回表
CREATE INDEX idx_cover_dept ON employees(dept_id, name, salary);
-- 或使用索引条件推送
ALTER TABLE employees ADD INDEX idx_dept_id (dept_id);
```

#### 技巧2：索引下推优化（MySQL 5.6+）
```sql
-- MySQL 5.6之前：先通过索引找到所有dept_id=10的记录，然后回表过滤status
-- MySQL 5.6+：直接在索引层过滤status，减少回表
SELECT * FROM employees 
WHERE dept_id = 10 AND status = 'active';

-- 创建复合索引
CREATE INDEX idx_dept_status ON employees(dept_id, status);
```

#### 技巧3：索引合并优化
```sql
-- 当多个单列索引时，MySQL可能使用索引合并
CREATE INDEX idx_name ON employees(name);
CREATE INDEX idx_dept ON employees(dept_id);

-- 可能使用索引合并
SELECT * FROM employees 
WHERE name LIKE '张%' OR dept_id = 10;
```

### 3.3 高性能索引设计模式

#### 模式1：时间范围查询优化
```sql
-- 问题：按时间范围查询大表性能差
SELECT * FROM orders 
WHERE create_time BETWEEN '2024-01-01' AND '2024-01-31';

-- 解决方案1：分区表 + 索引
CREATE TABLE orders (
    id BIGINT,
    create_time DATETIME,
    -- 按月分区
    PRIMARY KEY (id, create_time)
) PARTITION BY RANGE (YEAR(create_time)*100 + MONTH(create_time)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    -- ...
);

-- 解决方案2：时序索引设计
CREATE INDEX idx_time_compressed ON orders (
    DATE(create_time),  -- 日期部分
    HOUR(create_time),  -- 小时部分
    id                  -- 保证唯一性
);
```

#### 模式2：枚举类型优化
```sql
-- 问题：状态字段选择性差
CREATE INDEX idx_status ON orders(status);  -- status只有几个值

-- 解决方案：复合索引 + 高选择性列在前
CREATE INDEX idx_status_time ON orders(status, create_time);

-- 更好的方案：单独为每个状态创建部分索引（MySQL 8.0+）
CREATE INDEX idx_status_active ON orders(create_time) 
WHERE status = 'active';

CREATE INDEX idx_status_completed ON orders(create_time) 
WHERE status = 'completed';
```



#### 模式3：JSON字段索引（MySQL 5.7+）
```sql
-- 问题：JSON字段查询慢
SELECT * FROM products 
WHERE JSON_EXTRACT(attributes, '$.color') = 'red';

-- 解决方案：生成列 + 索引
ALTER TABLE products 
ADD COLUMN color VARCHAR(20) 
GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(attributes, '$.color'))) STORED;

CREATE INDEX idx_color ON products(color);
```



### 3.4 索引监控与维护

#### 监控脚本
```sql
-- 1. 查看索引使用情况
SELECT 
    OBJECT_SCHEMA,
    OBJECT_NAME,
    INDEX_NAME,
    COUNT_READ,
    COUNT_FETCH,
    COUNT_INSERT,
    COUNT_UPDATE,
    COUNT_DELETE
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE OBJECT_SCHEMA = 'your_database'
ORDER BY COUNT_READ DESC;

-- 2. 查找未使用的索引
SELECT 
    s.OBJECT_SCHEMA,
    s.OBJECT_NAME,
    s.INDEX_NAME
FROM performance_schema.table_io_waits_summary_by_index_usage s
LEFT JOIN information_schema.statistics t 
    ON s.OBJECT_SCHEMA = t.TABLE_SCHEMA 
    AND s.OBJECT_NAME = t.TABLE_NAME 
    AND s.INDEX_NAME = t.INDEX_NAME
WHERE s.INDEX_NAME IS NOT NULL
    AND s.COUNT_STAR = 0
    AND t.NON_UNIQUE = 1
    AND t.SEQ_IN_INDEX = 1;

-- 3. 查看索引统计信息
SHOW INDEX FROM your_table;
ANALYZE TABLE your_table;  -- 更新统计信息
```

#### 维护脚本
```sql
-- 1. 定期重建碎片化索引
OPTIMIZE TABLE your_table;  -- 会锁表

-- 2. 在线重建索引（MySQL 8.0+）
ALTER TABLE your_table ALTER INDEX idx_name INVISIBLE;
ALTER TABLE your_table ALTER INDEX idx_name VISIBLE;

-- 3. 分批删除数据保持索引健康
DELETE FROM large_table 
WHERE id BETWEEN 1 AND 10000 
ORDER BY id 
LIMIT 1000;

-- 4. 使用pt-online-schema-change在线修改索引
pt-online-schema-change \
    --alter="DROP INDEX idx_old, ADD INDEX idx_new(column)" \
    D=your_db,t=your_table \
    --execute
```

### 3.5 索引设计反模式

#### 反模式1：过多索引
```sql
-- 问题：每个查询都建索引
CREATE INDEX idx_a ON t(a);
CREATE INDEX idx_b ON t(b);
CREATE INDEX idx_c ON t(c);
CREATE INDEX idx_a_b ON t(a, b);
CREATE INDEX idx_a_c ON t(a, c);
-- 导致：写入性能下降，磁盘空间浪费

-- 解决方案：合并索引
CREATE INDEX idx_a_b_c ON t(a, b, c);  -- 替换前三个索引
```

#### 反模式2：大字段索引
```sql
-- 问题：对TEXT/BLOB字段建索引
CREATE INDEX idx_content ON articles(content(500));  -- 前缀索引过长

-- 解决方案：使用生成列
ALTER TABLE articles 
ADD COLUMN content_hash CHAR(32) AS (MD5(content)) STORED;
CREATE INDEX idx_content_hash ON articles(content_hash);
```

#### 反模式3：随机值索引
```sql
-- 问题：UUID作为索引
CREATE INDEX idx_uuid ON users(uuid);  -- UUID是随机的，插入会导致页分裂

-- 解决方案1：使用有序UUID
CREATE INDEX idx_ordered_uuid ON users(
    UUID_TO_BIN(uuid, TRUE)  -- MySQL 8.0+
);

-- 解决方案2：使用自增ID+UUID
CREATE TABLE users (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    uuid CHAR(36) UNIQUE,
    INDEX idx_uuid (uuid)  -- 二级索引影响较小
);
```

### 3.6 索引设计决策树
```
开始索引设计
    │
    ├── 查询有WHERE条件？
    │   ├── 是 → 为WHERE列建索引
    │   └── 否 → 跳过
    │
    ├── 查询有JOIN？
    │   ├── 是 → 为JOIN列建索引
    │   └── 否 → 跳过
    │
    ├── 查询有ORDER BY/GROUP BY？
    │   ├── 是 → 考虑复合索引包含这些列
    │   └── 否 → 跳过
    │
    ├── 查询SELECT所有列？
    │   ├── 是 → 考虑覆盖索引
    │   └── 否 → 只索引查询列
    │
    ├── 表是否频繁更新？
    │   ├── 是 → 谨慎添加索引
    │   └── 否 → 可适当增加索引
    │
    └── 评估索引选择性
        ├── 选择性 > 10% → 适合建索引
        └── 选择性 ≤ 10% → 考虑其他优化
```

## 四、实际案例优化

### 案例：电商订单查询优化
```sql
-- 原始表结构
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    status ENUM('pending','paid','shipped','completed','cancelled'),
    amount DECIMAL(10,2),
    create_time DATETIME,
    update_time DATETIME
);

-- 常见查询
-- 1. 用户查看自己的订单
SELECT * FROM orders WHERE user_id = ? ORDER BY create_time DESC;

-- 2. 后台按状态和时间查询
SELECT * FROM orders 
WHERE status = ? 
AND create_time BETWEEN ? AND ?
ORDER BY id DESC
LIMIT 100;

-- 3. 统计用户订单金额
SELECT user_id, SUM(amount) 
FROM orders 
WHERE create_time >= ?
GROUP BY user_id
HAVING SUM(amount) > 1000;

-- 优化方案
-- 索引1：用户查询优化
CREATE INDEX idx_user_time ON orders(user_id, create_time DESC);

-- 索引2：后台查询优化
CREATE INDEX idx_status_time ON orders(status, create_time, id);

-- 索引3：统计查询优化
CREATE INDEX idx_time_user_amount ON orders(create_time, user_id, amount);

-- 分区方案（按月分区）
ALTER TABLE orders 
PARTITION BY RANGE (TO_DAYS(create_time)) (
    PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```



## 五、工具与资源

### 5.1 索引分析工具
```bash
# 1. EXPLAIN分析
EXPLAIN FORMAT=JSON SELECT ...;

# 2. MySQL Workbench Visual Explain

# 3. Percona Toolkit
pt-index-usage slow.log
pt-duplicate-key-checker

# 4. 使用sys schema（MySQL 5.7+）
USE sys;
SELECT * FROM schema_redundant_indexes;
SELECT * FROM schema_unused_indexes;
```

### 5.2 性能测试模板
```sql
-- 1. 准备测试数据
SET @start_time = NOW();

-- 2. 执行查询
SELECT ...;

-- 3. 计算耗时
SET @end_time = NOW();
SELECT TIMEDIFF(@end_time, @start_time) AS execution_time;

-- 4. 查看执行计划
EXPLAIN SELECT ...;

-- 5. 查看索引使用
SHOW STATUS LIKE 'Handler_read%';
```



## 总结

### 索引设计黄金法则
1. **选择性原则**：只为高选择性的列创建索引
2. **覆盖原则**：尽量让查询使用覆盖索引
3. **最左前缀原则**：复合索引的列顺序很重要
4. **适度原则**：索引不是越多越好
5. **维护原则**：定期监控和维护索引

### 性能优化检查点
- ✅ 使用EXPLAIN分析查询计划
- ✅ 监控慢查询日志
- ✅ 定期更新统计信息
- ✅ 删除未使用的索引
- ✅ 考虑使用分区表处理历史数据

记住：**最好的索引是那些支持你最常见查询的索引，而不是理论上的完美索引**。在实际项目中，要根据具体的查询模式和数据分布来设计索引。
