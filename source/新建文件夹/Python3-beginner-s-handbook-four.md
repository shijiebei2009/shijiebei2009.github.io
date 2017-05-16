date: 2015-07-13 23:03:44
title: "Python3入门手册之四"
tags: [Python3]
categories: Programming Notes

---
*Version：Python 3.4.3 (v3.4.3:9b73f1c3e601, Feb 24 2015, 22:44:40) [MSC v.1600 64 bit (AMD64)] on win32*

###函数
####可接受任意数量参数的函数
为了能让一个函数接受任意数量的位置参数，可以使用一个`*参数`
```python
def avg(first, *rest):
    return (first + sum(rest)) / (1 + len(rest))

# Sample use
avg(1, 2) # 1.5
avg(1, 2, 3, 4) # 2.5
```
在这个例子中，`rest`是由所有其他位置参数组成的元组。然后我们在代码中把它当成了一个序列来进行后续的计算。

使用一个以`**开头的参数`来表示接受任意数量的关键字参数。
```python
import html
def make_element(name, value, **attrs):
    keyvals = [' %s="%s"' % item for item in attrs.items()]
    attr_str = ''.join(keyvals)
    element = '<{name}{attrs}>{value}</{name}>'.format(
        name=name,
        attrs=attr_str,
        value=html.escape(value))
    return element

# Example
# Creates '<item size="large" quantity="6">Albatross</item>'
res = make_element('item', 'Albatross', size='large', quantity=6)
print(res)

# Creates '<p><spam></p>'
res = make_element('p', '<spam>')
print(res)
```
在这里，`attrs`是一个包含所有被传入进来的关键字参数的字典。

如果你还希望某个函数能同时接受任意数量的位置参数和关键字参数，可以同时使用`*和**`。
```python
def anyargs(*args, **kwargs):
    print(args)  # A tuple  Prints ('a', 'b', 'c', 'd', 'e')
    print(kwargs)  # A dict  Prints{'key1': '1', 'key2': '2'}
anyargs('a', 'b', 'c', 'd', 'e', key1='1', key2='2')
```
使用这个函数时，所有位置参数会被放到`args元组`中，所有关键字参数会被放到`字典kwargs`中。
一个`*参数`只能出现在函数定义中最后一个位置参数后面，而 `**参数`只能出现在最后一个参数。 有一点要注意的是，在`*参数`后面仍然可以定义其他参数。

####只接受关键字参数的函数
利用这种技术，我们还能在接受任意多个位置参数的函数中指定关键字参数。
```python
def mininum(*values, clip=None):
    m = min(values)
    if clip is not None:
        m = clip if clip > m else m
    return m

minimum(1, 5, 2, -5, 10) # Returns -5
minimum(1, 5, 2, -5, 10, clip=0) # Returns 0</pre>
```
####返回多个值的函数
为了能返回多个值，函数直接return一个元组就行了。
```python
>>> def myfun():
... return 1, 2, 3
...
>>> a, b, c = myfun()
````
尽管`myfun()`看上去返回了多个值，实际上是先创建了一个元组然后返回的。这个语法看上去比较奇怪，实际上我们使用的是逗号来生成一个元组，而不是用括号。
```python
>>> a = (1, 2) # With parentheses
>>> a
(1, 2)
>>> b = 1, 2 # Without parentheses
>>> b
(1, 2)
```
####定义有默认参数的函数
定义一个有可选参数的函数是非常简单的，直接在函数定义中给参数指定一个默认值，并放到参数列表最后就行了。
```python
def spam(a, b=42):
    print(a, b)

spam(1) # Ok. a=1, b=42
spam(1, 2) # Ok. a=1, b=2
```
如果默认参数是一个可修改的容器比如一个列表、集合或者字典，可以使用None作为默认值
```python
# Using a list as a default value
def spam(a, b=None):
    if b is None:
        b = []
    ...
```
有一点是值得注意的，默认参数的值仅仅在函数定义的时候赋值一次
```python
>>> x = 42
>>> def spam(a, b=x):
...     print(a, b)
...
>>> spam(1)
1 42
>>> x = 23 # Has no effect
>>> spam(1)
1 42
>>>
```
注意到当我们改变`x`的值的时候对默认参数值并没有影响，这是因为在函数定义的时候就已经确定了它的默认值了。

其次，默认参数的值应该是不可变的对象，比如`None、True、False、`数字或字符串。 特别的，千万不要像下面这样写代码：
```python
def spam(a, b=[]): # NO!
    ...
