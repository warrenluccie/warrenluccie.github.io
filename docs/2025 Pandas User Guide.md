









# Group by: split-apply-combine

---

> https://pandas.pydata.org/pandas-docs/stable/user_guide/groupby.html



By “group by” we are referring to a process involving one or more of the following steps:

- **Splitting** the data into groups based on some criteria.
- **Applying** a function to each group independently.
- **Combining** the results into a data structure.

Out of these, the split step is the most **straightforward**. In the apply step, we might wish to do one of the following:

- **Aggregation聚合**: compute a summary statistic (or statistics) for each group. Some examples:

  > - Compute group sums or means.
  > - Compute group sizes / counts.

- **Transformation转换**: perform some group-specific computations and return a like-indexed object. Some examples:

  > - Standardize data (zscore) within a group.   对数据标准化操作
  > - Filling NAs within groups with a value derived from each group.

- **Filtration过滤**: discard some groups, according to a group-wise computation that evaluates to True or False. Some examples:

  > - Discard data that belong to groups with only a few members.
  > - Filter out data based on the group sum or mean.

Many of these operations are defined on GroupBy objects. These operations are similar to those of the [aggregating API](https://pandas.pydata.org/pandas-docs/stable/user_guide/basics.html#basics-aggregate), [window API](https://pandas.pydata.org/pandas-docs/stable/user_guide/window.html#window-overview), and [resample API](https://pandas.pydata.org/pandas-docs/stable/user_guide/timeseries.html#timeseries-aggregate).

It is possible that a given operation does not fall into one of these categories or is some combination of them. In such a case, it may be possible to compute the operation using GroupBy’s `apply` method. This method will examine the results of the apply step and try to **sensibly c**ombine them into a single result if it doesn’t fit into either of the above three categories.



> **:green_heart:NOTE:**
>
> **An operation that is split into multiple steps using built-in GroupBy operations will be more efficient than using the `apply` method with a user-defined Python function.**

**The name GroupBy should be quite familiar to those who have used a SQL-based tool (or `itertools`), in which you can write code like:**

```sql
SELECT Column1, Column2, mean(Column3), sum(Column4)
FROM SomeTable
GROUP BY Column1, Column2    # 先分组，再聚合
```



**We aim to make operations like this natural and easy to express using pandas. We’ll address each area of GroupBy functionality, then provide some non-trivial examples / use cases.**

**See the [cookbook](https://pandas.pydata.org/docs/user_guide/cookbook.html#cookbook-grouping) for some advanced strategies.**



## Splitting an object into groups[#](https://pandas.pydata.org/docs/user_guide/groupby.html#splitting-an-object-into-groups)

The abstract definition of grouping is to provide a mapping of labels to group names. To create a GroupBy object (more on what the GroupBy object is later), you may do the following:

```python
In [1]: speeds = pd.DataFrame(
   ...:     [
   ...:         ("bird", "Falconiformes", 389.0),
   ...:         ("bird", "Psittaciformes", 24.0),
   ...:         ("mammal", "Carnivora", 80.2),
   ...:         ("mammal", "Primates", np.nan),
   ...:         ("mammal", "Carnivora", 58),
   ...:     ],
   ...:     index=["falcon", "parrot", "lion", "monkey", "leopard"],
   ...:     columns=("class", "order", "max_speed"),
   ...: )
   ...: 

In [2]: speeds
Out[2]: 
          class           order  max_speed
falcon     bird   Falconiformes      389.0
parrot     bird  Psittaciformes       24.0
lion     mammal       Carnivora       80.2
monkey   mammal        Primates        NaN
leopard  mammal       Carnivora       58.0

In [3]: grouped = speeds.groupby("class")

In [4]: grouped = speeds.groupby(["class", "order"])
```



**The mapping can be specified many different ways:**

- **A Python function, to be called on each of the index labels.**
- **A list or NumPy array of the same length as the index.**
- **A dict or `Series`, providing a `label -> group name` mapping.**
- **For `DataFrame` objects, a string indicating either a column name or an index level name to be used to group.**
- **A list of any of the above things.**



> **A string passed to `groupby` may refer to either a column or an index level. If a string matches both a column name and an index level name, a `ValueError` will be raised.**

**Collectively we refer to the grouping objects as the keys. For example, consider the following `DataFrame`:**

```python
In [5]: df = pd.DataFrame(
   ...:     {
   ...:         "A": ["foo", "bar", "foo", "bar", "foo", "bar", "foo", "foo"],
   ...:         "B": ["one", "one", "two", "three", "two", "two", "one", "three"],
   ...:         "C": np.random.randn(8),
   ...:         "D": np.random.randn(8),
   ...:     }
   ...: )
   ...: 

In [6]: df
Out[6]: 
     A      B         C         D
0  foo    one  0.469112 -0.861849
1  bar    one -0.282863 -2.104569
2  foo    two -1.509059 -0.494929
3  bar  three -1.135632  1.071804
4  foo    two  1.212112  0.721555
5  bar    two -0.173215 -0.706771
6  foo    one  0.119209 -1.039575
7  foo  three -1.044236  0.271860
```



**On a DataFrame, we obtain a GroupBy object by calling [`groupby()`](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.groupby.html#pandas.DataFrame.groupby). This method returns a `pandas.api.typing.DataFrameGroupBy` instance. We could naturally group by either the `A` or `B` columns, or both:**

```python
In [7]: grouped = df.groupby("A")

In [8]: grouped = df.groupby("B")

In [9]: grouped = df.groupby(["A", "B"])
```



> **`df.groupby('A')` is just syntactic sugar for `df.groupby(df['A'])`.**

















# Indexing and selecting data



> https://pandas.pydata.org/docs/user_guide/indexing.html#different-choices-for-indexing

**The axis labeling information in pandas objects serves many purposes:**

- **Identifies data (i.e. provides *metadata*) using known indicators, important for analysis, visualization, and interactive console display.**
- **Enables automatic and explicit data alignment.**
- **Allows intuitive getting and setting of subsets of the data set.**

**In this section, we will focus on the final point: namely, how to slice, dice, and generally get and set subsets of pandas objects. The primary focus will be on Series and DataFrame as they have received more development attention in this area.**



 

> **:green_heart:NOTE:**
>
> **The Python and NumPy indexing operators `[]` and attribute operator `.` provide quick and easy access to pandas data structures across a wide range of use cases. This makes interactive work intuitive, as there’s little new to learn if you already know how to deal with Python dictionaries and NumPy arrays. However, since the type of the data to be accessed isn’t known in advance, directly using standard operators has some optimization limits. For production code, we recommended that you take advantage of the optimized pandas data access methods exposed in this chapter.**
>
> **indexing operator: 索引操作符    `[]`**
>
> **attribute operator:  属性操作符     `.`**



> **:warning:Warning**
>
> **Whether a copy or a reference is returned for a setting operation, may depend on the context. This is sometimes called `chained assignment` and should be avoided. See [Returning a View versus Copy](https://pandas.pydata.org/docs/user_guide/indexing.html#indexing-view-versus-copy).**
>
> **chained assignment** : 链式赋值尽量避免使用，返回view 还是 copy取决于上下文。







## Selection by label 基于标签的索引





## Selection by position 基于位置的索引







## Selection by callable 基于可调用对象的索引

**`.loc`, `.iloc`, and also `[]` indexing can accept a `callable` as indexer. **

**The `callable` must be a function with one argument (the calling Series or DataFrame) that returns valid output for indexing.**

`.loc`、`.iloc` 以及`[]` 索引可以接受一个可调用对象作为索引器。这个可调用对象必须是一个带有一个参数（调用的 Series 或 DataFrame）的函数，该函数返回用于索引的有效输出。

> **:information_source:NOTE:**
>
> **For `.iloc` indexing, returning a tuple from the callable is not supported, since tuple destructuring for row and column indexes occurs *before* applying callables.** 对于. iloc索引，不支持从可调用对象返回元组，因为行和列索引的元组解构发生在应用可调用对象之前。 



```python
In [102]: df1 = pd.DataFrame(np.random.randn(6, 4),
   .....:                    index=list('abcdef'),
   .....:                    columns=list('ABCD'))
   .....: 

In [103]: df1
Out[103]: 
          A         B         C         D
a -0.023688  2.410179  1.450520  0.206053
b -0.251905 -2.213588  1.063327  1.266143
c  0.299368 -0.863838  0.408204 -1.048089
d -0.025747 -0.988387  0.094055  1.262731
e  1.289997  0.082423 -0.055758  0.536580
f -0.489682  0.369374 -0.034571 -2.484478

# 筛选出A列中所有数值大于0的数据
In [104]: df1.loc[lambda df: df['A'] > 0, :]
Out[104]: 
          A         B         C         D
c  0.299368 -0.863838  0.408204 -1.048089
e  1.289997  0.082423 -0.055758  0.536580


# 筛选出所有行和A列及B列的数据
In [105]: df1.loc[:, lambda df: ['A', 'B']]
Out[105]: 
          A         B
a -0.023688  2.410179
b -0.251905 -2.213588
c  0.299368 -0.863838
d -0.025747 -0.988387
e  1.289997  0.082423
f -0.489682  0.369374

# 筛选出所有行和A列及B列的数据
In [106]: df1.iloc[:, lambda df: [0, 1]]
Out[106]: 
          A         B
a -0.023688  2.410179
b -0.251905 -2.213588
c  0.299368 -0.863838
d -0.025747 -0.988387
e  1.289997  0.082423
f -0.489682  0.369374


df1.columns
# Index(['A', 'B', 'C', 'D'], dtype='object')

# 筛选出A列所有的数据
In [107]: df1[lambda df: df.columns[0]]
Out[107]: 
a   -0.023688
b   -0.251905
c    0.299368
d   -0.025747
e    1.289997
f   -0.489682
Name: A, dtype: float64
```



**You can use callable indexing in `Series`.**

```python
In [108]: df1['A'].loc[lambda s: s > 0]   # passing series object to lambda.
Out[108]: 
c    0.299368
e    1.289997
Name: A, dtype: float64
```



**Using these methods / indexers, you can chain data selection operations without using a temporary variable.**



```python
In [109]: bb = pd.read_csv('data/baseball.csv', index_col='id')

In [110]: (bb.groupby(['year', 'team']).sum(numeric_only=True)
   .....:    .loc[lambda df: df['r'] > 100])
   .....: 
Out[110]: 
           stint    g    ab    r    h  X2b  ...     so   ibb   hbp    sh    sf  gidp
year team                                   ...                                     
2007 CIN       6  379   745  101  203   35  ...  127.0  14.0   1.0   1.0  15.0  18.0
     DET       5  301  1062  162  283   54  ...  176.0   3.0  10.0   4.0   8.0  28.0
     HOU       4  311   926  109  218   47  ...  212.0   3.0   9.0  16.0   6.0  17.0
     LAN      11  413  1021  153  293   61  ...  141.0   8.0   9.0   3.0   8.0  29.0
     NYN      13  622  1854  240  509  101  ...  310.0  24.0  23.0  18.0  15.0  48.0
     SFN       5  482  1305  198  337   67  ...  188.0  51.0   8.0  16.0   6.0  41.0
     TEX       2  198   729  115  200   40  ...  140.0   4.0   5.0   2.0   8.0  16.0
     TOR       4  459  1408  187  378   96  ...  265.0  16.0  12.0   4.0  16.0  38.0

[8 rows x 18 columns]
```























# 附录：如何理解Python callable object?

在Python中，**callable object**是指任何可以被调用（即使用圆括号`()`进行调用）的对象。**理解callable对象的关键在于知道它们的特性和用途。**

最为常见和常用的callable object是Python function.

### 1. **基本概念**
- **函数**：最常见的callable对象是函数（包括用户定义的函数和内置函数）。调用函数时，可以传入参数，并获取返回值。
  
  ```python
  def my_function():
      return "Hello, World!"
  
  print(my_function())  # 调用函数
  ```
  
- **类实例**：如果一个类定义了`__call__`方法，那么这个类的实例也可以被调用。调用实例时，会执行其`__call__`方法。
  
  ```python
  class MyCallable:
      def __call__(self):
          return "I am callable!"
  
  obj = MyCallable()
  print(obj())  # 调用实例
  ```

### 2. **判断一个对象是否可调用**
可以使用内置的`callable()`函数来检查一个对象是否是可调用的：
```python
print(callable(my_function))  # True
print(callable(obj))           # True
print(callable(42))            # False
```

### 3. **应用场景**
- **高阶函数**：在Python中，函数可以作为参数传递给其他函数，或者作为返回值返回。这使得函数可以用于回调机制、事件处理等场景。
  ```python
  def apply_function(func, value):
      return func(value)
  
  print(apply_function(lambda x: x + 1, 5))  # 输出 6
  ```

- **策略模式**：使用可调用对象作为策略，将不同的行为封装在不同的可调用对象中，便于动态选择和替换策略。
  
- **装饰器**：装饰器本质上是一个返回可调用对象的函数，用于修改或扩展其他函数的行为。
  
  ```python
  def my_decorator(func):
      def wrapper():
          print("Before call")
          func()
          print("After call")
      return wrapper
  
  @my_decorator
  def say_hello():
      print("Hello!")
  
  say_hello()  # 调用装饰过的函数
  ```

### 总结
在Python中，callable对象为代码的灵活性和可复用性提供了强大的支持。

它们使得编程更加抽象化，能够通过不同的方式实现功能，从而使得代码更易于维护和扩展。







# pandas.Series.apply

> **Invoke function on values of Series.**
>
> **Can be ufunc (a NumPy function that applies to the entire Series) or a Python function that only works on single values.**

- `pandas.Series.apply()` 是用于对 `pandas.Series` 中的每个元素应用一个函数的强大工具。

- 你可以传入自定义函数或 `pandas` 内置函数来对 `Series` 中的每个值进行逐一处理。

- 该方法特别适用于对数据进行逐项操作的场景，如转换、计算、分类等。

### `Series.apply()` 语法

```python
Series.apply(func, convert_dtype=True, args=(), **kwargs)
```

- **`func`**：应用到 `Series` 每个元素的函数，可以是内置函数或自定义函数。
- **`convert_dtype`**：是否转换输出的 `Series` 数据类型，默认为 `True`。
- **`args`**：传递给 `func` 的附加参数。
- **`**kwargs`**：传递给 `func` 的额外关键字参数。

### 示例及应用场景

#### 1. **使用内置函数**

可以直接使用 Python 的内置函数，如 `abs()`、`len()` 等，来处理每个元素。

##### 示例1：对每个元素取绝对值
```python
import pandas as pd

s = pd.Series([-1, 2, -3, 4])

# 对每个元素取绝对值
result = s.apply(abs)
print(result)
```
输出：
```
0    1
1    2
2    3
3    4
dtype: int64
```

#### 2. **使用自定义函数**
你可以传递一个自定义函数给 `apply()`，对 `Series` 中的每个元素进行自定义操作。

##### 示例2：对每个元素平方
```python
# 定义平方的函数
def square(x):
    return x ** 2

# 对每个元素平方
result = s.apply(square)
print(result)
```
输出：
```
0     1
1     4
2     9
3    16
dtype: int64
```

#### 3. **带参数的自定义函数**

自定义函数可以接受附加参数，并将这些参数传递给 `apply()`。

##### 示例3：对每个元素乘以一个常数
```python
# 定义乘法函数
def multiply(x, factor):
    return x * factor

# 对每个元素乘以 3
result = s.apply(multiply, args=(3,))
print(result)
```
输出：
```
0    -3
1     6
2    -9
3    12
dtype: int64
```

#### 4. **使用 `lambda` 表达式**

可以使用 `lambda` 表达式来简化代码，适用于简单的操作。

##### 示例4：用 `lambda` 表达式将每个元素加 10
```python
# 使用 lambda 表达式
result = s.apply(lambda x: x + 10)
print(result)
```
输出：
```
0     9
1    12
2     7
3    14
dtype: int64
```

#### 5. **处理字符串类型数据**

`apply()` 也可以用于处理字符串 `Series`，如字符串长度计算、大小写转换等。

##### 示例5：计算每个字符串的长度
```python
s_str = pd.Series(['apple', 'banana', 'cherry'])

# 计算每个字符串的长度
result = s_str.apply(len)
print(result)
```
输出：
```
0    5
1    6
2    6
dtype: int64
```

##### 示例6：将字符串转换为大写

```python
# 将字符串转换为大写
result = s_str.apply(lambda x: x.upper())
print(result)
```
输出：
```
0     APPLE
1    BANANA
2    CHERRY
dtype: object
```

#### 6. **处理复杂的数据转换**

`apply()` 可以用于更复杂的数据转换或计算操作，特别适合结合自定义逻辑和条件处理。

##### 示例7：根据条件对值进行分类
```python
# 自定义分类函数
def categorize(x):
    if x < 0:
        return 'Negative'
    elif x > 0:
        return 'Positive'
    else:
        return 'Zero'

# 对每个元素进行分类
result = s.apply(categorize)
print(result)
```
输出：
```
0    Negative
1    Positive
2    Negative
3    Positive
dtype: object
```

### 总结
- `apply()` 是 `pandas` 中用于对 `Series` 中每个元素进行自定义操作的工具。
- 可以传递内置函数、自定义函数或 `lambda` 表达式，甚至可以结合更多复杂的条件进行操作。
- 常见应用场景包括：数据清洗、数据转换、数值计算、字符串处理等。





# pandas.DataFrame.apply   

`DataFrame.apply()` 方法是 `pandas` 中非常强大和灵活的功能，它允许我们对 `DataFrame` 的行或列应用函数。

通过 `apply()`，我们可以实现自定义的操作，而不仅仅局限于内置的向量化方法。

`apply()` 可用于按行或按列应用函数，函数可以是内置的、lambda 表达式，或者是用户自定义函数。

### `DataFrame.apply()` 语法

```python
DataFrame.apply(func, axis=0, raw=False, result_type=None, args=(), **kwds)
```

- **`func`**：要应用的函数，可以是函数名、lambda 函数、或用户定义的函数。
- **`axis`**：
  - **`axis=0` 表示对列应用函数（按行操作，每次传入列数据，一列数据为Series对象）。**
  - **`axis=1` 表示对行应用函数（按列操作，每次传入行数据，一行数据为Series对象）。**
- **`raw`**：布尔值，如果为 `True`，则传递 `ndarray`，否则传递 `Series`（默认）。
- **`result_type`**：决定返回结果的形状。可以是 `'expand'`，'reduce'，或 `None`。

### 示例及应用场景

#### 1. **按列应用函数**

在这种情况下，我们对 `DataFrame` 的每一列应用一个函数。默认情况下，`apply()` 是按列进行操作的（即 `axis=0`）。

##### 示例1：对每一列求和
```python
import pandas as pd

df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': [4, 5, 6],
    'C': [7, 8, 9]
})

# 对每一列求和
result = df.apply(sum)
print(result)
```
输出：
```python
A    6
B    15
C    24
dtype: int64
```

##### 示例2：使用自定义函数计算每一列的平方和

```python
def sum_of_squares(series):
    return sum(x ** 2 for x in series)

result = df.apply(sum_of_squares)
print(result)
```
输出：
```python
A    14
B    77
C    194
dtype: int64
```

#### 2. **按行应用函数**

通过设置 `axis=1`，我们可以对 `DataFrame` 的每一行应用函数。

##### 示例3：计算每一行的最大值
```python
result = df.apply(max, axis=1)
print(result)
```
输出：
```
0    7
1    8
2    9
dtype: int64
```

##### 示例4：按行应用 lambda 表达式，将每一行的值相加

```python
result = df.apply(lambda row: row['A'] + row['B'] + row['C'], axis=1)
print(result)
```
输出：
```
0    12
1    15
2    18
dtype: int64
```

#### 3. **使用 `apply()` 与自定义函数结合**

`apply()` 的一个重要优势是可以将用户自定义函数应用到 `DataFrame` 的行或列上。

##### 示例5：按行计算并添加新列

假设我们有一组数据表示员工的工资和奖金，想要计算他们的总收入：
```python
data = {
    'Salary': [50000, 60000, 70000],
    'Bonus': [5000, 7000, 8000]
}
df = pd.DataFrame(data)

# 定义一个函数来计算总收入
def total_income(row):
    return row['Salary'] + row['Bonus']

# 应用函数,新增一列来表示每个员工的年收入
df['Total Income'] = df.apply(total_income, axis=1)
print(df)
```
输出：
```
   Salary  Bonus  Total Income
0   50000   5000         55000
1   60000   7000         67000
2   70000   8000         78000
```

#### 4. **处理缺失值**

我们还可以使用 `apply()` 方法来处理缺失值。

##### 示例6：对空值进行处理

```python
import numpy as np

df = pd.DataFrame({
    'A': [1, 2, np.nan],
    'B': [np.nan, 5, 6],
    'C': [7, np.nan, 9]
})

# 填充缺失值为列的平均值
df = df.apply(lambda col: col.fillna(col.mean()))
print(df)
```
输出：
```
     A    B    C
0  1.0  5.5  7.0
1  2.0  5.0  8.0
2  1.5  6.0  9.0
```

#### 5. **使用 `apply()` 返回多个值**

有时候我们希望通过 `apply()` 返回多个结果，并将其展开为多个列。

##### 示例7：分割字符串并返回多个值

```python
df = pd.DataFrame({
    'Full Name': ['John Doe', 'Jane Smith', 'Jack Black']
})

# 分割全名并返回姓和名
# 一列变成新增加的两列
df[['First Name', 'Last Name']] = df['Full Name'].apply(lambda x: pd.Series(x.split()))
print(df)
```
输出：
```
     Full Name First Name Last Name
0     John Doe       John       Doe
1   Jane Smith       Jane     Smith
2   Jack Black       Jack     Black
```

### 总结
- `DataFrame.apply()` 是 `pandas` 中用于按行或按列应用函数的灵活方法，可以用于数据清洗、特征提取等任务。
- 通过自定义函数、lambda 表达式等方式，可以实现非常复杂的操作，而不仅仅局限于简单的内置函数。
- 适合的应用场景包括数据转换、填充缺失值、数据合并与拆分等任务。







# pandas.DataFrame.assign

> https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.assign.html#pandas.DataFrame.assign



```python
DataFrame.assign(**kwargs)   # 传递关键字参数
```

**Assign new columns to a DataFrame.**

**Returns a new object with all original columns in addition to new ones. Existing columns that are re-assigned will be overwritten.**

将新列分配给数据框。返回一个新对象，除了新列之外还包含所有原始列。重新分配的现有列将被覆盖。

> **Parameters:**
>
> - **\*\*kwargs**: dict of {str: callable or Series}
>
>   **The column names are keywords.** 
>
>   **If the values are callable, they are computed on the DataFrame and assigned to the new columns.** 
>
>   **The callable must not change input DataFrame (though pandas doesn’t check it). If the values are not callable, (e.g. a Series, scalar, or array), they are simply assigned.**

> **Returns:**
>
> - DataFrame
>
>   A new DataFrame with the new columns in addition to all the existing columns.

Notes

Assigning multiple columns within the same `assign` is possible. Later items in ‘**kwargs’ may refer to newly created or modified columns in ‘df’; items are computed and assigned into ‘df’ in order.



----



**`DataFrame.assign()` 方法是 `pandas` 中用于向 `DataFrame` 添加新列或修改现有列的便捷方法。**

**该方法可以基于现有列的计算来创建新列，而不修改原始 `DataFrame`，返回的是修改后的新 `DataFrame`。**

**它非常适用于管道操作，使代码更加简洁和可读。**

## `DataFrame.assign()` 语法

```python
DataFrame.assign(**kwargs)
```

- `**kwargs`：表示多个参数，每个参数是要添加或修改的列名和列的值。值可以是标量、数组、Series 或者是一个返回值的函数。

  

 **示例及应用场景**

## 1. **添加新列**

我们可以通过 `assign()` 方法向 `DataFrame` 中添加一列新的数据，并将其值设为常量或基于现有列的计算结果。

##### 示例1：添加常量列

```python
import pandas as pd

df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': [4, 5, 6]
})

# 添加一个新列 C，值为常量 10
new_df = df.assign(C=10)
print(new_df)
```
输出：
```
   A  B   C
0  1  4  10
1  2  5  10
2  3  6  10
```

## 2. **基于现有列添加新列**

`assign()` 可以基于现有的列进行计算，来添加新列。

##### 示例2：添加新列为两列的和

```python
# 添加一列 D，D 的值是 A 和 B 的和
new_df = df.assign(D=lambda x: x['A'] + x['B'])
print(new_df)
```
输出：
```
   A  B  D
0  1  4  5
1  2  5  7
2  3  6  9
```

##### 示例3：添加新列为已有列的平方
```python
# 添加一列 E，E 的值是 A 列的平方
new_df = df.assign(E=lambda x: x['A'] ** 2)
print(new_df)
```
输出：
```
   A  B  E
0  1  4  1
1  2  5  4
2  3  6  9
```

## 3. **修改已有列的值**

`assign()` 还可以用于修改现有列的值。

##### 示例4：修改现有列的值

```python
# 修改 B 列，使其每个值加 10
new_df = df.assign(B=lambda x: x['B'] + 10)
print(new_df)
```
输出：
```
   A   B
0  1  14
1  2  15
2  3  16
```

## 4. **链式操作中的应用**

`assign()` 在链式操作中非常有用，因为它不会修改原始的 `DataFrame`，而是返回一个新的 `DataFrame`，这使得可以将多步操作串联在一起。

##### 示例5：链式操作
```python
new_df = (df
          .assign(C=lambda x: x['A'] + x['B'])  # 添加列 C
          .assign(D=lambda x: x['C'] * 2)      # 基于 C 添加列 D
         )
print(new_df)
```
输出：
```
   A  B  C   D
0  1  4  5  10
1  2  5  7  14
2  3  6  9  18
```

## 5. **添加多列**

`assign()` 可以一次添加多列，每个列的值可以是基于现有列的不同计算。

##### 示例6：一次性添加多列

```python
new_df = df.assign(C=lambda x: x['A'] + x['B'], 
                   D=lambda x: x['A'] * x['B'])
print(new_df)
```
输出：
```
   A  B  C   D
0  1  4  5   4
1  2  5  7  10
2  3  6  9  18
```

## 6. **结合 `pandas.Series` 添加新列**

我们还可以直接传递 `pandas.Series` 对象作为新列的值。新列的值将按索引对齐到现有的 `DataFrame`。

##### 示例7：结合 `Series` 添加新列
```python
# 创建一个 Series，并添加为新列
new_column = pd.Series([10, 20, 30])
new_df = df.assign(C=new_column)
print(new_df)
```
输出：
```
   A  B   C
0  1  4  10
1  2  5  20
2  3  6  30
```

## 总结

- **`DataFrame.assign()` 是一种非常便捷的方式来向 `DataFrame` 添加新列或修改现有列。**
- **该方法的优势在于它不会修改原始 `DataFrame`，而是返回一个新的 `DataFrame`，这使其非常适合管道化操作。**
- **`assign()` 支持标量、函数或 `pandas.Series`，灵活性极强。**
- **常见的应用场景包括数据处理、特征工程以及链式数据转换。**







# DataFrame.dropna()方法

> https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.dropna.html#pandas.DataFrame.dropna

`DataFrame.dropna()` 是 `pandas` 中用于删除包含缺失值（`NaN`）的行或列的一个方法。它可以帮助我们清理数据，删除那些无效或不完整的数据点。通过灵活的参数设置，可以选择删除带有 `NaN` 的行或列。

### `DataFrame.dropna()` 语法

```python
DataFrame.dropna(axis=0, how='any', thresh=None, subset=None, inplace=False)

DataFrame.dropna(*, axis=0, how=<no_default>, thresh=<no_default>, subset=None, inplace=False, ignore_index=False)  # 官方网站API说明

```

> **Remove missing values.**
>
> **See the [User Guide](https://pandas.pydata.org/docs/user_guide/missing_data.html#missing-data) for more on which values are considered missing, and how to work with missing data.**

> **Parameters:**
>
> - **axis: {0 or ‘index’, 1 or ‘columns’}, default 0**
>
>   **Determine if rows or columns which contain missing values are removed.**
>
>   - 0, or ‘index’ : Drop rows which contain missing values.
>   - 1, or ‘columns’ : Drop columns which contain missing value.Only a single axis is allowed.
>
> - **how**{‘any’, ‘all’}, default ‘any’
>
>   Determine if row or column is removed from DataFrame, when we have at least one NA or all NA.‘any’ : If any NA values are present, drop that row or column.‘all’ : If all values are NA, drop that row or column.
>
> - **thresh**int, optional
>
>   Require that many non-NA values. Cannot be combined with how.
>
> - **subset**column label or sequence of labels, optional
>
>   Labels along other axis to consider, e.g. if you are dropping rows these would be a list of columns to include.
>
> - **inplace**bool, default False
>
>   Whether to modify the DataFrame rather than creating a new one.
>
> - **ignore_index**bool, default `False`
>
>   If `True`, the resulting axis will be labeled 0, 1, …, n - 1.



> **Returns:**
>
> - **DataFrame  or None**
>
>   **DataFrame with NA entries dropped from it or None if `inplace=True`.**









- **`axis`**：表示沿哪个轴删除缺失值。`axis=0` 表示删除行（默认），`axis=1` 表示删除列。
- **`how`**：定义删除的标准。
  - `'any'`**（默认）：只要行或列中有任何 `NaN`，就删除。**
  - `'all'`：**行或列中所有值都是 `NaN` 时，才删除。**
- **`thresh`**：表示保留多少非 `NaN` 值才不会被删除（即至少要有多少个非 `NaN` 值）。
- **`subset`**：指定在哪些列上检查 `NaN`，默认对所有列进行检查。
- **`inplace`**：是否直接在原 `DataFrame` 上进行操作。如果为 `True`，会修改原始 `DataFrame`，否则返回新的 `DataFrame`。

### 示例及应用场景

#### 1. **删除包含 `NaN` 的行（默认行为）**
默认情况下，`dropna()` 会删除任意包含缺失值的行（`axis=0`）。

##### 示例1：删除包含 `NaN` 的行
```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'A': [1, 2, np.nan],
    'B': [4, np.nan, 6],
    'C': [7, 8, 9]
})

# 删除包含 NaN 的行
new_df = df.dropna()
print(new_df)
```
输出：
```
     A    B  C
0  1.0  4.0  7
```
- 这里删除了第 1 和第 2 行，因为它们包含 `NaN`。

#### 2. **删除包含 `NaN` 的列**
通过设置 `axis=1`，可以删除包含缺失值的列。

##### 示例2：删除包含 `NaN` 的列
```python
# 删除包含 NaN 的列
new_df = df.dropna(axis=1)
print(new_df)
```
输出：
```
   C
0  7
1  8
2  9
```
- 这里删除了 `A` 和 `B` 列，因为它们包含 `NaN`。

#### 3. **按条件删除 `NaN`**
`how='all'` 只删除全是 `NaN` 的行或列。

##### 示例3：删除全是 `NaN` 的行
```python
df2 = pd.DataFrame({
    'A': [1, np.nan, np.nan],
    'B': [np.nan, np.nan, np.nan],
    'C': [7, 8, 9]
})

# 仅删除所有值为 NaN 的行
new_df = df2.dropna(how='all')
print(new_df)
```
输出：
```
     A   B  C
0  1.0 NaN  7
1  NaN NaN  8
2  NaN NaN  9
```
- 这里没有删除任何行，因为没有行全是 `NaN`。

#### 4. **使用 `thresh` 参数**
`thresh` 参数用于保留至少包含一定数量的非 `NaN` 值的行或列。

##### 示例4：保留至少有 2 个非 `NaN` 值的行
```python
# 删除非 NaN 值少于 2 个的行
new_df = df.dropna(thresh=2)
print(new_df)
```
输出：
```
     A    B  C
0  1.0  4.0  7
1  2.0  NaN  8
```
- 这里删除了第 2 行，因为它只有 1 个非 `NaN` 值。

#### 5. **使用 `subset` 参数**
`subset` 参数可以指定要在哪些列中检查缺失值。

##### 示例5：只检查特定列的缺失值
```python
# 仅检查 'A' 和 'B' 列
new_df = df.dropna(subset=['A', 'B'])
print(new_df)
```
输出：
```
     A    B  C
0  1.0  4.0  7
```
- 这里只检查 `A` 和 `B` 列，所以第 1 和第 2 行被删除，因为这些列中包含 `NaN`。

#### 6. **`inplace=True` 修改原始 `DataFrame`**
使用 `inplace=True`，可以直接修改原始 `DataFrame`，而不是返回一个副本。

##### 示例6：直接修改 `DataFrame`
```python
df.dropna(inplace=True)
print(df)
```
输出：
```
     A    B  C
0  1.0  4.0  7
```
- 这里 `df` 本身被修改，删除了包含 `NaN` 的行。

### 总结
- `dropna()` 是清理数据时删除缺失值的常用工具，支持多种参数进行灵活控制。
- 可以删除包含 `NaN` 的行或列，也可以根据某些列或非 `NaN` 值的数量进行保留或删除操作。
- 常见的应用场景包括处理缺失数据、数据清洗、删除无效记录等。



-----





# DataFrame.fillna()

> https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.fillna.html#pandas.DataFrame.fillna

`pandas.DataFrame.fillna()` 是用于将 `NaN`（缺失值）替换为指定值的函数。在数据清洗过程中，填补缺失值是非常常见的操作，`fillna()` 提供了多种灵活的方式来处理缺失值，比如用标量、字典、方法、插值等替换 `NaN`。

### `DataFrame.fillna()` 语法

```python
DataFrame.fillna(value=None, method=None, axis=None, inplace=False, limit=None, downcast=None)

DataFrame.fillna(value=None, *, method=None, axis=None, inplace=False, limit=None, downcast=<no_default>)  # 来源于官网
```

> **Fill NA/NaN values using the specified method.**

> Parameters:
>
> - **value**: scalar, dict, Series, or DataFrame
>
>   Value to use to fill holes (e.g. 0), alternately a dict/Series/DataFrame of values specifying which value to use for each index (for a Series) or column (for a DataFrame). Values not in the dict/Series/DataFrame will not be filled. This value cannot be a list.
>
> - **method**{‘backfill’, ‘bfill’, ‘ffill’, None}, default None
>
>   Method to use for filling holes in reindexed Series:ffill: propagate last valid observation forward to next valid.backfill / bfill: use next valid observation to fill gap.
>
>   ***Deprecated since version 2.1.0:* Use ffill or bfill instead.**
>
>   
>
>   *axis**{0 or ‘index’} for Series, {0 or ‘index’, 1 or ‘columns’} for DataFrame
>
>   Axis along which to fill missing values. For Series this parameter is unused and defaults to 0.
>
>   **inplace**bool, default False
>
>   If True, fill in-place. Note: this will modify any other views on this object (e.g., a no-copy slice for a column in a DataFrame).
>
>   **limit**int, default None
>
>   If method is specified, this is the maximum number of consecutive NaN values to forward/backward fill. In other words, if there is a gap with more than this number of consecutive NaNs, it will only be partially filled. If method is not specified, this is the maximum number of entries along the entire axis where NaNs will be filled. Must be greater than 0 if not None.
>
>   **downcast**dict, default is None
>
>   A dict of item->dtype of what to downcast if possible, or the string ‘infer’ which will try to downcast to an appropriate equal type (e.g. float64 to int64 if possible).



> **Returns:**
>
> - **Series/DataFrame or None**
>
>   **Object with missing values filled or None if `inplace=True`.**

- **`value`**：用来填充 `NaN` 的值，可以是标量、字典、`Series` 或 `DataFrame`。 
- **`method`**：填充方法：
  - `'ffill'`（向前填充）：用前一个非 `NaN` 值替换 `NaN`。
  - `'bfill'`（向后填充）：用后一个非 `NaN` 值替换 `NaN`。
- **`axis`**：沿哪个轴填充缺失值。`axis=0` 表示按列填充（默认），`axis=1` 表示按行填充。
- **`inplace`**：是否直接修改原 `DataFrame`。`True` 表示直接修改原始数据。
- **`limit`**：最多填充的 `NaN` 数量。
- **`downcast`**：是否尝试将数据类型降级，比如将浮点型转换为整型。

### 示例及应用场景

#### 1. **使用标量替换 `NaN`**
最常见的场景是用一个固定值（如 0、均值等）来替换 `NaN`。

##### 示例1：用 0 替换 `NaN`
```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    'A': [1, 2, np.nan],
    'B': [np.nan, 3, 4],
    'C': [5, np.nan, 6]
})

# 用 0 替换所有的 NaN
new_df = df.fillna(0)
print(new_df)
```
输出：
```
     A    B    C
0  1.0  0.0  5.0
1  2.0  3.0  0.0
2  0.0  4.0  6.0
```

#### 2. **使用字典为不同列填充不同值**
可以为不同的列指定不同的填充值，通过传递字典来实现。

##### 示例2：为不同列设置不同的填充值
```python
# 用 0 替换 A 列的 NaN，用 100 替换 B 列的 NaN
new_df = df.fillna({'A': 0, 'B': 100})
print(new_df)
```
输出：
```
     A      B    C
0  1.0  100.0  5.0
1  2.0    3.0  NaN
2  0.0    4.0  6.0
```

#### 3. **使用方法（如前向填充或后向填充）**
可以用前一个或后一个非缺失值来填充缺失值。

##### 示例3：向前填充（ffill）
```python
# 用前一个非 NaN 值填充 NaN
new_df = df.fillna(method='ffill')
print(new_df)
```
输出：
```
     A    B    C
0  1.0  NaN  5.0
1  2.0  3.0  5.0
2  2.0  4.0  6.0
```

##### 示例4：向后填充（bfill）
```python
# 用后一个非 NaN 值填充 NaN
new_df = df.fillna(method='bfill')
print(new_df)
```
输出：
```
     A    B    C
0  1.0  3.0  5.0
1  2.0  3.0  6.0
2  NaN  4.0  6.0
```

#### 4. **限制填充的数量**
`limit` 参数可以控制填充的最大次数，防止填充所有的缺失值。

##### 示例5：限制最多填充一个 `NaN`
```python
# 仅填充最多一个 NaN
new_df = df.fillna(method='ffill', limit=1)
print(new_df)
```
输出：
```
     A    B    C
0  1.0  NaN  5.0
1  2.0  3.0  5.0
2  2.0  4.0  6.0
```

#### 5. **基于列或行填充**
通过 `axis` 参数可以指定按列或按行进行填充。

##### 示例6：按行填充
```python
# 按行进行填充
new_df = df.fillna(method='ffill', axis=1)
print(new_df)
```
输出：
```
     A    B    C
0  1.0  1.0  5.0
1  2.0  3.0  3.0
2  NaN  4.0  6.0
```

#### 6. **基于数据类型降级**
有时在填充数据后可以尝试将数据类型降级，使用 `downcast` 参数。

##### 示例7：降级数据类型
```python
# 用 0 填充 NaN，并将浮点数降级为整数
new_df = df.fillna(0, downcast='infer')
print(new_df)
print(new_df.dtypes)
```
输出：
```
     A    B  C
0  1.0  0.0  5
1  2.0  3.0  0
2  0.0  4.0  6

A    float64
B    float64
C      int64
dtype: object
```

### 总结
- `fillna()` 是填补缺失值的强大工具，支持标量替换、方法替换（前向、后向填充）、字典替换等多种方式。
- 它适合处理缺失值问题，如数据清洗、填充缺失数据、保证数据完整性。
- 在复杂的数据清洗过程中，通过 `fillna()` 可以保持数据的连续性、完整性，并灵活处理不同列的缺失值。



-------









# DataFrame.sort_values()

`DataFrame.sort_values()` 是 `pandas` 用于对 `DataFrame` 的某一列或多列进行排序的方法。可以按升序或降序排序，支持单列、多列排序，并可以选择是否保持原 `DataFrame` 结构。

### `DataFrame.sort_values()` 语法

```python
DataFrame.sort_values(by, axis=0, ascending=True, inplace=False, kind='quicksort', na_position='last', ignore_index=False, key=None)
```

- **`by`**：要排序的列或列的列表。可以按单列或多列排序。
- **`axis`**：如果为 0（默认），则对行排序；如果为 1，则对列排序。
- **`ascending`**：是否升序排列，默认为 `True`。可以为布尔值或布尔列表，布尔列表可用于按多列排序时指定每列的排序顺序。
- **`inplace`**：是否直接在原 `DataFrame` 上排序。如果为 `True`，则不会返回副本，而是修改原数据。
- **`kind`**：指定排序算法，默认为 `'quicksort'`。可选：`'mergesort'`、`'heapsort'` 等。
- **`na_position`**：表示缺失值（`NaN`）在排序时的位置，`'first'` 将 `NaN` 放在前面，`'last'` 将 `NaN` 放在最后。
- **`ignore_index`**：是否忽略索引。`True` 会重置索引。
- **`key`**：可以为排序指定一个自定义函数，如 `str.lower` 等。

### 示例及应用场景

#### 1. **按单列升序排序**
默认情况下，`sort_values()` 会按升序对指定列排序。

##### 示例1：按 `A` 列升序排序
```python
import pandas as pd

df = pd.DataFrame({
    'A': [2, 1, 3],
    'B': [5, 4, 6]
})

# 按 A 列升序排序
sorted_df = df.sort_values(by='A')
print(sorted_df)
```
输出：
```
   A  B
1  1  4
0  2  5
2  3  6
```

#### 2. **按单列降序排序**
通过设置 `ascending=False`，可以按降序排序。

##### 示例2：按 `A` 列降序排序
```python
# 按 A 列降序排序
sorted_df = df.sort_values(by='A', ascending=False)
print(sorted_df)
```
输出：
```
   A  B
2  3  6
0  2  5
1  1  4
```

#### 3. **按多列排序**
当有多个排序条件时，可以指定多个列，且每列可以有不同的排序顺序。

##### 示例3：按 `A` 列升序，`B` 列降序排序
```python
df = pd.DataFrame({
    'A': [1, 2, 2],
    'B': [3, 2, 1]
})

# 按 A 列升序，B 列降序排序
sorted_df = df.sort_values(by=['A', 'B'], ascending=[True, False])
print(sorted_df)
```
输出：
```
   A  B
0  1  3
1  2  2
2  2  1
```

#### 4. **缺失值排序**
可以指定 `NaN` 排序的位置，默认 `NaN` 放在最后。

##### 示例4：指定缺失值位置
```python
import numpy as np

df = pd.DataFrame({
    'A': [2, np.nan, 3],
    'B': [1, 4, 2]
})

# 按 A 列升序排序，并将 NaN 放在前面
sorted_df = df.sort_values(by='A', na_position='first')
print(sorted_df)
```
输出：
```
     A  B
1  NaN  4
0  2.0  1
2  3.0  2
```

#### 5. **使用 `inplace` 修改原 `DataFrame`**
使用 `inplace=True`，可以在原 `DataFrame` 上直接进行排序，而不是返回副本。

##### 示例5：直接在原数据上进行排序
```python
df.sort_values(by='A', inplace=True)
print(df)
```
输出：
```
   A  B
1  1  4
0  2  5
2  3  6
```

#### 6. **忽略索引**
通过设置 `ignore_index=True`，可以重置排序后的索引。

##### 示例6：排序后重置索引
```python
# 排序后重置索引
sorted_df = df.sort_values(by='A', ignore_index=True)
print(sorted_df)
```
输出：
```
   A  B
0  1  4
1  2  5
2  3  6
```

### 总结
- `DataFrame.sort_values()` 主要用于对 `DataFrame` 的某一列或多列进行排序，支持升序、降序、多列组合排序等操作。
- 可以根据不同的需求选择是否忽略索引、对 `NaN` 进行特殊处理、使用不同的排序算法等。
- 常见的应用场景包括数据分析时的排序操作、按条件进行数据筛选与排序等。



---



# pandas.read_csv方法

`pandas.read_csv()` 是 `pandas` 中读取 CSV 文件的主要方法，能够将 CSV 格式的数据文件加载为 `DataFrame`，并提供了多种参数选项来处理不同的数据结构和文件编码。

### `read_csv()` 语法

```python
pandas.read_csv(filepath_or_buffer, sep=',', delimiter=None, header='infer', names=None, index_col=None,
                usecols=None, dtype=None, engine=None, converters=None, true_values=None, false_values=None,
                skiprows=None, nrows=None, na_values=None, keep_default_na=True, parse_dates=False, 
                infer_datetime_format=False, encoding=None, skip_blank_lines=True, ... )


# 官方最新API pandas 2.2
pandas.read_csv(filepath_or_buffer, *, sep=<no_default>, delimiter=None, header='infer', names=<no_default>, index_col=None, usecols=None, dtype=None, engine=None, converters=None, true_values=None, false_values=None, skipinitialspace=False, skiprows=None, skipfooter=0, nrows=None, na_values=None, keep_default_na=True, na_filter=True, verbose=<no_default>, skip_blank_lines=True, parse_dates=None, infer_datetime_format=<no_default>, keep_date_col=<no_default>, date_parser=<no_default>, date_format=None, dayfirst=False, cache_dates=True, iterator=False, chunksize=None, compression='infer', thousands=None, decimal='.', lineterminator=None, quotechar='"', quoting=0, doublequote=True, escapechar=None, comment=None, encoding=None, encoding_errors='strict', dialect=None, on_bad_lines='error', delim_whitespace=<no_default>, low_memory=True, memory_map=False, float_precision=None, storage_options=None, dtype_backend=<no_default>)

```

### 常用参数说明

- **`filepath_or_buffer`**：文件路径或类似文件的对象。可以是本地路径、URL、文件对象等。
- **`sep`**：分隔符，默认是逗号 `','`。
- **`delimiter`**：用于分隔字段的单字符字符串。通常和 `sep` 二选一。
- **`header`**：指定用作列名的行号，默认是 0 行作为列名。`None` 表示没有行作为列名。
- **`names`**：**如果没有列名，可以传入一个列表用于指定列名。**
- **`index_col`**：指定哪一列作为行索引，可使用列名或列索引。
- **`usecols`**：指定读取的列，可以是列名列表或列索引列表。
- **`dtype`**：设置列的数据类型，如 `{'column1': str, 'column2': int}`。
- **`parse_dates`**：自动解析日期，支持 `True/False` 或列名列表。
- **`skiprows`**：跳过文件开头的指定行数。
- **`nrows`**：读取的行数。
- **`na_values`**：用于指定哪些值表示缺失值。
- **`encoding`**：指定文件编码，如 `'utf-8'` 或 `'gbk'`。
- **`skip_blank_lines`**：是否跳过空行，默认 `True`。

### 示例及应用场景

#### 1. **读取简单的 CSV 文件**
##### 示例1：默认读取
```python
import pandas as pd

# 假设文件 'data.csv' 内容如下：
# name, age, city
# John, 25, New York
# Anna, 30, Los Angeles
# Mike, 22, Chicago
df = pd.read_csv('data.csv')
print(df)
```
输出：
```
    name  age         city
0   John   25     New York
1   Anna   30  Los Angeles
2   Mike   22      Chicago
```

#### 2. **指定分隔符**
CSV 文件中可能使用不同的分隔符，如 `;`。可以通过 `sep` 参数指定。

##### 示例2：使用分号分隔符
```python
# 假设文件 'data_semicolon.csv' 内容如下：
# name; age; city
# John; 25; New York
df = pd.read_csv('data_semicolon.csv', sep=';')
print(df)
```

#### 3. **指定列名和索引列**

如果 CSV 文件没有列名，可以通过 `names` 参数手动指定，并且可以指定某一列为索引。

##### 示例3：手动设置列名并指定索引列
```python
# 假设文件 'data_no_header.csv' 内容如下：
# John, 25, New York
# Anna, 30, Los Angeles
# Mike, 22, Chicago
df = pd.read_csv('data_no_header.csv', names=['name', 'age', 'city'], index_col='name')
print(df)
```
输出：
```
       age         city
name                   
John     25     New York
Anna     30  Los Angeles
Mike     22      Chicago
```

#### 4. **指定数据类型**
使用 `dtype` 指定列的数据类型，避免读取时自动推断类型可能带来的错误。

##### 示例4：将 `age` 列指定为字符串
```python
df = pd.read_csv('data.csv', dtype={'age': str})
print(df.dtypes)
```
输出：
```
name    object
age     object
city    object
dtype: object
```

#### 5. **解析日期**
如果 CSV 文件包含日期列，可以使用 `parse_dates` 参数解析日期。

##### 示例5：解析日期列
```python
# 假设文件 'data_dates.csv' 内容如下：
# name, birth_date
# John, 1995-02-20
# Anna, 1990-05-30
df = pd.read_csv('data_dates.csv', parse_dates=['birth_date'])
print(df)
```
输出：
```
    name birth_date
0   John 1995-02-20
1   Anna 1990-05-30
```

#### 6. **指定要读取的列**
使用 `usecols` 参数只读取所需的列，可以节省内存。

##### 示例6：读取 `name` 和 `age` 列
```python
df = pd.read_csv('data.csv', usecols=['name', 'age'])
print(df)
```

#### 7. **处理缺失值**
使用 `na_values` 自定义哪些值应被视为缺失值。

##### 示例7：将特定值处理为缺失值
```python
# 假设文件 'data_na.csv' 内容如下：
# name, age, city
# John, 25, New York
# Anna, NA, Los Angeles
df = pd.read_csv('data_na.csv', na_values=['NA'])
print(df)
```

#### 8. **跳过特定行**
使用 `skiprows` 跳过文件中的某些行。

##### 示例8：跳过文件头的前两行
```python
# 假设文件 'data_skip.csv' 内容如下：
# Skip this line
# And this line too
# name, age, city
# John, 25, New York
df = pd.read_csv('data_skip.csv', skiprows=2)
print(df)
```

### 总结
- `pandas.read_csv()` 是读取 CSV 文件并将其转化为 `DataFrame` 的核心方法。
- 常见应用场景包括：设置分隔符、手动指定列名和索引、数据类型转换、日期解析、处理缺失值等。







---

# pandas.merge 



`pandas.merge()` 是用于合并两个 `DataFrame` 的方法，通过一个或多个键连接数据表。它类似 SQL 中的 `JOIN` 操作，可以指定连接类型以及连接的列。

### `merge()` 语法

```python
pandas.merge(left, right, how='inner', on=None, left_on=None, right_on=None, 
             left_index=False, right_index=False, sort=False, suffixes=('_x', '_y'), 
             indicator=False, validate=None)
```

### 参数说明

- **`left`** 和 **`right`**：要合并的两个 `DataFrame`。
- **`how`**：指定连接方式，有四种可选值，默认是 `'inner'`。
  - `'inner'`：取交集，相当于 SQL 的 INNER JOIN。
  - `'outer'`：取并集，相当于 SQL 的 FULL OUTER JOIN。
  - `'left'`：以左表为基准，相当于 SQL 的 LEFT JOIN。
  - `'right'`：以右表为基准，相当于 SQL 的 RIGHT JOIN。
- **`on`**：用于连接的列名，左右两张表必须有相同的列名。
- **`left_on`** 和 **`right_on`**：用于连接的列名，分别对应左表和右表，可以连接不同列名的情况。
- **`left_index`** 和 **`right_index`**：使用行索引进行合并。可以分别设定 `True` 或 `False`。
- **`sort`**：是否对合并结果根据连接键进行排序，默认为 `False`。
- **`suffixes`**：连接后重叠列的后缀，默认为 `('_x', '_y')`。
- **`indicator`**：如果设置为 `True`，则添加一列 `_merge`，标识每一行的来源（'left_only', 'right_only', 'both'）。
- **`validate`**：用于验证合并条件是否符合要求，可选 `'one_to_one'`, `'one_to_many'`, `'many_to_one'`, `'many_to_many'`。

### 示例及应用场景

#### 1. **简单合并**
两个 `DataFrame` 共有一个相同的列，通过 `on` 参数指定该列即可。

##### 示例1：使用相同列合并
```python
import pandas as pd

df1 = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['Alice', 'Bob', 'Charlie']
})
df2 = pd.DataFrame({
    'id': [1, 2, 4],
    'score': [85, 90, 95]
})

# 通过 'id' 列合并
result = pd.merge(df1, df2, on='id', how='inner')
print(result)
```

输出：
```
   id     name  score
0   1    Alice     85
1   2      Bob     90
```

#### 2. **左连接和右连接**
左连接和右连接会保留左表或右表中所有的行数据，即使另一表中没有对应值。

##### 示例2：左连接
```python
result = pd.merge(df1, df2, on='id', how='left')
print(result)
```

输出：
```
   id     name  score
0   1    Alice   85.0
1   2      Bob   90.0
2   3  Charlie    NaN
```

#### 3. **外连接**
外连接会保留两张表中所有的行，并在没有匹配的值处填充 `NaN`。

##### 示例3：外连接
```python
result = pd.merge(df1, df2, on='id', how='outer')
print(result)
```

输出：
```
   id     name  score
0   1    Alice   85.0
1   2      Bob   90.0
2   3  Charlie    NaN
3   4      NaN   95.0
```

#### 4. **指定左右不同的连接列**
可以通过 `left_on` 和 `right_on` 参数指定不同的列进行连接。

##### 示例4：不同列合并
```python
df1 = pd.DataFrame({
    'student_id': [1, 2, 3],
    'name': ['Alice', 'Bob', 'Charlie']
})
df2 = pd.DataFrame({
    'id': [1, 2, 3],
    'score': [85, 90, 95]
})

result = pd.merge(df1, df2, left_on='student_id', right_on='id', how='inner')
print(result)
```

输出：
```
   student_id     name  id  score
0           1    Alice   1     85
1           2      Bob   2     90
2           3  Charlie   3     95
```

#### 5. **使用索引合并**
`left_index` 和 `right_index` 可用于使用行索引进行合并。

##### 示例5：使用索引合并
```python
df1 = pd.DataFrame({'name': ['Alice', 'Bob', 'Charlie']}, index=[1, 2, 3])
df2 = pd.DataFrame({'score': [85, 90, 95]}, index=[1, 2, 4])

result = pd.merge(df1, df2, left_index=True, right_index=True, how='outer')
print(result)
```

输出：
```
       name  score
1     Alice   85.0
2       Bob   90.0
3   Charlie    NaN
4       NaN   95.0
```

#### 6. **使用 `indicator` 参数查看行来源**
设置 `indicator=True` 可以添加一列 `_merge`，显示每一行的来源。

##### 示例6：使用 `indicator`
```python
result = pd.merge(df1, df2, left_index=True, right_index=True, how='outer', indicator=True)
print(result)
```

输出：
```
       name  score      _merge
1     Alice   85.0        both
2       Bob   90.0        both
3   Charlie    NaN   left_only
4       NaN   95.0  right_only
```

#### 7. **验证连接方式**
使用 `validate` 可以对合并关系进行验证。例如，验证一对一关系。

##### 示例7：验证一对一关系

```python
# 验证一对一关系
result = pd.merge(df1, df2, on='id', how='inner', validate='one_to_one')
```

若合并关系不符合，则会抛出异常。

### 总结

- `pandas.merge()` 是合并两个 `DataFrame` 的主要方法，支持 `inner`、`outer`、`left`、`right` 四种连接方式。
- 常用于数据整合、特征合并、数据校验等场景。







---

# pandas.concat



`pandas.concat()` 用于沿一个指定轴（行或列）将多个 `DataFrame` 或 `Series` 对象连接起来。与 `merge()` 不同，`concat()` 更关注将数据直接堆叠到一起，而不是基于键的合并。

### `concat()` 语法

```python
pandas.concat(objs, axis=0, join='outer', ignore_index=False, keys=None,
              levels=None, names=None, verify_integrity=False, sort=False, copy=True)
```

### 参数说明

- **`objs`**：一个列表或字典，包含要连接的 `DataFrame` 或 `Series` 对象。
- **`axis`**：连接的轴，`axis=0` 表示按行连接（垂直堆叠），`axis=1` 表示按列连接（水平合并）。默认为 `0`。
- **`join`**：指定连接方式，`'outer'` 表示外连接（默认），`'inner'` 表示内连接。
- **`ignore_index`**：如果为 `True`，忽略原始索引并生成新的索引。默认为 `False`。
- **`keys`**：为每个连接的片段增加一个多级索引的键。
- **`verify_integrity`**：检查合并后的数据是否存在重复索引，如果有则抛出异常。
- **`sort`**：指定是否对未对齐的轴进行排序（在 `axis=1` 且 `join='outer'` 时常用）。

### 示例及应用场景

#### 1. **按行连接多个 DataFrame**
使用 `axis=0`（默认）进行垂直堆叠。

##### 示例1：按行连接
```python
import pandas as pd

df1 = pd.DataFrame({'A': ['A0', 'A1', 'A2'], 'B': ['B0', 'B1', 'B2']})
df2 = pd.DataFrame({'A': ['A3', 'A4', 'A5'], 'B': ['B3', 'B4', 'B5']})

# 垂直堆叠两个 DataFrame
result = pd.concat([df1, df2])
print(result)
```

输出：
```
    A   B
0  A0  B0
1  A1  B1
2  A2  B2
0  A3  B3
1  A4  B4
2  A5  B5
```

#### 2. **按列连接多个 DataFrame**
使用 `axis=1` 对齐行索引，将多个 `DataFrame` 水平拼接。

##### 示例2：按列连接
```python
df1 = pd.DataFrame({'A': ['A0', 'A1', 'A2']})
df2 = pd.DataFrame({'B': ['B0', 'B1', 'B2']})

result = pd.concat([df1, df2], axis=1)
print(result)
```

输出：
```
    A   B
0  A0  B0
1  A1  B1
2  A2  B2
```

#### 3. **指定连接方式**
使用 `join='inner'` 或 `join='outer'` 指定连接方式。在 `axis=1` 时，常用 `join='inner'` 对齐两个 `DataFrame` 的共同索引。

##### 示例3：外连接和内连接
```python
df1 = pd.DataFrame({'A': ['A0', 'A1', 'A2']}, index=[0, 1, 2])
df2 = pd.DataFrame({'B': ['B0', 'B1', 'B2']}, index=[1, 2, 3])

# 外连接
result_outer = pd.concat([df1, df2], axis=1, join='outer')
print(result_outer)

# 内连接
result_inner = pd.concat([df1, df2], axis=1, join='inner')
print(result_inner)
```

输出：
```
# 外连接（outer join）
      A    B
0    A0  NaN
1    A1   B0
2    A2   B1
3   NaN   B2

# 内连接（inner join）
      A    B
1    A1   B0
2    A2   B1
```

#### 4. **忽略原始索引**
使用 `ignore_index=True` 重新生成连续的索引。

##### 示例4：忽略索引
```python
result = pd.concat([df1, df2], ignore_index=True)
print(result)
```

输出：
```
    A   B
0  A0  B0
1  A1  B1
2  A2  B2
3  A3  B3
4  A4  B4
5  A5  B5
```

#### 5. **添加键作为多级索引**
使用 `keys` 参数为不同 `DataFrame` 添加键。

##### 示例5：添加多级索引
```python
result = pd.concat([df1, df2], keys=['df1', 'df2'])
print(result)
```

输出：
```
        A   B
df1 0  A0  B0
    1  A1  B1
    2  A2  B2
df2 0  A3  B3
    1  A4  B4
    2  A5  B5
```

#### 6. **使用验证连接完整性**
通过 `verify_integrity=True` 检查合并的 `DataFrame` 是否有重复索引值，若有则抛出异常。

##### 示例6：验证索引完整性
```python
# 若 df1 和 df2 索引有重复会抛出异常
result = pd.concat([df1, df2], verify_integrity=True)
```

### 总结
- `pandas.concat()` 常用于沿轴连接多个数据表。
- 常见的应用场景包括批量数据合并、按行或按列堆叠数据、重新设置索引、增加多级索引等。



------------







# Dataframe.groupby()

---

> https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.groupby.html#pandas.DataFrame.groupby

> **需要指定哪些数值列参与聚合运算。**
>
> **FutureWarning: The default value of numeric_only in DataFrameGroupBy.sum is deprecated.** 
>
> **In a future version, numeric_only will default to False. Either specify numeric_only or select only columns which should be valid for the function.**
>
>   **df.groupby(by="A").sum()**

`pandas.DataFrame.groupby()` 是一个强大的方法，用于将 `DataFrame` 按照某一列（或多列）分组，

**并对每个组应用聚合、转换或过滤操作。它常用于数据分析和汇总。**

## `groupby()` 语法

```python
DataFrame.groupby(by=None, axis=0, level=None, as_index=True, sort=True, group_keys=True, squeeze=False, observed=False, dropna=True)


# df.groupby pandas 2.2 版本官方API
DataFrame.groupby(by=None, axis=<no_default>, level=None, as_index=True, sort=True, group_keys=True, observed=<no_default>, dropna=True)
```

## 参数说明

- **as_index: bool, default True**:    **Return object with group labels as the index. Only relevant for DataFrame input. as_index=False is effectively “SQL-style” grouped output. This argument has no effect on filtrations (see the [filtrations in the user guide](https://pandas.pydata.org/docs/dev/user_guide/groupby.html#filtration)), such as `head()`, `tail()`, `nth()` and in transformations (see the [transformations in the user guide](https://pandas.pydata.org/docs/dev/user_guide/groupby.html#transformation)).**

- **`by`**：用于分组的列名、列的列表或函数。
- **`axis`**：指定分组的轴，默认为 `0`，表示按行分组。
- **`level`**：如果使用 MultiIndex，可以指定要分组的级别。
- **`as_index`**：**布尔值，默认为 `True`，表示返回的结果是否使用分组的列作为索引。 默认情况下，参与分组的列字段默认成为新DataFrame的索引。**
- **`sort`**：布尔值，是否对分组后的结果进行排序，默认为 `True`。
- **`group_keys`**：布尔值，默认为 `True`，**表示是否将分组的键加入结果的索引。**
- **`squeeze`**：布尔值，默认为 `False`，表示是否压缩结果的维度。
- **`observed`**：仅在分组时使用分类数据，默认为 `False`，返回所有类别。

### 示例及应用场景

#### 1. **基本分组**

按某一列分组，并计算每组的均值。

```python
import pandas as pd

data = {
    'category': ['A', 'B', 'A', 'B', 'A', 'B'],
    'values': [10, 20, 30, 40, 50, 60]
}

df = pd.DataFrame(data)

# 按照 'category' 列分组并计算均值
result = df.groupby('category').mean()
print(result)
```

输出：
```
           values
category        
A            30.0
B            40.0
```

#### 2. **多列分组**

可以同时按多列分组。

```python
data = {
    'category': ['A', 'B', 'A', 'B', 'A', 'B'],
    'sub_category': ['X', 'X', 'Y', 'Y', 'X', 'Y'],
    'values': [10, 20, 30, 40, 50, 60]
}

df = pd.DataFrame(data)

# 按照 'category' 和 'sub_category' 列分组并计算均值
result = df.groupby(['category', 'sub_category']).mean()
print(result)
```

输出：
```
                  values
category sub_category       
A        X          30.0
         Y          30.0
B        X          20.0
         Y          50.0
```

#### 3. **聚合操作**

可以使用 `agg()` 方法对每个组应用不同的聚合函数。

```python
# 按照 'category' 列分组，并计算总和和均值
result = df.groupby('category').agg({'values': ['sum', 'mean']})
print(result)
```

输出：
```
           values         
              sum  mean
category               
A              90  30.0
B             120  40.0
```

#### 4. **过滤操作**

可以使用 `filter()` 方法对分组后的数据进行过滤。

```python
# 仅保留值总和大于 100 的组
result = df.groupby('category').filter(lambda x: x['values'].sum() > 100)
print(result)
```

输出：
```
  category  values
1        B      20
3        B      40
5        B      60
```

#### 5. **变换操作**

使用 `transform()` 方法对每个组应用变换，返回与原始 DataFrame 相同形状的数据。

```python
# 计算每个组的均值，并将结果添加为新列
df['mean_value'] = df.groupby('category')['values'].transform('mean')
print(df)
```

输出：
```
  category  values  mean_value
0        A      10         30.0
1        B      20         40.0
2        A      30         30.0
3        B      40         40.0
4        A      50         30.0
5        B      60         40.0
```

### 应用场景

- **数据分析**：用于统计分析和数据汇总。
- **特征工程**：计算特征的聚合统计，用于机器学习模型。
- **数据清洗**：通过分组来查找和处理异常值或缺失数据。
- **报告生成**：生成分组汇总的报表。

### 总结

`pandas.DataFrame.groupby()` 是数据分析中一个非常重要的功能，提供了强大的分组、聚合和变换能力。通过合理使用，可以高效地处理和分析大规模数据。



----

# pandas.Series.unstack

`pandas.Series.unstack()` 方法用于将 `Series` 对象的某个层级的索引转换为列，从而将其“反转”成一个更宽的格式。这个方法通常在 `MultiIndex`（多重索引）系列中使用，将一个层级的索引变为列，使得数据更易于阅读和分析。

### 语法

```python
Series.unstack(level=-1, fill_value=None)
```

### 参数说明

- **`level`**：要转换为列的索引层级，默认为最后一个层级（`-1`）。可以是层级的名称或层级的数字索引。
- **`fill_value`**：用于填充缺失值的值，默认为 `None`。

### 示例及应用场景

#### 1. **基本用法**

创建一个带有多重索引的 `Series` 并使用 `unstack()` 方法。

```python
import pandas as pd

# 创建一个 MultiIndex Series
index = pd.MultiIndex.from_tuples([
    ('A', 'one'),
    ('A', 'two'),
    ('B', 'one'),
    ('B', 'two'),
])
data = pd.Series([1, 2, 3, 4], index=index)

print("Original Series:")
print(data)

# 使用 unstack() 方法
result = data.unstack()
print("\nUnstacked Series:")
print(result)
```

输出：
```
Original Series:
A  one    1
   two    2
B  one    3
   two    4
dtype: int64

Unstacked Series:
   one  two
A    1    2
B    3    4
```

#### 2. **指定层级**

如果有多个层级，可以指定要转换的层级。

```python
# 创建一个更复杂的 MultiIndex Series
index = pd.MultiIndex.from_tuples([
    ('A', 'one', 'x'),
    ('A', 'two', 'x'),
    ('A', 'one', 'y'),
    ('A', 'two', 'y'),
    ('B', 'one', 'x'),
    ('B', 'two', 'x'),
])
data = pd.Series([1, 2, 3, 4, 5, 6], index=index)

print("Original Series with multiple levels:")
print(data)

# 将第二个层级 ('one' 和 'two') 转为列
result = data.unstack(level=1)
print("\nUnstacked Series (level=1):")
print(result)
```

输出：
```
Original Series with multiple levels:
A  one  x    1
      y    3
   two  x    2
      y    4
B  one  x    5
   two  x    6
dtype: int64

Unstacked Series (level=1):
   x  y
A  1  3
   2  4
B  5  NaN
   6  NaN
```

#### 3. **填充缺失值**

在将数据转换为更宽的格式时，可以使用 `fill_value` 参数来填充缺失的值。

```python
# 使用 fill_value 填充缺失值
result = data.unstack(fill_value=0)
print("\nUnstacked Series with fill_value=0:")
print(result)
```

输出：
```
Unstacked Series with fill_value=0:
   x  y
A  1  3
   2  4
B  5  0
   6  0
```

### 应用场景

- **数据重塑**：将长格式数据转换为宽格式，以便更易于理解和分析。
- **数据可视化**：在绘图之前重新组织数据结构。
- **数据准备**：为机器学习模型准备输入数据，常常需要将数据以特定格式排列。

### 总结

`pandas.Series.unstack()` 是一个非常有用的方法，可以将多重索引的 `Series` 转换为一个更宽的 `DataFrame`，便于数据的进一步处理和分析。通过合理使用该方法，可以高效地管理和组织复杂的数据结构。





----------------

# pandas.Series.map



`pandas.Series.map()` 方法用于对 `Series` 中的每个元素应用一个函数或映射关系，并返回一个新的 `Series`。

这个方法非常灵活，常用于数据转换、数据清洗和数据分析。

### 语法

```python
Series.map(arg, na_action=None)
```

### 参数说明

- **`arg`**：可以是一个函数、字典或 Series。根据 `arg` 的类型，`map()` 会对每个元素进行相应的转换。
- **`na_action`**：可选参数，如果设置为 `'ignore'`，则在处理缺失值 (`NaN`) 时将其保留，不应用映射函数。

### 示例及应用场景

#### 1. **使用函数进行映射**

可以定义一个函数并将其应用于 `Series` 中的每个元素。

```python
import pandas as pd

# 创建一个 Series
data = pd.Series([1, 2, 3, 4, 5])

# 定义一个函数，将每个元素平方
def square(x):
    return x ** 2

# 使用 map() 方法
result = data.map(square)
print("Mapped Series (squared values):")
print(result)
```

输出：
```
Mapped Series (squared values):
0     1
1     4
2     9
3    16
4    25
dtype: int64
```

#### 2. **使用字典进行映射**

可以使用字典来定义旧值与新值之间的映射关系。

```python
# 创建一个 Series
data = pd.Series(['cat', 'dog', 'rabbit', 'dog'])

# 定义映射关系
mapping = {'cat': 'kitten', 'dog': 'puppy'}

# 使用 map() 方法进行映射
result = data.map(mapping)
print("Mapped Series (using dictionary):")
print(result)
```

输出：
```
Mapped Series (using dictionary):
0    kitten
1     puppy
2       NaN
3     puppy
dtype: object
```

#### 3. **处理缺失值**

使用 `na_action='ignore'` 来保留缺失值。

```python
data = pd.Series([1, 2, None, 4, 5])

# 使用 map() 方法，忽略缺失值
result = data.map(square, na_action='ignore')
print("Mapped Series with ignored NaN:")
print(result)
```

输出：
```
Mapped Series with ignored NaN:
0     1.0
1     4.0
2     NaN
3    16.0
4    25.0
dtype: float64
```

#### 4. **使用 lambda 函数进行映射**

可以直接使用 `lambda` 函数进行简单的操作。

```python
# 创建一个 Series
data = pd.Series([1, 2, 3, 4, 5])

# 使用 lambda 函数将每个元素加倍
result = data.map(lambda x: x * 2)
print("Mapped Series (doubled values):")
print(result)
```

输出：
```
Mapped Series (doubled values):
0    2
1    4
2    6
3    8
4    10
dtype: int64
```

### 应用场景

- **数据清洗**：将分类变量映射到数值或其他形式，以便进行后续分析。
- **数据转换**：快速修改数据格式或值，例如将字符串转换为日期。
- **特征工程**：在机器学习前对特征进行转换或处理。

### 总结

`pandas.Series.map()` 是一个非常强大的方法，可以灵活地应用于数据的转换和清洗。

通过使用函数、字典或其他 `Series`，用户可以轻松地处理和转换数据，为进一步的分析和处理做好准备。





# pandas.DataFrame.applymap

---

`pandas.DataFrame.applymap()` 方法用于对 `DataFrame` 中的每个元素应用一个函数。不同于 `apply()` 是作用于行或列，`applymap()` 是逐元素地应用，适合需要对 `DataFrame` 每个单元格进行相同操作的情况。

### 语法

```python
DataFrame.applymap(func)
```

### 参数说明

- **`func`**：一个函数，作用于 `DataFrame` 的每个元素。

### 示例及应用场景

#### 1. **基本用法**

对 `DataFrame` 中的每个元素进行操作，例如加倍。

```python
import pandas as pd

# 创建 DataFrame
df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': [4, 5, 6]
})

# 使用 applymap() 对每个元素加倍
result = df.applymap(lambda x: x * 2)
print("DataFrame after applying function with applymap():")
print(result)
```

输出：
```
DataFrame after applying function with applymap():
    A   B
0   2   8
1   4  10
2   6  12
```

#### 2. **字符串处理**

可以对每个元素执行字符串转换操作，例如将 `DataFrame` 中的每个字符串元素转换为大写。

```python
df = pd.DataFrame({
    'A': ['cat', 'dog', 'rabbit'],
    'B': ['apple', 'banana', 'cherry']
})

# 使用 applymap() 将每个字符串转换为大写
result = df.applymap(lambda x: x.upper())
print("DataFrame with uppercase strings:")
print(result)
```

输出：
```
DataFrame with uppercase strings:
        A       B
0     CAT   APPLE
1     DOG  BANANA
2  RABBIT  CHERRY
```

#### 3. **元素格式化**

可以将每个单元格格式化，例如将数值转换为带有小数的字符串格式。

```python
df = pd.DataFrame({
    'A': [1.1234, 2.5678, 3.1415],
    'B': [4.5678, 5.6789, 6.7890]
})

# 使用 applymap() 将每个元素保留两位小数
result = df.applymap(lambda x: f"{x:.2f}")
print("Formatted DataFrame with two decimal places:")
print(result)
```

输出：
```
Formatted DataFrame with two decimal places:
      A     B
0  1.12  4.57
1  2.57  5.68
2  3.14  6.79
```

### 应用场景

- **数据清洗和转换**：对每个数据单元格进行格式化或数据类型转换。
- **特征工程**：在数据预处理中，对每个单元格应用特征转换或归一化。
- **数据标准化**：例如，将数据中的特定符号替换成标准符号。

### 注意事项

`applymap()` 只适用于 `DataFrame` 对象，不适用于 `Series`。如果只对一行或一列数据进行操作，请使用 `apply()`。

### 总结

`applymap()` 在对整个 `DataFrame` 逐元素地应用相同转换时非常有用。它提供了极大的灵活性，可以方便地对每个单元格应用自定义函数。



# DataFrame.GroupBy.apply()

----

`DataFrame.groupby().apply()` 是 `pandas` 中一个强大的方法，用于在分组数据上执行自定义操作。

`apply()` 方法会将函数作用于每个分组的 `DataFrame`，并将结果组合成一个新的 `DataFrame`。这与 `agg()` 不同，`agg()` 主要用于汇总，而 `apply()` 则可用于更灵活的操作。

### 语法

```python
DataFrame.groupby(by).apply(func, *args, **kwargs)
```

### 参数说明

- **`by`**：用于分组的列或索引。
- **`func`**：自定义函数，可以对每个分组的 `DataFrame` 进行任意操作。函数可以返回标量、Series 或 DataFrame。
- **`*args, **kwargs`**：传递给 `func` 的附加参数。

### 示例及应用场景

#### 1. **计算每组的归一化值**

使用 `apply()` 对每组数据进行归一化操作。

```python
import pandas as pd

# 创建 DataFrame
df = pd.DataFrame({
    'Category': ['A', 'A', 'B', 'B', 'C', 'C'],
    'Value': [10, 15, 10, 30, 50, 60]
})

# 定义一个函数进行归一化
def normalize(group):
    group['Normalized'] = group['Value'] / group['Value'].sum()
    return group

# 使用 apply() 对每组应用归一化
result = df.groupby('Category').apply(normalize)
print("Grouped DataFrame with normalized values:")
print(result)
```

输出：
```
Grouped DataFrame with normalized values:
  Category  Value  Normalized
0        A     10    0.400000
1        A     15    0.600000
2        B     10    0.250000
3        B     30    0.750000
4        C     50    0.454545
5        C     60    0.545455
```

#### 2. **在分组数据上应用自定义操作**

可以通过 `apply()` 方法来计算每个组中最大值与最小值的差。

```python
# 定义函数计算 max-min
def max_min_diff(group):
    return group['Value'].max() - group['Value'].min()

# 使用 apply() 计算每组的 max-min 差值
result = df.groupby('Category').apply(max_min_diff)
print("\nMax-Min difference in each group:")
print(result)
```

输出：
```
Max-Min difference in each group:
Category
A     5
B    20
C    10
dtype: int64
```

#### 3. **更复杂的操作：对每个分组的前两名进行标记**

通过 `apply()` 和 `head()` 来提取每组中的前两条记录。

```python
# 创建一个更复杂的数据集
df = pd.DataFrame({
    'Category': ['A', 'A', 'A', 'B', 'B', 'C', 'C', 'C'],
    'Value': [10, 15, 7, 10, 30, 50, 60, 55]
})

# 使用 apply() 选择每组的前两名
result = df.groupby('Category').apply(lambda x: x.nlargest(2, 'Value'))
print("\nTop 2 values in each group:")
print(result)
```

输出：
```
Top 2 values in each group:
  Category  Value
1        A     15
0        A     10
4        B     30
3        B     10
6        C     60
7        C     55
```

### 应用场景

- **分组数据的复杂操作**：用于无法通过简单聚合函数（如 `sum`、`mean`）完成的复杂操作。
- **数据清洗和预处理**：可以在分组的基础上对数据进行逐组的清洗、转换等处理。
- **特征工程**：在分组数据中生成新的特征或进行数据变换。

### 注意事项

- `apply()` 返回结果的类型依赖于函数的输出。如果返回的是 `Series`，则最终得到一个分组的 `Series`；如果返回的是 `DataFrame`，则最终得到一个分组的 `DataFrame`。
- `apply()` 的计算开销较大，处理较大的数据集时性能可能较低，因此要谨慎使用。



```python
C:\Users\Administrator\AppData\Local\Temp\ipykernel_30376\1265208953.py:12: FutureWarning: Not prepending group keys to the result index of transform-like apply. In the future, the group keys will be included in the index, regardless of whether the applied function returns a like-indexed object.
To preserve the previous behavior, use

	>>> .groupby(..., group_keys=False)

To adopt the future behavior and silence this warning, use 

	>>> .groupby(..., group_keys=True)
  ratings = ratings.groupby("UserID").apply(ratings_norm)
```







# 如何理解数据的归一化？

----

数据归一化（Normalization）是一种将不同范围的数据缩放到相同尺度的技术。

它通常将数据缩放到指定范围（如0到1），在不改变数据本身特征的情况下，使特征的量级相似，从而更易于进行建模和分析。

### 为什么需要归一化？

1. **特征尺度一致性**：不同特征的量级差异可能会影响机器学习模型的表现。例如，价格特征可能以万为单位，而评分特征通常在1到5的范围。直接使用这些数据会导致模型对量级大的特征更加敏感。

2. **优化算法效率**：许多算法（如梯度下降）对特征尺度敏感，归一化后模型更快收敛。归一化使模型更易训练，减少训练时间。

3. **改善模型效果**：在距离度量敏感的模型（如K近邻和支持向量机）中，归一化可提升预测准确性，因为各特征在相同尺度上进行计算。

### 常见的归一化方法

1. **Min-Max归一化**
   - 公式：$$ X' = \frac{X - X_{\min}}{X_{\max} - X_{\min}} $$
   - 将数据缩放到$$[0, 1]$$之间，或指定的范围。
   - **适用于数据分布不明显或无极端异常值的场景。**
2. **Z-score标准化（标准化）**
   - 公式：$$ X' = \frac{X - \mu}{\sigma} $$
   - 将数据转换为均值为$$0$$，标准差为$$1$$的标准正态分布。
   - **适用于数据呈现正态分布的场景，且模型对数据的分布要求高（如线性回归）**。
3. **小数缩放法**
   - 将数据缩放至小数范围，公式为：$$ X' = \frac{X}{10^n} $$，其中n是使最大绝对值小于1的最小整数。
   - 简单，但使用较少。









### 归一化示例（Python）

以Min-Max归一化为例：

```python
import pandas as pd
from sklearn.preprocessing import MinMaxScaler

# 创建示例数据
data = pd.DataFrame({
    'price': [20000, 30000, 40000, 50000, 60000],
    'rating': [3.5, 4.0, 4.5, 5.0, 4.2]
})

# 初始化归一化器
scaler = MinMaxScaler()
data_normalized = scaler.fit_transform(data)

print("归一化后的数据：")
print(pd.DataFrame(data_normalized, columns=data.columns))
```

### 归一化的应用场景

- **机器学习预处理**：归一化可以避免特征因尺度差异影响模型训练，例如在神经网络、SVM、KNN等模型中尤为重要。
- **数据可视化**：缩放后的数据可以更容易地在同一图表上进行展示，便于直观分析和对比。
- **动态范围有限的系统**：在计算资源有限或要求有限精度的系统（如嵌入式设备）中，归一化可以使数据符合其要求。

### 总结

归一化是提升模型训练效果、改善计算效率的重要步骤。根据数据特征和模型要求选择合适的归一化方法，将能显著提升模型表现。











# pandas.DataFrame.groupby().agg()



`pandas.DataFrame.groupby().agg()` 是用于在分组数据上执行聚合操作的强大方法。通过 `groupby()` 对数据按指定列进行分组后，可以使用 `agg()` 对每个分组应用聚合函数，如求和、平均值、计数等。

### 语法

```python
DataFrame.groupby(by).agg(func)
```

### 参数说明

- **`by`**：指定分组依据的列名或索引。
- **`func`**：用于聚合的函数，可以是字符串（如 `'sum'`、`'mean'`）或者函数（如 `np.sum`），也可以是列表或字典用于指定不同列的聚合方法。

### 示例及应用场景

#### 1. 基本用法：单列分组并聚合

按 `Category` 列进行分组，并计算 `Value` 列的总和：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'Category': ['A', 'A', 'B', 'B', 'C'],
    'Value': [10, 15, 10, 30, 50]
})

# 按 Category 分组并聚合
result = df.groupby('Category').agg('sum')
print("Grouped sum of 'Value' by 'Category':")
print(result)
```

输出：

```
Grouped sum of 'Value' by 'Category':
          Value
Category       
A            25
B            40
C            50
```

#### 2. 多列分组并聚合

按 `Category` 和 `Subcategory` 两列进行分组，并对 `Value` 求和：

```python
df = pd.DataFrame({
    'Category': ['A', 'A', 'B', 'B', 'C'],
    'Subcategory': ['X', 'Y', 'X', 'Y', 'X'],
    'Value': [10, 15, 10, 30, 50]
})

result = df.groupby(['Category', 'Subcategory']).agg('sum')
print("\nGrouped sum by 'Category' and 'Subcategory':")
print(result)
```

输出：

```
Grouped sum by 'Category' and 'Subcategory':
                   Value
Category Subcategory       
A        X             10
         Y             15
B        X             10
         Y             30
C        X             50
```

#### 3. 指定不同列的聚合方法

使用字典为不同列指定不同的聚合操作，例如对 `Value` 列求和，对 `Count` 列求平均：

```python
df = pd.DataFrame({
    'Category': ['A', 'A', 'B', 'B', 'C'],
    'Value': [10, 15, 10, 30, 50],
    'Count': [1, 2, 3, 4, 5]
})

# 按 Category 分组，对 Value 求和，对 Count 求平均
result = df.groupby('Category').agg({'Value': 'sum', 'Count': 'mean'})
print("\nAggregated sum of 'Value' and mean of 'Count' by 'Category':")
print(result)
```

输出：

```
Aggregated sum of 'Value' and mean of 'Count' by 'Category':
          Value  Count
Category              
A            25    1.5
B            40    3.5
C            50    5.0
```

#### 4. 使用自定义聚合函数

可以使用自定义函数对分组数据执行复杂操作。例如，计算 `Value` 列的最大值与最小值之差：

```python
# 定义自定义聚合函数
def max_min_diff(series):
    return series.max() - series.min()

# 对每个组应用自定义聚合函数
result = df.groupby('Category').agg({'Value': max_min_diff})
print("\nCustom aggregation (max-min) on 'Value' by 'Category':")
print(result)
```

输出：

```
Custom aggregation (max-min) on 'Value' by 'Category':
          Value
Category       
A             5
B            20
C             0
```

### 应用场景

- **数据统计与分析**：例如，计算各类别的总和、均值、最大最小值、标准差等，用于描述性统计分析。
- **数据分组与汇总**：在销售数据、财务数据等领域，通常需要按地区、时间、产品等分组汇总，`groupby().agg()` 可以非常方便地实现。
- **特征工程**：通过分组聚合生成新的特征，在机器学习中用于增强数据集的表达能力。

### 总结

`DataFrame.groupby().agg()` 提供了强大的分组聚合能力，支持单一或多个聚合操作、自定义函数以及分组条件的灵活控制，是数据分析和处理中的重要工具。









# `pandas.DataFrame.unstack()` 

**`pandas.DataFrame.unstack()` 是用于将 `DataFrame` 的多级索引中的某一层次“旋转”或“展开”到列中，将行索引中的一部分转化为列索引。**

**这在数据透视、重新整形时非常有用。**

### 语法

```python
DataFrame.unstack(level=-1, fill_value=None)
```

### 参数说明

- **`level`：要展开的索引层级。可以是整数（索引级别）或级别名称。默认值 `-1` 表示最内层索引。**
- **`fill_value`：当展开操作产生缺失值时，用于填充缺失值的填充值。默认是 `None`。**

### 返回值  

返回一个 `DataFrame` 或 `Series`，索引减少一层，而列索引增加一层。

### 示例

#### 示例 1：基础用法

假设我们有一个多级索引的 `DataFrame`，使用 `unstack()` 方法将最内层索引转为列：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'Category': ['A', 'A', 'B', 'B'],
    'Subcategory': ['X', 'Y', 'X', 'Y'],
    'Value': [10, 15, 20, 25]
})
df = df.set_index(['Category', 'Subcategory'])

# 使用 unstack 展开最内层索引（Subcategory）
unstacked_df = df.unstack()
print("Unstacked DataFrame:")
print(unstacked_df)
```

输出：

```
Unstacked DataFrame:
           Value      
Subcategory     X     Y
Category                
A              10  15.0
B              20  25.0
```

`Subcategory` 索引被展开为列标签，原来的 `Category` 成为行索引。

#### 示例 2：指定索引层级

假设我们有多层索引数据，可以选择展开更外层的索引：

```python
# 添加一层索引
df = pd.DataFrame({
    'Region': ['North', 'North', 'South', 'South'],
    'Category': ['A', 'A', 'B', 'B'],
    'Subcategory': ['X', 'Y', 'X', 'Y'],
    'Value': [10, 15, 20, 25]
})
df = df.set_index(['Region', 'Category', 'Subcategory'])

# 展开第一个索引（Region）
unstacked_df = df.unstack(level=0)
print("\nUnstacked DataFrame by 'Region':")
print(unstacked_df)
```

输出：

```
Unstacked DataFrame by 'Region':
                  Value       
Region            North South
Category Subcategory             
A        X           10   NaN
         Y           15   NaN
B        X          NaN    20
         Y          NaN    25
```

这里将 `Region` 索引层展开到列位置，`Category` 和 `Subcategory` 仍保留在行索引中。

#### 示例 3：处理缺失值

在展开操作产生的 `NaN` 位置填入指定值：

```python
unstacked_df = df.unstack(fill_value=0)
print("\nUnstacked DataFrame with fill_value=0:")
print(unstacked_df)
```

输出：

```
Unstacked DataFrame with fill_value=0:
                  Value       
Region            North South
Category Subcategory             
A        X           10     0
         Y           15     0
B        X            0    20
         Y            0    25
```

### 应用场景

- **数据透视表**：将某些层级的索引展开为列，构建数据透视表。
- **数据重整与分析**：在多级索引数据中，通过展开特定索引简化数据的显示，使其更具可读性。
- **时间序列数据**：在时间序列数据上，利用 `unstack` 可以将日期等索引展开为列，更易于观察数据随时间的变化。

### 总结

`unstack()` 是处理多层索引 `DataFrame` 的关键方法，适用于将层级索引的一部分转化为列索引。











# `pandas.DataFrame.pivot()`



`pandas.DataFrame.pivot()` 是一个用于重新整形 `DataFrame` 的方法，它可以将长表格格式的数据转换成宽表格格式。它根据指定的列创建新的行索引和列，适合用于生成数据透视表。

### 语法

```python
DataFrame.pivot(index=None, columns=None, values=None)
```

### 参数说明

- **`index`**：用于生成新行索引的列标签（或标签列表）。
- **`columns`**：用于生成新列标签的列标签。
- **`values`**：要在透视表中填充的数据列。如果没有指定，`DataFrame` 中的所有剩余列都会被用作值。

### 返回值

返回一个新的 `DataFrame`，其中 `index` 列变成行索引，`columns` 列变成列标签，`values` 列作为填充值。

### 注意事项

- `pivot` 要求每个 `index` 和 `columns` 的组合都是唯一的，否则会引发 `ValueError`。如果数据存在重复组合的情况，可以使用 `pivot_table()` 方法。
- `pivot` 是非聚合的，即它只是重排数据，而不进行任何汇总或聚合。

### 示例

#### 示例 1：基本用法

假设我们有一个 `DataFrame`，其中每行表示一个年份、城市及相应的温度，我们希望将其转换为按年份和城市展开的表格格式：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'Year': [2020, 2020, 2021, 2021],
    'City': ['New York', 'Los Angeles', 'New York', 'Los Angeles'],
    'Temperature': [25, 30, 22, 28]
})

# 使用 pivot 方法重塑 DataFrame
pivot_df = df.pivot(index='Year', columns='City', values='Temperature')
print("Pivoted DataFrame:")
print(pivot_df)
```

输出：

```
Pivoted DataFrame:
City   Los Angeles  New York
Year                        
2020            30        25
2021            28        22
```

在这里，`Year` 列成为行索引，`City` 列成为新列，而 `Temperature` 列的值填充对应的位置。

#### 示例 2：多列索引

如果我们有更多维度的数据，比如增加一个季节列，可以组合多个列来生成行索引：

```python
df = pd.DataFrame({
    'Year': [2020, 2020, 2021, 2021],
    'Season': ['Winter', 'Summer', 'Winter', 'Summer'],
    'City': ['New York', 'New York', 'Los Angeles', 'Los Angeles'],
    'Temperature': [0, 25, 15, 30]
})

# 使用 pivot 方法生成多列索引
pivot_df = df.pivot(index=['Year', 'Season'], columns='City', values='Temperature')
print("\nPivoted DataFrame with MultiIndex:")
print(pivot_df)
```

输出：

```
Pivoted DataFrame with MultiIndex:
                City        Los Angeles  New York
Year Season                                      
2020 Winter               NaN        0.0
     Summer               NaN       25.0
2021 Winter              15.0       NaN
     Summer              30.0       NaN
```

在这个例子中，`Year` 和 `Season` 组合成一个多层次索引。

### 应用场景

- **数据透视表**：`pivot` 是构建数据透视表的核心工具，可以将长格式数据转换为宽格式，使其更具可读性。
- **数据分析**：尤其在财务、销售等领域，需要按时间、地区或产品进行数据展开，便于观察趋势。
- **数据展示**：转换表格数据格式，制作报表。

### 总结

`pivot()` 是 `pandas` 用于数据透视的核心方法，适用于长格式数据向宽格式的转换。当需要对数据进行聚合时，可使用 `pivot_table()`。





# `pandas.DataFrame.stack()`

`pandas.DataFrame.stack()` 是一种用于将 `DataFrame` 的列索引“旋转”或“压缩”到行索引中的方法。它的作用与 `unstack()` 相反，即将宽格式的表格数据转换为长格式，使数据更便于分析和处理。

### 语法

```python
DataFrame.stack(level=-1, dropna=True)
```

### 参数说明

- **`level`**：要压缩的列索引的级别。可以是整数（表示级别位置）或级别名称。默认值 `-1` 表示最内层索引。
- **`dropna`**：默认 `True`，表示移除生成的层次索引中的缺失值。如果设置为 `False`，保留 `NaN` 值。

### 返回值

返回一个 `Series` 或 `DataFrame`，其中列索引的一部分转化为行索引，数据以长格式显示。

### 示例

#### 示例 1：基础用法

假设我们有一个多级列的 `DataFrame`，使用 `stack()` 将列转换为行索引：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    ('A', 'Math'): [90, 85, 88],
    ('A', 'Science'): [92, 81, 79],
    ('B', 'Math'): [70, 65, 78],
    ('B', 'Science'): [75, 69, 72]
}, index=['Alice', 'Bob', 'Charlie'])

print("Original DataFrame:")
print(df)

# 使用 stack 方法压缩列索引
stacked_df = df.stack()
print("\nStacked DataFrame:")
print(stacked_df)
```

输出：

```
Original DataFrame:
           A              B        
        Math Science  Math Science
Alice     90      92    70      75
Bob       85      81    65      69
Charlie   88      79    78      72

Stacked DataFrame:
             Math  Science
Alice   A     90       92
        B     70       75
Bob     A     85       81
        B     65       69
Charlie A     88       79
        B     78       72
```

在这里，`A` 和 `B` 列被压缩为行索引的一部分，使数据从宽格式转换为长格式。

#### 示例 2：指定层级

假设有更深层次的多级索引时，可以指定具体层次来进行压缩：

```python
df = pd.DataFrame({
    ('Category', 'A', 'Math'): [90, 85, 88],
    ('Category', 'A', 'Science'): [92, 81, 79],
    ('Category', 'B', 'Math'): [70, 65, 78],
    ('Category', 'B', 'Science'): [75, 69, 72]
}, index=['Alice', 'Bob', 'Charlie'])

print("Original DataFrame with multiple levels:")
print(df)

# 指定 level 进行压缩
stacked_df = df.stack(level=1)
print("\nStacked DataFrame with specified level:")
print(stacked_df)
```

#### 示例 3：保留空值

通过将 `dropna=False`，保留空值的行：

```python
stacked_df = df.stack(dropna=False)
print("\nStacked DataFrame with NaN preserved:")
print(stacked_df)
```

### 应用场景

- **数据清洗**：在数据清理和整形中，将宽表格转为长表格便于处理，尤其是处理列数较多的数据。
- **长宽表转换**：在数据透视前需要先进行堆叠转换，使数据更具灵活性。
- **多维数据分析**：在科学数据、财务数据等多维度分析中，将宽表转换为长表可以便于观察各维度的交叉表现。

### 总结

`stack()` 是一种有效的长宽表转换方法，与 `unstack()` 相反，用于将指定列压缩至行索引，使数据更易于分析和计算。







# `pandas.to_datetime()`



- **`pandas.to_datetime()` 是一个用于将各种格式的日期和时间数据转换为 `pandas datetime64` 类型的函数。**
- **这在数据分析和处理时间序列数据时非常有用，可以确保日期时间数据的一致性和可操作性。**

### 语法

```python
pandas.to_datetime(arg, errors='raise', format=None, unit=None, infer_datetime_format=False, origin='unix', cache=True)
```

### 参数说明

- **`arg`：要转换为日期时间格式的参数，通常是字符串、时间戳、列表、数组或 `Series`。**
- **`errors`：控制遇到解析错误时的行为。选项包括：**
  - **`'raise'`（默认）：引发错误。**
  - **`'coerce'`：将无法解析的值设置为 `NaT`（缺失时间）。**
  - **`'ignore'`：返回原始输入。**
- **`format`：用于解析日期时间字符串的格式（例如：`'%Y-%m-%d %H:%M:%S'`）。**
- **`unit`：如果 `arg` 是整数，可以指定时间单位（如 `'s'`、`'ms'`、`'us'`、`'ns'`）。**
- **`infer_datetime_format`：如果设置为 `True`，会尝试自动推断日期格式，加快解析速度。**
- **`origin`：指定时间戳的起始点，默认是 `'unix'`，即从1970年1月1日开始。**
- **`cache`：如果设置为 `True`，会缓存转换结果以提高性能。**

### 返回值

返回一个 `DatetimeIndex`、`Series` 或 `DataFrame`（取决于输入），所有的日期时间数据均被转换为 `datetime64` 类型。

### 示例

#### 示例 1：字符串转换

将字符串转换为 `datetime` 对象：

```python
import pandas as pd

# 单个字符串转换
date_str = '2024-10-31 10:30:00'
date_obj = pd.to_datetime(date_str)
print("Converted datetime object:", date_obj)
```

输出：

```
Converted datetime object: 2024-10-31 10:30:00
```

#### 示例 2：列表转换

将列表中的字符串转换为 `DatetimeIndex`：

```python
date_list = ['2024-10-31', '2024-11-01', '2024-11-02']
date_index = pd.to_datetime(date_list)
print("\nConverted DatetimeIndex:")
print(date_index)
```

输出：

```
Converted DatetimeIndex:
DatetimeIndex(['2024-10-31', '2024-11-01', '2024-11-02'], dtype='datetime64[ns]', freq=None)
```

#### 示例 3：处理错误

使用 `errors='coerce'` 参数处理无法解析的日期：

```python
invalid_dates = ['2024-10-31', 'invalid_date', '2024-11-02']
converted_dates = pd.to_datetime(invalid_dates, errors='coerce')
print("\nConverted dates with error handling:")
print(converted_dates)
```

输出：

```
Converted dates with error handling:
DatetimeIndex(['2024-10-31', 'NaT', '2024-11-02'], dtype='datetime64[ns]', freq=None)
```

#### 示例 4：指定格式

如果知道日期字符串的格式，可以提高解析速度：

```python
date_strs = ['31-10-2024', '01-11-2024']
converted_dates = pd.to_datetime(date_strs, format='%d-%m-%Y')
print("\nConverted dates with specified format:")
print(converted_dates)
```

输出：

```
Converted dates with specified format:
DatetimeIndex(['2024-10-31', '2024-11-01'], dtype='datetime64[ns]', freq=None)
```

### 应用场景

- **数据清洗**：将混合格式的日期字符串标准化为统一的日期时间格式。
- **时间序列分析**：在时间序列数据处理中，将日期字符串或时间戳转换为可处理的 `datetime` 类型，以便进行索引和分析。
- **特征工程**：在机器学习中，处理日期时间特征时，将原始字符串特征转换为 `datetime` 格式。

### 总结

`pandas.to_datetime()` 是处理和转换日期时间数据的核心工具，支持多种输入格式和灵活的错误处理，能够显著提高数据分析的效率和准确性。











# NumPy.size

`numpy.size` 是一个用于返回数组的元素数量的函数。它可以用于获取多维数组中所有元素的总数，也可以用于特定轴的元素数量。

### 语法

```python
numpy.size(a, axis=None)
```

### 参数说明

- **`a`**：输入的数组。
- **`axis`**：可选参数，用于指定轴。如果为 `None`（默认值），则返回数组的总元素数量。如果指定为某个轴，则返回该轴上的元素数量。

### 返回值

返回一个整数，表示数组中元素的数量。

### 示例

#### 示例 1：计算总元素数量

```python
import numpy as np

# 创建一个示例数组
arr = np.array(
    [[1, 2, 3], 
     [4, 5, 6]])

# 获取数组的总元素数量
total_size = np.size(arr)
print("Total number of elements in the array:", total_size)
```

输出：

```
Total number of elements in the array: 6
```

#### 示例 2：计算特定轴上的元素数量

```python
# 计算每一列的元素数量
size_axis0 = np.size(arr, axis=0)
print("Number of elements along axis 0 (rows):", size_axis0)

# 计算每一行的元素数量
size_axis1 = np.size(arr, axis=1)
print("Number of elements along axis 1 (columns):", size_axis1)
```

输出：

```
Number of elements along axis 0 (rows): 2
Number of elements along axis 1 (columns): 3
```

#### 示例 3：在多维数组中使用

```python
# 创建一个三维数组
arr_3d = np.array([[[1, 2], [3, 4]], [[5, 6], [7, 8]]])

# 获取三维数组的总元素数量
total_size_3d = np.size(arr_3d)
print("Total number of elements in the 3D array:", total_size_3d)

# 获取第一维（层数）的元素数量
size_axis0_3d = np.size(arr_3d, axis=0)
print("Number of elements along axis 0 (layers):", size_axis0_3d)
```

输出：

```
Total number of elements in the 3D array: 8
Number of elements along axis 0 (layers): 2
```

### 应用场景

- **数据分析**：在处理数据时，获取数组的大小可以帮助了解数据的维度和结构。
- **调试**：在调试时，可以用来检查数组的形状和元素数量，确保数据处理正确。
- **算法设计**：在实现一些算法时，获取数组的元素数量有助于设置循环或条件。

### 总结

`numpy.size` 是一个简单而实用的函数，适用于获取数组的元素数量，能够处理多维数组和特定轴的查询。









# `pandas.DataFrame.reset_index()`



`pandas.DataFrame.reset_index()` 是一个用于重置 `DataFrame` 索引的方法。该方法通常用于将当前索引转换为列，并为 `DataFrame` 分配新的默认整数索引。这在数据清洗和整理过程中非常有用，尤其是在进行分组或透视操作后。

### 语法

```python
DataFrame.reset_index(level=None, drop=False, inplace=False, col_level=0, col_fill='', **kwargs)
```

### 参数说明

- **`level`**：可选，指定要重置的索引级别。如果 `DataFrame` 是多层索引，可以通过提供级别名称或位置来选择。
- **`drop`**：布尔值，默认值为 `False`。如果为 `True`，则不会将当前索引列转换为列，而是直接丢弃索引。
- **`inplace`**：布尔值，默认值为 `False`。如果为 `True`，则在原地修改 `DataFrame`，不返回新的 `DataFrame`。
- **`col_level`**：当 `DataFrame` 有多级列时，指定要放置新列的级别。
- **`col_fill`**：当 `DataFrame` 有多级列时，填充新列的名称。

### 返回值

返回一个新的 `DataFrame`（如果 `inplace=False`），其中索引被重置为默认的整数索引，原索引列可以被保留或丢弃。

### 示例

#### 示例 1：基本用法

假设我们有一个以某列作为索引的 `DataFrame`，我们希望重置索引：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': ['a', 'b', 'c']
})
df.set_index('A', inplace=True)

print("Original DataFrame with A as index:")
print(df)

# 使用 reset_index 重置索引
reset_df = df.reset_index()
print("\nDataFrame after reset_index:")
print(reset_df)
```

输出：

```
Original DataFrame with A as index:
   B
A   
1  a
2  b
3  c

DataFrame after reset_index:
   A  B
0  1  a
1  2  b
2  3  c
```

在这个例子中，原索引列 `A` 被转换为 `DataFrame` 的一列，并且创建了一个新的整数索引。

#### 示例 2：丢弃索引

如果我们只想丢弃当前索引而不保留它，可以使用 `drop=True`：

```python
reset_df_drop = df.reset_index(drop=True)
print("\nDataFrame after reset_index with drop=True:")
print(reset_df_drop)
```

输出：

```
DataFrame after reset_index with drop=True:
   B
0  a
1  b
2  c
```

在这里，原索引被完全丢弃，只保留了列 `B`。

#### 示例 3：在原地修改

如果想要在原地修改 `DataFrame`，可以设置 `inplace=True`：

```python
df.reset_index(inplace=True)
print("\nDataFrame after in-place reset_index:")
print(df)
```

输出：

```
DataFrame after in-place reset_index:
   A  B
0  1  a
1  2  b
2  3  c
```

### 应用场景

- **数据清理**：在数据分析过程中，经常需要将索引重置为默认整数索引，以便于后续的操作。
- **合并操作后**：在进行合并或连接操作后，重置索引可以确保结果的整洁性。
- **转换索引类型**：在某些情况下，将当前索引转换为列可以使数据的处理更加方便。

### 总结

`pandas.DataFrame.reset_index()` 是一种灵活的工具，用于处理 `DataFrame` 的索引，使其在数据分析和清洗过程中更加整洁和易于操作。













# `pandas.DataFrame.set_index()` 

`pandas.DataFrame.set_index()` 是一个用于设置 `DataFrame` 的索引的方法。

通过此方法，可以将某一列或多列设为新的行索引，以便于更方便地进行数据检索和操作。

### 语法

```python
DataFrame.set_index(keys, drop=True, append=False, inplace=False, verify_integrity=False)
```

### 参数说明

- **`keys`**：用于设置索引的列名（或列名列表），可以是字符串或字符串的列表。
- **`drop`**：布尔值，默认值为 `True`。如果为 `True`，则在设置索引后丢弃指定的列。如果为 `False`，则保留这些列。
- **`append`**：布尔值，默认值为 `False`。如果为 `True`，则将新索引附加到现有索引中，而不是替换它。
- **`inplace`**：布尔值，默认值为 `False`。如果为 `True`，则在原地修改 `DataFrame`，不返回新的 `DataFrame`。
- **`verify_integrity`**：布尔值，默认值为 `False`。如果为 `True`，则检查新索引是否有重复值，若有则引发错误。

### 返回值

返回一个新的 `DataFrame`（如果 `inplace=False`），其索引被设置为指定的列。

### 示例

#### 示例 1：基本用法

将某一列设置为索引：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': ['a', 'b', 'c']
})

print("Original DataFrame:")
print(df)

# 使用 set_index 设置 A 列为索引
df_set_index = df.set_index('A')
print("\nDataFrame after set_index:")
print(df_set_index)
```

输出：

```
Original DataFrame:
   A  B
0  1  a
1  2  b
2  3  c

DataFrame after set_index:
   B
A   
1  a
2  b
3  c
```

在这个例子中，列 `A` 被设置为新的索引，且原索引（默认整数索引）被替换。

#### 示例 2：保留原列

如果希望在设置索引后保留原列，可以将 `drop` 参数设置为 `False`：

```python
df_set_index_keep = df.set_index('A', drop=False)
print("\nDataFrame after set_index with drop=False:")
print(df_set_index_keep)
```

输出：

```
DataFrame after set_index with drop=False:
   A  B
A      
1  1  a
2  2  b
3  3  c
```

在这里，列 `A` 作为索引的同时仍然保留在 `DataFrame` 中。

#### 示例 3：使用多列设置索引

可以通过传递列名列表来设置多列索引：

```python
df_multi_index = pd.DataFrame({
    'A': ['foo', 'foo', 'bar', 'bar'],
    'B': ['one', 'two', 'one', 'two'],
    'C': [1, 2, 3, 4]
})

print("\nOriginal Multi-Column DataFrame:")
print(df_multi_index)

# 使用多列设置索引
df_multi_index_set = df_multi_index.set_index(['A', 'B'])
print("\nDataFrame after set_index with multiple columns:")
print(df_multi_index_set)
```

输出：

```
Original Multi-Column DataFrame:
     A    B  C
0  foo  one  1
1  foo  two  2
2  bar  one  3
3  bar  two  4

DataFrame after set_index with multiple columns:
        C
A   B    
foo one  1
    two  2
bar one  3
    two  4
```

#### 示例 4：原地修改

可以通过设置 `inplace=True` 来原地修改 `DataFrame`：

```python
df.set_index('A', inplace=True)
print("\nDataFrame after in-place set_index:")
print(df)
```

输出：

```
DataFrame after in-place set_index:
   B
A   
1  a
2  b
3  c
```

### 应用场景

- **数据清理**：在数据处理时，将特定列设为索引可以提高数据的检索效率。
- **时间序列分析**：通常使用日期或时间列作为索引，以便进行时间序列分析。
- **分组和聚合**：设置索引有助于后续的数据分组和聚合操作。

### 总结

`pandas.DataFrame.set_index()` 是一个强大的工具，用于灵活地设置 `DataFrame` 的索引，有助于提高数据操作的效率和可读性。通过适当的参数配置，可以根据需要保留或丢弃原列，甚至可以设置多级索引。













# `pandas.DatetimeIndex` 



- `pandas.DatetimeIndex` 是一个特殊类型的索引，用于处理时间序列数据。
- 它专门设计用于存储时间戳，并支持时间戳的操作和计算。使用 `DatetimeIndex` 可以更高效地对时间序列数据进行切片、对齐、重采样等操作。

### 创建 DatetimeIndex

`DatetimeIndex` 可以通过多种方式创建，例如从字符串、时间戳、`datetime` 对象或 `pandas.to_datetime()` 函数等。

#### 示例 1：从字符串创建

```python
import pandas as pd

# 从字符串创建 DatetimeIndex
date_strings = ['2024-01-01', '2024-01-02', '2024-01-03']
dt_index = pd.DatetimeIndex(date_strings)

print("DatetimeIndex from strings:")
print(dt_index)
```

输出：

```
DatetimeIndex(['2024-01-01', '2024-01-02', '2024-01-03'], dtype='datetime64[ns]', freq=None)
```

#### 示例 2：使用 pd.date_range()

可以使用 `pd.date_range()` 创建一系列连续的日期时间。

```python
# 创建一个日期范围
dt_index_range = pd.date_range(start='2024-01-01', end='2024-01-05')

print("\nDatetimeIndex using date_range:")
print(dt_index_range)
```

输出：

```
DatetimeIndex(['2024-01-01', '2024-01-02', '2024-01-03', '2024-01-04',
               '2024-01-05'],
              dtype='datetime64[ns]', freq='D')
```

### 使用 DatetimeIndex

`DatetimeIndex` 支持许多强大的功能，例如切片、对齐、重采样等。

#### 示例 3：切片

可以使用日期时间切片来访问特定的时间段。

```python
# 创建带有 DatetimeIndex 的 DataFrame
df = pd.DataFrame({
    'value': [10, 20, 30, 40, 50]
}, index=dt_index_range)

print("\nDataFrame with DatetimeIndex:")
print(df)

# 切片访问特定日期
print("\nSlicing DataFrame:")
print(df['2024-01-02':'2024-01-04'])
```

输出：

```
DataFrame with DatetimeIndex:
            value
2024-01-01     10
2024-01-02     20
2024-01-03     30
2024-01-04     40
2024-01-05     50

Slicing DataFrame:
            value
2024-01-02     20
2024-01-03     30
2024-01-04     40
```

#### 示例 4：重采样

`DatetimeIndex` 使得重采样操作变得简单方便。

```python
# 创建一个更复杂的时间序列数据
ts = pd.Series([1, 2, 3, 4, 5], index=pd.date_range('2024-01-01', periods=5, freq='D'))

# 重采样为每两天的总和
resampled = ts.resample('2D').sum()
print("\nResampled Series (sum every 2 days):")
print(resampled)
```

输出：

```
Resampled Series (sum every 2 days):
2024-01-01    3
2024-01-03    7
2024-01-05    5
Freq: 2D, dtype: int64
```

### 常用方法和属性

- **`.day`**、**`.month`**、**`.year`**：提取日期的各个部分。
- **`.to_period()`**：将 `DatetimeIndex` 转换为 `PeriodIndex`。
- **`.normalize()`**：将时间戳的时间部分归零，只保留日期。

### 示例 5：提取日期部分

```python
# 提取年份
print("\nYears:")
print(dt_index.year)

# 提取月份
print("\nMonths:")
print(dt_index.month)

# 提取日期
print("\nDays:")
print(dt_index.day)
```

### 应用场景

- **时间序列分析**：使用 `DatetimeIndex` 可以有效管理和操作时间序列数据，支持复杂的时间序列分析和可视化。
- **数据清洗和准备**：在数据清理阶段，使用 `DatetimeIndex` 可以轻松地处理和转换时间数据。
- **金融数据分析**：广泛应用于金融数据分析，以支持股票、商品等市场的时间序列数据处理。

### 总结

`pandas.DatetimeIndex` 是处理时间序列数据的重要工具，提供了强大的功能以支持日期时间数据的操作。通过使用 `DatetimeIndex`，用户可以更方便地进行数据分析、重采样、切片和聚合等操作。





## Timestamp、DatetimeIndex支持大量的属性可以获取日期分量：

> https://pandas.pydata.org/pandas-docs/stable/user_guide/timeseries.html#time-date-components

### Time/date components

There are several time/date properties that one can access from `Timestamp` or a collection of timestamps like a `DatetimeIndex`.

| Property         | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| year             | The year of the datetime                                     |
| month            | The month of the datetime                                    |
| day              | The days of the datetime                                     |
| hour             | The hour of the datetime                                     |
| minute           | The minutes of the datetime                                  |
| second           | The seconds of the datetime                                  |
| microsecond      | The microseconds of the datetime                             |
| nanosecond       | The nanoseconds of the datetime                              |
| date             | Returns datetime.date (does not contain timezone information) |
| time             | Returns datetime.time (does not contain timezone information) |
| timetz           | Returns datetime.time as local time with timezone information |
| dayofyear        | The ordinal day of year                                      |
| day_of_year      | The ordinal day of year                                      |
| **weekofyear**   | The week ordinal of the year                                 |
| **week**         | The week ordinal of the year                                 |
| dayofweek        | The number of the day of the week with Monday=0, Sunday=6    |
| day_of_week      | The number of the day of the week with Monday=0, Sunday=6    |
| weekday          | The number of the day of the week with Monday=0, Sunday=6    |
| quarter          | Quarter of the date: Jan-Mar = 1, Apr-Jun = 2, etc.          |
| days_in_month    | The number of days in the month of the datetime              |
| is_month_start   | Logical indicating if first day of month (defined by frequency) |
| is_month_end     | Logical indicating if last day of month (defined by frequency) |
| is_quarter_start | Logical indicating if first day of quarter (defined by frequency) |
| is_quarter_end   | Logical indicating if last day of quarter (defined by frequency) |
| is_year_start    | Logical indicating if first day of year (defined by frequency) |
| is_year_end      | Logical indicating if last day of year (defined by frequency) |
| is_leap_year     | Logical indicating if the date belongs to a leap year        |

Furthermore, if you have a `Series` with datetimelike values, then you can access these properties via the `.dt` accessor, as detailed in the section on [.dt accessors](https://pandas.pydata.org/pandas-docs/stable/user_guide/basics.html#basics-dt-accessors).

> **DatetimeIndex.week was deprecated in 1.1.0 and now it's suggested to use `DatetimeIndex.isocalendar().week` instead**









# `pandas.date_range()`



**`pandas.date_range()` 是一个用于生成指定时间范围内的日期时间序列的函数。它允许用户创建一系列连续的日期，常用于时间序列数据分析和处理。**

### 语法

```python
pandas.date_range(start=None, end=None, periods=None, freq='D', tz=None, normalize=False, closed=None, name=None, inclusive='both', **kwargs)
```

### 参数说明

- **`start`**：指定序列的起始日期，可以是字符串、`datetime` 对象或 `Timestamp`。
- **`end`**：指定序列的结束日期，格式同 `start`。
- **`periods`**：要生成的日期数量。如果同时提供 `start` 和 `end`，则 `periods` 会被忽略。
- **`freq`**：指定日期的频率，默认值为 `'D'`（每天）。可以是 `'M'`（每月），`'H'`（每小时），`'T'` 或 `'min'`（每分钟）等。
- **`tz`**：时区信息，可以指定时区字符串，例如 `'UTC'`。
- **`normalize`**：布尔值，默认值为 `False`。如果为 `True`，则将时间归零，仅保留日期部分。
- **`closed`**：指定范围的封闭端点，可以是 `'right'`、`'left'`、`'both'` 或 `None`。
- **`inclusive`**：指定是否包含起始和结束日期，默认值为 `'both'`。

### 返回值

返回一个 `DatetimeIndex` 对象，包含指定范围内的日期时间序列。

### 示例

#### 示例 1：基本用法

生成一个从指定日期开始的日期范围：

```python
import pandas as pd

# 创建一个从2024年1月1日开始的日期范围
date_range = pd.date_range(start='2024-01-01', periods=5)

print("Date range from 2024-01-01:")
print(date_range)
```

输出：

```
Date range from 2024-01-01:
DatetimeIndex(['2024-01-01', '2024-01-02', '2024-01-03', '2024-01-04',
               '2024-01-05'],
              dtype='datetime64[ns]', freq='D')
```

#### 示例 2：使用结束日期

生成一个结束日期的日期范围：

```python
# 创建一个从2024年1月1日到2024年1月10日的日期范围
date_range_end = pd.date_range(start='2024-01-01', end='2024-01-10')

print("\nDate range from 2024-01-01 to 2024-01-10:")
print(date_range_end)
```

输出：

```
Date range from 2024-01-01 to 2024-01-10:
DatetimeIndex(['2024-01-01', '2024-01-02', '2024-01-03', '2024-01-04',
               '2024-01-05', '2024-01-06', '2024-01-07', '2024-01-08',
               '2024-01-09', '2024-01-10'],
              dtype='datetime64[ns]', freq='D')
```

#### 示例 3：指定频率

生成每小时的日期时间范围：

```python
# 创建一个每小时的日期范围
hourly_range = pd.date_range(start='2024-01-01', periods=5, freq='H')

print("\nHourly date range from 2024-01-01:")
print(hourly_range)
```

输出：

```
Hourly date range from 2024-01-01:
DatetimeIndex(['2024-01-01 00:00:00', '2024-01-01 01:00:00',
               '2024-01-01 02:00:00', '2024-01-01 03:00:00',
               '2024-01-01 04:00:00'],
              dtype='datetime64[ns]', freq='H')
```

#### 示例 4：使用时区

创建带有时区信息的日期范围：

```python
# 创建带有时区的日期范围
tz_range = pd.date_range(start='2024-01-01', periods=5, tz='UTC')

print("\nTimezone-aware date range:")
print(tz_range)
```

输出：

```
Timezone-aware date range:
DatetimeIndex(['2024-01-01 00:00:00+00:00', '2024-01-02 00:00:00+00:00',
               '2024-01-03 00:00:00+00:00', '2024-01-04 00:00:00+00:00',
               '2024-01-05 00:00:00+00:00'],
              dtype='datetime64[ns, UTC]', freq='D')
```

### 应用场景

- **时间序列分析**：生成用于时间序列分析的日期范围，方便创建时间索引。
- **数据生成**：在模拟或生成数据时，可以使用 `date_range` 创建时间戳。
- **可视化**：在绘图时，可以使用时间序列数据进行 X 轴标记。

### 总结

`pandas.date_range()` 是一个强大的函数，用于生成指定范围内的日期时间序列，支持多种频率和时区设置。它在时间序列分析和数据处理过程中非常有用，帮助用户更方便地管理和操作时间数据。





# pandas.DataFrame.reindex()

**`pandas.DataFrame.reindex()` 是一个用于改变 `DataFrame` 的索引或列的函数。**

**它可以根据新的索引或列来调整数据，并允许用户对缺失的数据进行处理（如填充或丢弃）。**

### 语法

```python
DataFrame.reindex(index=None, columns=None, method=None, fill_value=None, limit=None, tolerance=None, level=None, fill_value=None)
```

### 参数说明

- **`index`**：新的行索引。如果提供，将替换现有的行索引。
- **`columns`**：新的列索引。如果提供，将替换现有的列索引。
- **`method`**：填充缺失值的方法，常用的有 `'ffill'`（前向填充）和 `'bfill'`（后向填充）。
- **`fill_value`**：用于填充缺失值的标量值。
- **`limit`**：填充时的最大数量。
- **`tolerance`**：填充时允许的误差范围。
- **`level`**：用于多层索引时，指定要重新索引的层级。

### 返回值

返回一个新的 `DataFrame`，其索引和/或列被替换为指定的值。

### 示例

#### 示例 1：基本用法

创建一个简单的 `DataFrame` 并使用 `reindex()` 更改行索引：

```python
import pandas as pd

# 创建示例 DataFrame
df = pd.DataFrame({
    'A': [1, 2, 3],
    'B': [4, 5, 6]
}, index=['a', 'b', 'c'])

print("Original DataFrame:")
print(df)

# 使用 reindex 更改行索引
df_reindexed = df.reindex(['b', 'c', 'd'])
print("\nReindexed DataFrame:")
print(df_reindexed)
```

输出：

```
Original DataFrame:
   A  B
a  1  4
b  2  5
c  3  6

Reindexed DataFrame:
     A    B
b  2.0  5.0
c  3.0  6.0
d  NaN  NaN
```

在这个例子中，行索引被更改为 `['b', 'c', 'd']`，原有的行索引 `['a']` 被丢弃，并且新索引 `d` 的值为 NaN。

#### 示例 2：更改列索引

可以使用 `reindex()` 更改列索引：

```python
# 使用 reindex 更改列索引
df_reindexed_columns = df.reindex(columns=['B', 'A', 'C'])
print("\nReindexed DataFrame with new columns:")
print(df_reindexed_columns)
```

输出：

```
Reindexed DataFrame with new columns:
     B    A   C
a  4.0  1.0 NaN
b  5.0  2.0 NaN
c  6.0  3.0 NaN
```

这里，列索引被更改为 `['B', 'A', 'C']`，而 `C` 列的值为 NaN，因为原 `DataFrame` 中没有这一列。

#### 示例 3：使用填充方法

使用填充方法处理缺失值：

```python
# 创建示例 DataFrame
df2 = pd.DataFrame({
    'A': [1, 2, 3],
    'B': [4, 5, 6]
}, index=['a', 'b', 'c'])

# 重新索引并使用前向填充
df_reindexed_ffill = df2.reindex(['a', 'b', 'c', 'd'], method='ffill')
print("\nReindexed DataFrame with forward fill:")
print(df_reindexed_ffill)
```

输出：

```
Reindexed DataFrame with forward fill:
     A  B
a  1.0  4
b  2.0  5
c  3.0  6
d  3.0  6
```

在这里，`d` 行的值通过前向填充方法填充，使用了 `c` 行的值。

### 应用场景

- **数据对齐**：在合并或连接多个数据集时，可以使用 `reindex()` 进行数据对齐。
- **时间序列分析**：在处理时间序列数据时，可以通过 `reindex()` 使得索引对齐到特定的时间点。
- **数据预处理**：在数据预处理阶段，可以使用 `reindex()` 重新定义行列索引并填充缺失值。

### 总结

`pandas.DataFrame.reindex()` 是一个强大的工具，能够灵活地重新定义 `DataFrame` 的行索引和列索引，并允许用户通过多种方式处理缺失数据。它在数据清理、对齐和分析等方面具有重要的应用价值。









# `pandas.DataFrame.resample()`



`pandas.DataFrame.resample()` 是一个用于对时间序列数据进行重采样的函数。它允许用户按指定的时间频率对数据进行重新分组，以便于汇总、分析和处理时间序列数据。

### 语法

```python
DataFrame.resample(rule, axis=0, closed='right', label='right', convention='start', base=0, on=None, level=None, kind=None, origin='epoch', offset=None, fill_value=None, limit=None, closed=None)
```

### 参数说明

- **`rule`**：重采样的频率规则，例如 `'D'`（每天）、`'M'`（每月）、`'H'`（每小时）等。
- **`axis`**：指定要重采样的轴，默认是 0（行）。
- **`closed`**：指定区间的闭合端点，默认为 `'right'`。
- **`label`**：返回值标签的位置，默认为 `'right'`。
- **`convention`**：在处理不同时间间隔时，指定开始或结束。
- **`base`**：用于偏移的基准数值。
- **`on`**：指定重采样的列名。
- **`level`**：用于多层索引时，指定重采样的层级。
- **`origin`**：指定时间序列的起始时间，默认为 `'epoch'`。
- **`fill_value`**：指定填充缺失值的标量。
- **`limit`**：填充时的最大数量。

### 返回值

返回一个新的 `DataFrame`，其索引为重采样后的时间戳，数据根据指定的汇总方法进行了处理。

### 示例

#### 示例 1：基本用法

首先，创建一个简单的时间序列 `DataFrame`：

```python
import pandas as pd

# 创建一个时间序列 DataFrame
date_rng = pd.date_range(start='2024-01-01', end='2024-01-10', freq='D')
df = pd.DataFrame(date_rng, columns=['date'])
df['data'] = range(1, len(df) + 1)
df.set_index('date', inplace=True)

print("Original DataFrame:")
print(df)
```

输出：

```
Original DataFrame:
            data
date             
2024-01-01     1
2024-01-02     2
2024-01-03     3
2024-01-04     4
2024-01-05     5
2024-01-06     6
2024-01-07     7
2024-01-08     8
2024-01-09     9
2024-01-10    10
```

#### 示例 2：按周重采样

将数据按周进行重采样，计算每周的总和：

```python
# 每周重采样并计算总和
weekly_resampled = df.resample('W').sum()
print("\nWeekly Resampled DataFrame (sum):")
print(weekly_resampled)
```

输出：

```
Weekly Resampled DataFrame (sum):
            data
date             
2024-01-07    28
2024-01-14    10
```

#### 示例 3：按月重采样

按月进行重采样并计算平均值：

```python
# 每月重采样并计算平均值
monthly_resampled = df.resample('M').mean()
print("\nMonthly Resampled DataFrame (mean):")
print(monthly_resampled)
```

输出：

```
Monthly Resampled DataFrame (mean):
            data
date             
2024-01-31    5.5
```

#### 示例 4：使用其他汇总方法

可以使用 `.agg()` 方法应用多种汇总统计：

```python
# 每天重采样并计算总和和平均值
daily_resampled = df.resample('D').agg(['sum', 'mean'])
print("\nDaily Resampled DataFrame (sum and mean):")
print(daily_resampled)
```

输出：

```
Daily Resampled DataFrame (sum and mean):
            data          
             sum mean
date                   
2024-01-01     1  1.0
2024-01-02     2  2.0
2024-01-03     3  3.0
2024-01-04     4  4.0
2024-01-05     5  5.0
2024-01-06     6  6.0
2024-01-07     7  7.0
2024-01-08     8  8.0
2024-01-09     9  9.0
2024-01-10    10 10.0
```

### 应用场景

- **时间序列分析**：用于处理和分析时间序列数据，如股票价格、气象数据等。
- **数据汇总**：可以轻松进行数据的汇总与统计分析，按日、周、月等不同时间频率对数据进行汇总。
- **数据清理**：在数据预处理阶段，使用重采样来填补缺失值或修复时间序列数据。

### 总结

`pandas.DataFrame.resample()` 是一个强大的工具，专为处理时间序列数据而设计。通过它，用户可以方便地对数据进行重采样、汇总和分析，是时间序列分析中不可或缺的一部分。





## Offset aliases

A number of string aliases are given to useful common time series frequencies. We will refer to these aliases as *offset aliases*.

> https://pandas.pydata.org/pandas-docs/stable/user_guide/timeseries.html#offset-aliases

| Alias | Description                                      |
| ----- | ------------------------------------------------ |
| B     | business day frequency                           |
| C     | custom business day frequency                    |
| D     | calendar day frequency                           |
| W     | weekly frequency                                 |
| ME    | month end frequency                              |
| SME   | semi-month end frequency (15th and end of month) |
| BME   | business month end frequency                     |
| CBME  | custom business month end frequency              |
| MS    | month start frequency                            |
| SMS   | semi-month start frequency (1st and 15th)        |
| BMS   | business month start frequency                   |
| CBMS  | custom business month start frequency            |
| QE    | quarter end frequency                            |
| BQE   | business quarter end frequency                   |
| QS    | quarter start frequency                          |
| BQS   | business quarter start frequency                 |
| YE    | year end frequency                               |
| BYE   | business year end frequency                      |
| YS    | year start frequency                             |
| BYS   | business year start frequency                    |
| h     | hourly frequency                                 |
| bh    | business hour frequency                          |
| cbh   | custom business hour frequency                   |
| min   | minutely frequency                               |
| s     | secondly frequency                               |
| ms    | milliseconds                                     |
| us    | microseconds                                     |
| ns    | nanoseconds                                      |

>  *Deprecated since version 2.2.0:* Aliases `H`, `BH`, `CBH`, `T`, `S`, `L`, `U`, and `N` are deprecated in favour of the aliases `h`, `bh`, `cbh`, `min`, `s`, `ms`, `us`, and `ns`.









































