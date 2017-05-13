title: "Python3入门手册之二"
tags: [Python3]
categories: Programming Notes
date: 2015-07-04 21:16:41

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*
###数字日期和时间
####数字的四舍五入
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
####执行精确的浮点数运算
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
####无穷大与NaN
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
####分数运算
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
####大型数组运算
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
####基本的日期与时间转换
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
####字符串转换为日期
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
####结合时区的日期操作
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
当涉及到时区操作的时候，有个问题就是我们如何得到时区的名称。比如，在这个例子中，我们如何知道`“Asia/Shanghai”`就是中国对应的时区名呢？为了查找，可以使用`ISO3166`国家代码作为关键字去查阅字典`pytz.country_timezones`
```python
# Urumqi  乌鲁木齐（新疆首府）
print(pytz.country_timezones['CN'])  # Prints ['Asia/Shanghai', 'Asia/Urumqi']
```

###迭代器与生成器
####代理迭代
Python的迭代器协议需要`__iter__()`方法返回一个实现了`__next__()`方法的迭代器对象。如果你只是迭代遍历其他容器的内容，你无须担心底层是怎样实现的。你所要做的只是传递迭代请求既可。这里的`iter()`函数的使用简化了代码，`iter(s)`只是简单的通过调用`s.__iter__()`方法来返回对应的迭代器对象，就跟`len(s)`会调用`s.__len__()`原理是一样的。
```python
#!/usr/bin/python
# coding: UTF-8
"""
Created on 2015/7/4  18:33
@author: 'WX'
"""
class Node:
    def __init__(self, value):
        self._value = value
        self._children = []

    def __repr__(self):
        return 'Node({!r})'.format(self._value)

    def add_child(self, node):
        self._children.append(node)

    def __iter__(self):
        return iter(self._children)
#Example
if __name__=='__main__':
    root = Node(0)
    child1 = Node(1)
    child2 = Node(2)
    root.add_child(child1)
    root.add_child(child2)
    for ch in root:
        print(ch)
```
####反向迭代
反向迭代仅仅当对象的大小可预先确定或者对象实现了`__reversed__()`的特殊方法时才能生效。如果两者都不符合，那你必须先将对象转换为一个列表才行
```python
a = [1, 2, 3, 4]
for x in reversed(a):
    print(x, end=' ')
print()
# Print a file backwards
f = open('test.csv', mode='r', encoding='utf-8')
for line in reversed(list(f)):
    print(line, end='')

```
要注意的是如果可迭代对象元素很多的话，将其预先转换为一个列表要消耗大量的内存。这时候需要在定义的类上实现`__reversed__()`方法来实现反向迭代
```python
class Countdown:
    def __init__(self, start):
        self.start = start

    # Forward iterator
    def __iter__(self):
        n = self.start
        while n > 0:
            yield n
            n -= 1

    # Reverse iterator
    def __reversed__(self):
        n = 1
        while n <= self.start:
            yield n
            n += 1

for rr in reversed(Countdown(30)):
    print(rr)
for rr in Countdown(30):
    print(rr)
```
定义一个反向迭代器可以使得代码非常的高效，因为它不再需要将数据填充到一个列表中然后再去反向迭代这个列表。
####迭代器切片
函数`itertools.islice()`正好适用于在迭代器和生成器上做切片操作
```python
>>> def count(n):
...     while True:
...         yield n
...         n += 1
...
>>> c = count(0)
>>> c[10:20]
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
TypeError: 'generator' object is not subscriptable

>>> # Now using islice()
>>> import itertools
>>> for x in itertools.islice(c, 10, 20):
...     print(x)
...
10
11
12
13
14
15
16
17
18
19
```
函数`islice()`返回一个可以生成指定元素的迭代器，它通过遍历并丢弃直到切片开始索引位置的所有元素。然后才开始一个个的返回元素，并直到切片结束索引位置。这里要着重强调的一点是`islice()`会消耗掉传入的迭代器中的数据。必须考虑到迭代器是不可逆的这个事实。所以如果你需要之后再次访问这个迭代器的话，那你就得先将它里面的数据放入一个列表中。
####跳过可迭代对象的开始部分
为了演示，假定你在读取一个开始部分是几行注释的源文件
```python
>>> with open('/etc/passwd') as f:
... for line in f:
...     print(line, end='')
...
##
# User Database
#
# Note that this file is consulted directly only when the system is running
# in single-user mode. At other times, this information is provided by
# Open Directory.
...
##
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
...
```
如果你想跳过开始部分的注释行的话，可以这样做
```python
>>> from itertools import dropwhile
>>> with open('/etc/passwd') as f:
...     for line in dropwhile(lambda line: line.startswith('#'), f):
...         print(line, end='')
...
nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false
root:*:0:0:System Administrator:/var/root:/bin/sh
...
```
####排列组合的迭代
`itertools`模块提供了三个函数来解决这类问题。其中一个是`itertools.permutations()`，它接受一个集合并产生一个元组序列，每个元组由集合中所有元素的一个可能排列组成。也就是说通过打乱集合中元素排列顺序生成一个元组
```python
>>> items = ['a', 'b', 'c']
>>> from itertools import permutations
>>> for p in permutations(items):
...     print(p)
...
('a', 'b', 'c')
('a', 'c', 'b')
('b', 'a', 'c')
('b', 'c', 'a')
('c', 'a', 'b')
('c', 'b', 'a')
```
如果你想得到指定长度的所有排列，你可以传递一个可选的长度参数
```python
>>> for p in permutations(items, 2):
...     print(p)
...
('a', 'b')
('a', 'c')
('b', 'a')
('b', 'c')
('c', 'a')
('c', 'b')
```
使用`itertools.combinations()`可得到输入集合中元素的所有的组合
```python
>>> from itertools import combinations
>>> for c in combinations(items, 3):
...     print(c)
...
('a', 'b', 'c')

>>> for c in combinations(items, 2):
...     print(c)
...
('a', 'b')
('a', 'c')
('b', 'c')

>>> for c in combinations(items, 1):
...     print(c)
...
('a',)
('b',)
('c',)
```
对于`combinations()`来讲，元素的顺序已经不重要了。也就是说，组合`('a', 'b')`跟`('b', 'a')`其实是一样的(最终只会输出其中一个)。

