其他竞品：Polars、Dask、DuckDB、Modin、FireDucks、Datatable

```python
!uv add pandas==3.0.3 openpyxl==3.1.5
# 导入 Pandas 库
import pandas as pd
import numpy as np
```

# Pandas 核心数据结构

Pandas 是 Python 数据分析的核心库，它提供了两种核心数据结构：**Series** 和 **DataFrame**。

简单来说：
- **Series** → 一列带标签的数据（类似一个带索引的数组或 Excel 中的一列）
- **DataFrame** → 一个带标签的二维表格（类似 Excel 工作表或 SQL 表）

## Series

Series 可以理解为「带标签的一维数组」。它由两个核心部分组成：
- `values`：数据本身（类似 NumPy 数组）
- `index`：每个数据点的标签

```python
# pd.Series: 从列表创建 Series
pd.Series([10, 20, 30, 40])

```

```python
# 从列表创建，指定索引
pd.Series([10, 20, 30, 40], index=['a', 'b', 'c', 'd'])

```

```python
# 从字典创建，key 自动成为索引
pd.Series({'a': 10, 'b': 20, 'c': 30})

```

```python
# 核心属性
s = pd.Series([10, 20, 30, 40], index=['a', 'b', 'c', 'd'])
print('values:', s.values, type(s.values))
print('index:', s.index, type(s.index))
print('dtype:', s.dtype)
print('shape:', s.shape)
```

## DataFrame 创建

DataFrame 是一个二维表格结构，可以来自多种数据源。

```python
# pd.DataFrame: 从字典创建——key 是列名，value 是列数据
pd.DataFrame({
    '姓名': ['张三', '李四', '王五', '赵六'],
    '年龄': [25, 30, 35, 28],
    '城市': ['北京', '上海', '广州', '深圳'],
    '薪资': [15000, 20000, 25000, 18000]
})

```

```python
# pd.DataFrame: 从嵌套列表创建——通过 columns 指定列名
pd.DataFrame([
    ['张三', 25, '北京', 15000],
    ['李四', 30, '上海', 20000],
    ['王五', 35, '广州', 25000],
], columns=['姓名', '年龄', '城市', '薪资'])

```

```python
# pd.DataFrame: 从 NumPy 数组创建
arr = np.array([
    [25, 15000],
    [30, 20000],
    [35, 25000],
])
pd.DataFrame(arr, columns=['年龄', '薪资'])

```

### 从文件读取

实际工作中数据很少手动输入，更多是从文件加载。Pandas 支持各种格式。

```python
# 从 CSV 文件读取（实际文件路径）——展示用法，不实际执行
df = pd.read_csv('../linking.csv')
df
# pd.read_csv 常用参数
# df = pd.read_csv('data.csv', encoding='utf-8')       # 指定编码
# df = pd.read_csv('data.csv', header=None)            # 无表头
# df = pd.read_csv('data.csv', index_col='id')         # 指定索引列
# df = pd.read_csv('data.csv', usecols=['a', 'b'])     # 只读指定列
# df = pd.read_csv('data.csv', nrows=100)              # 只读前 100 行

```

```python

# pd.read_excel: 从 Excel 文件读取
df_xlx = pd.read_excel('../linking.xlsx', sheet_name='Sheet1')
df_xlx
# pd.read_json: 从 JSON 文件读取
# df = pd.read_json('data.json')
```

### 从数据库读取

Pandas 还可以直接读取 SQL 查询结果。这里以 Python 内置的 SQLite 为例。

```python
# pd.read_sql: 从数据库读取 SQL 查询结果
import sqlite3

# 创建内存数据库并写入数据
conn = sqlite3.connect(':memory:')
conn.execute('''
    CREATE TABLE employees (
        name TEXT, age INT, city TEXT, salary INT
    )
''')
conn.execute("INSERT INTO employees VALUES ('张三', 25, '北京', 15000)")
conn.execute("INSERT INTO employees VALUES ('李四', 30, '上海', 20000)")
conn.execute("INSERT INTO employees VALUES ('王五', 35, '广州', 25000)")
conn.execute("INSERT INTO employees VALUES ('赵六', 28, '深圳', 18000)")
conn.commit()

df = pd.read_sql('SELECT * FROM employees', conn)
conn.close()
df

```

#### 💡**场景**

你的团队用 MySQL/PostgreSQL 存储业务数据，直接 `pd.read_sql('SELECT ...', connection)` 就能把数据拉到 DataFrame 里做分析，不用手动导出成 CSV。

## DataFrame 的组成结构

一个 DataFrame 内部由以下核心部分组成。理解它们，你就掌握了一半。

```python
df_xlx
```

```python
# columns: 列名
print('columns:', df_xlx.columns)
print('类型:', type(df_xlx.columns))
```

```python
# index: 行索引
print('index:', df_xlx.index)
print('类型:', type(df_xlx.index))
```

```python
# values: 核心数据（二维 NumPy 数组）
print('类型:', type(df_xlx.values))
print('values:')
df_xlx.to_numpy()
```

```python
# dtypes: 每列的数据类型
print('dtypes:')
df.dtypes
```

```python
# shape: 形状 (行数, 列数)
print('shape:', df.shape)
print('行数:', df.shape[0])
print('列数:', df.shape[1])
```

```python
# size: 元素总数
print('size:', df.size)
# ndim: 维度数
print('ndim:', df.ndim)
```
