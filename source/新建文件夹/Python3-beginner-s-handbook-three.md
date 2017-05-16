title: "Python3入门手册之三"
tags: [Python3]
categories: Programming Notes
date: 2015-07-06 20:50:25

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*

###文件与I/O
####读写文本数据
使用带有`rt`模式的`open()`函数读取文本文件
```python
with open('test.csv', mode='rt', encoding='utf-8') as f:
    data = f.read()
    print(data)

with open('test.csv', mode='rt', encoding='utf-8') as f:
    for line in f:
        print(line, end='')

```

类似的，为了写入一个文本文件，使用带有`wt`模式的`open()`函数，如果之前文件内容存在则清除并覆盖掉
```python
with open('dest.csv', mode='wt', encoding='utf-8') as f:
    f.write('text1')  # 注意该方式写入文件默认没有换行符，需手动添加
    f.write('text2')

with open('dest.csv', mode='wt', encoding='utf-8') as f:
    print('text1', file=f)  # 该方式写入文件默认换行，无需手动添加
    print('text2', file=f)
```
如果是在已存在文件中添加内容，使用模式为`at`的`open()`函数。`with`控制块结束时，文件会自动关闭。你也可以不使用`with`语句，但是这时候你就必须记得手动关闭文件。

**使用print输出元组**
```python
row = ('a', 'b', 'c')
print(*row, sep=',', end='。') # Prints a,b,c。
```
####读写字节数据
使用模式为`rb`或`wb`的`open()`函数来读取或写入二进制数据
```python
with open('dest2.csv', mode='wb') as f:
    f.write(b'HelloWorld!')

with open('dest2.csv', mode='rb') as f:
    print(f.read())
```
在读取二进制数据的时候，字节字符串和文本字符串的语义差异可能会导致一个潜在的陷阱。特别需要注意的是，索引和迭代动作返回的是字节的值而不是字节字符串。
```python
>>> # Text string
>>> t = 'Hello World'
>>> t[0]
'H'
>>> for c in t:
...     print(c)
...
H
e
l
l
o
...
>>> # Byte string
>>> b = b'Hello World'
>>> b[0]
72
>>> for c in b:
...     print(c)
...
72
101
108
108
111
...
```
####向不存在的文件中写入数据
如果只允许向不存在的文件写入数据，可以在`open()`函数中使用`x`模式来代替`w`模式来解决这个问题。
```python
with open('dest2.csv', mode='xt') as f:
    f.write('test')

Traceback (most recent call last):
  File "E:/workspaces/Python3/edu/shu/python/study/FileAndIO.py", line 33, in <module>
    with open('dest2.csv', mode='xt') as f:
FileExistsError: [Errno 17] File exists: 'dest2.csv'
```
由提示信息可知，当该文件在系统中已经存在的时候，是不允许继续向其中写入内容的。如果文件是二进制的，使用`xb`来代替`xt`。
####字符串的I/O操作
使用`io.StringIO()`和`io.BytesIO()`类来创建类文件对象操作字符串数据
```python
import io

s = io.StringIO()
s.write('Hello World!')

print('\nthis is a test', file=s)
# 向控制台输出内容
print(s.getvalue())

s = io.StringIO('Hello World!\n')
# 先读取4个字符
print(s.read(4))
# 读取剩下的字符
print(s.read())
```
`io.StringIO`只能用于文本。如果你要操作二进制数据，要使用`io.BytesIO`类来代替
```python
>>> s = io.BytesIO()
>>> s.write(b'binary data')
>>> s.getvalue()
b'binary data'
```
####读写压缩文件
`gzip`和`bz2`模块可以很容易的处理这些文件。两个模块都为`open()`函数提供了另外的实现来解决这个问题
```python
# gzip compression
import gzip
with gzip.open('somefile.gz', 'rt') as f:
    text = f.read()

# bz2 compression
import bz2
with bz2.open('somefile.bz2', 'rt') as f:
    text = f.read()
```
类似的，为了写入压缩数据，可以这样做
```python
# gzip compression
import gzip
with gzip.open('somefile.gz', 'wt') as f:
    f.write(text)

# bz2 compression
import bz2
with bz2.open('somefile.bz2', 'wt') as f:
    f.write(text)
```
大部分情况下读写压缩数据都是很简单的。但是要注意的是选择一个正确的文件模式是非常重要的。如果你不指定模式，那么默认的就是二进制模式，如果这时候程序想要接受的是文本数据，那么就会出错。
当写入压缩数据时，可以使用`compresslevel`这个可选的关键字参数来指定一个压缩级别。
```python
with gzip.open('somefile.gz', 'wt', compresslevel=5) as f:
    f.write(text)
```
默认的等级是`9`，也是最高的压缩等级。等级越低性能越好，但是数据压缩程度也越低。