```

如果你这么做了，当默认值在其他地方被修改后你将会遇到各种麻烦。这些修改会影响到下次调用这个函数时的默认值。
```python
>>> def spam(a, b=[]):
...     print(b)
...     return b
...
>>> x = spam(1)
>>> x
[]
>>> x.append(99)
>>> x.append('Yow!')
>>> x
[99, 'Yow!']
>>> spam(1) # Modified list gets returned!
[99, 'Yow!']
```

这种结果应该不是你想要的。为了避免这种情况的发生，最好是将默认值设为`None`， 然后在函数里面检查它，前面的例子就是这样做的。

在测试`None`值时使用`is`操作符是很重要的，也是这种方案的关键点。

```python
def spam(a, b=None):
    if not b: # NO! Use 'b is None' instead
        b = []
    ...
```

####定义匿名或内联函数
当一些函数很简单，仅仅只是计算一个表达式的值的时候，就可以使用`lambda`表达式来代替了。
```python
>>> add = lambda x, y: x + y
>>> add(2,3)
5
>>> add('hello', 'world')
'helloworld'
```

`lambda`表达式典型的使用场景是排序或数据`reduce`等：
```python
>>> names = ['David Beazley', 'Brian Jones',
...         'Raymond Hettinger', 'Ned Batchelder']
>>> sorted(names, key=lambda name: name.split()[-1].lower())
['Ned Batchelder', 'David Beazley', 'Raymond Hettinger', 'Brian Jones']
```
有一点要注意的是，`lambda`表达式中的变量x是自由变量，在运行时绑定值，而不是定义时就绑定，这跟函数的默认值参数定义是不同的。而如果你想让某个匿名函数在定义时就捕获到值，可以将那个参数值定义成默认参数即可，类似如下：

```python
>>> x = 10
>>> a = lambda y, x=x: x + y
>>> x = 20
>>> b = lambda y, x=x: x + y
>>> a(10)
20
>>> b(10)
30
```
如果你不设定为默认参数的话，运行如下
```python
>>> x = 10
>>> a = lambda y: x + y
>>> x = 20
>>> b = lambda y: x + y
>>> a(10)
30
>>> b(10)
30
```
**Notes**
在这里列出来的问题是新手很容易犯的错误，有些新手可能会不恰当的`lambda`表达式。 比如，通过在一个循环或列表推导中创建一个`lambda`表达式列表，并期望函数能在定义时就记住每次的迭代值。

```python
>>> funcs = [lambda x: x+n for n in range(5)]
>>> for f in funcs:
... print(f(0))
...
4
4
4
4
4
>>>
```
但是实际效果是运行是n的值为迭代的最后一个值。现在我们用另一种方式修改一下：
```python
>>> funcs = [lambda x, n=n: x+n for n in range(5)]
>>> for f in funcs:
... print(f(0))
...
0
1
2
3
4
>>>
```
通过使用函数默认值参数形式，`lambda`函数在定义时就能绑定到值。

####减少可调用对象的参数个数
如果需要减少某个函数的参数个数，你可以使用`functools.partial()`。`partial()`允许你给一个或多个参数设置固定的值，减少接下来被调用时的参数个数。
```python
from functools import partial
def spam(a, b, c, d):
    print(a, b, c, d)
spam(1, 2, 3, 4)  # Prints 1 2 3 4
s1 = partial(spam, 1)  # a = 1
s1(5, 6, 7)
s2 = partial(spam, 3, 4, d=12)  # a=3,b=4,d=12
s2(1)

```
可以看出`partial()`固定某些参数并返回一个新的`callable`对象。这个新的`callable`接受未赋值的参数，然后跟之前已经赋值过的参数合并起来，最后将所有参数传递给原始函数。

假设你有一个点的列表来表示`(x,y)`坐标元组。你可以使用下面的函数来计算两点之间的距离：
```python
points = [ (1, 2), (3, 4), (5, 6), (7, 8) ]
import math
def distance(p1, p2):
    x1, y1 = p1
    x2, y2 = p2
    return math.hypot(x2 - x1, y2 - y1)
