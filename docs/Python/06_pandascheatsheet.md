# Pandas库速查表及最佳实践

关于Pandas库的速查表，包括数据结构（Series和DataFrame）、数据导入导出、数据清洗、数据转换、数据筛选、数据聚合、时间序列、性能优化以及最佳实践。
 主要内容：

  1. 数据结构：Series和DataFrame
  2. 数据导入和导出
  3. 数据清洗（处理缺失值、重复值、异常值）
  4. 数据转换（类型转换、字符串处理、apply函数）
  5. 数据筛选（选择列、行、条件筛选）
  6. 数据聚合（groupby、聚合函数、透视表）
  7. 时间序列处理
  8. 性能优化和最佳实践



## 1.快速概览

Pandas 是 Python 数据科学生态系统的核心库，提供了高效、易用的数据结构和数据分析工具。它是数据处理、清洗和分析的首选工具。

---

## 2.核心数据结构

### 2.1 Series 创建和操作
```python
import pandas as pd
import numpy as np

# 创建 Series
s1 = pd.Series([1, 3, 5, np.nan, 6, 8])
s2 = pd.Series([10, 20, 30], index=['a', 'b', 'c'])
s3 = pd.Series({'a': 1, 'b': 2, 'c': 3})

print("Series 1:\n", s1)
print("Series 2:\n", s2)
print("Series 索引:", s2.index)
print("Series 值:", s2.values)

# Series 操作
print("前3个元素:", s1.head(3))
print("描述统计:", s1.describe())
print("非空值数量:", s1.count())
```

### DataFrame 创建和操作
```python
# 从字典创建 DataFrame
data = {
    'Name': ['Alice', 'Bob', 'Charlie', 'David'],
    'Age': [25, 30, 35, 28],
    'City': ['New York', 'London', 'Tokyo', 'Paris'],
    'Salary': [50000, 70000, 80000, 60000]
}
df = pd.DataFrame(data)
print("DataFrame:\n", df)

# 从列表创建
data_list = [
    ['Alice', 25, 'New York', 50000],
    ['Bob', 30, 'London', 70000],
    ['Charlie', 35, 'Tokyo', 80000],
    ['David', 28, 'Paris', 60000]
]
df2 = pd.DataFrame(data_list, columns=['Name', 'Age', 'City', 'Salary'])

# 从 NumPy 数组创建
arr = np.random.randn(4, 3)
df3 = pd.DataFrame(arr, columns=['A', 'B', 'C'])

print("DataFrame 基本信息:")
print("形状:", df.shape)
print("列名:", df.columns.tolist())
print("索引:", df.index.tolist())
print("数据类型:\n", df.dtypes)
```

---

## 📥 数据导入导出

### 读取各种格式
```python
# 读取 CSV 文件
df_csv = pd.read_csv('data.csv', encoding='utf-8')

# 读取 Excel 文件
df_excel = pd.read_excel('data.xlsx', sheet_name='Sheet1')

# 读取 JSON 文件
df_json = pd.read_json('data.json')

# 读取 SQL 数据库
import sqlite3
conn = sqlite3.connect('database.db')
df_sql = pd.read_sql_query('SELECT * FROM table_name', conn)

# 读取 HTML 表格
df_html = pd.read_html('https://example.com/table.html')[0]

# 最佳实践参数
df = pd.read_csv('large_file.csv',
                 encoding='utf-8',
                 parse_dates=['date_column'],  # 解析日期
                 dtype={'category_col': 'category'},  # 指定数据类型
                 usecols=['col1', 'col2', 'col3'],  # 只读取需要的列
                 nrows=1000)  # 只读取前1000行进行测试
```

### 数据导出
```python
# 导出为 CSV
df.to_csv('output.csv', index=False, encoding='utf-8')

# 导出为 Excel
df.to_excel('output.xlsx', sheet_name='Data', index=False)

# 导出为 JSON
df.to_json('output.json', orient='records', indent=2)

# 导出到 SQL
df.to_sql('table_name', conn, if_exists='replace', index=False)

# 导出为 HTML 表格
df.to_html('output.html', index=False)

# 导出为 Markdown（用于文档）
print(df.to_markdown(index=False))
```