在计算组合的时候，一旦元素被选取就会从候选中剔除掉(比如如果元素’a’已经被选取了，那么接下来就不会再考虑它了)。而函数`itertools.combinations_with_replacement()`允许同一个元素被选择多次
```python
>>> for c in combinations_with_replacement(items, 3):
...     print(c)
...
('a', 'a', 'a')
('a', 'a', 'b')
('a', 'a', 'c')
('a', 'b', 'b')
('a', 'b', 'c')
('a', 'c', 'c')
('b', 'b', 'b')
('b', 'b', 'c')
('b', 'c', 'c')
('c', 'c', 'c')
```
####序列上索引值迭代
```python
>>> my_list = ['a', 'b', 'c']
>>> for idx, val in enumerate(my_list):
...     print(idx, val)
...
0 a
1 b
2 c
```
为了按传统行号输出(行号从1开始)，你可以传递一个开始参数：
```python
>>> my_list = ['a', 'b', 'c']
>>> for idx, val in enumerate(my_list, 1):
...     print(idx, val)
...
1 a
2 b
3 c
```
`enumerate()`对于跟踪某些值在列表中出现的位置是很有用的。 所以，如果你想将一个文件中出现的单词映射到它出现的行号上去，可以很容易的利用`enumerate()`来完成
```python
word_summary = defaultdict(list)

with open('myfile.txt', 'r') as f:
    lines = f.readlines()

for idx, line in enumerate(lines):
    # Create a list of words in current line
    words = [w.strip().lower() for w in line.split()]
    for word in words:
        word_summary[word].append(idx)
```
如果你处理完文件后打印 `word_summary`，会发现它是一个字典(准确来讲是一个`defaultdict`)，对于每个单词有一个`key`，每个`key`对应的值是一个由这个单词出现的行号组成的列表。如果某个单词在一行中出现过两次，那么这个行号也会出现两次，同时也可以作为文本的一个简单统计。

`enumerate()`函数返回的是一个`enumerate`对象实例，它是一个迭代器，返回连续的包含一个计数和一个值的元组，元组中的值通过在传入序列上调用`next()`返回。