```
现在假设你想以某个点为基点，根据点和基点之间的距离来排序所有的这些点。列表的`sort()`方法接受一个关键字参数来自定义排序逻辑，但是它只能接受一个单个参数的函数(`distance()`很明显是不符合条件的)。现在我们可以通过使用`partial()`来解决这个问题：

```python
>>> pt = (4, 3)
>>> points.sort(key=partial(distance,pt))
>>> points
[(3, 4), (1, 2), (5, 6), (7, 8)]
```
更进一步，`partial()` 通常被用来微调其他库函数所使用的回调函数的参数。 例如，下面是一段代码，使用 `multiprocessing` 来异步计算一个结果值， 然后这个值被传递给一个接受一个`result`值和一个可选`logging`参数的回调函数：

```python
def output_result(result, log=None):
    if log is not None:
        log.debug('Got: %r', result)

# A sample function
def add(x, y):
    return x + y

if __name__ == '__main__':
    import logging
    from multiprocessing import Pool
    from functools import partial

    logging.basicConfig(level=logging.DEBUG)
    log = logging.getLogger('test')

    p = Pool()
    p.apply_async(add, (3, 4), callback=partial(output_result, log=log))
    p.close()
    p.join()
```
###类与对象
####改变对象的字符串显示
要改变一个实例的字符串表示，可重新定义它的`__str__()`和`__repr__()`方法。
```python
class Pair:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return 'Pair({0.x!r}, {0.y!r})'.format(self)

    def __str__(self):
        return '({0.x!s}, {0.y!s})'.format(self)
```
`__repr__()`方法返回一个实例的代码表示形式，通常用来重新构造这个实例。内置的`repr()`函数返回这个字符串，跟我们使用交互式解释器显示的值是一样的。`__str__()`方法将实例转换为一个字符串，使用`str()`或`print()`函数会输出这个字符串。
```python
>>> p = Pair(3, 4)
>>> p
Pair(3, 4) # __repr__() output
>>> print(p)
(3, 4) # __str__() output
```
`!r`格式化代码指明输出使用`__repr__()`来代替默认的`__str__()`。你还可以做如下的测试：
```python
>>> p = Pair(3, 4)
>>> print('p is {0!r}'.format(p))
p is Pair(3, 4)
>>> print('p is {0}'.format(p))
p is (3, 4)
```
定义`__repr__()`和`__str__()`通常是很好的习惯，因为它能简化调试和实例输出。例如，如果仅仅只是打印输出或日志输出某个实例，那么程序员会看到实例更加详细与有用的信息。
`__repr__()`生成的文本字符串标准做法是需要让`eval(repr(x)) == x`为真。如果实在不能这样子做，应该创建一个有用的文本表示，并使用<和>括起来。
```python
>>> f = open('file.dat')
>>> f
<_io.TextIOWrapper name='file.dat' mode='r' encoding='UTF-8'>
>>>
```
如果`__str__()`没有被定义，那么就会使用`__repr__()`来代替输出。上面的`format()`方法的使用看上去很有趣，格式化代码`{0.x}`对应的是第1个参数的x属性。因此，在下面的函数中，0实际上指的就是`self`本身：
```python
def __repr__(self):
    return 'Pair({0.x!r}, {0.y!r})'.format(self)
```
作为这种实现的一个替代，你也可以使用`%`操作符，就像下面这样
```python
def __repr__(self):
    return 'Pair(%r, %r)' % (self.x, self.y)
```
####让对象支持上下文管理协议
为了让一个对象兼容`with`语句，你需要实现`__enter()__`和`__exit__()`方法。例如，考虑如下的一个类，它能为我们创建一个网络连接：

```python
from socket import socket, AF_INET, SOCK_STREAM

class LazyConnection:
    def __init__(self, address, family=AF_INET, type=SOCK_STREAM):
        self.address = address
        self.family = family
        self.type = type
        self.sock = None

    def __enter__(self):
        if self.sock is not None:
            raise RuntimeError('Already connected')
        self.sock = socket(self.family, self.type)
        self.sock.connect(self.address)
        return self.sock

    def __exit__(self, exc_ty, exc_val, tb):
        self.sock.close()
        self.sock = None
