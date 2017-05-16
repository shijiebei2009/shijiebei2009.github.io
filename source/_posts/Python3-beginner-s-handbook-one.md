title: "Python3入门手册之一"
date: 2015-07-02 21:41:03
tags: [Python3]
categories: Programming Notes

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*

**There are two ways of constructing a software design: one way is to make it so simple that there are obviously no deficiences; the other is to make it so complicated that there are no obvious deficiences.**
---- C.A.R.Hoare

**Success in life is a matter not so much of talent and opportunity as of concentration and perseverance**
---- C.W.Wendte


### 选择Python的原因
* 简单易学，功能强大，具有高效的高层数据结构，支持面向对象编程
* 解释性语言，可扩展性，可嵌入性，丰富的库

### 选择一个编辑器
工欲善其事必先利其器，所以选择编辑器首当其冲。
* IDLE：Python自带，极简利器，支持语法高亮
* Vim/Emacs：`Linux/FreeBSD`平台上的开发利器，二者择其一
* PyCharm：号称最智能的Python编辑器，的确是的，但是略微复杂

### 数据结构和算法
#### 解压序列赋值给多个变量
```python
# 变量的数量需要和序列元素的数量一致
data = ['a', 'b', 'c', 'd']
a, b, c, d = data
print(a, b, c, d)

data = ['ACME', 50, 91.1, (2012, 12, 21)]
name, shares, price, (year, month, day) = data
print(year, month, day)

# 解压一部分，其他值丢弃
data = ['ACME', 50, 91.1, (2012, 12, 21)]
_, shares, price, _ = data
print(shares, price)

# 只取第一个和最后一个，中间的所有元素用通配符接收
record = ('Dave', 'dave@example.com', '773-555-1212', '847-555-1212')
name, *_, phone = record
print(name, phone)

'''
OutPut:
a b c d
2012 12 21
50 91.1
Dave 847-555-1212
'''
```
#### 查找最大或最小的N个元素
```python
import heapq

nums = [1, 8, 2, 23, 7, -4, 18, 23, 42, 37, 2]
print(heapq.nlargest(3, nums))  # Prints [42, 37, 23]
print(heapq.nsmallest(3, nums))  # Prints [-4, 1, 2]
# heapify会先将集合数据进行堆排序后放入一个列表中
# heappop会先将第一个元素弹出来，然后用下一个最小的元素来取代被弹出元素
heapq.heapify(nums)
print(heapq.heappop(nums))  # Prints -4
print(heapq.heappop(nums))  # Prints 1
print(heapq.heappop(nums))  # Prints 2

# 下面代码在对每个元素进行对比的时候，会以price的值进行比较。
portfolio = [
    {'name': 'IBM', 'shares': 100, 'price': 91.1},
    {'name': 'AAPL', 'shares': 50, 'price': 543.22},
    {'name': 'FB', 'shares': 200, 'price': 21.09},
    {'name': 'HPQ', 'shares': 35, 'price': 31.75},
    {'name': 'YHOO', 'shares': 45, 'price': 16.35},
    {'name': 'ACME', 'shares': 75, 'price': 115.65}
]
cheap = heapq.nsmallest(3, portfolio, key=lambda s: s['price'])
expensive = heapq.nlargest(3, portfolio, key=lambda s: s['price'])
print(cheap)
print(expensive)
```
#### 字典操作相关
```python
# 求字典中值的最小值、最大值
prices = {
    'ACME': 45.23,
    'AAPL': 612.78,
    'IBM': 205.55,
    'HPQ': 37.20,
    'FB': 10.75
}
min_price = min(zip(prices.values(), prices.keys()))
# min_price is (10.75, 'FB')
max_price = max(zip(prices.values(), prices.keys()))
# max_price is (612.78, 'AAPL')

# 还可以使用sorted()和zip()函数来排列字典数据
prices_sorted = sorted(zip(prices.values(), prices.keys()))
# prices_sorted is [(10.75, 'FB'), (37.2, 'HPQ'),
#                   (45.23, 'ACME'), (205.55, 'IBM'),
#                   (612.78, 'AAPL')]
# 执行这些计算的时候，需要注意的是zip()函数创建的是一个只能访问一次的迭代器。 比如，下面的代码就会产生错误：
prices_and_names = zip(prices.values(), prices.keys())
print(min(prices_and_names))  # OK
print(max(prices_and_names))  # ValueError: max() arg is an empty sequence

# 需要注意的是在计算操作中使用到了(值，键)对。当多个实体拥有相同的值的时候，键会决定返回结果。 比如，在执行min()和max()操作的时候，如果恰巧最小或最大值有重复的，那么拥有最小或最大键的实体会返回：
prices = {'AAA': 45.23, 'ZZZ': 45.23}
min(zip(prices.values(), prices.keys()))
# OutPut:(45.23, 'AAA')
max(zip(prices.values(), prices.keys()))
# OutPut:(45.23, 'ZZZ')

# 查找两字典的相同点
a = {
    'x': 1,
    'y': 2,
    'z': 3
}

b = {
    'w': 10,
    'x': 11,
    'y': 2
}
# Find keys in common
a.keys() & b.keys()  # { 'x', 'y' }
# Find keys in a that are not in b
a.keys() - b.keys()  # { 'z' }
# Find (key,value) pairs in common
a.items() & b.items()  # { ('y', 2) }
```

