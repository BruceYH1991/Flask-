# *Flask* 请求处理

## 1. 接收请求入口

``` python
# venv/lib/python3.8/site-packages/flask/app.py:2459
class Flask(_PackageBoundObject):
    def __call__(self, environ, start_response):
        return self.wsgi_app(environ, start_response)
```



## 2. 整体流程

``` python
# venv/lib/python3.8/site-packages/flask/app.py:2416
class Flask(_PackageBoundObject):
    def wsgi_app(self, environ, start_response):
        # ctx: 请求相关上下文 是一个 RequestContext 对象
        ctx = self.request_context(environ)
        error = None
        try:
            try:
                # 将ctx压入栈(当前上下文)
                ctx.push()
                # 处理请求入口
                response = self.full_dispatch_request()
            except Exception as e:
                error = e
                response = self.handle_exception(e)
            except:  # noqa: B001
                error = sys.exc_info()[1]
                raise
            # 返回请求结果
            return response(environ, start_response)
        finally:
            if self.should_ignore_error(error):
                error = None
            # 清理上下文
            ctx.auto_pop(error)
```

> **请求处理流程可以概括成5步**
>
> 1. 封装请求对象
>
>     ``` python
>     ctx = self.request_context(environ)
>     ```
>
> 2. 将请求对象加入到上下文中
>
>     ``` python
>     ctx.push()
>     ```
>
> 3. 处理请求的到 response
>
>     ``` python
>     response = self.full_dispatch_request()
>     ```
>
> 4. 返回请求结果
>
>     ```python
>     return response(environ, start_response)
>     ```
>
> 5. 清理上下文
>
>     ```python
>     ctx.auto_pop(error)
>     ```
>
>     
>



## 3. RequestContext 对象

``` python
# flask.app.Flask.request_context
def request_context(self, environ):
        return RequestContext(self, environ)
```

> + 主要属性以及方法
>
>     ``` python
>     # flask.ctx.RequestContext
>     class RequestContext(object):
>         def __init__(self, app, environ, request=None, session=None):
>             self.app = app
>             if request is None:
>                 # 通过 app.request_class(environ) 来封装 request 对象
>                 # app.request_class = flask.wrappers.Request
>                 request = app.request_class(environ)
>             self.request = request
>             # 省略部分代码
>     
>             # 
>             self._implicit_app_ctx_stack = []
>     
>             # 请求后置处理器, 扩展功能用
>             self._after_request_functions = []
>     
>         @property
>         def g(self):
>             return _app_ctx_stack.top.g
>     
>         @g.setter
>         def g(self, value):
>             _app_ctx_stack.top.g = value
>     
>         # 省略部分代码
>     
>         # 最重要的 push 将请求上下文绑定到当前上下文 _request_ctx_stack 中
>         # _request_ctx_stack = LocalStack()  (后面在看)
>         def push(self):
>             top = _request_ctx_stack.top
>             if top is not None and top.preserved:
>                 top.pop(top._preserved_exc)
>     
>             # 拿到当前的app上下文
>             app_ctx = _app_ctx_stack.top
>             if app_ctx is None or app_ctx.app != self.app:
>                 app_ctx = self.app.app_context()
>                 app_ctx.push()
>                 self._implicit_app_ctx_stack.append(app_ctx)
>             else:
>                 self._implicit_app_ctx_stack.append(None)
>     
>             if hasattr(sys, "exc_clear"):
>                 sys.exc_clear()
>     		
>             # 将当前的 RequestContext 加到 _request_ctx_stack 中
>             _request_ctx_stack.push(self)
>     
>             # 省略部分session相关代码
>     
>         def pop(self, exc=_sentinel):
>             # 略
>             pass
>     
>         def auto_pop(self, exc):
>             if self.request.environ.get("flask._preserve_context") or (
>                 exc is not None and self.app.preserve_context_on_exception
>             ):
>                 self.preserved = True
>                 self._preserved_exc = exc
>             else:
>                 self.pop(exc)
>     
>         def __enter__(self):
>             self.push()
>             return self
>     
>         def __exit__(self, exc_type, exc_value, tb):
>             self.auto_pop(exc_value)
>     
>             if BROKEN_PYPY_CTXMGR_EXIT and exc_type is not None:
>                 reraise(exc_type, exc_value, tb)
>     ```
>
>     

## 4. LocalStack 对象

``` python
# venv/lib/python3.8/site-packages/flask/globals.py:57
# _request_ctx_stack 是全局变量 是在Flask项目初始话的时候执行的
# 所以里面一定有个标识来标注不同线程下的 RequestContext 对象
_request_ctx_stack = LocalStack()
_app_ctx_stack = LocalStack()
```

> **可以看出 请求上下文 和 app上下文的结构是一样的 都是一个 LocalStack 对象    **
>
> **_request_ctx_stack.push(self: RequestContext) 实际上就是调用 LocalStack.push(obj)**

```python
# werkzeug.local.LocalStack
class LocalStack(object):
    def __init__(self):
        # 可以看出实际上 LocalStack 就是封装了 Local 对象
        self._local = Local()

    def __release_local__(self):
        self._local.__release_local__()

    @property
    def __ident_func__(self):
        return self._local.__ident_func__

    @__ident_func__.setter
    def __ident_func__(self, value):
        object.__setattr__(self._local, "__ident_func__", value)

    def __call__(self):
        def _lookup():
            rv = self.top
            if rv is None:
                raise RuntimeError("object unbound")
            return rv

        return LocalProxy(_lookup)

    def push(self, obj):
        # 实际上数据都是存储在 Local 对象的 "stack" 属性上的 默认是个 空list
        # 当执行 push 的时候就是在执行 List(stack).append(obj)
        rv = getattr(self._local, "stack", None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        stack = getattr(self._local, "stack", None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        # 只是拿到 _local.stack 最后一个元素的引用, 并没有将这个元素弹出栈
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None

def release_local(local):
    local.__release_local__()
```