```
这个类的关键特点在于它表示了一个网络连接，但是初始化的时候并不会做任何事情(比如它并没有建立一个连接)。连接的建立和关闭是使用`with`语句自动完成的
```python
from functools import partial

conn = LazyConnection(('www.python.org', 80))
# Connection closed
with conn as s:
    # conn.__enter__() executes: connection open
    s.send(b'GET /index.html HTTP/1.0\r\n')
    s.send(b'Host: www.python.org\r\n')
    s.send(b'\r\n')
    resp = b''.join(iter(partial(s.recv, 8192), b''))
    # conn.__exit__() executes: connection closed
```
编写上下文管理器的主要原理是你的代码会放到`with`语句块中执行。当出现`with`语句的时候，对象的`__enter__()`方法被触发，它返回的值(如果有的话)会被赋值给`as`声明的变量。然后，`with`语句块里面的代码开始执行。最后，`__exit__()`方法被触发进行清理工作。

不管`with`代码块中发生什么，上面的控制流都会执行完，就算代码块中发生了异常也是一样的。事实上，`__exit__()`方法的第三个参数包含了异常类型、异常值和追溯信息(如果有的话)。`__exit__()`方法能自己决定怎样利用这个异常信息，或者忽略它并返回一个None值。如果`__exit__()`返回`True`，那么异常会被清空，就好像什么都没发生一样，`with`语句后面的程序继续在正常执行。

如果你想要获得嵌套`with`语句的效果，代码需要如下修改
```python
from socket import socket, AF_INET, SOCK_STREAM

class LazyConnection:
    def __init__(self, address, family=AF_INET, type=SOCK_STREAM):
        self.address = address
        self.family = family
        self.type = type
        self.connections = []

    def __enter__(self):
        sock = socket(self.family, self.type)
        sock.connect(self.address)
        self.connections.append(sock)
        return sock

    def __exit__(self, exc_ty, exc_val, tb):
        self.connections.pop().close()

# Example use
from functools import partial

conn = LazyConnection(('www.python.org', 80))
with conn as s1:
    pass
        with conn as s2:
        pass
        # s1 and s2 are independent sockets
```
在第二个版本中，`LazyConnection`类可以被看做是某个连接工厂。在内部，一个列表被用来构造一个栈。每次`__enter__()`方法执行的时候，它复制创建一个新的连接并将其加入到栈里面。`__exit__()`方法简单的从栈中弹出最后一个连接并关闭它。这里稍微有点难理解，不过它能允许嵌套使用`with`语句创建多个连接。

在需要管理一些资源比如文件、网络连接和锁的编程环境中，使用上下文管理器是很普遍的。这些资源的一个主要特征是它们必须被手动的关闭或释放来确保程序的正确运行。例如，如果你请求了一个锁，那么你必须确保之后释放了它，否则就可能产生死锁。通过实现`__enter__()`和`__exit__()`方法并使用`with`语句可以很容易的避免这些问题，因为`__exit__()`方法可以让你无需担心这些了。

####创建大量对象时节省内存方法
对于主要是用来当成简单的数据结构的类而言，你可以通过给类添加`__slots__`属性来极大的减少实例所占的内存。
```python
class Date:
    __slots__ = ['year', 'month', 'day']
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
```

当你定义`__slots__`后，Python就会为实例使用一种更加紧凑的内部表示。实例通过一个很小的固定大小的数组来构建，而不是为每个实例定义一个字典，这跟元组或列表很类似。在`__slots__`中列出的属性名在内部被映射到这个数组的指定小标上。使用`slots`一个不好的地方就是我们不能再给实例添加新的属性了，只能使用在`__slots__`中定义的那些属性名。

####在类中封装属性名
`Python`程序员不去依赖语言特性去封装数据，而是通过遵循一定的属性和方法命名规约来达到这个效果。第一个约定是任何以单下划线`_`开头的名字都应该是内部实现。

```python
class A:
    def __init__(self):
        self._internal = 0 # An internal attribute
        self.public = 1 # A public attribute

    def public_method(self):
        '''
        A public method
        '''
        pass

    def _internal_method(self):
        pass
