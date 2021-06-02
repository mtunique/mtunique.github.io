---
title: python实现各基本数据结构（1）——离散和线性
date: 2016-01-27T14:30:27+08:00
category: python
tags:
- python
- algorithm
---

# 离散
## 集合（set）

Operation | Average case | Worst Case | notes
:---------| :------------|:---------------|:----------
x in s|O(1)|O(n)|
Union s|t|O(len(s)+len(t))|
Intersection s&t|O(min(len(s), len(t))|O(len(s) * len(t))|replace "min" with "max" if t is not a set
Multiple intersection s1&s2&..&sn||(n-1)*O(l) where l is max(len(s1),..,len(sn))|
Difference s-t|O(len(s))||
s.difference_update(t)|O(len(t))||
Symmetric Difference s^t|O(len(s))|O(len(s) * len(t))|
s.symmetric_difference_update(t)|O(len(t))|O(len(t) * len(s))|

<!-- more -->

## dict
Operation | Average case | Worst Case
:---------| :------------|:----------
Copy|O(n)|O(n)
Get Item|O(1)|O(n)
Set Item|O(1)|O(n)
Delete Item|O(1)|O(n)
Iteration|O(n)|O(n)

## defaultdict（collections.defaultdict）
defaultdict在处理不存在的key时和dict不同：若某key不存在则用调用default_factory并将其值作为该key的value。例如
```python
from collections import defaultdict
defdict = defaultdict(list)
defdict['boo'].append(1)
```


# 线性数据结构
## 队列
### 栈（list）
|Operation | Average Case |Amortized Worst Case
|:---------| :------------|:---------------
|Copy|O(n)|O(n)
|Append|O(1)|O(1)
|Insert|O(n)|O(n)
|Get Item|O(1)|O(1)
|Set Item|O(1)|O(1)
|Delete Item|O(n)|O(n)
|Iteration|O(n)|O(n)
|Get Slice|O(k)|O(k)
|Del Slice|O(n)|O(n)
|Set Slice|O(k+n)|O(k+n)
|Extend|O(k)|O(k)
|[Sort][1]|O(n log n)|O(n log n)
|Multiply|O(nk)|O(nk)
|x in s|O(n)
|min(s), max(s)|O(n)
|Get Length|O(1)|O(1)

### array
相对list它固定了数据类型，可以节约内存。

### 队列 （Queue，LifoQueue）
Queue为先进先出（fifo）
LifoQueue为后进先出（lifo）

```python
q = Queue.LifoQueue()
q.put(1)
q.put(2)
q.put(3)
while not q.empty():
    print q.get(),

q = Queue.Queue()
q.put(1)
q.put(2)
q.put(3)
while not q.empty():
    print q.get(),
```
output:
```
3 2 1 1 2 3
```

### 双向队列 double-ended（collections.deque）

线程安全，且deque的`popleft()`, `appendleft(item)`比list的`pop(0)`和`insert(0, v)`要快的多。

|Operation | Average Case |Amortized Worst Case
|:---------| :------------|:---------------
|Copy|O(n)|O(n)
|append|O(1)|O(1)
|appendleft|O(1)|O(1)
|pop|O(1)|O(1)
|popleft|O(1)|O(1)
|extend|O(k)|O(k)
|extendleft|O(k)|O(k)
|rotate|O(k)|O(k)
|remove|O(n)|O(n)


### priority queue（Queue.PriorityQueue）
优先队列是利用 heepq实现的
```python
from Queue import PriorityQueue

q = PriorityQueue()
q.put(2)
q.put(1)
q.put(3)
while not q.empty():
	print q.get(),

q.put((2,'b'))
q.put((1,'c'))
q.put((3,'a'))
while not q.empty():
    print q.get(),


class A(object):
    def __init__(self, priority, v):
        self.priority = priority
        self.value = v

    def __cmp__(self, other):
        return cmp(self.priority, other.priority)

q.put(A(2, 'B'))
q.put(A(1, 'C'))
q.put(A(3, 'A'))

while not q.empty():
    next_item = q.get()
    print next_item.value,
```
output:
```
1 2 3 (1, 'c') (2, 'b') (3, 'a') C B A
```

[1]: http://svn.python.org/projects/python/trunk/Objects/listsort.txt
