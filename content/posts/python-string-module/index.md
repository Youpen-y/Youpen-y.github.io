---
title: "Python string module"
date: 2026-02-11T23:42:53+08:00
draft: false
tags: [python]
comments: true
---
`Python` 标准库中的 `string` 模块提供了一些与字符串处理相关的常量、类和辅助函数。用于使用预定义字符集（如所有字母、数字等）、自定义格式化规则、使用简单的模板替换字符串内容等场景。

---
#### 字符串常量

| 常量 | 说明 |
|---|---|
| `string.ascii_letters` | `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| `string.ascii_lowercase` | `abcdefghijklmnopqrstuvwxyz` |
| `string.ascii_uppercase` | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| `string.digits` | `0123456789` |
| `string.octdigits` | `01234567` |
| `string.hexdigits` | `0123456789abcdefABCDEF` |
| `string.punctuation` | <code>!"#$%&\'()*+,-./:;<=>?@[\\]^_`{\|}~</code> |
| `string.printable` | <code>0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&\'()*+,-./:;<=>?@[\\]^_`{\|}~ \t\n\r\x0b\x0c</code> |
---

#### 类
1. `Formatter` -- 自定义字符串格式化类
Python 最常使用的字符串格式化方式是 `str.format()` 或 f-strings。在 CPython 中，它们由 C 代码直接实现，性能极高。但这两种方式的格式化规则是完全固定的，无法胜任需要自定义字段来源、限制可访问属性以及实现领域专用格式规则的场景。
`string.Formatter`是对同一底层 C 格式化引擎的 Python 封装，将字符串格式化过程拆成多个可重写步骤，每步骤暴露对应的 `get_value()`, `format_field()`, `parse()` 关键钩子方法，使得可以通过继承 `Formatter` 类来创建定制的格式化行为。更多可参见 [User-Defined Formatting](https://peps.python.org/pep-3101/#user-defined-formatting)

```python
class Formatter(builtins.object)

|   format(self, format_string, /, *args, **kwargs)
|       -- 外部调用入口，等价于 str.format()
|          内部调用 vformat() 执行真正的格式化流程
|
|       示例:
|       Formatter().format("Hello {name}", name="Alice")
|       -> "Hello Alice"
|
|   vformat(self, format_string, args, kwargs)
|       -- 格式化引擎核心入口
|          负责：解析模板 → 查找字段 → 应用格式说明 → 拼接结果
|          可重写以完全接管整个流程
|
|   parse(self, format_string)
|       -- 将模板字符串拆解为四元组序列:
|          (literal_text, field_name, format_spec, conversion)
|
|       示例:
|       for literal, field, spec, conv in Formatter().parse("Hi {name!r:>10}"):
|           print(literal, field, spec, conv)
|       # "Hi "   name   >10   r
|
|   get_field(self, field_name, args, kwargs)
|       -- 解析字段路径（点/索引访问）
|          返回 (value, used_key)
|          默认支持：
|              {user.name}
|              {data[0]}
|
|   get_value(self, key, args, kwargs)
|       -- 决定字段“根值”的来源
|          当遇到 {x} 时被调用:
|          "{x}".format(x=1) -> get_value("x", (), {"x": 1})
|
|   format_field(self, value, format_spec)
|       -- 将字段值根据格式说明转换为字符串
|          等价于：format(value, format_spec)
|
|       示例:
|       format_field(3.14159, ".2f") -> "3.14"
|
|   convert_field(self, value, conversion)
|       -- 处理转换标志 !r !s !a
|
|       示例:
|       "{x!r}" -> convert_field(x, "r") -> repr(x)
|
|   check_unused_args(self, used_args, args, kwargs)
|       -- 校验是否存在未使用的参数
|          默认行为：抛出异常
|          可重写以放宽或禁用该检查
```

示例：新增一个格式说明符（format spec） `:upper`
```python
class UpperFormatter(Formatter):
    def format_field(self, value, spec):
        if spec == "upper":
            return str(value).upper()
        return super().format_field(value, spec)

fmt = UpperFormatter()
print(fmt.format("{name:upper}", name="alice")) # ALICE
```

示例：新增一个转换标志（conversion） `!u`
```python
class UpperConvFormatter(Formatter):
    def convert_field(self, value, conv):
        if conv == "u":     # 使用 {name!u}
            return str(value).upper()
        return super().convert_field(value, conv)

fmt = UpperConvFormatter()
print(fmt.format("{name!u}", name="alice")) # ALICE
```
> 转换标志（conversion） vs 格式说明符（format spec）
>
> | 语法 | 名称 | 用途 | 钩子 | 
> |---|---|---|---|
> | `{!r}` | 转换标志 | repr/str/ascii | `convert_field(value, conversion)` |
> | `{x:upper}` | 格式说明 | 对齐、宽度、数字格式等  | `format_field(value, format_spec)` |


2. `Template`` -- 模板字符串类
`string.Template` 提供一种简单、安全的字符串替换机制，使用 `$VAR_NAME` 的方式进行替换。
```python
from string import Template

t = Template("Hello, $name!")
print(t.substitute(name="Alice"))
```
---
#### 辅助函数
`string.capwords(s, sep=None)`: 将字符串中每个单词的首字母大写，而其他字母小写，可以指定单词分隔符。
```python
import string

s = "hello world, python!"
print(string.capwords(s))   # Hello World, Python!
```