```
`Python`并不会真的阻止别人访问内部名称。但是如果你这么做肯定是不好的，可能会导致脆弱的代码。同时还要注意到，使用下划线开头的约定同样适用于模块名和模块级别函数。例如，如果你看到某个模块名以单下划线开头(比如`_socket`)，那它就是内部实现。类似的，模块级别函数比如`sys._getframe()`在使用的时候就得加倍小心了。

使用双下划线开始会导致访问名称变成其他形式。比如，在前面的类B中，私有属性会被分别重命名为`_B__private`和`_B__private_method`。这时候你可能会问这样重命名的目的是什么，答案就是继承——这种属性通过继承是无法被覆盖的。比如：
```python
class C(B):
    def __init__(self):
        super().__init__()
        self.__private = 1 # Does not override B.__private

    # Does not override B.__private_method()
    def __private_method(self):
        pass
```
这里，私有名称`__private`和`__private_method`被重命名为`_C__private`和`_C__private_method`，这个跟父类B中的名称是完全不同的。

通常的，如果定义的变量与保留关键字冲突，那么可以单划线作为后缀。一般的以单划线开头定义变量即可，如果你清楚你的代码会涉及到子类，那么推荐是用双下划线方案。
#### 创建可管理的属性

自定义某个属性的一种简单方法是将它定义为一个`property`。例如，下面的代码定义了一个`property`，增加对一个属性简单的类型检查：
```python
class Person:
    def __init__(self, first_name):
        self.first_name = first_name

    # Getter function
    @property
    def first_name(self):
        return self._first_name

    # Setter function
    @first_name.setter
    def first_name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._first_name = value

    # Deleter function (optional)
    @first_name.deleter
    def first_name(self):
        raise AttributeError("Can't delete attribute")
```
上述代码中有三个相关联的方法，这三个方法的名字都必须一样。第一个方法是一个`getter`函数，它使得`first_name`成为一个属性。其他两个方法给`first_name`属性添加了`setter`和`deleter`函数。需要强调的是只有在`first_name`属性被创建后，后面的两个装饰器`@first_name.setter`和`@first_name.deleter`才能被定义。

`property`的一个关键特征是它看上去跟普通的`attribute`没什么两样，但是访问它的时候会自动触发`getter`、`setter`和`deleter`方法。例如：

```python
>>> a = Person('Guido')
>>> a.first_name # Calls the getter
'Guido'
>>> a.first_name = 42 # Calls the setter
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    File "prop.py", line 14, in first_name
        raise TypeError('Expected a string')
TypeError: Expected a string
>>> del a.first_name
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
AttributeError: can't delete attribute
```
在实现一个`property`时候，底层数据(如果有的话)仍然需要存储在某个地方。因此，在get和set方法中，你会看到对`_first_name`属性的操作，这也是实际数据保存的地方。另外，你可能还会问为什么`__init__()`方法中设置了`self.first_name`而不是`self._first_name`。在这个例子中，我们创建一个`property`的目的就是在设置`attribute`的时候进行检查。因此，你可能想在初始化的时候也进行这种类型检查。通过设置`self.first_name`，自动调用`setter`方法，这个方法里面会进行参数的检查，否则就是直接访问`self._first_name`了。

还能在已存在的`get`和`set`方法基础上定义`property`。
```python
class Person:
    def __init__(self, first_name):
        self.set_first_name(first_name)

    # Getter function
    def get_first_name(self):
        return self._first_name

    # Setter function
    def set_first_name(self, value):
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._first_name = value

    # Deleter function (optional)
    def del_first_name(self):
        raise AttributeError("Can't delete attribute")

    # Make a property from existing get/set methods
    name = property(get_first_name, set_first_name, del_first_name)
```
####调用父类方法
```python
class A:
    def spam(self):
        print('A.spam')

class B(A):
    def spam(self):
        print('B.spam')
        super().spam()  # Call parent spam()
```

`super()`函数的一个常见用法是在`__init__()`方法中确保父类被正确的初始化了：

```python
class A:
    def __init__(self):
        self.x = 0