最后一点，`gzip.open()`和`bz2.open()`还有一个很少被知道的特性，它们可以作用在一个已存在并以二进制模式打开的文件上。比如，下面代码是可行的：
```python
import gzip
f = open('somefile.gz', 'rb')
with gzip.open(f, 'rt') as g:
    text = g.read()
```
这样就允许`gzip`和`bz2`模块可以工作在许多类文件对象上，比如套接字，管道和内存中文件等。
####读取二进制数据到可变缓冲区中
```python
import os.path

def read_into_buffer(filename):
    buf = bytearray(os.path.getsize(filename))
    with open(filename, 'rb') as f:
        f.readinto(buf)
    return buf
```

```python
>>> # Write a sample file
>>> with open('sample.bin', 'wb') as f:
...     f.write(b'Hello World')
...
>>> buf = read_into_buffer('sample.bin')
>>> buf
bytearray(b'Hello World')
>>> buf[0:5] = b'Hallo'
>>> buf
bytearray(b'Hallo World')
>>> with open('newsample.bin', 'wb') as f:
...     f.write(buf)
...
11
```
文件对象的`readinto()`方法能被用来为预先分配内存的数组填充数据，甚至包括由`array`模块或`numpy`库创建的数组。和普通`read()`方法不同的是，`readinto()`填充已存在的缓冲区而不是为新对象重新分配内存再返回它们。因此，你可以使用它来避免大量的内存分配操作。比如，如果你读取一个由相同大小的记录组成的二进制文件时，你可以像下面这样写：
```python
record_size = 32 # Size of each record (adjust value)

buf = bytearray(record_size)
with open('somefile', 'rb') as f:
    while True:
        n = f.readinto(buf)
        if n < record_size:
            break
        # Use the contents of buf
        ...
```
####文件路径名的操作
```python
>>> import os
>>> path = '/Users/beazley/Data/data.csv'

>>> # Get the last component of the path
>>> os.path.basename(path)
'data.csv'

>>> # Get the directory name
>>> os.path.dirname(path)
'/Users/beazley/Data'

>>> # Join path components together
>>> os.path.join('tmp', 'data', os.path.basename(path))
'tmp/data/data.csv'

>>> # Expand the user's home directory
>>> path = '~/Data/data.csv'
>>> os.path.expanduser(path)
'/Users/beazley/Data/data.csv'

>>> # Split the file extension
>>> os.path.splitext(path)
('~/Data/data', '.csv')
```
对于任何的文件名的操作，你都应该使用`os.path`模块，而不是使用标准字符串操作来构造自己的代码。特别是为了可移植性考虑的时候更应如此，因为`os.path`模块知道`Unix`和`Windows`系统之间的差异并且能够可靠地处理类似`Data/data.csv`和`Data\data.csv`这样的文件名。其次，你真的不应该浪费时间去重复造轮子。通常最好是直接使用已经为你准备好的功能。

