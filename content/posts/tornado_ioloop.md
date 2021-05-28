---
title: 【Tornado笔记】tornado.web.IOLoop和tornado.util.Configurable
date: 2014-11-28T13:30:27+08:00
categories:
- python
tags:
- python
---

[上一篇文章](https://www.mtunique.com/tornado_application.html)tornado的 Hello,
world 还没有分析完 还差最后一句

```python
import tornado.ioloop
import tornado.web

class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world")

application = tornado.web.Application([
    (r"/", MainHandler),
])

if __name__ == "__main__":
    application.listen(8888)
    tornado.ioloop.IOLoop.instance().start()
```

IOLoop就是整个tornado区别于其他框架的最关键的地方。

<!-- more -->
#### 一、`tornado.ioloop.IOLoop.instance`
```python
@staticmethod
def instance():
    if not hasattr(IOLoop, "_instance"):
        with IOLoop._instance_lock:
            if not hasattr(IOLoop, "_instance"):
                # New instance after double check
                IOLoop._instance = IOLoop()
    return IOLoop._instance
```

每个tornado进程都会有一个全局的`IOLoop`实例，这个方法就是用来获得这个实例的。

`with IOLoop._instance_lock`保证了创建的这个实例的过程是线程安全的。

`IOLoop`继承了`tornado.util.Configurable`。`Configurable`
（是一个“抽象”类）实现了`__new__`，所以将自动调用了`__new__`生成`IOLoop`的实例。

#### 二、`tornado.util.Configurable.__new__()`

```python
def __new__(cls, **kwargs):
    base = cls.configurable_base()
    args = {}
    if cls is base:
        impl = cls.configured_class()
        if base.__impl_kwargs:
            args.update(base.__impl_kwargs)
    else:
        impl = cls
    args.update(kwargs)
    instance = super(Configurable, cls).__new__(impl)
    # initialize vs __init__ chosen for compatibility with AsyncHTTPClient
    # singleton magic.  If we get rid of that we can switch to __init__
    # here too.
    instance.initialize(**args)
    return instance</pre></div>
```