---

## 🔍 数据查看和基本信息

### 数据探索
```python
# 创建示例数据
np.random.seed(42)
df = pd.DataFrame({
    'Name': ['Alice', 'Bob', 'Charlie', 'David', 'Eva'] * 20,
    'Age': np.random.randint(18, 65, 100),
    'Salary': np.random.normal(50000, 15000, 100),
    'Department': np.random.choice(['IT', 'HR', 'Finance', 'Marketing'], 100),
    'Join_Date': pd.date_range('2020-01-01', periods=100, freq='D')
})

print("=== 数据基本信息 ===")
print("前5行:\n", df.head())
print("\n后3行:\n", df.tail(3))
print("\n随机5行:\n", df.sample(5))
print("\n形状:", df.shape)
print("列名:", df.columns.tolist())
print("索引:", df.index.tolist())
print("\n数据类型:\n", df.dtypes)
print("\n信息摘要:")
df.info()
```

### 统计信息
```python
print("=== 统计信息 ===")
print("描述统计:\n", df.describe())
print("\n包含对象类型的描述统计:\n", df.describe(include='all'))
print("\n唯一值统计:")
for col in df.columns:
    print(f"{col}: {df[col].nunique()} 个唯一值")

print("\n缺失值统计:")
print(df.isnull().sum())
print("\n重复行统计:", df.duplicated().sum())

# 内存使用
print(f"\n内存使用: {df.memory_usage(deep=True).sum() / 1024**2:.2f} MB")
```

---

## 🧹 数据清洗

### 处理缺失值
```python
# 创建包含缺失值的数据
df_missing = df.copy()
df_missing.loc[::10, 'Salary'] = np.nan  # 每10行设置一个缺失值
df_missing.loc[::15, 'Age'] = np.nan

print("原始缺失值:")
print(df_missing.isnull().sum())

# 删除缺失值
df_dropna = df_missing.dropna()  # 删除任何包含缺失值的行
df_dropna_cols = df_missing.dropna(axis=1)  # 删除任何包含缺失值的列
df_dropna_thresh = df_missing.dropna(thresh=3)  # 至少3个非空值的行

# 填充缺失值
df_filled = df_missing.fillna({
    'Age': df_missing['Age'].median(),  # 用中位数填充年龄
    'Salary': df_missing['Salary'].mean()  # 用均值填充薪资
})

# 向前/向后填充
df_ffill = df_missing.fillna(method='ffill')  # 向前填充
df_bfill = df_missing.fillna(method='bfill')  # 向后填充

# 插值
df_interpolate = df_missing.interpolate()  # 线性插值

print("\n处理后缺失值:")
print(df_filled.isnull().sum())
```

### 处理重复值
```python
# 创建包含重复值的数据
df_dup = pd.concat([df, df.iloc[:5]], ignore_index=True)  # 添加重复行

print("重复行数:", df_dup.duplicated().sum())
print("重复行:\n", df_dup[df_dup.duplicated(keep=False)])  # 显示所有重复行

# 删除重复值
df_no_dup = df_dup.drop_duplicates()  # 删除完全重复的行
df_no_dup_subset = df_dup.drop_duplicates(subset=['Name', 'Age'])  # 基于特定列删除

print("删除重复后形状:", df_no_dup.shape)
```

### 数据类型转换
```python
# 数据类型优化
print("转换前数据类型:\n", df.dtypes)

# 转换数据类型
df_optimized = df.copy()
df_optimized['Age'] = df_optimized['Age'].astype('int16')  # 减小整数类型
df_optimized['Department'] = df_optimized['Department'].astype('category')  # 分类数据
df_optimized['Join_Date'] = pd.to_datetime(df_optimized['Join_Date'])  # 日期类型

print("\n转换后数据类型:\n", df_optimized.dtypes)
print(f"内存减少: {(1 - df_optimized.memory_usage(deep=True).sum() / df.memory_usage(deep=True).sum()) * 100:.1f}%")
```

