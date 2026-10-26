```python
# 导入 NumPy 库
import numpy as np
```

# NumPy 数据操作

上一节课我们理解了 ndarray 的核心概念（shape、dtype、strides、axis），本节课聚焦于**如何操作数组中的数据**。

先说明一下，这节课我们只讲基础操作。以后讲到具体应用（模型推理、数据预处理等）时，涉及到的 NumPy 操作会再拿出来细讲，到时候理解起来也更具体。

## 数组创建

除了从列表直接创建，NumPy 提供了很多便捷的构造方法。

```python
# np.zeros: 创建全零数组
print('zeros(2,3):\n', np.zeros((2,3)), '\n')
# np.ones: 创建全一数组
print('ones(2,3):\n', np.ones((2,3), dtype=np.float32), '\n')
# np.eye: 创建单位矩阵
print('eye(3):\n', np.eye(3))
```

```python
# np.arange: 创建等差数组
print('arange(0, 10, 2):', np.arange(0, 10, 2))
# np.linspace: 创建等间隔数组
print('linspace(0, 1, 5):', np.linspace(0, 1, 5))
# np.linspace: 创建等间隔数组
print('linspace(0, 1, 5, endpoint=False):', np.linspace(0, 1, 5, endpoint=False))
```

```python
# default_rng: 创建随机数生成器
rng = np.random.default_rng(41)
# uniform: 生成均匀分布随机数
print('uniform(0, 1, (2,3)):\n', rng.uniform(0, 1, (2,3)), '\n')
# normal: 生成正态分布随机数
print('normal(0, 1, (2,3)):\n', rng.normal(0, 1, (2,3)), '\n')
# integers: 生成随机整数
print('integers(0, 10, (2,3)):\n', rng.integers(0, 10, (2,3)))
```

#### 💡**场景**

快速造一批模拟数据来测试逻辑。比如模拟 10 个商品的价格和销量，测试排序或筛选功能。

```python
# default_rng: 创建随机数生成器
rng = np.random.default_rng(42)
# uniform: 生成均匀分布随机数
prices = rng.uniform(10, 100, 10).round(2)
# integers: 生成随机整数
sales = rng.integers(0, 1000, 10)
print('价格:', prices)
print('销量:', sales)
```

## 索引与切片

NumPy 的索引语法与 Python 列表类似，但支持多维操作。

### 基本索引与切片

语法：`arr[start:stop:step]`，多维就用逗号分隔。

- start：起始索引，默认 0
- stop：结束索引（不包含），默认数组长度
- step：步长，默认 1

**注意：切片返回的是视图（view），不是拷贝**——改切片会影响原数组。

```python
# np.arange: 创建等差数组
arr = np.arange(12).reshape(3, 4)
print('原数组:\n', arr, '\n')
print('arr[0]:', arr[0])
print('arr[:, 1]:\n', arr[:, 1:])
print('arr[1:, 2:]:\n', arr[1:, 2:])
```

```python
view = arr[1:, 2:]
view[0, 0] = 99
print('修改视图后原数组:\n', arr)
# copy: 创建数组的副本
copy = arr[1:, 2:].copy()
```

### 花式索引（Fancy Indexing）

用整数数组选取不连续的行/列。**返回的是拷贝。**

```python
# np.arange: 创建等差数组
arr = np.arange(12).reshape(4, 3)
print('数组:\n', arr, '\n')
print('arr[[0, 2]]:\n', arr[[0, 2]], '\n')
rows = [0, 1, 3]
cols = [2, 1, 0]
print('arr[rows, cols]:', arr[rows, cols])
```

### 布尔索引（Boolean Indexing）

