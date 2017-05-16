title: "使用Python的CSV模块读写CSV文件"
tags: [Python3]
categories: Programming Notes
date: 2015-10-08 12:01:00

---

**开发环境：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32**

####自序
也许你会说，我为什么要学习使用CSV模块呢？没有CSV模块我一样可以解析操作CSV文件，比如下面这种代码：
```python
with open('stocks.csv') as f:
for line in f:
    row = line.split(',')
    # process row
    ...
```
使用这种方式的一个缺点就是你仍然需要去处理一些棘手的细节问题。比如，如果某些字段值被引号包围，你不得不去除这些引号。另外，如果一个被引号包围的字段碰巧含有一个逗号，那么程序就会因为产生一个错误而停止。

默认情况下，`CSV`库可识别`Microsoft Excel`所使用的`CSV`编码规则。这或许也是最常见的形式，并且也会给你带来最好的兼容性。然而，如果你查看`CSV`的文档，就会发现有很多种方法将它应用到其他编码格式上（如修改分隔字符等，用Tab分隔）。所以你应该总是优先选择`CSV`模块分割或解析`CSV`数据。


####CSV格式简介
逗号分隔值（`Comma-Separated Values`，`CSV`，有时也称为字符分隔值，因为分隔字符也可以不是逗号），其文件以纯文本形式存储表格数据（数字和文本）。纯文本意味着该文件是一个字符序列，不含必须像二进制数字那样被解读的数据。`CSV`文件由任意数目的记录组成，记录间以某种换行符分隔；每条记录由字段组成，字段间的分隔符是其它字符或字符串，最常见的是逗号或制表符。通常，所有记录都有完全相同的字段序列。

####CSV文件注意点
如果你正在读取`CSV`数据并将它们转换为命名元组，需要注意对列名进行合法性认证。一个`CSV`格式文件有一个包含非法标识符的列头行，这样最终会导致在创建一个命名元组时产生一个`ValueError`异常而失败。为了解决这问题，你可能不得不先去修正列标题。
```python
import re
with open('stock.csv') as f:
    f_csv = csv.reader(f)
    headers = [ re.sub('[^a-zA-Z_]', '_', h) for h in next(f_csv) ]
    Row = namedtuple('Row', headers)
    for r in f_csv:
        row = Row(*r)
        # Process row
        ...
```
还有重要的一点需要强调的是，`CSV`产生的数据都是字符串类型的，它不会做任何其他类型的转换。如果你需要做这样的类型转换，你必须自己手动去实现。

####CSV模块操作示例
输入文件：stocks.csv
```html
Symbol,Price,Date,Time,Change,Volume
"AA",39.48,"6/11/2007","9:36am",-0.18,181800
"AIG",71.38,"6/11/2007","9:36am",-0.15,195500
"AXP",62.58,"6/11/2007","9:36am",-0.46,935000
"BA",98.31,"6/11/2007","9:36am",+0.12,104800
"C",53.08,"6/11/2007","9:36am",-0.25,360900
"CAT",78.29,"6/11/2007","9:36am",-0.23,225400
```