class B(A):
    def __init__(self):
        super().__init__()
        self.y = 1
```
####定义接口或者抽象基类
使用`abs`模块可以很轻松的定义抽象基类:
```python
from abc import ABCMeta, abstractmethod

class IStream(metaclass=ABCMeta):
    @abstractmethod
    def read(self, maxbytes=-1):
        pass

    @abstractmethod
    def write(self, data):
        pass
```
抽象类的一个特点是它不能直接被实例化，比如你想像下面这样做是不行的:
```python
a = IStream() # TypeError: Can't instantiate abstract class
                # IStream with abstract methods read, write
```
抽象类的目的就是让别的类继承它并实现特定的抽象方法：
```python
class SocketStream(IStream):
    def read(self, maxbytes=-1):
        pass

    def write(self, data):
        pass
```
抽象基类的一个主要用途是在代码中检查某些类是否为特定类型，实现了特定接口：
```python
def serialize(obj, stream):
    if not isinstance(stream, IStream):
        raise TypeError('Expected an IStream')
    pass
```
除了继承这种方式外，还可以通过注册方式来让某个类实现抽象基类：
```python
import io

# Register the built-in I/O classes as supporting our interface
IStream.register(io.IOBase)

# Open a normal file and type check
f = open('foo.txt')
isinstance(f, IStream) # Returns True
```
`@abstractmethod`还能注解静态方法、类方法和`properties`。你只需保证这个注解紧靠在函数定义前即可：

```python
class A(metaclass=ABCMeta):
    @property
    @abstractmethod
    def name(self):
        pass

    @name.setter
    @abstractmethod
    def name(self, value):
        pass

    @classmethod
    @abstractmethod
    def method1(cls):
        pass

    @staticmethod
    @abstractmethod
    def method2():
        pass
```
标准库中有很多用到抽象基类的地方。`collections`模块定义了很多跟容器和迭代器(序列、映射、集合等)有关的抽象基类。`numbers`库定义了跟数字对象(整数、浮点数、有理数等)有关的基类。`io`库定义了很多跟`I/O`操作相关的基类。

你可以使用预定义的抽象类来执行更通用的类型检查，例如：
```python
import collections

# Check if x is a sequence
if isinstance(x, collections.Sequence):
...

# Check if x is iterable
if isinstance(x, collections.Iterable):
...

# Check if x has a size
if isinstance(x, collections.Sized):
...

# Check if x is a mapping
if isinstance(x, collections.Mapping):
```
尽管`ABCs`可以让我们很方便的做类型检查，但是我们在代码中最好不要过多的使用它。因为`Python`的本质是一门动态编程语言，其目的就是给你更多灵活性，强制类型检查或让你代码变得更复杂，这样做无异于舍本求末。
#### 实现自定义容器
`collections`定义了很多抽象基类，当你想自定义容器类的时候它们会非常有用。比如你想让你的类支持迭代，那就让你的类继承`collections.Iterable`即可：
```python
import collections
class A(collections.Iterable):
    pass
```
不过你需要实现`collections.Iterable`所有的抽象方法，否则会报错:
```python
>>> a = A()
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
TypeError: Can't instantiate abstract class A with abstract methods __iter__
```
`collections`中很多抽象类会为一些常见容器操作提供默认的实现，这样一来你只需要实现那些你最感兴趣的方法即可。假设你的类继承自`collections.MutableSequence`，如下：
```python
class Items(collections.MutableSequence):
    def __init__(self, initial=None):
        self._items = list(initial) if initial is not None else []

    # Required sequence methods
    def __getitem__(self, index):
        print('Getting:', index)
        return self._items[index]

    def __setitem__(self, index, value):
        print('Setting:', index, value)
        self._items[index] = value

    def __delitem__(self, index):
        print('Deleting:', index)
        del self._items[index]

    def insert(self, index, value):
        print('Inserting:', index, value)
        self._items.insert(index, value)

    def __len__(self):
        print('Len')
        return len(self._items)
```
#### 在类中定义多个构造器

为了实现多个构造器，你需要使用到类方法。例如：
```python
import time