## 5. Local对象

```python
# werkzeug.local.Local
class Local(object):
    # 只能动态绑定这两个属性
    __slots__ = ("__storage__", "__ident_func__")

    def __init__(self):
        # 初始化 __storage__ 和 __ident_func__
        # __storage__是存储数据的地方, 一个 嵌套dict 结构 -- {key(线程id): {name: value}}
        object.__setattr__(self, "__storage__", {})
        object.__setattr__(self, "__ident_func__", get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        # 当前线程id
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```



![](/home/bruce/my_code/Flask-/请求调用流程.png)



## 6. 处理request

```python
#flask.app.Flask.wsgi_app
class Flask(_PackageBoundObject):
    # 省略部分代码
    def wsgi_app(self, environ, start_response):
        # 省略部分代码
		ctx.push()
        # 处理i请求并返回结果
		response = self.full_dispatch_request()
       # 省略部分代码
    
    def full_dispatch_request(self):
        # 调用每个app实例的前置处理器,每个app实例只会触发一次
        self.try_trigger_before_first_request_functions()
        try:
            request_started.send(self)
            # 调用当前请求的前置处理器
            rv = self.preprocess_request()
            if rv is None:
                # 调用视图函数
                rv = self.dispatch_request()
        except Exception as e:
            rv = self.handle_user_exception(e)
        # 执行后置处理器并返回 response
        return self.finalize_request(rv)
    
    def try_trigger_before_first_request_functions(self):
        if self._got_first_request:
            return
        with self._before_request_lock:
            if self._got_first_request:
                return
            for func in self.before_first_request_funcs:
                # 函数不能带参数
                func()
            self._got_first_request = True

    def preprocess_request(self):
        bp = _request_ctx_stack.top.request.blueprint

        funcs = self.url_value_preprocessors.get(None, ())
        if bp is not None and bp in self.url_value_preprocessors:
            funcs = chain(funcs, self.url_value_preprocessors[bp])
        for func in funcs:
            func(request.endpoint, request.view_args)

        funcs = self.before_request_funcs.get(None, ())
        if bp is not None and bp in self.before_request_funcs:
            funcs = chain(funcs, self.before_request_funcs[bp])
        for func in funcs:
            # 函数不能带参数
            rv = func()
            if rv is not None:
                return rv

    def dispatch_request(self):
        req = _request_ctx_stack.top.request
        if req.routing_exception is not None:
            self.raise_routing_exception(req)
        rule = req.url_rule
        # if we provide automatic options for this URL and the
        # request came with the OPTIONS method, reply automatically
        if (
            getattr(rule, "provide_automatic_options", False)
            and req.method == "OPTIONS"
        ):
            return self.make_default_options_response()
        # 执行视图函数
        return self.view_functions[rule.endpoint](**req.view_args)
    

    def finalize_request(self, rv, from_error_handler=False):
        response = self.make_response(rv)
        try:
            # 执行后置处理器
            response = self.process_response(response)
            request_finished.send(self, response=response)
        except Exception:
            if not from_error_handler:
                raise
            self.logger.exception(
                "Request finalizing failed with an error while handling an error"
            )
        return response
    
    def process_response(self, response):
        ctx = _request_ctx_stack.top
        bp = ctx.request.blueprint
        funcs = ctx._after_request_functions
        if bp is not None and bp in self.after_request_funcs:
            funcs = chain(funcs, reversed(self.after_request_funcs[bp]))
        if None in self.after_request_funcs:
            funcs = chain(funcs, reversed(self.after_request_funcs[None]))
        for handler in funcs:
            response = handler(response)
        if not self.session_interface.is_null_session(ctx.session):
            self.session_interface.save_session(self, ctx.session, response)
        return response
```





## 7. 清理上下文

**入口:**

​	**ctx.auto_pop(error)**

```python
# flask.ctx.RequestContext.auto_pop
class RequestContext(object):
	def auto_pop(self, exc):
        if self.request.environ.get("flask._preserve_context") or (
            exc is not None and self.app.preserve_context_on_exception
        ):
            self.preserved = True
            self._preserved_exc = exc
        else:
            self.pop(exc)
```

```python
# flask.ctx.RequestContext.pop
class RequestContext(object):
    def pop(self, exc=_sentinel):
        app_ctx = self._implicit_app_ctx_stack.pop()

        try:
            clear_request = False
            if not self._implicit_app_ctx_stack:
                self.preserved = False
                self._preserved_exc = None
                if exc is _sentinel:
                    exc = sys.exc_info()[1]
                # 执行没有执行完的方法,包括bluePrint(蓝图)里的方法
                self.app.do_teardown_request(exc)

                if hasattr(sys, "exc_clear"):
                    sys.exc_clear()

                request_close = getattr(self.request, "close", None)
                if request_close is not None:
                    request_close()
                clear_request = True
        finally:
            # 将request弹出栈
            rv = _request_ctx_stack.pop()

            if clear_request:
                rv.request.environ["werkzeug.request"] = None

            # Get rid of the app as well if necessary.
            if app_ctx is not None:
                app_ctx.pop(exc)

            assert rv is self, "Popped wrong request context. (%r instead of %r)" % (
                rv,
                self,
            )
```









