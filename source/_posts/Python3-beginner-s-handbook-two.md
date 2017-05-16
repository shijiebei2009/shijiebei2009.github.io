title: "Python3入门手册之二"
tags: [Python3]
categories: Programming Notes
date: 2015-07-04 21:16:41

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*
### 数字日期和时间
#### 数字的四舍五入
```python
#!/usr/bin/python
# coding: UTF-8
"""
Created on 2015/7/3  23:01
@author: 'WX'
"""

print(round(1.23, 1))  # Prints 1.2
print(round(1.27, 1))  # Prints 1.3
print(round(-1.27, 1))  # Prints -1.3
print(round(1.25361, 3))  # Prints 1.254
# 当一个值刚好在两个边界的中间的时候，round 函数返回离它最近的偶数
print(round(1.25, 1))  # Prints 1.2
print(round(1.35, 1))  # Prints 1.4

# 传给 round() 函数的 ndigits 参数可以是负数，这种情况下， 舍入运算会作用在十位、百位、千位等上面
a = 1627731
print(round(a, -1))
print(round(a, -2))
print(round(a, -3))

# 格式化浮点数，保留指定的小数点后保留位数
print(format(1.23456, '0.2f'))
```
#### 执行精确的浮点数运算
如果需要精确的浮点数计算，那么推荐使用`decimal`模块的`Decimal`
```python
from decimal import Decimal

a = Decimal('1.3')
b = Decimal('1.7')
print(a / b)

a = Decimal('4.2')
b = Decimal('2.1')
c = a.__add__(b)
d = a + b
print(c)  # Prints 6.3
print(d)  # Prints 6.3
```
#### 无穷大与NaN
```python
a = float('inf')
b = float('-inf')
c = float('nan')
print(a)  # Prints inf
print(b)  # Prints -inf
print(c)  # Prints nan
import math
# 可以使用如下方式测试这些值的存在，测试一个NaN值的唯一安全的方法就是使用math.isnan()
print(math.isinf(a))  # Prints True
print(math.isnan(c))  # Prints True
```
#### 分数运算
```python
# fractions模块可以被用来执行包含分数的数学运算
from fractions import Fraction

a = Fraction(5, 4)
b = Fraction(7, 16)
print(a + b)  # Prints 27/16
c = a * b
print(c)  # Prints 35/64
print(c.numerator)  # 获取分子
print(c.denominator)  # 获取分母
print(float(c))  # 强转成float类型
d = 3.75
e = Fraction(*d.as_integer_ratio())  # 强转float类型到分数类型
print(e)
```
#### 大型数组运算
涉及到数组的重量级运算操作，可以使用`NumPy`库。`NumPy`的一个主要特征是它会给Python提供一个数组对象，相比标准的Python列表而言更适合用来做数学运算
```python
>>> # Python lists
>>> x = [1, 2, 3, 4]
>>> y = [5, 6, 7, 8]
>>> x * 2
[1, 2, 3, 4, 1, 2, 3, 4]
>>> x + 10
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
TypeError: can only concatenate list (not "int") to list
>>> x + y
[1, 2, 3, 4, 5, 6, 7, 8]
```

