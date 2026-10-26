# Pandas 数据清洗

数据清洗是数据分析中最耗时但也最重要的环节。
真实世界的数据往往是「脏」的——类型不统一、格式不一致、缺失值、异常值……

本节课我们以一份真实的程序员薪资调查数据为例，动手做一次完整的数据清洗。

**核心目标**：统一各列数据类型，让数据「干净、可用」。

```python
import pandas as pd
import numpy as np
import re

df = pd.read_excel('../linking.xlsx')
```

```python
# loc: 按索引标签名定位（行名、列名）
# 语法: df.loc[行, 列]
# df.loc[0:5]  # 索引0到5的所有行
# df.loc[0:5, "城市"]  # 取前5行的城市列
# df["城市"] == "北京"
# df.loc[df["城市"] == "北京"]  # 条件筛选所有列

# # iloc: 按整数位置定位
# # 语法: df.iloc[行位置, 列位置]
# df.iloc[0:5]  # 前5行
# df.iloc[0:5, 0:3]  # 前5行、前3列
# df.iloc[0, 0]  # 第一行第一列的值

```

## 1. 数据初探

拿到一份新数据，先别急着动手——先「观察」。

`df.info()` 可以快速了解：数据量、各列类型、缺失情况。

```python
df.info()
```

`df.head()` 看一下前几行，对数据内容有个直观感受。

```python
df.head()
```

再单独看看每列的类型和缺失值数量。

这里我们发现：虽然有 679 行，但很多列都有缺失值；而且**类型全是 object/str**——
数值型的字段（经验、薪资）被读成了字符串，这是我们今天要解决的核心问题。

```python
df.dtypes
```

```python
df.isna().sum()
```

## 2. 应届信息提取

很多列都混入了「26应届」「25应届」这类值——它其实是「毕业年份」信息，不是该列的正常数据。
统一做法：把「X应届」提取为独立的一列「毕业年份」，原列留空（经验列写 0）。

先看看「应届」值分布在哪些列：

```python
col_name = '学历'
temp_mask = df[col_name].astype(str).str.contains('应届', na=False)
# temp_mask
df.loc[temp_mask, col_name].head(10)
# df[temp_mask][['城市','经验','薪资','学历']].head(10)
```

用 `str.extract()` 可以从字符串中提取正则匹配的部分。先试试一列：

```python
df.loc[temp_mask, '城市'].astype(str).str.extract(r'(\d+)\s*应届')
```

拿到数字后加上 2000 就是真实的毕业年份。现在统一处理所有列：

```python
df['毕业年份'] = np.nan

for col in ['城市', '经验']:
    mask = df[col].astype(str).str.contains(r'应届', na=False)
    year = df.loc[mask, col].astype(str).str.extract(r'(\d+)\s*应届')
    if not year.empty:
        df.loc[mask, '毕业年份'] = 2000 + year[0].dropna().astype(int)
    if col == '经验':
        df.loc[mask, col] = 0.0
    else:
        df.loc[mask, col] = None
```

```python
df['毕业年份'].value_counts(dropna=False)
```

```python
df['城市'].value_counts(dropna=False)
```

```python
df['经验'].value_counts(dropna=False)
```

```python
df
```

## 3. 学历清洗

先看「学历」列——它是分类数据，但写法很不统一。

```python
df['学历'].value_counts()
```

问题很明显：
- 「本」「本科」「本211」「985本」本质都是「本科」
- 「专」「专科」「大专」本质都是「大专」
- 「硕」「硕士」「985硕士」本质都是「硕士」
- 「招人」「留美」等是异常值

我们用 `map()` 做**分类映射**，不认识的统一设为 `NaN`。

```python
edu_map = {
    '本': '本科', 
    '本科': '本科', 
    '本211': '本科', 
    '985本': '本科',
    '专': '大专', 
    '专科': '大专', 
    '大专': '大专',
    '硕': '硕士', 
    '硕士': '硕士', 
    '985硕士': '硕士',
    '高中': '高中', 
    '高': '高中', 
    '高二': '高中',
    '初': '初中', 
    '初中': '初中',
    '中专': '中专',
    '小学': '小学',
}
df['学历'] = df['学历'].map(edu_map)
```

```python
df['学历'].value_counts(dropna=False)
```

清洗后类别清晰多了。还可以进一步把 `object` 转为 `category` 类型——
既节省内存，也明确这是分类数据。

```python
df['学历'] = df['学历'].astype('category')
df['学历'].dtype
```

## 4. 城市清洗

城市列的主要问题是**混入了不属于城市的数据**。

```python
df['城市'].value_counts()
```

可以看到「焦虑型人格」「在读」「-」等明显不是城市名。

处理方法：先 `str.strip()` 去空格，再用布尔索引剔除非城市值。
（「应届」相关值已在上述预处理中提走，不会再出现在这里。）