---

## 🔄 数据选择和索引

### 基本选择方法
```python
# 选择列
names = df['Name']  # Series
name_age = df[['Name', 'Age']]  # DataFrame

# 选择行
first_5 = df.iloc[:5]  # 按位置选择
specific_rows = df.iloc[[0, 2, 4]]  # 按位置列表选择

# 按标签选择
df_indexed = df.set_index('Name')
alice_data = df_indexed.loc['Alice']  # 按索引标签选择
alice_bob = df_indexed.loc[['Alice', 'Bob']]  # 多个标签

print("列选择示例:")
print(names.head())
print("\n位置选择示例:")
print(first_5)
print("\n标签选择示例:")
print(alice_data)
```

### 条件筛选
```python
# 简单条件筛选
high_salary = df[df['Salary'] > 60000]
it_dept = df[df['Department'] == 'IT']
young_employees = df[df['Age'] < 30]

# 多条件筛选
young_high_salary = df[(df['Age'] < 30) & (df['Salary'] > 55000)]  # 且
it_or_hr = df[df['Department'].isin(['IT', 'HR'])]
name_starts_a = df[df['Name'].str.startswith('A')]

# 复杂条件
complex_condition = df[
    (df['Age'].between(25, 35)) & 
    (df['Salary'] > df['Salary'].median()) &
    (~df['Department'].isin(['HR']))
]

print("高薪资员工:\n", high_salary.head())
print("\nIT部门员工:\n", it_dept.head())
print("\n复杂条件筛选:\n", complex_condition.head())
```

### 复杂查询方法
```python
# 使用 query 方法
result1 = df.query('Salary > 60000 and Age < 35')
result2 = df.query('Department in ["IT", "Finance"]')
result3 = df.query('Name.str.startswith("C")', engine='python')

# 使用变量@var
min_salary = 50000
max_age = 40
result4 = df.query('Salary > @min_salary and Age < @max_age')

print("Query 结果:")
print(result1.head())
```

---

## 📈 数据转换

### 添加和修改列
```python
# 添加新列
df_transformed = df.copy()

# 基于现有列计算
df_transformed['Salary_K'] = df_transformed['Salary'] / 1000
df_transformed['Age_Group'] = pd.cut(df_transformed['Age'], 
                                   bins=[18, 25, 35, 45, 65],
                                   labels=['18-25', '26-35', '36-45', '46-65'])
df_transformed['Seniority'] = (df_transformed['Salary'] > df_transformed['Salary'].median()).map({True: 'High', False: 'Low'})

# 使用 apply
df_transformed['Name_Length'] = df_transformed['Name'].apply(len)
df_transformed['Salary_Category'] = df_transformed['Salary'].apply(
    lambda x: 'High' if x > 70000 else 'Medium' if x > 50000 else 'Low'
)

print("转换后的数据:")
print(df_transformed[['Name', 'Age', 'Salary', 'Age_Group', 'Seniority', 'Salary_Category']].head())
```



### 字符串操作
```python
# 字符串方法
df_str = df.copy()

df_str['Name_Upper'] = df_str['Name'].str.upper()
df_str['Name_Lower'] = df_str['Name'].str.lower()
df_str['Name_Length'] = df_str['Name'].str.len()
df_str['First_Letter'] = df_str['Name'].str[0]
df_str['Has_A'] = df_str['Name'].str.contains('a', case=False)

# 字符串分割和提取
df_str[['First_Name', 'Last_Name']] = df_str['Name'].str.split(' ', expand=True).fillna('')

print("字符串操作结果:")
print(df_str[['Name', 'Name_Upper', 'Name_Length', 'First_Letter', 'Has_A']].head())
```



