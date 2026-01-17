---
title: "Python inspect 模块"
date: 2026-01-12T20:15:01+08:00
draft: false
tags: [python, debugging]
comments: true
---
`inspect` 是 Python 标准库中用于获取实时对象信息的模块，这里的实时对象可以是模块(modules)、类(classes)、函数(methods)、方法(functions)、回溯(tracebacks)、帧对象(frame objects)和代码对象(code objects)。 该模块可以用来检查类的的内容，检索方法的源代码，提取并格式化函数的参数列表，或者获取显示详细回溯(traceback)所需的所有信息。

> Tip: Python 函数（function）与方法（method）的区别体现在**定义位置**，**调用方式**以及是否绑定对象上下文上。函数是独立的功能模块，不依赖于类或对象，而方法是类中定义的功能模块，会根据调用方式自动绑定到对象上。当类中定义的功能模块绑定到类上时则是函数：意味着我们可以像调用普通函数一样，使用 `T.func()` 调用绑定到类 `T` 上的函数 `func()`，不过需要考虑是否包含 `self` 参数。
> 
> ```python
> # 示例：一个普通函数
> def func(t):
>     print(f"func(): value = {t.value}")
> 
> class T:
>     def __init__(self, value):
>         self.value = value
>
>     # 可复用代码块
>     def func(self):
>         print(f"T.func(): value = {self.value}")
>
>     # 可通过 T.func2() 像普通函数一样调用
>     def func2():
>         print(f"Hello, World")
>
> # 创建实例
> t = T(42)
>
> # 1. 普通函数调用（显式传入实例）
> func(t)     # func(): value = 42
>
> # 2. 绑定方法调用（Python 自动把 t 作为 self 传入）
> t.func()    # T.func(): value = 42
>
> # 3. 非绑定方法调用（从类上取方法，手动传入实例）
> T.func(t)   # T.func(): value = 42
>
> # inspect.isfunction(T.func)    # True
> # inspect.ismethod(T().func)    # True   
> ```

### 公共接口与核心类
`inspect` 模块主要提供四种服务：类型检查、获取源代码、检查类与函数以及检查解释器堆栈。下面是通过 `help(inspect)` 获得的帮助文档：
```
ispackage(), ismodule(), isclass(), ismethod(), isfunction(), 
isgeneratorfunction(), isasyncgenfunction(), iscoroutinefunction(),
isgenerator(), istraceback(), isabstract(),
isframe(), iscode(), isbuiltin(), isroutine(),
isasyncgen(), iscoroutine(), isawaitable(),...
-- 判断函数，检查对象类型

// inspect.ismodule(math)   # True
// inspect.isclass(str)     # True
// inspect.isroutine(len)   # True, function OR builtin

getmembers(object[, predicate]) -- 返回满足给定条件对象的成员，以 name-value 形式呈现，其中 predicate 是上述的 is* 判断函数，用以过滤成员

getfile() -- 获取定义对象的文件路径

getsourcefile() -- 返回对象的源文件路径（仅限 `.py`）

getsource() -- 返回对象的源代码文本

getdoc(), getcomments() -- 获取对象文档

getmodule() -- 获取对象源模块，返回 None 表示无法确定

getclasstree() -- 返回嵌套结构，表示继承层级

getargvalues(), getcallargs() -- 获取函数参数信息
getfullargspec() -- 同上，支持 Python 3 特征

formatargvalues() -- 函数参数和值，格式化成类似源码调用形式的字符串

getouterframes() -- 从某个桟帧开始，往外列出调用链

currentframe() -- 返回当前执行点的桟帧对象

stack(), trace() -- 获取当前线程的完整调用桟

signature() -- 统一解析所有 callable 的调用签名

get_annotations() -- 安全获取类型注解
```

`inspect` 使用 `Signature` 对象来表示可调用对象及其返回注解（callable object and its return annotation）。可以使用 `signature()` 方法来检索 `Signature` 对象。

```python
>>> help(inspect.Signature)
class Signature(builtins.object)
    Signature(parameters=None, *, return_annotation=Signature.empty)
    Signature 对象用来表示函数的整体签名。为函数的每一个参数保存一个 Parameter 对象，以及特定于函数的信息。

    * parameters: OrderedDict
        参数名到 Parameter 对象的有序映射

        示例： 
        def f(a, b=1, *, c: int): ...

        Signature
         |--- Parameter(name='a', kind=POSITIONAL_OR_KEYWORD)
         |--- Parameter(name='b', kind=POSITIONAL_OR_KEYWORD, default=1)
         |--- Parameter(name='c', kind=KEYWORD_ONLY, annotation=int)

    * return_annotation: object
        函数的返回类型的注解（如果指定）。如果无返回类型注解，该属性设置为 Signature.empty 。


    bind(*args, **kwargs) -> BoundArguments
        按照函数的签名规则，把一组实参”绑定“到参数上

    bind_partial(*args, **kwargs)
        允许“只绑定一部分参数”
```

### `inspect` 常见使用示例：

1. 查看函数定义、签名
```python
import inspect

def example_func():
    """This is an example function."""
    pass

def example_func2(a, b=2, *args, **kwargs) -> None:
    pass

print(inspect.getsource(example_func)) # 将打印函数定义

print(inspect.signature(example_func2))
# (a, b=2, *args, **kwargs) -> None
```

2. 用于轻量级日志
```python
import inspect

def log(msg: str):
    # log_caller()    <- currentframe().f_back
    #   |--- log()    <- currentframe()
    frame = inspect.currentframe().f_back

    # 解析原始 frame 对象为 filename, lineno, function, code_context, index
    info = inspect.getframeinfo(frame)

    # <调用 log() 的文件>:<调用 log() 的行号> - <msg>
    print(f"{info.filename}:{info.lineno} - {msg})
``` 