# Python 装饰器详解笔记

装饰器（Decorator）是 Python 中一种强大的语法特性，用于在不修改原函数代码的前提下，动态地为函数或类添加额外功能。本文将从**原理**、**参数传入**和**函数返回值处理**三个方面详细讲解装饰器的使用与实现。

---

## 一、装饰器的原理

### 1.1 函数是一等公民

在 Python 中，函数是“一等公民”（first-class object），意味着：

- 函数可以赋值给变量；
- 函数可以作为参数传递；
- 函数可以作为其他函数的返回值。

```python
def greet():
    return "Hello!"

say = greet  # 函数赋值
print(say())  # 输出: Hello!
```

### 1.2 闭包（Closure）

装饰器依赖于**闭包**机制：内部函数可以访问外部函数的作用域中的变量，即使外部函数已经执行完毕。

```python
def outer(x):
    def inner(y):
        return x + y
    return inner

add5 = outer(5)
print(add5(3))  # 输出: 8
```

### 1.3 装饰器的本质

装饰器本质上是一个**接收函数作为参数并返回一个新函数的高阶函数**。

```python
def my_decorator(func):
    def wrapper():
        print("Before function call")
        func()
        print("After function call")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

# 等价于: say_hello = my_decorator(say_hello)
say_hello()
```

输出：

```
Before function call
Hello!
After function call
```

> `@my_decorator` 是语法糖，等价于 `say_hello = my_decorator(say_hello)`

---

## 二、参数传入：如何让装饰器支持任意参数

原始装饰器只适用于无参函数。为了通用性，需要使用 `*args` 和 `**kwargs`。

### 2.1 支持任意位置参数和关键字参数

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print("Calling function with args:", args, "and kwargs:", kwargs)
        result = func(*args, **kwargs)
        print("Function returned:", result)
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

print(add(3, 5))
```

输出：

```
Calling function with args: (3, 5) and kwargs: {}
Function returned: 8
8
```

### 2.2 带参数的装饰器（装饰器工厂）

有时我们需要向装饰器本身传参，例如 `@retry(times=3)`。这需要**三层嵌套函数**：

```python
def retry(times):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for i in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"Attempt {i+1} failed: {e}")
            raise Exception(f"Failed after {times} attempts")
        return wrapper
    return decorator

@retry(times=3)
def might_fail():
    import random
    if random.random() < 0.7:
        raise ValueError("Random failure")
    return "Success!"

print(might_fail())
```

结构说明：

- `retry(times)`：接收装饰器参数，返回真正的装饰器 `decorator`
- `decorator(func)`：接收被装饰函数，返回包装函数 `wrapper`
- `wrapper(*args, **kwargs)`：实际执行逻辑

---

## 三、函数返回值的处理

装饰器必须正确处理并返回原函数的返回值，否则会破坏函数行为。

### 3.1 必须显式返回原函数结果

错误示例（丢失返回值）：

```python
def bad_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before")
        func(*args, **kwargs)  # 没有 return！
        print("After")
    return wrapper

@bad_decorator
def get_data():
    return [1, 2, 3]

result = get_data()  # result 是 None！
```

正确做法：

```python
def good_decorator(func):
    def wrapper(*args, **kwargs):
        print("Before")
        result = func(*args, **kwargs)  # 保存返回值
        print("After")
        return result  # 返回原函数结果
    return wrapper
```

### 3.2 处理异常时也要注意返回值

如果装饰器中捕获异常并重试，需确保最终成功时返回正确值：

```python
def safe_call(func):
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            print(f"Error: {e}")
            return None  # 或抛出、返回默认值等
    return wrapper
```

---

## 四、补充：保留原函数元信息（functools.wraps）

装饰器会掩盖原函数的 `__name__`、`__doc__` 等属性。使用 `functools.wraps` 可修复：

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        """This is wrapper doc"""
        return func(*args, **kwargs)
    return wrapper

@my_decorator
def example():
    """Original function doc"""
    pass

print(example.__name__)  # 输出: example（而非 wrapper）
print(example.__doc__)   # 输出: Original function doc
```

---

## 总结

|方面|关键点|
|---|---|
|**原理**|函数是一等对象 + 闭包 + 高阶函数|
|**参数传入**|使用 `*args, **kwargs` 支持任意参数；带参装饰器需三层嵌套|
|**返回值处理**|必须 `return func(...)`，否则丢失结果|
|**最佳实践**|使用 `@functools.wraps` 保留元信息|

掌握这三方面，就能灵活编写和使用各种装饰器，提升代码复用性和可维护性。