### 数据分箱和离散化
```python
# 等宽分箱
df['Salary_Bins'] = pd.cut(df['Salary'], bins=5)

# 等频分箱
df['Salary_Quantile'] = pd.qcut(df['Salary'], q=4, labels=['Low', 'Medium', 'High', 'Very High'])

# 自定义分箱
bins = [0, 30000, 50000, 70000, 1000000]
labels = ['Very Low', 'Low', 'Medium', 'High']
df['Salary_Custom'] = pd.cut(df['Salary'], bins=bins, labels=labels)

print("分箱结果:")
print(df[['Salary', 'Salary_Bins', 'Salary_Quantile', 'Salary_Custom']].head(10))
```

---



## 📊 数据聚合和分组

### GroupBy 操作
```python
# 基本分组
grouped = df.groupby('Department')

print("分组统计:")
print("各组大小:\n", grouped.size())
print("\n平均薪资:\n", grouped['Salary'].mean())
print("\n多列聚合:\n", grouped[['Age', 'Salary']].mean())

# 多列分组
multi_grouped = df.groupby(['Department', 'Age_Group'])
print("\n多列分组统计:\n", multi_grouped['Salary'].mean())
```

### 聚合函数
```python
# 多种聚合函数
agg_results = df.groupby('Department').agg({
    'Age': ['mean', 'min', 'max', 'std'],
    'Salary': ['mean', 'median', 'sum', 'count'],
    'Name': 'count'  # 员工数量
}).round(2)

print("详细聚合结果:")
print(agg_results)

# 自定义聚合函数
def salary_range(group):
    return group.max() - group.min()

custom_agg = df.groupby('Department').agg({
    'Salary': [salary_range, 'mean', 'std'],
    'Age': ['mean', 'std']
})

print("\n自定义聚合:")
print(custom_agg)
```

### 透视表和交叉表

```python
# 透视表
pivot_table = pd.pivot_table(df, 
                           values='Salary', 
                           index='Department', 
                           columns='Age_Group', 
                           aggfunc=['mean', 'count'],
                           fill_value=0,
                           margins=True)

print("透视表:")
print(pivot_table)

# 交叉表
cross_tab = pd.crosstab(df['Department'], df['Age_Group'], 
                       values=df['Salary'], 
                       aggfunc='mean',
                       margins=True)

print("\n交叉表:")
print(cross_tab.round(2))
```

---



## 🔄 数据合并和连接

### 合并操作

```python
# 创建示例数据
df1 = pd.DataFrame({
    'ID': [1, 2, 3, 4],
    'Name': ['Alice', 'Bob', 'Charlie', 'David'],
    'Age': [25, 30, 35, 28]
})

df2 = pd.DataFrame({
    'ID': [1, 2, 3, 5],
    'Salary': [50000, 70000, 80000, 60000],
    'Department': ['IT', 'HR', 'Finance', 'Marketing']
})

# 内连接
inner_join = pd.merge(df1, df2, on='ID', how='inner')

# 左连接
left_join = pd.merge(df1, df2, on='ID', how='left')

# 右连接
right_join = pd.merge(df1, df2, on='ID', how='right')

# 外连接
outer_join = pd.merge(df1, df2, on='ID', how='outer')

print("内连接:\n", inner_join)
print("\n左连接:\n", left_join)
print("\n右连接:\n", right_join)
print("\n外连接:\n", outer_join)
```

### 连接和拼接

```python
# 垂直拼接
df_top = df.head(3)
df_bottom = df.tail(3)
vertical_concat = pd.concat([df_top, df_bottom], axis=0)

# 水平拼接
df_left = df[['Name', 'Age']].head(3)
df_right = df[['Salary', 'Department']].head(3)
horizontal_concat = pd.concat([df_left, df_right], axis=1)

print("垂直拼接:\n", vertical_concat)
print("\n水平拼接:\n", horizontal_concat)

# 使用 join
df_indexed1 = df1.set_index('ID')
df_indexed2 = df2.set_index('ID')
joined = df_indexed1.join(df_indexed2, how='inner')

print("\nJoin 操作:\n", joined)
```