还有一点可能并不很重要，但是也值得注意，有时候当你在一个已经解压后的元组序列上使用`enumerate()`函数时很容易调入陷阱。你得像下面正确的方式这样写：
```python
data = [ (1, 2), (3, 4), (5, 6), (7, 8) ]
# Correct!
for n, (x, y) in enumerate(data):
    ...
# Error!
for n, x, y in enumerate(data):
    ...
```
####同时迭代多个序列
为了同时迭代多个序列，使用`zip()`函数
```python
>>> xpts = [1, 5, 4, 2, 10, 7]
>>> ypts = [101, 78, 37, 15, 62, 99]
>>> for x, y in zip(xpts, ypts):
...     print(x,y)
...
1 101
5 78
4 37
2 15
10 62
7 99
>>>
```
`zip(a, b)`会生成一个可返回元组`(x, y)`的迭代器，其中`x`来自`a`，`y`来自`b`。一旦其中某个序列到底结尾，迭代宣告结束。因此迭代长度跟参数中最短序列长度一致。
```python
>>> a = [1, 2, 3]
>>> b = ['w', 'x', 'y', 'z']
>>> for i in zip(a,b):
...     print(i)
...
(1, 'w')
(2, 'x')
(3, 'y')
```
如果这个不是你想要的效果，那么还可以使用`itertools.zip_longest()`函数来代替。
```python
>>> from itertools import zip_longest
>>> for i in zip_longest(a,b):
...     print(i)
...
(1, 'w')
(2, 'x')
(3, 'y')
(None, 'z')

>>> for i in zip_longest(a, b, fillvalue=0):
...     print(i)
...
(1, 'w')
(2, 'x')
(3, 'y')
(0, 'z')
```
最后强调一点就是，`zip()`会创建一个迭代器来作为结果返回。如果你需要将结对的值存储在列表中，要使用`list()`函数。
```python
>>> zip(a, b)
<zip object at 0x1007001b8>
>>> list(zip(a, b))
[(1, 10), (2, 11), (3, 12)]
```
####不同集合上元素的迭代
当可迭代对象类型不一样的时候`chain()`同样可以很好的工作，它接受一个可迭代对象列表作为输入，并返回一个迭代器，有效的屏蔽掉在多个容器中迭代细节。
```python
>>> from itertools import chain
>>> a = [1, 2, 3, 4]
>>> b = ['x', 'y', 'z']
>>> for x in chain(a, b):
... print(x)
...
1
2
3
4
x
y
z
>>>
```
####展开嵌套的序列
可以写一个包含`yield from`语句的递归生成器来轻松解决这个问题
```python
from collections import Iterable

def flatten(items, ignore_types=(str, bytes)):
    for x in items:
        if isinstance(x, Iterable) and not isinstance(x, ignore_types):
            yield from flatten(x)
        else:
            yield x

items = [1, 2, [3, 4, [5, 6], 7], 8]
# Produces 1 2 3 4 5 6 7 8
for x in flatten(items):
    print(x)
```
额外的参数`ignore_types`和检测语句`isinstance(x,ignore_types)`用来将字符串和字节排除在可迭代对象外，防止将它们再展开成单个的字符。这样的话字符串数组就能最终返回我们所期望的结果了。
```python
>>> items = ['Dave', 'Paula', ['Thomas', 'Lewis']]
>>> for x in flatten(items):
...     print(x)
...
Dave
Paula
Thomas
Lewis
```
####顺序迭代合并后的排序迭代对象
```python
>>> import heapq
>>> a = [1, 4, 7, 10]
>>> b = [2, 5, 6, 11]
>>> for c in heapq.merge(a, b):
...     print(c)
...
1
2
4
5
6
7
10
11
```
`heapq.merge`可迭代特性意味着它不会立马读取所有序列。这就意味着你可以在非常长的序列中使用它，而不会有太大的开销。比如，下面是一个例子来演示如何合并两个排序文件
```python
with open('sorted_file_1', 'rt') as file1, \
    open('sorted_file_2', 'rt') as file2, \
    open('merged_file', 'wt') as outf:

    for line in heapq.merge(file1, file2):
        outf.write(line)
```
有一点要强调的是`heapq.merge()`需要所有输入序列必须是排过序的。特别的，它并不会预先读取所有数据到堆栈中或者预先排序，也不会对输入做任何的排序检测。它仅仅是检查所有序列的开始部分并返回最小的那个，这个过程一直会持续直到所有输入序列中的元素都被遍历完。

参考资料
【1】[A Byte of Python3][id]
【2】[python3-cookbook](http://python3-cookbook.readthedocs.org/zh_CN/latest/c01/p08_calculating_with_dict.html)
[id]:http://code.google.com/p/proden/downloads/detail?name=A%20Byte%20of%20Python3(%E4%B8%AD%E6%96%87%E7%89%88).pdf