```python
# 去空格
df['城市'] = df['城市'].str.strip()

# 省/市格式，保留斜杠后的城市名
# 如「湖南/株洲」→「株洲」
df['城市'] = df['城市'].str.split('/').str[-1].str.strip()

# 剔除异常值
not_cities = ['焦虑型人格', '在读', '-']
df.loc[df['城市'].isin(not_cities), '城市'] = None
df.loc[df['城市'] == '广西柳州', '城市'] = '柳州'
df['城市'].value_counts()
```

## 5. 经验清洗

经验列的目标是转为**数值（年）**。但现有的值五花八门——

```python
df['经验'].value_counts()
```

我们需要处理的情况：
- `「3年」「2.5年」` → 提取数字
- `「3个月」「4个月」` → 转为年
- （应届已在预处理中统一处理为 0）
- `「<1年」「<3年」` → 取中间值
- `「10+年」` → 取下限
- `「转行」「读研」「上海」` → 无法解析为经验值，设为 NaN

这些规则适合用 **自定义函数 + `apply()`** 来实现。

```python
def parse_experience(val):
    if pd.isna(val):
        return np.nan
    if isinstance(val, (int, float)):
        return float(val)
    s = str(val).strip()
    if s in ['-', '']:
        return np.nan
    if s in ['转行', '读研', '上海']:
        return np.nan
    if '应届' in s or '毕业' in s:
        return 0.0
    if s in ['大一', '大三']:
        return 0.0
    plus_match = re.search(r'(\d+(?:\.\d+)?)\s*\+\s*年', s)
    if plus_match:
        return float(plus_match.group(1))
    lt_match = re.search(r'<(\d+(?:\.\d+)?)\s*年', s)
    if lt_match:
        v = float(lt_match.group(1))
        return max(v - 0.5, 0)
    years = re.findall(r'(\d+(?:\.\d+)?)\s*年', s)
    if years:
        return sum(float(y) for y in years)
    months = re.findall(r'(\d+)\s*个月', s)
    if months:
        return round(sum(float(m) for m in months) / 12, 1)
    return np.nan

df['经验'] = df['经验'].apply(parse_experience)
```

```python
df['经验'].describe()
```

```python
df['经验'].value_counts(dropna=False)
```

经验被成功转为了 `float64` 类型。

## 6. 薪资清洗

薪资是最复杂的一列——来看看它有多少种写法。

```python
df['薪资'].value_counts()
```

薪资列的情况：
- `9k`、`10k` → 标准千元月薪
- `20w/年` → 年薪，需转月薪
- `13k*15` → 带月数，取基本月薪
- `30k+`、`25+k` → 带加号
- `前端：20k\n转行：28k` → 多行，取最后一行
- `25k->22k->18k` → 变动历史，取最新值
- `16k（广州）` → 带地点说明
- `50k~60k` → 薪资范围
- `实习`、`？？` → 异常值

还是用 **自定义函数 + `apply()`** 处理。

```python
def parse_salary(val):
    if pd.isna(val):
        return np.nan
    s = str(val).strip()
    if s in ['实习', '-', '？？', '']:
        return np.nan
    if '\n' in s:
        s = s.split('\n')[-1].strip()
    if '->' in s:
        s = s.split('->')[-1].strip()
    s = re.sub(r'[（(][^)）]*[)）]', '', s).strip()
    w_match = re.search(r'(\d+(?:\.\d+)?)\s*w', s, re.IGNORECASE)
    if w_match:
        return round(float(w_match.group(1)) * 10 / 12, 1)
    slash_match = re.search(r'(\d+(?:\.\d+)?)\s*k\s*/\s*(\d+(?:\.\d+)?)\s*k', s, re.IGNORECASE)
    if slash_match:
        return float(slash_match.group(2))
    k_match = re.search(r'(\d+(?:\.\d+)?)\s*\+?\s*k', s, re.IGNORECASE)
    if k_match:
        return float(k_match.group(1))
    return np.nan

df['薪资'] = df['薪资'].apply(parse_salary)
```

```python
df['薪资'].value_counts(dropna=False)
```

```python
df['薪资'].describe()
```

清洗后薪资变为 `float64`，单位统一为**千元/月**，可以正常做统计分析了。

（注：`200w/年` 等个例被正确转为月薪约 166.7k。个别极端值可根据业务需要后续处理。）

```python
df = df.rename(columns={'薪资': '薪资(k)', '经验': '经验(年)'})
```

## 7. 清洗结果验证

最后整体检查一遍清洗成果。

```python
df.info()
```

```python
df.dtypes
```

```python
df.head()
```

```python
df = df[['昵称', '学历', '城市', '经验(年)', '薪资(k)', '毕业年份', '备注']]
df['毕业年份'] = df['毕业年份'].astype('Int32')
# 保存到 Excel
df.to_excel('../linking_clean.xlsx', index=False)

# 保存到 CSV
df.to_csv('../linking_clean.csv', index=False, encoding='utf-8-sig')
```