---



## 📅 时间序列处理

### 时间序列操作

```python
# 创建时间序列数据
dates = pd.date_range('2023-01-01', periods=100, freq='D')
ts_data = pd.DataFrame({
    'Date': dates,
    'Value': np.random.randn(100).cumsum() + 100,
    'Category': np.random.choice(['A', 'B', 'C'], 100)
})
ts_data = ts_data.set_index('Date')

print("时间序列数据:")
print(ts_data.head())

# 时间序列重采样
daily_data = ts_data['Value'].resample('D').mean()  # 每日（已经是每日）
weekly_data = ts_data['Value'].resample('W').mean()  # 每周
monthly_data = ts_data['Value'].resample('M').mean()  # 每月

print("\n重采样结果:")
print("每周均值:\n", weekly_data.head())
print("月度均值:\n", monthly_data.head())
```

### 时间序列特征工程

```python
# 创建时间特征
ts_features = ts_data.copy()
ts_features['DayOfWeek'] = ts_features.index.dayofweek
ts_features['DayOfMonth'] = ts_features.index.day
ts_features['Month'] = ts_features.index.month
ts_features['Quarter'] = ts_features.index.quarter
ts_features['IsWeekend'] = ts_features.index.dayofweek.isin([5, 6])

# 滚动统计
ts_features['Value_7D_Mean'] = ts_features['Value'].rolling(window=7).mean()
ts_features['Value_7D_Std'] = ts_features['Value'].rolling(window=7).std()
ts_features['Value_30D_Max'] = ts_features['Value'].rolling(window=30).max()

# 差分和百分比变化
ts_features['Value_Diff'] = ts_features['Value'].diff()
ts_features['Value_Pct_Change'] = ts_features['Value'].pct_change()

print("时间序列特征:")
print(ts_features[['Value', 'DayOfWeek', 'Month', 'Value_7D_Mean', 'Value_Pct_Change']].head(10))
```

---



## ⚡ 性能优化

### 数据类型优化

```python
def optimize_dtypes(df):
    """优化 DataFrame 的数据类型以减少内存使用"""
    original_memory = df.memory_usage(deep=True).sum() / 1024**2
    
    df_optimized = df.copy()
    
    # 优化数值列
    for col in df_optimized.select_dtypes(include=['int']).columns:
        c_min = df_optimized[col].min()
        c_max = df_optimized[col].max()
        
        if c_min > np.iinfo(np.int8).min and c_max < np.iinfo(np.int8).max:
            df_optimized[col] = df_optimized[col].astype(np.int8)
        elif c_min > np.iinfo(np.int16).min and c_max < np.iinfo(np.int16).max:
            df_optimized[col] = df_optimized[col].astype(np.int16)
        elif c_min > np.iinfo(np.int32).min and c_max < np.iinfo(np.int32).max:
            df_optimized[col] = df_optimized[col].astype(np.int32)
        else:
            df_optimized[col] = df_optimized[col].astype(np.int64)
    
    # 优化浮点数列
    for col in df_optimized.select_dtypes(include=['float']).columns:
        df_optimized[col] = df_optimized[col].astype(np.float32)
    
    # 转换对象列为分类数据
    for col in df_optimized.select_dtypes(include=['object']).columns:
        if df_optimized[col].nunique() / len(df_optimized[col]) < 0.5:
            df_optimized[col] = df_optimized[col].astype('category')
    
    optimized_memory = df_optimized.memory_usage(deep=True).sum() / 1024**2
    reduction = (1 - optimized_memory / original_memory) * 100
    
    print(f"内存使用: {original_memory:.2f} MB -> {optimized_memory:.2f} MB")
    print(f"减少: {reduction:.1f}%")
    
    return df_optimized

# 应用优化
df_optimized = optimize_dtypes(df)
```

