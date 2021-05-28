---
title: 【Tornado笔记】tornado.web.Application
date: 2014-11-28T13:30:27+08:00
categories:
- python
tags:
- python
---

从tornado的 Hello,world 开始分析tornado的源码

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

很容易可以看出，通过继承`RequestHandler`类定义自己的处理类，来处理请求。`Application`类的对象来处理URI的路由（将URI`r"/"`于处理类`MainHandler`组成tuple，关联起来）。

<!-- more -->
### tornado.web.Application类

#### 一、init

简化版代码：

```python
def __init__(self, handlers=None, default_host="", transforms=None,
             **settings):
    if transforms is None:
        self.transforms = []
        if settings.get("compress_response") or settings.get("gzip"):
            self.transforms.append(GZipContentEncoding)
    else:
        self.transforms = transforms
    ......
    self.ui_modules = {'linkify': _linkify,
                       'xsrf_form_html': _xsrf_form_html,
                       'Template': TemplateModule,
                       }
    self.ui_methods = {}
    self._load_ui_modules(settings.get("ui_modules", {}))
    self._load_ui_methods(settings.get("ui_methods", {}))

    if self.settings.get("static_path"):
        ......
    if handlers:
        self.add_handlers(".*$", handlers)

    if self.settings.get('debug'):
        self.settings.setdefault('autoreload', True)
        ......
    # Automatically reload modified modules
    if self.settings.get('autoreload'):
        from tornado import autoreload
        autoreload.start()
```

参数handlers是一个list，list里每个object是一个URLSpec的对象tuple。tuple可以是二到四个element，分别是URI的正则、handler类、用于初始化URLSpec的kwargs、handler的name。 （下面add_handlers详细说明）

参数settings是一个dict，所有settings的具体用法

1.  初始化transforms（HTTP传输压缩等，默认GZipContentEncoding 和 ChunkedTransferEncoding 。也可以自己实现，需要实现 transform_first_chunk 和 transform_chunk 接口，RequestHandler 中的 flush 调用，剖析RequestHandler时详细介绍），UI模块

2.  通过settings的值来初始化静态文件处理Handler，包括：

    *   static_path
    *   static_url_prefix
    *   static_handler_class
    *   static_handler_args
    *   static_hash_cache

3.  初始化其他settings

4.  调用add_handlers方法添加handlers。 5.加载自动重新加载模块（当检测到代码被修改后重构启动）

#### 二、add_handle

```python
def add_handlers(self, host_pattern, host_handlers):
    if not host_pattern.endswith("$"):
        host_pattern += "$"
    handlers = []
    if self.handlers and self.handlers[-1][0].pattern == '.*$':
        self.handlers.insert(-1, (re.compile(host_pattern), handlers))
    else:
        self.handlers.append((re.compile(host_pattern), handlers))

    for spec in host_handlers:
        if isinstance(spec, (tuple, list)):
            assert len(spec) in (2, 3, 4)
            spec = URLSpec(*spec)
        handlers.append(spec)
        if spec.name:
            if spec.name in self.named_handlers:
                app_log.warning(
                    "Multiple handlers named %s; replacing previous value",
                    spec.name)
            self.named_handlers[spec.name] = spec
```

将`host_pattern`和`handlers`，组成tuple加到`self.handlers`的末尾但是在匹配所有域名的tuple前。

由`spec = URLSpec(*spec)`易看出初始化Application的时候的第一个参数存的tuple是用来初始化URLSpec的所以参数顺序应该和URLSpec要求的一样（`def __init__(self, pattern, handler, kwargs=None, name=None)`）。

用过第四个参数name来构造反响代理，储存在`Application`的`named_handlers`（dict）里。

hello world里调用了`Application.listen`和`tornado.ioloop.IOLoop.instance().start()`（以后会详细介绍`IOLoop`），来真正启动。

#### 三、listen

```python
def listen(self, port, address="", **kwargs):
        from tornado.httpserver import HTTPServer
        server = HTTPServer(self, **kwargs)
        server.listen(port, address)
```

实例化一个`HTTPServer`，将`Application`绑定上去。

当有请求的时候`HTTPServer`将会调用`Application.start_request`来将`application`和`connection`绑定在一起初始化一个`_RequestDispatcher`的对象，由其来处理请求的路由，从而利用已经通过`add_handler`建立的路由规则。