使用如下方式测试文件的类型，如果测试的文件不存在的时候，结果都会返回**False**
```python
>>> # Is a regular file
>>> os.path.isfile('/etc/passwd')
True

>>> # Is a directory
>>> os.path.isdir('/etc/passwd')
False

>>> # Is a symbolic link
>>> os.path.islink('/usr/local/bin/python3')
True

>>> # Get the file linked to
>>> os.path.realpath('/usr/local/bin/python3')
'/usr/local/bin/python3.3'
```
如果你还想获取元数据(比如文件大小或者是修改日期)，也可以使用`os.path`模块来解决
```python
>>> os.path.getsize('/etc/passwd')
3669
>>> os.path.getmtime('/etc/passwd')
1272478234.0
>>> import time
>>> time.ctime(os.path.getmtime('/etc/passwd'))
'Wed Apr 28 13:10:34 2010'
```
使用`os.listdir()`函数来获取某个目录中的文件列表
```python
import os
names = os.listdir('somedir')
```
获取目录中的列表是很容易的，但是其返回结果只是目录中实体名列表而已。如果你还想获取其他的元信息，比如文件大小，修改时间等等，你或许还需要使用到`os.path`模块中的函数或者`os.stat()`函数来收集数据。
```python
# Example of getting a directory listing

import os
import os.path
import glob

pyfiles = glob.glob('*.py')

# Get file sizes and modification dates
name_sz_date = [(name, os.path.getsize(name), os.path.getmtime(name))
                for name in pyfiles]
for name, size, mtime in name_sz_date:
    print(name, size, mtime)

# Alternative: Get file metadata
file_metadata = [(name, os.stat(name)) for name in pyfiles]
for name, meta in file_metadata:
    print(name, meta.st_size, meta.st_mtime)
```
####增加或改变已打开文件的编码
如果你想给一个以二进制模式打开的文件添加`Unicode`编码/解码方式，可以使用`io.TextIOWrapper()`对象包装它。
```python
import urllib.request
import io

u = urllib.request.urlopen('http://www.python.org')
f = io.TextIOWrapper(u, encoding='utf-8')
text = f.read()
```
如果你想修改一个已经打开的文本模式的文件的编码方式，可以先使用`detach()`方法移除掉已存在的文本编码层，并使用新的编码方式代替。
```python
>>> import sys
>>> sys.stdout.encoding
'UTF-8'
>>> sys.stdout = io.TextIOWrapper(sys.stdout.detach(), encoding='latin-1')
>>> sys.stdout.encoding
'latin-1
```
在文本模式打开的文件中写入原始的字节数据，将字节数据直接写入文件的缓冲区即可。
```python
>>> import sys
>>> sys.stdout.write(b'Hello\n')
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
TypeError: must be str, not bytes
>>> sys.stdout.buffer.write(b'Hello\n')
Hello
5
>>>
```
类似的，能够通过读取文本文件的`buffer`属性来读取二进制数据。
####创建临时文件和文件夹
`tempfile`模块中有很多的函数可以完成这任务。为了创建一个匿名的临时文件，可以使用`tempfile.TemporaryFile`：
```python
from tempfile import TemporaryFile

with TemporaryFile('w+t') as f:
    # Read/write to the file
    f.write('Hello World\n')
    f.write('Testing\n')

    # Seek back to beginning and read the data
    f.seek(0)
    data = f.read()

# Temporary file is destroyed
```
`TemporaryFile()`的第一个参数是文件模式，通常来讲文本模式使用`w+t`，二进制模式使用`w+b`。这个模式同时支持读和写操作，在这里是很有用的，因为当你关闭文件去改变模式的时候，文件实际上已经不存在了。`TemporaryFile()`另外还支持跟内置的`open()`函数一样的参数。