class Date:
    """方法一：使用类方法"""
    # Primary constructor
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    # Alternate constructor
    @classmethod
    def today(cls):
        t = time.localtime()
        return cls(t.tm_year, t.tm_mon, t.tm_mday)
```
直接调用类方法即可，下面是使用示例：
```python
a = Date(2012, 12, 21) # Primary
b = Date.today() # Alternate
```
类方法的一个主要用途就是定义多个构造器。它接受一个`class`作为第一个参数`(cls)`。你应该注意到了这个类被用来创建并返回最终的实例。在继承时也能工作的很好：
```python
class NewDate(Date):
    pass

c = Date.today() # Creates an instance of Date (cls=Date)
d = NewDate.today() # Creates an instance of NewDate (cls=NewDate)
```
####创建不调用init方法的实例
可以通过`__new__()`方法创建一个未初始化的实例。例如考虑如下这个类：

```python
class Date:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
```
下面演示如何不调用`__init__()`方法来创建这个Date实例：
```python
>>> d = Date.__new__(Date)
>>> d
<__main__.Date object at 0x1006716d0>
>>> d.year
Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
AttributeError: 'Date' object has no attribute 'year'
>>>
```
结果可以看到，这个`Date`实例的属性`year`还不存在，所以你需要手动初始化：
```python
>>> data = {'year':2012, 'month':8, 'day':29}
>>> for key, value in data.items():
...     setattr(d, key, value)
...
>>> d.year
2012
>>> d.month
8
>>>
```
####实现状态对象或者状态机
你想实现一个状态机或者是在不同状态下执行操作的对象，但是又不想在代码中出现太多的条件判断语句。那么如下是最好的实现方式
```python

class Connection1:
    """新方案——对每个状态定义一个类"""

    def __init__(self):
        self.new_state(ClosedConnectionState)

    def new_state(self, newstate):
        self._state = newstate
        # Delegate to the state class

    def read(self):
        return self._state.read(self)

    def write(self, data):
        return self._state.write(self, data)

    def open(self):
        return self._state.open(self)

    def close(self):
        return self._state.close(self)


# Connection state base class
class ConnectionState:
    @staticmethod
    def read(conn):
        raise NotImplementedError()

    @staticmethod
    def write(conn, data):
        raise NotImplementedError()

    @staticmethod
    def open(conn):
        raise NotImplementedError()

    @staticmethod
    def close(conn):
        raise NotImplementedError()


# Implementation of different states
class ClosedConnectionState(ConnectionState):
    @staticmethod
    def read(conn):
        raise RuntimeError('Not open')

    @staticmethod
    def write(conn, data):
        raise RuntimeError('Not open')

    @staticmethod
    def open(conn):
        conn.new_state(OpenConnectionState)

    @staticmethod
    def close(conn):
        raise RuntimeError('Already closed')


class OpenConnectionState(ConnectionState):
    @staticmethod
    def read(conn):
        print('reading')

    @staticmethod
    def write(conn, data):
        print('writing')

    @staticmethod
    def open(conn):
        raise RuntimeError('Already open')

    @staticmethod
    def close(conn):
        conn.new_state(ClosedConnectionState)


c = Connection1()
print(c._state)
c.open()
print(c._state)
c.read()
print(c._state)
c.write('Hello')
print(c._state)
c.close()
print(c._state)#### 通过字符串调用对象方法
```
#### 让类支持比较操作

Python类对每个比较操作都需要实现一个特殊方法来支持。例如为了支持>=操作符，你需要定义一个`__ge__()`方法。尽管定义一个方法没什么问题，但如果要你实现所有可能的比较方法那就有点烦人了。

装饰器`functools.total_ordering`就是用来简化这个处理的。使用它来装饰一个来，你只需定义一个`__eq__()`方法，外加其他方法`(__lt__, __le__, __gt__, or __ge__)`中的一个即可。然后装饰器会自动为你填充其它比较方法。
```python
from functools import total_ordering

class Room:
    def __init__(self, name, length, width):
        self.name = name
        self.length = length
        self.width = width
        self.square_feet = self.length * self.width