### 高效操作方法

```python
import time

# 创建大型数据集
large_df = pd.DataFrame({
    'A': np.random.randn(1000000),
    'B': np.random.randint(0, 100, 1000000),
    'C': np.random.choice(['cat1', 'cat2', 'cat3', 'cat4'], 1000000)
})

# ❌ 不推荐：使用 iterrows()
def slow_method(df):
    result = []
    for index, row in df.iterrows():
        result.append(row['A'] * 2)
    return result

# ✅ 推荐：使用向量化操作
def fast_method(df):
    return df['A'] * 2

# ✅ 推荐：使用 apply（比 iterrows 快）
def apply_method(df):
    return df['A'].apply(lambda x: x * 2)

# 性能比较
start = time.time()
result1 = slow_method(large_df.head(1000))  # 只测试1000行
time1 = time.time() - start

start = time.time()
result2 = fast_method(large_df)
time2 = time.time() - start

start = time.time()
result3 = apply_method(large_df)
time3 = time.time() - start

print(f"iterrows 时间: {time1:.4f}秒 (1000行)")
print(f"向量化时间: {time2:.4f}秒")
print(f"apply 时间: {time3:.4f}秒")
print(f"向量化比 iterrows 快: {time1*1000/time2:.1f}x (按比例)")
```

### 使用查询优化

```python
# ❌ 不推荐：链式操作
result_slow = df[df['Age'] > 30][df['Salary'] > 50000][['Name', 'Department']]

# ✅ 推荐：单次查询
result_fast = df.loc[(df['Age'] > 30) & (df['Salary'] > 50000), ['Name', 'Department']]

# ✅ 推荐：使用 query
result_query = df.query('Age > 30 and Salary > 50000')[['Name', 'Department']]

print("查询结果一致:", result_slow.equals(result_fast) and result_fast.equals(result_query))
```

---



## 🎯 最佳实践

### 数据处理管道

```python
def create_data_pipeline():
    """创建数据处理管道"""
    
    # 1. 数据加载
    @pd.api.extensions.register_dataframe_accessor("pipeline")
    class PipelineAccessor:
        def __init__(self, pandas_obj):
            self._obj = pandas_obj
        
        def load_data(self, filepath, **kwargs):
            """加载数据"""
            return pd.read_csv(filepath, **kwargs)
        
        def clean_data(self, drop_duplicates=True, fill_missing=True):
            """数据清洗"""
            df = self._obj.copy()
            
            if drop_duplicates:
                df = df.drop_duplicates()
            
            if fill_missing:
                # 数值列用中位数填充
                numeric_cols = df.select_dtypes(include=[np.number]).columns
                df[numeric_cols] = df[numeric_cols].fillna(df[numeric_cols].median())
                
                # 类别列用众数填充
                category_cols = df.select_dtypes(include=['object', 'category']).columns
                for col in category_cols:
                    df[col] = df[col].fillna(df[col].mode()[0] if not df[col].mode().empty else 'Unknown')
            
            return df
        
        def feature_engineering(self):
            """特征工程"""
            df = self._obj.copy()
            
            # 添加时间特征（如果有日期列）
            date_columns = df.select_dtypes(include=['datetime64']).columns
            for col in date_columns:
                df[f'{col}_year'] = df[col].dt.year
                df[f'{col}_month'] = df[col].dt.month
                df[f'{col}_day'] = df[col].dt.day
            
            # 添加统计特征
            numeric_cols = df.select_dtypes(include=[np.number]).columns
            for col in numeric_cols:
                df[f'{col}_zscore'] = (df[col] - df[col].mean()) / df[col].std()
            
            return df
        
        def optimize(self):
            """优化数据类型"""
            return optimize_dtypes(self._obj)
    
    return PipelineAccessor

# 使用管道
Pipeline = create_data_pipeline()

# 模拟数据处理流程
# df_clean = df.pipeline.clean_data()
# df_features = df_clean.pipeline.feature_engineering()
# df_final = df_features.pipeline.optimize()
```

