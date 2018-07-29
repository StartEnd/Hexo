---
title: 'Python 迭代器,生成器'
date: 2018-5-15
updated: 2018-5-18
categories:
  - python
tags:
  - 迭代器
  - 生成器
abbrlink: f74d6115
---

## Python 迭代器,生成器

在程序设计中，通常会有 loop、iterate、traversal 和 recursion 等概念，他们各自的含义如下：

* 循环（loop），指的是在满足条件的情况下，重复执行同一段代码。比如 Python 中的 while 语句。
* 迭代（iterate），指的是按照某种顺序逐个访问列表中的每一项。比如 Python 中的 for 语句。
* 递归（recursion），指的是一个函数不断调用自身的行为。比如，以编程方式输出著名的斐波纳契数列。
* 遍历（traversal），指的是按照一定的规则访问树形结构中的每个节点，而且每个节点都只访问一次。

> 在其他语言中，for 与 while 都用于循环，而 Python 则没有类似其他语言的 for 循环，只有 while 来实现循环。在 Python 中， for 用来实现迭代，它的结构是 for ... in ...，其在迭代时会产生迭代器，实际是将可迭代对象转换成迭代器，再重复调用 next() 方法实现的。

在了解Python的数据结构时，容器(container)、可迭代对象(iterable)、迭代器(iterator)、生成器(generator)、列表/集合/字典推导式(list,set,dict comprehension)众多概念参杂在一起，难免让初学者一头雾水，我将用一篇文章试图将这些概念以及它们之间的关系捋清楚。
![](http://pizisong.qiniudn.com/2018-07-20-15320518011493.png?imageView2/2/h/640)


## 容器(container)

容器是一种把多个元素组织在一起的数据结构，容器中的元素可以逐个地迭代获取，可以用in, not in关键字判断元素是否包含在容器中。通常这类数据结构把所有的元素存储在内存中（也有一些特例，并不是所有的元素都放在内存，比如迭代器和生成器对象）在Python中，常见的容器对象有：

* list, deque, ....
* set, frozensets, ....
* dict, defaultdict, OrderedDict, Counter, ....
* tuple, namedtuple, …
* str

容器比较容易理解，因为你就可以把它看作是一个盒子、一栋房子、一个柜子，里面可以塞任何东西。从技术角度来说，当它可以用来询问某个元素是否包含在其中时，那么这个对象就可以认为是一个容器，比如 list，set，tuples都是容器对象：

```py
>>> assert 1 in [1, 2, 3]      # lists
>>> s = 'foobar'
>>> assert 'b' in s
>>> >>> d = {1: 'foo', 2: 'bar', 3: 'qux'}
>>> assert 1 in d
>>> assert 'foo' not in d  # 'foo' 不是dict中的元素
```

尽管绝大多数容器都提供了某种方式来获取其中的每一个元素，但这并不是容器本身提供的能力，而是可迭代对象赋予了容器这种能力，当然并不是所有的容器都是可迭代的，比如：Bloom filter，虽然Bloom filter可以用来检测某个元素是否包含在容器中，但是并不能从容器中获取其中的每一个值，因为Bloom filter压根就没把元素存储在容器中，而是通过一个散列函数映射成一个值保存在数组中。

## 可迭代对象(iterable)
刚才说过，很多容器都是可迭代对象，此外还有更多的对象同样也是可迭代对象，比如处于打开状态的files，sockets等等。但凡是可以返回一个迭代器的对象都可称之为可迭代对象，听起来可能有点困惑，没关系，先看一个例子

```py
>>> x = [1, 2, 3]
>>> y = iter(x)
>>> z = iter(x)
>>> next(y)
1
>>> next(y)
2
>>> next(z)
1
>>> type(x)
<class 'list'>
>>> type(y)
<class 'list_iterator'>
```
这里x是一个可迭代对象，可迭代对象和容器一样是一种通俗的叫法，并不是指某种具体的数据类型，list是可迭代对象，dict是可迭代对象，set也是可迭代对象。y和z是两个独立的迭代器，迭代器内部持有一个状态，该状态用于记录当前迭代所在的位置，以方便下次迭代的时候获取正确的元素。迭代器有一种具体的迭代器类型，比如list_iterator，set_iterator。可迭代对象实现了__iter__方法，该方法返回一个迭代器对象。

当运行代码:

```py
x = [1, 2, 3]
for elem in x:
    ...
```
实际执行情况是:
![](http://pizisong.qiniudn.com/2018-07-20-15320528705479.png?imageView2/2/h/640)

反编译该段代码，你可以看到解释器显示地调用GET_ITER指令，相当于调用iter(x)，FOR_ITER指令就是调用next()方法，不断地获取迭代器中的下一个元素，但是你没法直接从指令中看出来，因为他被解释器优化过了。

```py
>>> import dis
>>> x = [1, 2, 3]
>>> dis.dis('for _ in x: pass')
  1           0 SETUP_LOOP              12 (to 14)
              2 LOAD_NAME                0 (x)
              4 GET_ITER
        >>    6 FOR_ITER                 4 (to 12)
              8 STORE_NAME               1 (_)
             10 JUMP_ABSOLUTE            6
        >>   12 POP_BLOCK
        >>   14 LOAD_CONST               0 (None)
             16 RETURN_VALUE
```

可迭代对象具有`__iter__` 方法，用于返回一个迭代器，或者定义了 `__getitem__` 方法，可以按 index 索引的对象（并且能够在没有值时抛出一个 IndexError 异常），因此，可迭代对象就是能够通过它得到一个迭代器的对象。所以，可迭代对象都可以通过调用内建的 iter() 方法返回一个迭代器。

可迭代器对象具有如下的特性：

* 可以 for 循环: for i in iterable；
* 可以按 index 索引的对象，也就是定义了 `__getitem__` 方法，比如 list,str;
* 定义了`__iter__ `方法，可以随意返回；
* 可以调用 iter(obj) 的对象，并且返回一个iterator。

可以通过`isinstance(obj, collections.Iterable) `来判断对象是否为可迭代对象。

### 迭代器对象(iterator)

迭代器对象是一个含有 next (Python 2) 或者 `__next__` (Python 3) 方法的对象。如果需要自定义迭代器，则需要满足如下迭代器协议：

* 定义了`__iter__` 方法，但是必须返回自身
* 定义了 next 方法,在 python3.x 是 `__next__`。用来返回下一个值，并且当没有数据了，抛出 StopIteration
* 可以保持当前的状态

可以通过 isinstance(obj, collections.Iterator) 来判断对象是否为迭代器。

**(用一句来总结就是，一个实现了 `__iter__()` 方法的对象是可迭代的，一个实现了 next() 方法的对象则是迭代器。**

那么什么迭代器呢？它是一个带状态的对象，他能在你调用`next()`方法的时候返回容器中的下一个值，任何实现了`__iter__`和`__next__()`（python2中实现next()）方法的对象都是迭代器，`__iter__`返回迭代器自身，`__next__`返回容器中的下一个值，如果容器中没有更多元素了，则抛出StopIteration异常，至于它们到底是如何实现的这并不重要。



所以，迭代器就是实现了工厂模式的对象，它在你每次你询问要下一个值的时候给你返回。有很多关于迭代器的例子，比如itertools函数返回的都是迭代器对象。

生成无限序列：

```py
>>> from itertools import count
>>> counter = count(start=13)
>>> next(counter)
13
>>> next(counter)
14
```

从一个有限序列中生成无限序列：

```py
>>> from itertools import cycle
>>> colors = cycle(['red', 'white', 'blue'])
>>> next(colors)
'red'
>>> next(colors)
'white'
>>> next(colors)
'blue'
>>> next(colors)
'red'
```

从无限的序列中生成有限序列：

```py
>>> from itertools import islice
>>> colors = cycle(['red', 'white', 'blue'])  # infinite
>>> limited = islice(colors, 0, 4)            # finite
>>> for x in limited:                         
...     print(x)
red
white
blue
red
```

为了更直观地感受迭代器内部的执行过程，我们自定义一个迭代器，以斐波那契数列为例：

```py
class Fib:
    def __init__(self):
        self.prev = 0
        self.curr = 1

    def __iter__(self):
        return self

    def __next__(self):
        value = self.curr
        self.curr += self.prev
        self.prev = value
        return value

>>> f = Fib()
>>> list(islice(f, 0, 10))
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```

Fib既是一个可迭代对象（因为它实现了__iter__方法），又是一个迭代器（因为实现了__next__方法）。实例变量prev和curr用户维护迭代器内部的状态。每次调用next()方法的时候做两件事：

1. 为下一次调用next()方法修改状态
2. 为当前这次调用生成返回结果
迭代器就像一个懒加载的工厂，等到有人需要的时候才给它生成值返回，没调用的时候就处于休眠状态等待下一次调用。

### 可迭代对象和迭代器的分开自定义
使用迭代器时,需要注意的一点是:
> 迭代器只能迭代一次，每次调用调用 next() 方法就会向前一步，不能回退，只能如过河的卒子，不断向前。另外，迭代器也不适合在多线程环境中对可变集合使用。

```py
class MyRange(object):
    def __init__(self, n):
        self.idx = 0
        self.n = n

    def __iter__(self):
        return self

    def next(self):
        if self.idx < self.n:
            val = self.idx
            self.idx += 1
            return val
        else:
            raise StopIteration()

myRange = MyRange(3)

print [i for i in myRange]
print [i for i in myRange]
```
运行结果

```
True
[0, 1, 2]
[]
```

也就是说一个迭代器无法多次使用。为了解决这个问题，可以将可迭代对象和迭代器分开自定义：

```py
class Zrange:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return ZrangeIterator(self.n)

class ZrangeIterator:
    def __init__(self, n):
        self.i = 0
        self.n = n

    def __iter__(self):
        return self

    def next(self):
        if self.i < self.n:
            i = self.i
            self.i += 1
            return i
        else:
            raise StopIteration()

zrange = Zrange(3)
print zrange is iter(zrange)

print [i for i in zrange]
print [i for i in zrange]
```

### 生成器(generator)
生成器算得上是Python语言中最吸引人的特性之一，生成器其实是一种特殊的迭代器，不过这种迭代器更加优雅。它不需要再像上面的类一样写`__iter__()`和`__next__()`方法了，只需要一个yiled关键字。 生成器一定是迭代器（反之不成立），因此任何生成器也是以一种懒加载的模式生成值。用生成器来实现斐波那契数列的例子是：

```py
def fib():
    prev, curr = 0, 1
    while True:
        yield curr
        prev, curr = curr, curr + prev

>>> f = fib()
>>> list(islice(f, 0, 10))
[1, 1, 2, 3, 5, 8, 13, 21, 34, 55]
```
fib就是一个普通的python函数，它特殊的地方在于函数体中没有return关键字，函数的返回值是一个生成器对象。当执行f=fib()返回的是一个生成器对象，此时函数体中的代码并不会执行，只有显示或隐示地调用next的时候才会真正执行里面的代码。

生成器在Python中是一个非常强大的编程结构，可以用更少地中间变量写流式代码，此外，相比其它容器对象它更能节省内存和CPU，当然它可以用更少的代码来实现相似的功能。现在就可以动手重构你的代码了，但凡看到类似：

```py
def something():
    result = []
    for ... in ...:
        result.append(x)
    return result
```

都可以用生成器函数来替代

```py
def iter_something():
    for ... in ...:
        yield x
```

#### 生成器表达式(generator expression)
生成器表达式是列表推倒式的生成器版本，看起来像列表推导式，但是它返回的是一个生成器对象而不是列表对象。
生成器表达式有一个特点，就是惰性计算。

惰性计算这个特点很有用

> 惰性计算想像成水龙头，需要的时候打开，接完水了关掉，这时候数据流就暂停了，再需要的时候再打开水龙头，这时候数据仍是接着输出，不需要从头开始循环.

```py
def add(s, x):
    return s + x

def gen():
    for  i in range(4):
        yield i

base = gen()
for n in [1, 10]:
    base = (add(i, n) for i in base)

print list(base)
```

结果输出是 [20,21,22,23]。很多人可能会想不明白，这里确实也很难理解，主要是因为生成器惰性计算的原因。生成器 base 在最后 list(base) 时被检索，此时生成器被赋值并开始计算。但此时 base 生成器一共被创建了三次，而且 n=10，这里注意 add(i+n) 绑定的是 n 这个变量而不是它当时的值（因为生成器在被检索时被赋值）。这样，首先通过 gen() 得到 (0, 1, 2, 3)，然后是第一次循环得到 (10 + 0, 10 + 1, 10 + 2, 10 +3)，最后是第二次循环得到 (10 + 10, 11 + 10, 12 + 10, 13 + 10)。

这里可以用管道的思路来理解这个例子。首先 gen() 函数是第一个生成器，下一个是第一次循环的 base = (add(i, n) for i in base)， 最后一个生成器是第二次循环的 base = (add(i, n) for i in base)。这样就相当于三个管道依次连接，但是水(数据)还没有流过，现在到了 list(base)，就相当于驱动器，打开了水的开关，这时候，按照管道的顺序，由第一个产生一个数据，yield 0，然后第一个管道关闭。之后传递给第二个管道就是第一次循环,此时执行了add(0, 10)，然后水继续流，到第二次循环，再执行add(10, 10),此时到管道尾巴了，此时产生了第一个数据20，然后第一个管道再开放：yield 1， 流程跟上面的一样，依次产生21,22,23；直到没有数据。

上面的例子就类似与下面这样的简单写法：

```py
def gen():
    for i in range(4):
        yield i  #  第一个管道

base = (add(i, 10) for i in base) #  第二个管道
base = (add(i, 10) for i in base) #  第三个管道

list(base) #  开关驱动器
```


```py
>>> a = (x*x for x in range(10))
>>> a
<generator object <genexpr> at 0x401f08>
>>> sum(a)
285
```

### 迭代器节省内存的真相

迭代器能够很好的节能内存，这是因为它不必一次性将数据全部加载到内存中，而是在需要的时候产生一个结果。这在数据量的时候是非常有用的。

```py
l = range(100000000)

for i in l:
    pass
 ```
 这个例子只是去遍历一个超大的列表，并没有做其他任何多余的操作。但是，在我的机器上运行时内存已经被占满，而且系统几乎卡死。但如果使用迭代器结果就不一样了：
 
 ```py
 l = xrange(100000000)

for i in l:
    pass
 ```
 这样修改后程序只在 4s 左右就执行完成了，并且对系统没有任何影响。

但是，需要注意的一点是：并非所有的迭代器都能很好的节省内存。例如：

```py
l = range(100000000)

for i in iter(l):
    pass
 ```
 这里虽然在迭代时把列表转化成了迭代器，但是所有的数据已经放在内存中，并不会带来任何的效益。

所以，并不是所有的迭代器都能节省内存，只有那些在需要时才产生一个结果的迭代器才有节省内存的特性。

### 迭代器速度

有听说迭代器的速度比列表、元组等容器对象快，这个说法太绝对，我也没有找到一个有力的证据证明迭代器总是比容器对象快。但在某些情况下，迭代器的效率确实会高些，容器对象需要把所有的数据加载到内存中，而读写内存也要消耗时间。因此，在某些情况下，速度会比较快。但是，要明白一点，不是所有的迭代器都能节省内存。

说到速度，这里提一点：在 python 中， map和列表解析要比手动的 for 运行更快，而且更加精简、优雅。因为他们的迭代在解析器内部是以 C 语言的速度执行的，而不是以手动 python 代码执行的，特别对于较大的数据集合，这也是使用 map 函数和列表解析的一个主要的性能优点。但需要注意的一点是，在 python3 之后，map 函数不再返回一个 list，而是返回一个迭代器。

## 总结
* 容器是一系列元素的集合，str、list、set、dict、file、sockets对象都可以看作是容器，容器都可以被迭代（用在for，while等语句中），因此他们被称为可迭代对象。
* 可迭代对象实现了`__iter__`方法，该方法返回一个迭代器对象。
* 迭代器持有一个内部状态的字段，用于记录下次迭代返回值，它实现了`__next__`和`__iter__`方法，迭代器不会一次性把所有元素加载到内存，而是需要的时候才生成返回结果。
* 生成器是一种特殊的迭代器，它的返回值不是通过return而是用yield。