#### 序列中出现次数最多的元素
```python
words = [
    'look', 'into', 'my', 'eyes', 'look', 'into', 'my', 'eyes',
    'the', 'eyes', 'the', 'eyes', 'the', 'eyes', 'not', 'around', 'the',
    'eyes', "don't", 'look', 'around', 'the', 'eyes', 'look', 'into',
    'my', 'eyes', "you're", 'under'
]
from collections import Counter

word_counts = Counter(words)
# 出现频率最高的3个单词
top_three = word_counts.most_common(3)
print(top_three)
# Outputs [('eyes', 8), ('the', 5), ('look', 4)]

# Counter 对象可以接受任意的 hashable 序列对象。 在底层实现上，一个 Counter 对象就是一个字典，将元素映射到它出现的次数上
print(word_counts['not'])  # Prints 1
print(word_counts['eyes'])  # Prints 8

# 注意，对于Counter对象可以直接进行数学运算操作
test = Counter(['addone'])
word_counts = word_counts + test
print(word_counts)
word_counts = word_counts - test
print(word_counts)
```

#### 通过某个关键字排序一个字典列表
```python
# 通过某个关键字排序一个字典列表
rows = [
    {'fname': 'Brian', 'lname': 'Jones', 'uid': 1003},
    {'fname': 'David', 'lname': 'Beazley', 'uid': 1002},
    {'fname': 'John', 'lname': 'Cleese', 'uid': 1001},
    {'fname': 'Big', 'lname': 'Jones', 'uid': 1004}
]
# 根据任意的字典字段来排序输入结果行是很容易实现的，代码示例：

from operator import itemgetter

rows_by_fname = sorted(rows, key=itemgetter('fname'))
rows_by_uid = sorted(rows, key=itemgetter('uid'))
print(rows_by_fname)
print(rows_by_uid)

# 排序不支持原生比较的对象
class User:
    def __init__(self, userId):
        self.userId = userId

    def __repr__(self):
        return 'User({})'.format(self.userId)


from operator import attrgetter

users = [User(23), User(28), User(26)]
print(sorted(users, key=attrgetter('userId')))
```
#### 通过某个字段将记录分组
```python
rows = [
    {'address': '5412 N CLARK', 'date': '07/01/2012'},
    {'address': '5148 N CLARK', 'date': '07/04/2012'},
    {'address': '5800 E 58TH', 'date': '07/02/2012'},
    {'address': '2122 N CLARK', 'date': '07/03/2012'},
    {'address': '5645 N RAVENSWOOD', 'date': '07/02/2012'},
    {'address': '1060 W ADDISON', 'date': '07/02/2012'},
    {'address': '4801 N BROADWAY', 'date': '07/01/2012'},
    {'address': '1039 W GRANVILLE', 'date': '07/04/2012'},
]
# 现在假设你想在按date分组后的数据块上进行迭代。为了这样做，你首先需要按照指定的字段(这里就是date)排序， 然后调用 itertools.groupby() 函数：

from operator import itemgetter
from itertools import groupby

# Sort by the desired field first
rows.sort(key=itemgetter('date'))
# Iterate in groups
for date, items in groupby(rows, key=itemgetter('date')):
    print(date)
    for i in items:
        print(' ', i)

'''
OutPut:
07/01/2012
  {'date': '07/01/2012', 'address': '5412 N CLARK'}
  {'date': '07/01/2012', 'address': '4801 N BROADWAY'}
07/02/2012
  {'date': '07/02/2012', 'address': '5800 E 58TH'}
  {'date': '07/02/2012', 'address': '5645 N RAVENSWOOD'}
  {'date': '07/02/2012', 'address': '1060 W ADDISON'}
07/03/2012
  {'date': '07/03/2012', 'address': '2122 N CLARK'}
07/04/2012
  {'date': '07/04/2012', 'address': '5148 N CLARK'}
  {'date': '07/04/2012', 'address': '1039 W GRANVILLE'}
'''
```

#### 过滤序列元素
```pytho