# statistics
> Note: 本文涉及的python源码的版本为 python 3.10.12

`statistics.py` 是Python的一个标准库，主要提供了用于**常见数据处理**的函数。

### <span style="border-bottom:2px dashed yellow;">The First Glance</span>

`__all__`可以定义用户在导入某个库/模块时，可以被导入的成员（如变量、函数和类），在`statistics.py`的文件开头就可以看到其定义的可见成员非常简单：
```python
__all__ = [
    'NormalDist',
    'StatisticsError',
    'correlation',
    'covariance',
    'fmean',
    'geometric_mean',
    'harmonic_mean',
    'linear_regression',
    'mean',
    'median',
    'median_grouped',
    'median_high',
    'median_low',
    'mode',
    'multimode',
    'pstdev',
    'pvariance',
    'quantiles',
    'stdev',
    'variance',
]
```
首先，`statistics`模块提供了三大类基础的函数组：
- 平均值以及对中心位置的评估函数：`mean`、`fmean`、`geometric_mean`、`harmonic_mean`、`median`、`median_low`、`median_high`、`median_grouped`、`mode`、`multimode`、`quantiles`
- 对分散程度的评估函数：`pstdev`、`pvariance`、`stdev`、`variance`
- 对两个输入之间关系的统计函数：`covariance`、`correlation`、`linear_regression`

此外，模块还提供了：
- 用于创建和操纵随机变量的正态分布的类：`NormalDist`
- 表示统计相关的异常：`StatisticsError`

### <span style="border-bottom:2px dashed yellow;">Take Away</span>

> 1.Python的`doctest`可以通过增加`#doctest: +ELLIPSIS`的注释来规避某些随机情况和复杂输出
```python
"""
>>> stdev([2.5, 3.25, 5.5, 11.25, 11.75])  #doctest: +ELLIPSIS
4.38961843444...
"""
```
> 2.Python中判断变量是否为迭代器/生成器
```python
def mean(data):
    if iter(data) is data:   # 生成器和迭代器可以同时判断
        data = list(data)
    ...

# or
from inspect import isgenerator
if isgenerator(data):       # 最完整的判断方法，具体判断方法可以看其源码
    ...

# or not
from typing import Iterable
if isinstance(data, Iterable):  # 只要是可迭代（list、set...）的都是True
    ...
```

> 简单线性回归 `linear_regression` 函数使用 **最小二乘法** 进行计算，返回预定义的具名元组 `LinearRegression`
```python
LinearRegression = namedtuple('LinearRegression', ('slope', 'intercept')
```

> 可以使用 `__slots__` 限制类可以被绑定的属性和方法
```python
class NormalDist:
    __slots__ = {
        '_mu': 'Arithmetic mean of a normal distribution',
        '_sigma': 'Standard deviation of a normal distribution',
    }
```