用布尔数组作为掩码（mask），选取满足条件的元素。**返回的是拷贝。**

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([10, 25, 3, 47, 8, 19])
mask = arr > 15
print('mask:', mask)
print('arr[arr > 15]:', arr[mask])
print('arr[arr % 2 == 0]:', arr[arr % 2 == 0])
```

```python
arr = np.array([5, 12, 18, 25, 30, 7, 42])
print('(>10) & (<30):', arr[(arr > 10) & (arr < 30)])
print('(<10) | (>30):', arr[(arr < 10) | (arr > 30)])
```

## 变形与重塑

改变数组的形状而不改数据。总元素数必须一致，用 `-1` 让 NumPy 自动算。

```python
# np.arange: 创建等差数组
arr = np.arange(12)
print('原始:', arr, '\n')
# reshape: 重塑数组形状
print('reshape(3, 4):\n', arr.reshape(3, 4), '\n')
# reshape: 重塑数组形状
print('reshape(2, -1):\n', arr.reshape(2, -1), '\n')
# reshape: 重塑数组形状
print('reshape(2, 2, 3):\n', arr.reshape(2, 2, 3))
```

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([1, 2, 3, 4])
# np.resize: 改变数组大小（可重复填充）
print('np.resize(arr, (2, 3)):\n', np.resize(arr, (2, 3)))
# np.resize: 改变数组大小（可重复填充）
print('np.resize(arr, (2, 2)):\n', np.resize(arr, (2, 2)))
```

#### 💡**场景**

API 返回的数据是一维的，但你需要按行列组织成表格，用 reshape 搞定。

```python
# np.array: 从列表创建 NumPy 数组
data = np.array([1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12])
# reshape: 重塑数组形状
table = data.reshape(4, 3)
print('转换为 4行3列的表格:\n', table)
```

## 数学运算

NumPy 的所有运算符都是**向量化**的——对整个数组一次性操作，不用写循环。

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([1, 2, 3, 4])
print('原数组:', arr)
print('arr + 10:', arr + 10)
print('arr * 2:', arr * 2)
print('arr ** 2:', arr ** 2)
print('arr > 2:', arr > 2)
```

```python
# np.array: 从列表创建 NumPy 数组
a = np.array([10, 20, 30])
# np.array: 从列表创建 NumPy 数组
b = np.array([1, 2, 3])
print('a + b:', a + b)
print('a - b:', a - b)
print('a * b:', a * b)
print('a / b:', a / b)
```

#### 💡**场景**

对一批数据做批量转换。比如商品价格全部打八折，温度从摄氏度转华氏度。

```python
# np.array: 从列表创建 NumPy 数组
prices = np.array([99, 199, 299, 399], dtype=float)
discounted = prices * 0.8
print('原价:', prices)
print('打八折:', discounted)