@total_ordering
class House:
    def __init__(self, name, style):
        self.name = name
        self.style = style
        self.rooms = list()

    @property
    def living_space_footage(self):
        return sum(r.square_feet for r in self.rooms)

    def add_room(self, room):
        self.rooms.append(room)

    def __str__(self):
        return '{}: {} square foot {}'.format(self.name,
                self.living_space_footage,
                self.style)

    def __eq__(self, other):
        return self.living_space_footage == other.living_space_footage

    def __lt__(self, other):
        return self.living_space_footage < other.living_space_footage
```
这里我们只是给`House`类定义了两个方法：`__eq__()`和`__lt__()`，它就能支持所有的比较操作。
```python
# Build a few houses, and add rooms to them
h1 = House('h1', 'Cape')
h1.add_room(Room('Master Bedroom', 14, 21))
h1.add_room(Room('Living Room', 18, 20))
h1.add_room(Room('Kitchen', 12, 16))
h1.add_room(Room('Office', 12, 12))
h2 = House('h2', 'Ranch')
h2.add_room(Room('Master Bedroom', 14, 21))
h2.add_room(Room('Living Room', 18, 20))
h2.add_room(Room('Kitchen', 12, 16))
h3 = House('h3', 'Split')
h3.add_room(Room('Master Bedroom', 14, 21))
h3.add_room(Room('Living Room', 18, 20))
h3.add_room(Room('Office', 12, 16))
h3.add_room(Room('Kitchen', 15, 17))
houses = [h1, h2, h3]
print('Is h1 bigger than h2?', h1 > h2) # prints True
print('Is h2 smaller than h3?', h2 < h3) # prints True
print('Is h2 greater than or equal to h1?', h2 >= h1) # Prints False
print('Which one is biggest?', max(houses)) # Prints 'h3: 1101-square-foot Split'
print('Which is smallest?', min(houses)) # Prints 'h2: 846-square-foot Ranch'
```

####创建缓存实例
在创建一个类的对象时，如果之前使用同样参数创建过这个对象，你想返回它的缓存引用。为了达到这样的效果，你需要使用一个和类本身分开的工厂函数，例如：
```python
# The class in question
class Spam:
    def __init__(self, name):
        self.name = name

# Caching support
import weakref
_spam_cache = weakref.WeakValueDictionary()
def get_spam(name):
    if name not in _spam_cache:
        s = Spam(name)
        _spam_cache[name] = s
    else:
        s = _spam_cache[name]
    return s
```
对于大部分程序而已，这里代码已经够用了。不过还是有一些更高级的实现值得了解下。首先是这里使用到了一个全局变量，并且工厂函数跟类放在一块。我们可以通过将缓存代码放到一个单独的缓存管理器中：
```python
import weakref

class CachedSpamManager:
    def __init__(self):
        self._cache = weakref.WeakValueDictionary()

    def get_spam(self, name):
        if name not in self._cache:
            s = Spam(name)
            self._cache[name] = s
        else:
            s = self._cache[name]
        return s

    def clear(self):
            self._cache.clear()

class Spam:
    manager = CachedSpamManager()
    def __init__(self, name):
        self.name = name

    def get_spam(name):
        return Spam.manager.get_spam(name)
```
但是这样暴露了类的实例化给用户，如果想阻止用户实例化该类的话，可以通过如下方法解决
```python
# ------------------------最后的修正方案------------------------
class CachedSpamManager2:
    def __init__(self):
        self._cache = weakref.WeakValueDictionary()

    def get_spam(self, name):
        if name not in self._cache:
            temp = Spam3._new(name)  # Modified creation
            self._cache[name] = temp
        else:
            temp = self._cache[name]
        return temp

    def clear(self):
            self._cache.clear()

class Spam3:
    def __init__(self, *args, **kwargs):
        raise RuntimeError("Can't instantiate directly")

    # Alternate constructor
    @classmethod
    def _new(cls, name):
        self = cls.__new__(cls)
        self.name = name
        return self
```
参考资料
【1】[A Byte of Python3][id]
【2】[python3-cookbook](http://python3-cookbook.readthedocs.org/zh_CN/latest/c01/p08_calculating_with_dict.html)
[id]:http://code.google.com/p/proden/downloads/detail?name=A%20Byte%20of%20Python3(%E4%B8%AD%E6%96%87%E7%89%88).pdf
