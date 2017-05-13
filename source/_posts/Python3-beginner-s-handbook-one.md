title: "Python3入门手册之一"
date: 2015-07-02 21:41:03
tags: [Python3]
categories: Programming Notes

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*

**There are two ways of constructing a software design: one way is to make it so simple that there are obviously no deficiences; the other is to make it so complicated that there are no obvious deficiences.**
--- C.A.R.Hoare

**Success in life is a matter not so much of talent and opportunity as of concentration and perseverance**
--- C.W.Wendte


###选择Python的原因
* 简单易学，功能强大，具有高效的高层数据结构，支持面向对象编程
* 解释性语言，可扩展性，可嵌入性，丰富的库

###选择一个编辑器
工欲善其事必先利其器，所以选择编辑器首当其冲。
* IDLE：Python自带，极简利器，支持语法高亮
* Vim/Emacs：`Linux/FreeBSD`平台上的开发利器，二者择其一
* PyCharm：号称最智能的Python编辑器，的确是的，但是略微复杂

###数据结构和算法
####解压序列赋值给多个变量
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
####查找最大或最小的N个元素
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
####字典操作相关
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

####序列中出现次数最多的元素
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

####通过某个关键字排序一个字典列表
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
####通过某个字段将记录分组
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

####过滤序列元素
```python
values = ['1', '2', '-3', '-', '4', 'N/A', '5']
def is_int(val):
    try:
        x = int(val)
        return True
    except ValueError:
        return False
ivals = list(filter(is_int, values))
print(ivals)
# Outputs ['1', '2', '-3', '4', '5']
```
####转换并同时计算数据
```python
# Determine if any .py files exist in a directory
import os
files = os.listdir('dirname')
if any(name.endswith('.py') for name in files):
    print('There be python!')
else:
    print('Sorry, no python.')
# Output a tuple as CSV
s = ('ACME', 50, 123.45)
print(','.join(str(x) for x in s)) # Prints ACME,50,123.45
# Data reduction across fields of a data structure
portfolio = [
    {'name':'GOOG', 'shares': 50},
    {'name':'YHOO', 'shares': 75},
    {'name':'AOL', 'shares': 20},
    {'name':'SCOX', 'shares': 65}
]
min_shares = min(s['shares'] for s in portfolio)

# Original: Returns 20
min_shares = min(s['shares'] for s in portfolio)
# Alternative: Returns {'name': 'AOL', 'shares': 20}
min_shares = min(portfolio, key=lambda s: s['shares'])
```
####合并多个字典或映射
```python
a = {'x': 1, 'z': 3 }
b = {'y': 2, 'z': 4 }

from collections import ChainMap
c = ChainMap(a,b)
print(c['x']) # Outputs 1 (from a)
print(c['y']) # Outputs 2 (from b)
print(c['z']) # Outputs 3 (from a)

# ChianMap使用原来的字典，它自己不创建新的字典。所以它并不会产生上面所说的结果，比如：

a['x'] = 42
print(c['x']) # Outputs 42 (from a)
```
###字符串和文本
####使用多个界定符分割字符串
`string`对象的`split()`方法只适应于非常简单的字符串分割情形，它并不允许有多个分隔符或者是分隔符周围不确定的空格。当你需要更加灵活的切割字符串的时候，最好使用`re.split()`方法。
```python
import re

line = 'abcd efg; hijk , lmn'
res = re.split(r'[;,\s]\s*', line)
print(res) # Prints ['abcd', 'efg', 'hijk', '', 'lmn']
```
函数`re.split()`是非常实用的，因为它允许你为分隔符指定多个正则模式。比如，在上面的例子中，分隔符可以是逗号(,)，分号(;)或者是空格，并且后面紧跟着任意个的空格。只要这个模式被找到，那么匹配的分隔符两边的实体都会被当成是结果中的元素返回。 返回结果为一个字段列表，这个跟`str.split()`返回值类型是一样的。
当你使用`re.split()`函数时候，需要特别注意的是正则表达式中是否包含一个括号捕获分组。如果使用了捕获分组，那么被匹配的文本也将出现在结果列表中。比如，观察一下这段代码运行后的结果。
```python
fields = re.split(r'(;|,|\s)\s*', line)
print(fields) # Prints ['abcd', ' ', 'efg', ';', 'hijk', ' ', '', ',', 'lmn']
```
如果你不想保留分割字符串到结果列表中去，但仍然需要使用到括号来分组正则表达式的话，确保你的分组是非捕获分组，形如`(?:...)`。
```python
res = re.split(r'(?:,|;|\s)\s*', line)
print(res) # Prints ['abcd', 'efg', 'hijk', '', 'lmn']
```
####字符串匹配
```python
# 检查多种匹配可能，需要将所有的匹配项放入到一个元组中，然后传给startswith()或者endswith()方法
strs = list()
strs.append('a.txt')
strs.append('b.doc')
for s in strs:
    # startswith方法类似
    if s.endswith(('.txt', '.doc')):
        print(str(s))

# 请注意，该方法必须接收一个元组作为输入参数，如果你有一个list或者set类型的选择项，需要先调用tuple()将其转换为元组类型
choices = ['.txt', '.doc']
for s in strs:
    if s.endswith(tuple(choices)):
        print(str(s))
```