在大多数`Unix`系统上，通过`TemporaryFile()`创建的文件都是匿名的，甚至连目录都没有。如果你想打破这个限制，可以使用`NamedTemporaryFile()`来代替。比如：
```python
from tempfile import NamedTemporaryFile
with NamedTemporaryFile('w+t') as f:
    print('filename is:', f.name)
    ...
# File automatically destroyed
```
这里，被打开文件的`f.name`属性包含了该临时文件的文件名。当你需要将文件名传递给其他代码来打开这个文件的时候，这个就很有用了。和`TemporaryFile()`一样，结果文件关闭时会被自动删除掉。如果你不想这么做，可以传递一个关键字参数`delte=False`即可。比如：
```python
with NamedTemporaryFile('w+t', delete=False) as f:
    print('filename is:', f.name)
    ...
```
为了创建一个临时目录，可以使用`tempfile.TemporaryDirectory()`
```python
from tempfile import TemporaryDirectory

with TemporaryDirectory() as dirname:
    print('dirname is:', dirname)
    # Use the directory
    ...
# Directory and all contents destroyed
```
`TemporaryFile()`、`NamedTemporaryFile()`和`TemporaryDirectory()`函数应该是处理临时文件目录的最简单的方式了，因为它们会自动处理所有的创建和清理步骤。在一个更低的级别，你可以使用`mkstemp()`和`mkdtemp()`来创建临时文件和目录。
####与串行端口的数据通信
尽管你可以通过使用Python内置的I/O模块来完成这个任务，但对于串行通信最好的选择是使用[pySerial](http://pyserial.sourceforge.net/)包。这个包的使用非常简单，先安装`pySerial`，使用类似下面这样的代码就能很容易的打开一个串行端口：
```python
import serial
ser = serial.Serial('/dev/tty.usbmodem641', # Device name varies
                    baudrate=9600,
                    bytesize=8,
                    parity='N',
                    stopbits=1)
```
设备名对于不同的设备和操作系统是不一样的。比如，在`Windows`系统上，你可以使用0，1等表示的一个设备来打开通信端口`COM0`和`COM1`。一旦端口打开，那就可以使用`read()`，`readline()`和`write()`函数读写数据了。
```python
ser.write(b'G1 X50 Y50\r\n')
resp = ser.readline()
```
####序列化Python对象
对于序列化最普遍的做法就是使用`pickle`模块。为了将一个对象保存到一个文件中，可以这样做：
```python
import pickle

data = ... # Some Python object
f = open('somefile', 'wb')
pickle.dump(data, f)
```
为了将一个对象转储为一个字符串，可以使用`pickle.dumps()`：
`s = pickle.dumps(data)`
为了从字节流中恢复一个对象，使用`picle.load()`或`pickle.loads()`函数。比如：
```python
# Restore from a file
f = open('somefile', 'rb')
data = pickle.load(f)

# Restore from a string
data = pickle.loads(s)
```
###数据编码和处理
####读写CSV数据
```python
import csv
from collections import namedtuple
# 将CSV中的数据读入元组中，并允许使用列名如row.Symbol和row.Change（Symbol和Change是CSV文件中的列名）
with open('stock.csv') as f:
    f_csv = csv.reader(f)
    headings = next(f_csv)
    Row = namedtuple('Row', headings)
    for r in f_csv:
        row = Row(*r)
        # Process row
        ...
```
另外一个选择就是将数据读取到一个字典序列中去。
```python
import csv
with open('stocks.csv') as f:
    f_csv = csv.DictReader(f)
    for row in f_csv:
        # process row
        ...
```
在这个版本中，你可以使用列名去访问每一行的数据了。比如，`row['Symbol']`或者`row['Change']`。

为了写入`CSV`数据，你仍然可以使用`CSV`模块，不过这时候先创建一个`writer`对象。
```python
headers = ['Symbol','Price','Date','Time','Change','Volume']
rows = [('AA', 39.48, '6/11/2007', '9:36am', -0.18, 181800),
         ('AIG', 71.38, '6/11/2007', '9:36am', -0.15, 195500),
         ('AXP', 62.58, '6/11/2007', '9:36am', -0.46, 935000),
       ]

with open('stocks.csv','w') as f:
    f_csv = csv.writer(f)
    f_csv.writerow(headers)
    f_csv.writerows(rows)
```
如果你有一个字典序列的数据，可以像这样做
```python
headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
rows = [{'Symbol':'AA', 'Price':39.48, 'Date':'6/11/2007',
        'Time':'9:36am', 'Change':-0.18, 'Volume':181800},
        {'Symbol':'AIG', 'Price': 71.38, 'Date':'6/11/2007',
        'Time':'9:36am', 'Change':-0.15, 'Volume': 195500},
        {'Symbol':'AXP', 'Price': 62.58, 'Date':'6/11/2007',
        'Time':'9:36am', 'Change':-0.46, 'Volume': 935000},
        ]

with open('stocks.csv','w') as f:
    f_csv = csv.DictWriter(f, headers)
    f_csv.writeheader()
    f_csv.writerows(rows)
```
####读写JSON数据
`json`模块提供了一种很简单的方式来编码和解码JSON数据。其中两个主要的函数是`json.dumps()`和`json.loads()`，要比其他序列化函数库如`pickle`的接口少得多。
```python
import json
data = {
    'name' : 'ACME',
    'shares' : 100,
    'price' : 542.23
}
json_str = json.dumps(data)
```
将JSON编码的字符串转回一个Python数据结构
`data = json.loads(json_str)`
如果你要处理的是文件而不是字符串，你可以使用`json.dump()`和`json.load()`来编码和解码JSON数据。
```python
# Writing JSON data
with open('data.json', 'w') as f:
    json.dump(data, f)

# Reading data back
with open('data.json', 'r') as f:
    data = json.load(f)
```
####解析简单的XML数据
可以使用`xml.etree.ElementTree`模块从简单的XML文档中提取数据。
```python
from urllib.request import urlopen
from xml.etree.ElementTree import parse

# Download the RSS feed and parse it
u = urlopen('http://planet.python.org/rss20.xml')
doc = parse(u)

# Extract and output tags of interest
for item in doc.iterfind('channel/item'):
    title = item.findtext('title')
    date = item.findtext('pubDate')
    link = item.findtext('link')

    print(title)
    print(date)
    print(link)
    print()
```
`xml.etree.ElementTree.parse()`函数解析整个XML文档并将其转换成一个文档对象。然后，你就能使用`find()`、`iterfind()`和`findtext()`等方法来搜索特定的XML元素了。这些函数的参数就是某个指定的标签名，例如`channel/item`或`title`。

`ElementTree`模块中的每个元素有一些重要的属性和方法，在解析的时候非常有用。`tag`属性包含了标签的名字，`text`属性包含了内部的文本，而`get()`方法能获取属性值。

有一点要强调的是`xml.etree.ElementTree`并不是`XML`解析的唯一方法。对于更高级的应用程序，你需要考虑使用`lxml`。它使用了和`ElementTree`同样的编程接口，因此上面的例子同样也适用于`lxml`。你只需要将刚开始的`import`语句换成`from lxml.etree import parse`就行了。`lxml`完全遵循`XML`标准，并且速度也非常快，同时还支持验证，`XSLT`和`XPath`等特性。

####增量式解析大型XML文件
任何时候只要你遇到增量式的数据处理时，第一时间就应该想到迭代器和生成器。 下面是一个很简单的函数，只使用很少的内存就能增量式的处理一个大型XML文件。
```python
from xml.etree.ElementTree import iterparse

def parse_and_remove(filename, path):
    path_parts = path.split('/')
    doc = iterparse(filename, ('start', 'end'))
    # Skip the root element
    next(doc)

    tag_stack = []
    elem_stack = []
    for event, elem in doc:
        if event == 'start':
            tag_stack.append(elem.tag)
            elem_stack.append(/)
        elif event == 'end':
            if tag_stack == path_parts:
                yield elem
                elem_stack[-2].remove(elem)
            try:
                tag_stack.pop()
                elem_stack.pop()
            except IndexError:
                pass
```
`iterparse()`方法允许对`XML`文档进行增量操作。使用时，你需要提供文件名和一个包含下面一种或多种类型的事件列表：`start`,`end`,`start-ns`和`end-ns`。由`iterparse()`创建的迭代器会产生形如(event, elem)的元组，其中`event`是上述事件列表中的某一个，而`elem`是相应的XML元素。

`start`事件在某个元素第一次被创建并且还没有被插入其他数据(如子元素)时被创建。而`end`事件在某个元素已经完成时被创建。尽管没有在例子中演示，`start-ns`和`end-ns`事件被用来处理`XML`文档命名空间的声明。
####将字典转换为XML
```python
from xml.etree.ElementTree import Element

def dict_to_xml(tag, d):
'''
Turn a simple dict of key/value pairs into XML
'''
elem = Element(tag)
for key, val in d.items():
    child = Element(key)
    child.text = str(val)
    elem.append(child)
return elem
```
对于`I/O`操作，使用`xml.etree.ElementTree`中的`tostring()`函数很容易就能将它转换成一个字节字符串。
```python
>>> from xml.etree.ElementTree import tostring
>>> tostring(e)
b'<stock><price>490.1</price><shares>100</shares><name>GOOG</name></stock>'
>>>
```
`xml.sax.saxutils`中的`escape()`和`unescape()`函数提供对HTML实体的转换方法
```python
>>> from xml.sax.saxutils import escape, unescape
>>> escape('<spam>')
'&lt;spam&gt;'
>>> unescape(_)
'<spam>'
```
####解析和修改XML
```python
>>> from xml.etree.ElementTree import parse, Element
>>> doc = parse('pred.xml')
>>> root = doc.getroot()
>>> root
<Element 'stop' at 0x100770cb0>

>>> # Remove a few elements
>>> root.remove(root.find('sri'))
>>> root.remove(root.find('cr'))
>>> # Insert a new element after <nm>...</nm>
>>> root.getchildren().index(root.find('nm'))
1
>>> e = Element('spam')
>>> e.text = 'This is a test'
>>> root.insert(2, e)

>>> # Write back to a file
>>> doc.write('newpred.xml', xml_declaration=True)
```
####编码和解码十六进制数
```python
>>> # Initial byte string
>>> s = b'hello'
>>> # Encode as hex
>>> import binascii
>>> h = binascii.b2a_hex(s)
>>> h
b'68656c6c6f'
>>> # Decode back to bytes
>>> binascii.a2b_hex(h)
b'hello'
```
类似的功能同样可以在`base64`模块中找到
```python
>>> import base64
>>> h = base64.b16encode(s)
>>> h
b'68656C6C6F'
>>> base64.b16decode(h)
b'hello'
```
大部分情况下，通过使用上述的函数来转换十六进制是很简单的。上面两种技术的主要不同在于大小写的处理。函数`base64.b16decode()`和`base64.b16encode()`只能操作大写形式的十六进制字母，而`binascii`模块中的函数大小写都能处理。
还有一点需要注意的是编码函数所产生的输出总是一个字节字符串。如果想强制以`Unicode`形式输出，你需要增加一个额外的界面步骤。例如：
```python
>>> h = base64.b16encode(s)
>>> print(h)
b'68656C6C6F'
>>> print(h.decode('ascii'))
68656C6C6F
```
在解码十六进制数时，函数`b16decode()`和`a2b_hex()`可以接受字节或`unicode`字符串。但是，`unicode`字符串必须仅仅只包含`ASCII`编码的十六进制数。
####编码解码Base64数据
`base64`模块中有两个函数`b64encode()`and`b64decode()`可以帮你解决这个问题。
```python
>>> # Some byte data
>>> s = b'hello'
>>> import base64

>>> # Encode as Base64
>>> a = base64.b64encode(s)
>>> a
b'aGVsbG8='

>>> # Decode from Base64
>>> base64.b64decode(a)
b'hello'
```
####读写二进制数组数据
可以使用`struct`模块处理二进制数据。下面是一段示例代码将一个Python元组列表写入一个二进制文件，并使用`struct`将每个元组编码为一个结构体。
```python
from struct import Struct
def write_records(records, format, f):
    '''
    Write a sequence of tuples to a binary file of structures.
    '''
    record_struct = Struct(format)
    for r in records:
        f.write(record_struct.pack(*r))

# Example
if __name__ == '__main__':
    records = [ (1, 2.3, 4.5),
                (6, 7.8, 9.0),
                (12, 13.4, 56.7) ]
    with open('data.b', 'wb') as f:
        write_records(records, '<idd', f)
```
如果你打算以块的形式增量读取文件，你可以这样做：
```python
from struct import Struct

def read_records(format, f):
    record_struct = Struct(format)
    chunks = iter(lambda: f.read(record_struct.size), b'')
    return (record_struct.unpack(chunk) for chunk in chunks)

# Example
if __name__ == '__main__':
    with open('data.b','rb') as f:
        for rec in read_records('<idd', f):
            # Process rec
            ...
```
如果你想将整个文件一次性读取到一个字节字符串中，然后在分片解析。那么你可以这样做：
```python
from struct import Struct

def unpack_records(format, data):
    record_struct = Struct(format)
    return (record_struct.unpack_from(data, offset)
            for offset in range(0, len(data), record_struct.size))

# Example
if __name__ == '__main__':
    with open('data.b', 'rb') as f:
        data = f.read()
    for rec in unpack_records('<idd', data):
        # Process rec
        ...
```

参考资料
【1】[A Byte of Python3][id]
【2】[python3-cookbook](http://python3-cookbook.readthedocs.org/zh_CN/latest/c01/p08_calculating_with_dict.html)
[id]:http://code.google.com/p/proden/downloads/detail?name=A%20Byte%20of%20Python3(%E4%B8%AD%E6%96%87%E7%89%88).pdf