# np.array: 从列表创建 NumPy 数组
celsius = np.array([0, 10, 20, 30, 40])
fahrenheit = celsius * 9 / 5 + 32
print('摄氏:', celsius)
print('华氏:', fahrenheit)
```

## 统计运算

对数组做聚合统计，可以用 `axis` 指定沿哪个方向算。

```python
# np.arange: 创建等差数组
arr = np.arange(12).reshape(3, 4).astype(float)
print('数组:\n', arr, '\n')
# sum: 计算所有元素之和
print('sum:', arr.sum())
# mean: 计算算术平均值
print('mean:', arr.mean())
# sum: 计算所有元素之和
print('sum(axis=0) 按列求和:', arr.sum(axis=0))
# sum: 计算所有元素之和
print('sum(axis=1) 按行求和:', arr.sum(axis=1))
# min: 找出最小值
print('min:', arr.min(), ', max:', arr.max())
# argmin: 找出最小值所在的索引
print('argmin:', arr.argmin(), ', argmax:', arr.argmax())
```

#### 标准差

标准差（Standard Deviation）是统计学中衡量数据波动程度（即离散程度）最常用的指标。简单来说，它告诉你数据围绕平均值有多“散”。

标准差越小：说明数据越集中，平均值越有代表性（大家成绩都差不多）。

标准差越大：说明数据越分散，平均值越不靠谱（有人考100分，有人考0分）。

$$ 
\textbf{1. 总体标准差（Population Standard Deviation）}\\[1em]
\sigma = \sqrt{\frac{\sum_{i=1}^{N} (x_i - \mu)^2}{N}}\\[2em]
\textbf{2. 样本标准差（Sample Standard Deviation）}\\[1em]
s = \sqrt{\frac{\sum_{i=1}^{n} (x_i - \bar{x})^2}{n-1}}\\[1em]
$$

```python
print("arr", arr)
print("mean", arr.mean())
# std: 计算总体标准差
print('std总体标准差:', arr.std())
# std: 计算样本标准差
print('std(ddof=1)样本标准差:', arr.std(ddof=1))
# 变异系数
print('变异系数CV:', arr.std() / arr.mean())
```

| CV 范围       | 离散程度   | 直观感受                       | 典型场景举例                             |
| :------------ | :--------- | :----------------------------- | :--------------------------------------- |
| **< 10%**     | **非常小** | 数据极度集中，几乎一模一样     | 工厂自动化生产的零件尺寸、精密仪器读数   |
| **10% ~ 20%** | **较小**   | 有波动但很稳定，平均值可靠     | 成年人的身高、体重、日常气温             |
| **20% ~ 30%** | **中等**   | 正常波动范围，尚可接受         | 学生考试成绩、员工绩效评分               |
| **30% ~ 50%** | **较大**   | 数据比较分散，平均值代表性下降 | 城市房价、不同地区的薪资水平             |
| **> 50%**     | **非常大** | 极度分散，平均值几乎没意义     | 股票日收益率、创业公司估值、个人消费支出 |

## 条件筛选与 np.where

`np.where(condition, x, y)` —— 如果条件成立取 x 的值，否则取 y 的值。类似向量化的 if-else。

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([10, -5, 3, -8, 0, 15])
# np.where: 根据条件筛选或替换值
result = np.where(arr < 0, 0, arr)
print('原数组:', arr)
print('负数变 0:', result)
```

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([10, 20, 30, 40, 50])
# np.where: 根据条件筛选或替换值
indices = np.where(arr > 25)
print('arr > 25 的位置:', indices)
print('对应值:', arr[indices])
```

#### 💡**场景**

对数据进行分类处理。比如成绩单中，60 分以上标记为「通过」，否则为「不通过」。

```python
# np.array: 从列表创建 NumPy 数组
scores = np.array([85, 42, 73, 59, 90, 31, 68])
# np.where: 根据条件筛选或替换值
result = np.where(scores >= 60, '通过', '不通过')
print('成绩:', scores)
print('结果:', result)
```

## 拼接与分割

把多个数组合并，或把一个数组拆成多份。

```python
# np.array: 从列表创建 NumPy 数组
a = np.array([[1, 2], [3, 4]])
# np.array: 从列表创建 NumPy 数组
b = np.array([[5, 6]])
# np.vstack: 垂直堆叠数组
print('vstack (垂直拼):\n', np.vstack([a, b]), '\n')
# np.concatenate: 沿指定轴拼接数组
print('concatenate axis=0:\n', np.concatenate([a, b], axis=0))
```

```python
# np.array: 从列表创建 NumPy 数组
a = np.array([[1, 2], [3, 4]])
# np.array: 从列表创建 NumPy 数组
b = np.array([[5], [6]])
# np.hstack: 水平堆叠数组
print('hstack (水平拼):\n', np.hstack([a, b]), '\n')
# np.concatenate: 沿指定轴拼接数组
print('concatenate axis=1:\n', np.concatenate([a, b], axis=1))
```

```python
# np.arange: 创建等差数组
arr = np.arange(16).reshape(4, 4)
print('数组:\n', arr, '\n')
# np.split: 沿行方向分割数组
first, second, rest = np.split(arr, [1, 3])
print('[:1]:\n', first, '\n')
print('[1:3]:\n', second, '\n')
print('[3:]:\n', rest)
```

```python
# np.arange: 创建等差数组
arr = np.arange(12).reshape(3, 4)
print('数组:\n', arr, '\n')
# np.hsplit: 沿列方向分割数组
left, right = np.hsplit(arr, 2)
print('左:\n', left, '\n')
print('右:\n', right)
```

## 排序

排序在很多场景中都会用到。

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([3, 1, 4, 1, 5, 9, 2, 6])
print('原数组:', arr)
# np.sort: 返回排序后的数组
print('np.sort(arr):', np.sort(arr))
# np.argsort: 返回排序后的索引数组
print('np.argsort(arr):', np.argsort(arr))
# np.unique: 返回唯一值
print('np.unique(arr):', np.unique(arr))
```