```python
>>> # Numpy arrays
>>> import numpy as np
>>> ax = np.array([1, 2, 3, 4])
>>> ay = np.array([5, 6, 7, 8])
>>> ax * 2
array([2, 4, 6, 8])
>>> ax + 10
array([11, 12, 13, 14])
>>> ax + ay
array([ 6, 8, 10, 12])
>>> ax * ay
array([ 5, 12, 21, 32])
```
`NumPy`还为数组操作提供了大量的通用函数，这些函数可以作为`math`模块中类似函数的替代
```python
>>> np.sqrt(ax)
array([ 1. , 1.41421356, 1.73205081, 2. ])
>>> np.cos(ax)
array([ 0.54030231, -0.41614684, -0.9899925 , -0.65364362])
```
使用`NumPy`构造一个二维数组的例子
```python
>>> grid = np.zeros(shape=(10000,10000), dtype=float)
>>> grid
    array([[ 0., 0., 0., ..., 0., 0., 0.],
    [ 0., 0., 0., ..., 0., 0., 0.],
    [ 0., 0., 0., ..., 0., 0., 0.],
    ...,
    [ 0., 0., 0., ..., 0., 0., 0.],
    [ 0., 0., 0., ..., 0., 0., 0.],
    [ 0., 0., 0., ..., 0., 0., 0.]])
```
对于`NumPy`构造的数组，如何选择行和列呢？
```python
>>> a = np.array([[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12]])
>>> a
array([[ 1, 2, 3, 4],
[ 5, 6, 7, 8],
[ 9, 10, 11, 12]])

>>> # Select row 1
>>> a[1]
array([5, 6, 7, 8])

>>> # Select column 1
>>> a[:,1]
array([ 2, 6, 10])

>>> # Select a subregion and change it
>>> a[1:3, 1:3]
array([[ 6, 7],
        [10, 11]])
>>> a[1:3, 1:3] += 10
>>> a
array([[ 1, 2, 3, 4],
        [ 5, 16, 17, 8],
        [ 9, 20, 21, 12]])

>>> # Broadcast a row vector across an operation on all rows
>>> a + [100, 101, 102, 103]
array([[101, 103, 105, 107],
        [105, 117, 119, 111],
        [109, 121, 123, 115]])
>>> a
array([[ 1, 2, 3, 4],
        [ 5, 16, 17, 8],
        [ 9, 20, 21, 12]])

>>> # Conditional assignment on an array
>>> np.where(a < 10, a, 10)
array([[ 1, 2, 3, 4],
        [ 5, 10, 10, 8],
        [ 9, 10, 10, 10]])
```
通常我们导入`NumPy`模块的时候会使用语句`import numpy as np`。这样的话你就不用再你的程序里面一遍遍的敲入`numpy`，只需要输入`np`就行了，节省了不少时间。
#### 基本的日期与时间转换
为了执行不同时间单位的转换和计算，请使用`datetime`模块
```python
>>> from datetime import timedelta
>>> a = timedelta(days=2, hours=6)
>>> b = timedelta(hours=4.5)
>>> c = a + b
>>> c.days
2
>>> c.seconds
37800
>>> c.seconds / 3600
10.5
>>> c.total_seconds() / 3600
58.5
```
如果你想表示指定的日期和时间，先创建一个`datetime`实例然后使用标准的数学运算来操作它们
```python
>>> from datetime import datetime
>>> a = datetime(2012, 9, 23)
>>> print(a + timedelta(days=10))
2012-10-03 00:00:00
>>>
>>> b = datetime(2012, 12, 21)
>>> d = b - a
>>> d.days
89
>>> now = datetime.today()
>>> print(now)
2012-12-21 14:54:43.094063
>>> print(now + timedelta(minutes=10))
2012-12-21 15:04:43.094063
```
对大多数基本的日期和时间处理问题，`datetime`模块以及足够了。 如果你需要执行更加复杂的日期操作，比如处理时区，模糊时间范围，节假日计算等等， 可以考虑使用`dateutil`模块
```python
>>> a = datetime(2012, 9, 23)
>>> a + timedelta(months=1)
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
TypeError: 'months' is an invalid keyword argument for this function
>>>
>>> from dateutil.relativedelta import relativedelta
>>> a + relativedelta(months=+1)
datetime.datetime(2012, 10, 23, 0, 0)
>>> a + relativedelta(months=+4)
datetime.datetime(2013, 1, 23, 0, 0)
>>>
>>> # Time between two dates
>>> b = datetime(2012, 12, 21)
>>> d = b - a
>>> d
datetime.timedelta(89)
>>> d = relativedelta(b, a)
>>> d
relativedelta(months=+2, days=+28)
>>> d.months
2
>>> d.days
28
```
#### 字符串转换为日期
使用Python的标准模块`datetime`可以很容易的解决这个问题
```python
from datetime import datetime

text = '2012-09-20'
y = datetime.strptime(text, '%Y-%m-%d')
print(y)  # Prints 2012-09-20 00:00:00
z = datetime.now()
print(z)  # Prints 2015-07-04 15:49:17.419612
diff = z - y
print(diff)  # Prints 1017 days, 15:49:17.419612
```
`datetime.strptime()`方法支持很多的格式化代码，比如`%Y`代表4位数年份，`%m`代表两位数月份。还有一点值得注意的是这些格式化占位符也可以反过来使用，将日期输出为指定的格式字符串形式
```python
>>> z
datetime.datetime(2012, 9, 23, 21, 37, 4, 177393)
>>> nice_z = datetime.strftime(z, '%A %B %d, %Y')
>>> nice_z
'Sunday September 23, 2012'
```
#### 结合时区的日期操作
对几乎所有涉及到时区的问题，你都应该使用`pytz`模块。这个包提供了`Olson`时区数据库，它是时区信息的事实上的标准，在很多语言和操作系统里面都可以找到。`pytz`模块一个主要用途是将`datetime`库创建的简单日期对象本地化。
```python
from datetime import datetime
from pytz import timezone

d = datetime(2015, 1, 1, 9, 30, 0)
print(d)  # Prints 2015-01-01 09:30:00
# central = timezone('Asia/Shanghai')
# Localize the date for Chicago
central = timezone('US/Central')
loc_d = central.localize(d)
print(loc_d)  # Prints 2015-01-01 09:30:00-06:00
```
一旦日期被本地化了， 它就可以转换为其他时区的时间了。 
```python
bang_d = loc_d.astimezone(timezone('Asia/Shanghai'))
print(bang_d)  # Prints 2015-01-01 23:30:00+08:00
```
当涉及到