### 错误处理和验证

```python
def validate_dataframe(df, rules):
    """验证 DataFrame 是否符合业务规则"""
    
    errors = []
    warnings = []
    
    # 检查缺失值
    missing_cols = df.isnull().sum()
    if missing_cols.any():
        for col, count in missing_cols[missing_cols > 0].items():
            warnings.append(f"列 '{col}' 有 {count} 个缺失值")
    
    # 检查数据类型
    for col, expected_type in rules.get('dtypes', {}).items():
        if col in df.columns and df[col].dtype != expected_type:
            errors.append(f"列 '{col}' 的数据类型应该是 {expected_type}，但实际是 {df[col].dtype}")
    
    # 检查数值范围
    for col, (min_val, max_val) in rules.get('ranges', {}).items():
        if col in df.columns:
            if df[col].min() < min_val or df[col].max() > max_val:
                errors.append(f"列 '{col}' 的值超出范围 [{min_val}, {max_val}]")
    
    # 检查唯一性约束
    for col in rules.get('unique_columns', []):
        if col in df.columns and df[col].duplicated().any():
            errors.append(f"列 '{col}' 应该有唯一值，但存在重复")
    
    # 输出结果
    if errors:
        print("❌ 数据验证错误:")
        for error in errors:
            print(f"  - {error}")
    
    if warnings:
        print("⚠️  数据验证警告:")
        for warning in warnings:
            print(f"  - {warning}")
    
    if not errors and not warnings:
        print("✅ 数据验证通过!")
    
    return len(errors) == 0

# 定义验证规则
validation_rules = {
    'dtypes': {
        'Age': 'int64',
        'Salary': 'float64'
    },
    'ranges': {
        'Age': (18, 100),
        'Salary': (0, 1000000)
    },
    'unique_columns': ['Name']
}

# 执行验证
validate_dataframe(df, validation_rules)
```

### 配置和设置

```python
def setup_pandas_environment():
    """设置 Pandas 环境配置"""
    
    # 显示配置
    pd.set_option('display.max_rows', 100)
    pd.set_option('display.max_columns', 50)
    pd.set_option('display.width', 1000)
    pd.set_option('display.precision', 2)
    
    # 性能配置
    pd.set_option('compute.use_bottleneck', True)
    pd.set_option('compute.use_numexpr', True)
    
    # 绘图配置
    pd.set_option('plotting.backend', 'matplotlib')
    
    print("Pandas 环境配置完成!")

# 应用配置
setup_pandas_environment()
```

---



## 📚 总结

**核心优势：**

- ✅ **灵活的数据结构**：Series 和 DataFrame
- ✅ **强大的IO能力**：支持多种数据格式
- ✅ **丰富的数据操作**：清洗、转换、聚合、合并
- ✅ **高效的时间序列**：专业的时间序列处理
- ✅ **优秀的性能**：基于 NumPy 的高效计算

**最佳实践：**
- 🎯 使用向量化操作替代循环
- 🎯 合理选择数据类型减少内存使用
- 🎯 利用方法链构建数据处理管道
- 🎯 使用查询方法提高代码可读性
- 🎯 建立数据验证规则确保数据质量

**性能技巧：**
- ⚡ 使用 `astype()` 优化数据类型
- ⚡ 避免链式索引，使用 `.loc[]` 或 `.iloc[]`
- ⚡ 对分类数据使用 `category` 类型
- ⚡ 使用 `pd.eval()` 进行复杂表达式计算

**常见陷阱：**
- ❌ 在大型数据集上使用 `iterrows()`
- ❌ 忽略 `SettingWithCopyWarning`
- ❌ 不处理缺失值直接分析
- ❌ 内存不足时仍加载整个文件

掌握这些 Pandas 技巧将帮助你高效地进行数据分析和处理，构建可靠的数据科学工作流！