输出文件：dest.csv
```html
Symbol,Price,Date,Time,Change,Volume
AA,39.48,6/11/2007,9:36am,-0.18,181800
AIG,71.38,6/11/2007,9:36am,-0.15,195500
AXP,62.58,6/11/2007,9:36am,-0.46,935000
```
代码
```python
#!/usr/bin/python
# coding: UTF-8
"""
Created on 2015/9/30  14:55
@author: 'WX'
"""
from collections import namedtuple
import csv

if __name__ == "__main__":
    fileName = 'stocks.csv'
    writeFileName = 'dest.csv'
    with open(fileName) as f:
        reader = csv.reader(f)
        headers = next(reader)
        print("使用下标进行访问")
        for row in reader:
            # 可以使用下标进行访问，row是一个元组
            length = len(row)
            for i in range(length):
                print(row[i], end='\t')
            print()

        print('------------------------------------------------')
    with open(fileName) as f:
        # 使用命名元组访问
        reader = csv.reader(f)
        headers = next(reader)
        Row = namedtuple('Row', headers)
        print("使用命名元组进行访问")
        for r in reader:
            row = Row(*r)
            # 这时候就可以使用首行的列名来进行访问了
            line = '%s\t%s\t%s\t%s\t%s\t%s' % (row.Symbol, row.Price, row.Date, row.Time, row.Change, row.Volume)
            print(line)
        print('------------------------------------------------')

    with open(fileName) as f:
        # 将内容读取到字典序列中，然后用key去读取
        dict_reader = csv.DictReader(f)
        print("使用字典序列进行访问")
        for row in dict_reader:
            line = '%s\t%s\t%s\t%s\t%s\t%s' % (
                row['Symbol'], row['Price'], row['Date'], row['Time'], row['Change'], row['Volume'])
            print(line)
    print('------------------------------------------------')

    print("使用元组方式写入，SUCCESS！")
    with open(writeFileName, mode='w', newline='') as wf:
        headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
        rows = [('AA', 39.48, '6/11/2007', '9:36am', -0.18, 181800),
                ('AIG', 71.38, '6/11/2007', '9:36am', -0.15, 195500),
                ('AXP', 62.58, '6/11/2007', '9:36am', -0.46, 935000),
                ]
        writer = csv.writer(wf)
        writer.writerow(headers)
        writer.writerows(rows)
    print("使用字典方式写入，SUCCESS！")
    # 在Windows平台需要指定newline=''，否则在两行内容之间会多出一行空行
    with open(writeFileName, mode='w', newline='') as wf:
        headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
        rows = [{'Symbol': 'AA', 'Price': 39.48, 'Date': '6/11/2007',
                 'Time': '9:36am', 'Change': -0.18, 'Volume': 181800},
                {'Symbol': 'AIG', 'Price': 71.38, 'Date': '6/11/2007',
                 'Time': '9:36am', 'Change': -0.15, 'Volume': 195500},
                {'Symbol': 'AXP', 'Price': 62.58, 'Date': '6/11/2007',
                 'Time': '9:36am', 'Change': -0.46, 'Volume': 935000}, ]
        writer = csv.DictWriter(wf, headers)
        writer.writeheader()
        writer.writerows(rows)
    # 更换一种分隔符写入文件
    with open(writeFileName, mode='w', newline='') as wf:
        headers = ['Symbol', 'Price', 'Date', 'Time', 'Change', 'Volume']
        rows = [('AA', 39.48, '6/11/2007', '9:36am', -0.18, 181800),
                ('AIG', 71.38, '6/11/2007', '9:36am', -0.15, 195500),
                ('AXP', 62.58, '6/11/2007', '9:36am', -0.46, 935000),
                ]
        writer = csv.writer(wf, delimiter=';')
        writer.writerow(headers)
        writer.writerows(rows)
    print("使用分好作为分隔符写入，SUCCESS！")
```

控制台输出
```html
C:\Python34\python.exe E:/workspaces/Python3/edu/shu/python/shumo/CsvUtil.py
使用下标进行访问
AA    39.48    6/11/2007    9:36am    -0.18    181800
AIG    71.38    6/11/2007    9:36am    -0.15    195500
AXP    62.58    6/11/2007    9:36am    -0.46    935000
BA    98.31    6/11/2007    9:36am    +0.12    104800
C    53.08    6/11/2007    9:36am    -0.25    360900
CAT    78.29    6/11/2007    9:36am    -0.23    225400
------------------------------------------------
使用命名元组进行访问
AA    39.48    6/11/2007    9:36am    -0.18    181800
AIG    71.38    6/11/2007    9:36am    -0.15    195500
AXP    62.58    6/11/2007    9:36am    -0.46    935000
BA    98.31    6/11/2007    9:36am    +0.12    104800
C    53.08    6/11/2007    9:36am    -0.25    360900
CAT    78.29    6/11/2007    9:36am    -0.23    225400
------------------------------------------------
使用字典序列进行访问
AA    39.48    6/11/2007    9:36am    -0.18    181800
AIG    71.38    6/11/2007    9:36am    -0.15    195500
AXP    62.58    6/11/2007    9:36am    -0.46    935000
BA    98.31    6/11/2007    9:36am    +0.12    104800
C    53.08    6/11/2007    9:36am    -0.25    360900
CAT    78.29    6/11/2007    9:36am    -0.23    225400
------------------------------------------------
使用元组方式写入，SUCCESS！
使用字典方式写入，SUCCESS！

Process finished with exit code 0
```

####分析和统计
最后，如果你读取`CSV`数据的目的是做数据分析和统计的话，你可能需要看一看`Pandas`包。`Pandas`包含了一个非常方便的函数叫`pandas.read_csv()`，它可以加载`CSV`数据到一个`DataFrame`对象中去。然后利用这个对象你就可以生成各种形式的统计、过滤数据以及执行其他高级操作了。


参考资料
【1】http://baike.baidu.com/subview/468993/5926031.htm
【2】http://python3-cookbook.readthedocs.org/zh_CN/latest/c06/p01_read_write_csv_data.html?highlight=csv