```python
# np.array: 从列表创建 NumPy 数组
matrix = np.array([[3, 1, 2], [6, 5, 4]])
print('矩阵:\n', matrix, '\n')
# np.sort: 返回排序后的数组
print('每行排序:\n', np.sort(matrix, axis=1))
# np.sort: 返回排序后的数组
print('每列排序:\n', np.sort(matrix, axis=0))
```

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([3, 1, 4, 1, 5])
# sort: 原地排序数组
arr.sort()
# sort: 原地排序数组
print('arr.sort() 后:', arr)
```

#### 💡**场景**

找出数据中最大/最小的前几个。比如找出销量最高的前 3 个商品。

```python
# np.array: 从列表创建 NumPy 数组
sales = np.array([230, 150, 890, 340, 560, 120, 670])
# np.argsort: 返回排序后的索引数组
top3_idx = np.argsort(sales)[::-1][:3]
print('销量:', sales)
print('前 3 名索引:', top3_idx)
print('前 3 名销量:', sales[top3_idx])
```

## 广播（Broadcasting）

广播是 NumPy 的特色：**对不同形状的数组做运算时，NumPy 会自动把小数组「扩展」成和大数组一样的形状**，无需手动复制。

规则：从尾部维度开始比，维度相同或有一个是 1 就能广播。

```python
# np.array: 从列表创建 NumPy 数组
arr = np.array([1, 2, 3])
print('arr + 10:', arr + 10)
print('arr * 5:', arr * 5)
```

```python
# np.arange: 创建等差数组
matrix = np.arange(12).reshape(3, 4)
# np.array: 从列表创建 NumPy 数组
row = np.array([10, 20, 30, 40])
# np.array: 从列表创建 NumPy 数组
col = np.array([[100], [200], [300]])

print('matrix:\n', matrix, '\n')
print('matrix + row (行广播):\n', matrix + row, '\n')
# reshape: 重塑数组形状
print('matrix + col[:, None] (列广播):\n', matrix + col)
```

```python
# np.array: 从列表创建 NumPy 数组
a = np.array([[1], [2], [3]])
# np.array: 从列表创建 NumPy 数组
b = np.array([10, 20, 30, 40])
print('(3,1) + (4,) -> (3,4):\n', a + b)
```

#### 💡**场景**

给一个二维表格的每一列加上不同的偏移量，或者每行乘以不同的系数。

```python
# np.array: 从列表创建 NumPy 数组
data = np.array([[80, 90, 85],
                 [70, 75, 80],
                 [60, 65, 70]])
# np.array: 从列表创建 NumPy 数组
bonus = np.array([5, 10, 3])
print('原始成绩:\n', data, '\n')
print('加上加分 (每人加的不同):\n', data + bonus)
```

## 总结

本节课覆盖了 NumPy 最常用的数据操作：

| 操作 | 常用方法 |
|------|----------|
| **创建** | `array`, `zeros`, `ones`, `arange`, `linspace`, `random` |
| **索引** | `arr[i]`, `arr[i:j]`, 花式索引, 布尔索引 |
| **变形** | `reshape`, `resize` |
| **数学运算** | `+`, `-`, `*`, `/`, `**`, `sin`, `cos`, `exp`, `sqrt` |
| **统计运算** | `sum`, `mean`, `std`, `min`, `max`, `cumsum` |
| **条件筛选** | `np.where`, 布尔掩码 |
| **拼接分割** | `concatenate`, `vstack`, `hstack`, `split` |
| **排序** | `sort`, `argsort`, `unique` |
| **广播** | 自动维度扩展 |


