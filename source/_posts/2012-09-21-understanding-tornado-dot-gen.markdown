---
layout: post
title: "理解 tornado.gen"
date: 2012-09-21 01:56
comments: true
categories: [Python, Tornado]
---

Tornado 通过 `@asynchronous` decorator 来实现异步请求，但使用的时候必须将 request handler 和 callback 分离开，`tornado.gen` 模块可以帮助我们在一个函数里完成这两个工作。下面是官方的一个例子：

{% codeblock lang:python %}
class GenAsyncHandler(RequestHandler):
    @asynchronous
    @gen.engine
    def get(self):
        http_client = AsyncHTTPClient()
        response = yield gen.Task(http_client.fetch, "http://example.com")
        do_something_with_response(response)
        self.render("template.html")
{% endcodeblock %}

这里用到了两个 decorator 稍显复杂，第一个 `@asynchronous` 会首先被执行，它的主要工作就是将 `RequestHandler` 的 `_auto_finish` 属性置为 `false`，如下：

{% codeblock web.py https://github.com/facebook/tornado/blob/master/tornado/web.py#L1116 download %}
def asynchronous(method):
    @functools.wraps(method)
    def wrapper(self, *args, **kwargs):
        if self.application._wsgi:
            raise Exception("@asynchronous is not supported for WSGI apps")
        self._auto_finish = False
        with stack_context.ExceptionStackContext(
            self._stack_context_handle_exception):
            return method(self, *args, **kwargs)
    return wrapper
{% endcodeblock %}

接着就是最重要的 `@gen.engine`，这里充分利用了 generator 的各种特性，首先来看 `@gen.engine` 的实现（我删减了部分代码以简化理解）：

{% codeblock gen.py https://github.com/facebook/tornado/blob/master/tornado/gen.py#L91 download %}
def engine(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        gen = func(*args, **kwargs)
        if isinstance(gen, types.GeneratorType):
            runner = Runner(gen)
            runner.run()
            return
    return wrapper
{% endcodeblock %}

局部变量 `gen` 代表第一段代码里的 `get` 函数，因为 `get` 包含了 `yield` 语句，因此成为了一个 generator。注意这里 `get` 并没有被执行，只是赋给了 `gen`。接下来是运行 `Runner` 对象的 `run` 函数。在理解 `run` 之前需要知道 generator 是通过调用 `next()` 或者 `send()` 来启动，启动之后会在遇到 `yield` 的地方 hold 住，然后将 `yield` 后面的语句的返回值返回给调用者，generator 此时即处于暂停运行状态，所有上下文都会保存。再次调用 `next()` 或 `send()` 便会恢复 generator 的运行，如果不再遇到 `yield` 语句就会抛出 `StopIteration` 异常。在恢复运行的同时 `yield` 语句本身会有返回值，如果是通过调用 `next()` 来恢复的，那么返回值永远是 `None`，而如果是通过 `send()` 则返回值取决于传给 `send()` 的参数。更多关于 generator 的说明请参考[官方文档](http://docs.python.org/reference/expressions.html#yield-expressions)。

结合第一段的示例代码，可以想到 `run` 干的工作可能就是启动 generator，然后获得 `gen.Task` 对象并调用 `http_client.fetch` 函数，等回调回来之后恢复 generator 的运行，最后将回调的返回值通过 `send()` 赋给 `response`。下面是我简化后的代码。

{% codeblock gen.py https://github.com/facebook/tornado/blob/master/tornado/gen.py#L322 download %}
def run(self):
    while True:
        if not self.yield_point.is_ready():
            return
        next = self.yield_point.get_result()
        try:
            yielded = self.gen.send(next)
        except StopIteration:
            return
        if isinstance(yielded, YieldPoint):
            self.yield_point = yielded
            self.yield_point.start(self)
{% endcodeblock %}

第 3 行检查回调是否完成，第一次运行 `run` 总是会返回 `True`。第 5 行获取回调的返回值，同样的第一次运行返回的是 `None`。将 `None` 传给 `send()` 启动 generator，`yielded` 即是 `gen.Task` 对象，第 12 行调用 `start` 开始运行我们真正需要运行的函数，对应到示例代码就是 `http_client.fetch` 函数，同时将 `Runner` 的 `result_callback` 作为回调函数。如下：

{% codeblock gen.py https://github.com/facebook/tornado/blob/master/tornado/gen.py#L374 download %}
def result_callback(self, key):
    def inner(*args, **kwargs):
        if kwargs or len(args) > 1:
            result = Arguments(args, kwargs)
        elif args:
            result = args[0]
        else:
            result = None
        self.results[key] = result
        self.run()
    return inner
{% endcodeblock %}

在得到回调返回值之后再次调用 `run`，通过 `get_result` 获取返回值，最后将返回值返回赋给 `response`，继续 request handler 的代码流程。