```python
# fnmatch和fnmatchcase的使用
from fnmatch import fnmatch, fnmatchcase

print(fnmatch('foo.txt', '*.TXT'))  # On OS X (Mac) Prints False
print(fnmatch('foo.txt', '*.TXT'))  # On Windows Prints True

# 完全使用你的模式大小写进行匹配
print(fnmatchcase('foo.txt', '*.TXT'))  # Prints False
```
####字符串搜索和替换
一个替换回调函数的参数是一个`match`对象，也就是`match()`或者`find()`返回的对象。使用`group()`方法来提取特定的匹配部分。回调函数最后返回替换字符串。如果除了替换后的结果外，你还想知道有多少替换发生了，可以使用`re.subn()`来代替。
```python
# 使用re模块进行匹配和搜索
text = 'Today is 11/27/2012. PyCon starts 3/13/2013.'
datepat = re.compile(r'(\d+)/(\d+)/(\d+)')
list_match = datepat.findall(text)  # findall方法返回的是所有匹配的列表
print(list_match)
for m in datepat.finditer(text):  # finditer方法返回的是迭代器
    print(m.groups())


# 注意点，在正则的开头字母‘r’是指定不去解析反斜杠
# r'(\d+)/(\d+)/(\d+)' 相当于 '(\\d+)/(\\d+)/(\\d+)'
m = datepat.match('11/27/2012')
print(m.group(0))  # Prints 11/27/2012
print(m.group(1))  # Prints 11
print(m.group(2))  # Prints 27
print(m.group(3))  # Prints 2012

newtext, n = datepat.subn(r'\3-\1-\2', text)
print(newtext)  # Prints Today is 2012-11-27. PyCon starts 2013-3-13.
print(n)  # Prints 2
```

为了在文本操作时忽略大小写，你需要在使用re模块的时候给这些操作提供`re.IGNORECASE`标志参数。
```python
text = 'UPPER PYTHON, lower python, Mixed Python'
re.findall('python', text, flags=re.IGNORECASE) # ['PYTHON', 'python', 'Python']
re.sub('python', 'snake', text, flags=re.IGNORECASE) # 'UPPER snake, lower snake, Mixed snake'
```
####最短匹配模式
在这个例子中，模式`r'\"(.*)\"'`的意图是匹配被双引号包含的文本。但是在正则表达式中*操作符是贪婪的，因此匹配操作会查找最长的可能匹配。 
```python
str_pat = re.compile(r'\"(.*)\"')
text = 'Computer says "no." Phone says "yes."'
str_pat.findall(text)  # ['no." Phone says "yes.']

# 如果希望获取最短匹配的结果，需要添加'?'
str_pat = re.compile(r'\"(.*?)\"')
str_pat.findall(text)  # ['no.', 'yes.']
```
这样就使得匹配变成非贪婪模式，从而得到最短的匹配，也就是我们想要的结果。

