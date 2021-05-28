---
title: 另一个Lambda表达式教程
date: 2015-01-24T14:30:00+08:00
categories:
- python
tags:
- python
---

有很多Python的[lambda](http://docs.python.org/tutorial/controlflow.html#lambda-forms)教程[1]。最近我偶然发现一个，真挺有用的。是Mike Driscoll在[Mouse VS Python](http://www.blog.pythonlibrary.org/) 博客上的[关于lambda的讨论](http://www.blog.pythonlibrary.org/2010/07/19/the-python-lambda/)) 。

```
当我刚开始学习Python，最容易困惑的概念之一，是lambda语句。我敢肯定，
其他新的程序员也对它很困惑...
```
<!-- more -->
Mike的讨论非常好：清晰，直接，且含有实用的示例。它帮助我终于领会了lambda，并导致我写的另一篇lambda教程。

一个用来构造函数的工具 基本上，Python的lambda是用于构造函数（或更精确地说，函数对象）的工具。这意味着，Python有两个构造函数的工具：def和lambda。

下面是一个例子。您可以以正常的方式用def构造一个函数，就像这样：

```python
def square_root(x): return math.sqrt(x)
```

或者你可以用lambda

```python
square_root = lambda x: math.sqrt(x)
```

下面是lambda的其他的一些有趣的例子：

```python
sum = lambda x, y:   x + y   #  def sum(x,y): return x + y
out = lambda   *x:   sys.stdout.write(" ".join(map(str,x)))
lambda event, name=button8.getLabel(): self.onButton(event, name)
```

lambda的好处在哪里？ 已经困扰我有很长一段时间的一个问题是：lambda的好处在哪里？为什么我们需要lambda？

答案是： 我们并不需要lambda，我们不用它一样可以做所有的事情。但是… 在特定的情况下，很是方便 - 它让编写代码更容易一些，而且编写的代码更整洁。

什么样的情况？ 好，其中一个情况是，我们需要一个简单的一次性功能：将被只使用一次函数。

通常，写函数有两个目的：(a)以减少代码重复（b）模块化代码。

*   如果你的应用程序在不同的地方包含重复的代码块，那么你就可以把代码拷贝到一个函数，给函数名，然后 – 使用该函数名 - 在代码中的不同位置调用它。
*   如果你有一个代码块执行一个明确的操作 - 但真的是冗长、粗糙、破坏程序的可读性，那么你可以把那么长的粗糙的所有代码变成一个函数。

但是，假设你需要创建一个函数，将只被使用一次 - 只从应用程序中的一个地方调用。好吧，首先，你不需要给函数的名称。它可以是“匿名的”。而且你可以把它定义在你想使用它的地方。这就是lambda是非常有用的时候。

但是，但是，但是…你会说。

*   首先是，为什么你想要一个只调用一次函数？排除原因（a）。
*   一个lambda的函数体只能包含单个表达式。这意味着，lambda表达式必须很短。排除了原因（b）。

创造一个短的匿名函数可能的原因是什么？

那么，考虑一下代码片段，使用lambda来定义一个Tkinter的GUI界面按钮的行为。 （这个例子是来自Mike的教程。）

```python
frame = tk.Frame(parent)
frame.pack()

btn22 = tk.Button(frame, 
        text="22", command=lambda: self.printNum(22))
btn22.pack(side=tk.LEFT)

btn44 = tk.Button(frame, 
        text="44", command=lambda: self.printNum(44))
btn44.pack(side=tk.LEFT)
```

这里要记住的一点是，tk.Button需要一个函数对象作为参数传递给该函数的参数。该函数对象将是它（按钮）点击按钮时调用的函数。基本上，该函数指定了点击该按钮时，GUI会做什么。

因此，我们必须通过函数参数传递一个函数对象到一个按钮。并注意 – 因为不同的按钮做不同的事情 - 我们需要为每个按钮对象提供不同的函数对象。每个函数将只使用一次。 所以，尽管我们可以这样写


```python
def __init__(self, parent):
    """Constructor"""
    frame = tk.Frame(parent)
    frame.pack()

    btn22 = tk.Button(frame, 
        text="22", command=self.buttonCmd22)
    btn22.pack(side=tk.LEFT)

    btn44 = tk.Button(frame, 
        text="44", command=self.buttonCmd44)
    btn44.pack(side=tk.LEFT)

def buttonCmd22(self):
    self.printNum(22)

def buttonCmd44(self):
    self.printNum(44)
```

这样写更容易（且更清楚）

```python
def __init__(self, parent):
    """Constructor"""
    frame = tk.Frame(parent)
    frame.pack()

    btn22 = tk.Button(frame, 
        text="22", command=lambda: self.printNum(22))
    btn22.pack(side=tk.LEFT)

    btn44 = tk.Button(frame, 
        text="44", command=lambda: self.printNum(44))
    btn44.pack(side=tk.LEFT)
```

当一个GUI程序有这样的代码，该按钮对象需要“call back”到被提供给作为其命令的函数对象。 因此，我们可以说，lambda的最常见的用途之一是在GUI框架，如Tkinter和wxPython中写“回调（callback）”，。

这一切似乎很简单。所以… 为什么lambda如此难以理解？ 我能想到四个原因:

首先Lambda难以理解，因为：一个lambda只能用一个表达式：什么是表达式？

很多人想知道这个问题的答案。如果你在Google上搜索了一下，你会看到很多的帖子，“在Python中，表达式和语句之间的区别是什么？”

一个很好的答案是，表达式返回（或计算结果为）值，而语句则没有。不幸的是，在Python中表达式也可以是一个语句，这种情况很容易造成糊涂。 – 赋值语句就像 A = B = 0，Python支持链式赋值。 （Python不是C）

很多情况下在当人们问这个问题时，他们真正想知道的是：什么样的情况下可以放入lambda，什么情况下不可以？ 而对于这个问题，我觉得遵循一些简单的规则就足够了。

*   如果它不返回一个值，它不是一个表达式，不能放入一个lambda。
*   如果你能想象它在赋值语句中放在等号的右边，那它是一个表达式，可以放进一个lambda。

利用这些规则意味着：

1.  赋值语句不能在lambda中使用。在Python中，赋值语句不返回任何东西，甚至没有None（null）。
2.  如数学运算，字符串操作，列表解析等都是一个lambda。
3.  函数调用是表达式。可以在lambda中放置函数调用，并将参数传递给该函数。这样就在一个新的匿名函数中封装了原函数调用（参数其他内容）。
4.  在Python3，print成了一个函数，所以在Python3+，print（…）可以在lambda中使用。
5.  即使函数是返回None，就像在Python3print函数，可以在一个lambda中使用。
6.  [条件表达式]，它是在Python2.5中引入，是表达式（而不是仅仅是一个语法不同的if / else语句）。它们返回一个值，并且可以在一个lambda使用。

```python
lambda: a if some_condition() else b
lambda x: ‘big’ if x > 100 else ‘small’
```

难以理解的第二个原因是：一个lambda只有一个表达式：为什么？为什么只有一个？为什么不能多表达式？为什么不能是语句？

对于一些开发人员来说，这个问题的意思是为什么Python的lambda语法如此怪异？对于其他人，尤其是那些有Lisp的背景的，这个问题是指为什么Python的lambda这么残废？为什么不像Lisp的lambda那么强大？

答案是很复杂，它涉及Python语法的“pythonicity”。lambda是一个相对较晚加入Python的。它加入的时候，Python语法已经固定下来了。在这种情况下，语法的lambda必须用“Pythonic”的方式硬塞进一个已经建立好的Python语法中。导致可以在lambda表达式上来完成一些事情有一定的局限性。

坦率地说，我仍然认为lambda语法看起来有点怪异。尽管那样，但是Guido解释了为什么lambda的语法是不会改变的。 Python不会成为Lisp。

难以理解的第三个原因是：：lambda通常被描述为一种工具，用于创建函数，但lambda语句中不含有返回语句。

在某种意义上，return语句隐含在lambda中。lambda规范必须包含只有一个表达式，表达式必须返回一个值，由lambda创建一个匿名函数隐式地返回表达式的返回值。这非常有意义。

还是 - 我想缺乏一个明确的return语句使得很难理解lambda，或者至少很难迅速理解。

难以理解的第四个原因是在lambda教程中通常会用作为创建匿名函数来引入lambda，其实最常见的lambda用途是用于创建匿名过程。

在编程的上古时期，我们就将子程序区分为两种不同的形式：过程和函数。过程是用来做事情的，并没有返回任何东西。函数是用于计算和返回值。函数和过程之间的差异已经成为一些编程语言的一部分了。在Pascal，例如，程序和函数是不同的关键字。

在大多数现代语言中，语言的语法中不再区分过程和函数。 例如Python的函数，可以像过程，函数，或两者兼而有之。（不是完全理想的）结果是一个Python函数总是被称为“函数”，即使它是本质上充当过程。

虽然过程和函数之间的区别已经基本消失的语言结构中，当思考有关程序如何工作的时候我们仍然时常用它。例如，当我读一个程序的源代码，并看到一些函数F，我揣摩F是做什么的。我经常可以把它归类到一个过程或函数 - 我会对自己说“F的目的是做这个的”，或“F的目的是计算和返回等这个和这个的”。

所以现在我想我们可以明白为什么lambda的许多解释是难以理解。 First of all, the Python language itself masks the distinction between a function and a procedure. 首先，Python语言本身模糊了函数和过程的区别。

第二，大多数教程介绍把lambda作为创建匿名函数的工具来介绍，其主要目的是要计算并返回结果。在大多数教程看到（这个包含）的第一个例子展示了如何编写一个lambda来返回值，x的平方根。

但是，这不是lambda最常用的方式，不是当他们在Google上搜索“python lambda教程”的时候要找的。对于lambda最常见的用途是创建匿名的过程，在GUI回调中使用。在这些用例中，我们不关心什么lambda返回什么，我们关心它做了什么。

这就解释了为什么典型的Python程序员难以理解大多数的lambda说明。因为他尝试学习如何编写一些GUI框架的代码：Tkinter，wxPython。运行这些lambda，想理解他们。Google“python lambda教程”。他发现那些以例子开始的教程是完全不适合他。

所以，如果你是这样的程序员 - 本教程是给你写的。我希望它能帮助到你。对不起，我们在本教程的结尾看到了这点，而不是开头。我们希望有一天，有人会写一个lambda教程，而不是以这种方式开头

*   lambda是一个用来构造匿名函数的工具

而以这样的句子开始：

*   lambda是一个用来构造回调的工具

所以你需要有它。另一个lambda教程。