####多行匹配模式
这个问题很典型的出现在当你用点(.)去匹配任意字符的时候，忘记了点(.)不能匹配换行符的事实。为了修正这个问题，你可以修改模式字符串，增加对换行的支持。比如：
```python
text = '''/* this is a
multiline comment */
'''
comment = re.compile(r'/\*((?:.|\n)*?)\*/')
com = comment.findall(text)
print(com)  # Prints [' this is a\nmultiline comment ']
```
在这个模式中，`(?:.|\n)` 指定了一个非捕获组(也就是它定义了一个仅仅用来做匹配，而不能通过单独捕获或者编号的组)。
`re.compile()`函数接受一个标志参数叫 `re.DOTALL`，在这里非常有用。它可以让正则表达式中的.匹配包括换行符在内的任意字符。比如：
```python
comment = re.compile(r'/\*(.*?)\*/', re.DOTALL)
com = comment.findall(text)
print(com)  # Prints [' this is a\nmultiline comment ']
```
####删除字符串中不需要的字符
`strip()`方法能用于删除开始或结尾的字符。`lstrip()`和`rstrip()`分别从左和从右执行删除操作。默认情况下，这些方法会去除空白字符，但是你也可以指定其他字符。
如果你想处理中间的空格，那么你需要求助其他技术。比如使用`replace()`方法或者是用正则表达式替换。
```python
import re
s = 'Hello        World!'
m = re.sub('\s+', ' ', s)
print(m) # Prints Hello World!
```
####字符串对齐
对于基本的字符串对齐操作，可以使用字符串的`ljust()`,`rjust()`和`center()`方法。
```python
text = 'Hello World!'
text = text.ljust(20)
print(text)  # Hello World!
print(len(text))  # 20

text = 'Hello World!'
text = text.rjust(20)
print(text)  #      Hello World!
print(len(text))  # 20
```
所有这些方法都能接受一个可选的填充字符。
```python
text = 'Hello World!'
print(text.rjust(20, '='))
print(text.center(20, '*'))
# 当格式化多个值的时候，这些格式代码也可以被用在 format() 方法中
print('{:+>10s} {:->10s}'.format('Hello', 'World')) # Prints +++++Hello -----World
```
####字符串中插入变量
```python
s = '{name} has {n} messages.'
print(s.format(name='Guido', n=37))

# 如果要被替换的变量能在变量域中找到，那么你可以结合使用format_map()和vars() 。就像下面这样：
s = '{name} has {n} messages.'
name = 'Guido'
n = 37
print(s.format_map(vars()))
```
####以指定列宽格式化字符串
```python
>>> import textwrap
>>> print(textwrap.fill(s, 70))
Look into my eyes, look into my eyes, the eyes, the eyes, the eyes,
not around the eyes, don't look around the eyes, look into my eyes,
you're under.

>>> print(textwrap.fill(s, 40))
Look into my eyes, look into my eyes,
the eyes, the eyes, the eyes, not around
the eyes, don't look around the eyes,
look into my eyes, you're under.

>>> print(textwrap.fill(s, 40, initial_indent='    '))
    Look into my eyes, look into my
eyes, the eyes, the eyes, the eyes, not
around the eyes, don't look around the
eyes, look into my eyes, you're under.

>>> print(textwrap.fill(s, 40, subsequent_indent='    '))
Look into my eyes, look into my eyes,
    the eyes, the eyes, the eyes, not
    around the eyes, don't look around
    the eyes, look into my eyes, you're
    under.
```
####在字符串中处理html和xml
如果你想替换文本字符串中的`‘<’`或者`‘>’`，使用html.escape()函数可以很容易的完成。比如：
```python
>>> s = 'Elements are written as "<tag>text</tag>".'
>>> import html
>>> print(s)
Elements are written as "<tag>text</tag>".
>>> print(html.escape(s))
Elements are written as "<tag>text</tag>".

>>> # Disable escaping of quotes
>>> print(html.escape(s, quote=False))
Elements are written as "<tag>text</tag>".
>>>
```
如果你接收到了一些含有编码值的原始文本，需要手动去做替换，通常你只需要使用`HTML`或者`XML`解析器的一些相关工具函数/方法即可。比如：
```python
>>> s = 'Spicy "Jalape&#241;o&quot.'
>>> from html.parser import HTMLParser
>>> p = HTMLParser()
>>> p.unescape(s)
'Spicy "Jalapeño".'
>>>
>>> t = 'The prompt is >>>'
>>> from xml.sax.saxutils import unescape
>>> unescape(t)
'The prompt is >>>'
>>>
```

####字节字符串上的字符串操作
```python
>>> data = b'Hello World'
>>> data[0:5]
b'Hello'
>>> data.startswith(b'Hello')
True
>>> data.split()
[b'Hello', b'World']
>>> data.replace(b'Hello', b'Hello Cruel')
b'Hello Cruel World'
>>>
```
这些操作同样也适用于字节数组。
```python
>>> data = bytearray(b'Hello World')
>>> data[0:5]
bytearray(b'Hello')
>>> data.startswith(b'Hello')
True
>>> data.split()
[bytearray(b'Hello'), bytearray(b'World')]
>>> data.replace(b'Hello', b'Hello Cruel')
bytearray(b'Hello Cruel World')
>>>
```
大多数情况下，在文本字符串上的操作均可用于字节字符串。然而，这里也有一些需要注意的不同点。首先，字节字符串的索引操作返回整数而不是单独字符。
```python
>>> a = 'Hello World' # Text string
>>> a[0]
'H'
>>> a[1]
'e'
>>> b = b'Hello World' # Byte string
>>> b[0]
72
>>> b[1]
101
>>>
```
第二点，字节字符串不会提供一个美观的字符串表示，也不能很好的打印出来，除非它们先被解码为一个文本字符串。
```python
>>> s = b'Hello World'
>>> print(s)
b'Hello World' # Observe b'...'
>>> print(s.decode('ascii'))
Hello World
>>>
```

参考资料
【1】[A Byte of Python3][id]
【2】[python3-cookbook](http://python3-cookbook.readthedocs.org/zh_CN/latest/c01/p08_calculating_with_dict.html)
[id]:http://code.google.com/p/proden/downloads/detail?name=A%20Byte%20of%20Python3(%E4%B8%AD%E6%96%87%E7%89%88).pdf
