> 本笔记基于问答式探索整理，涵盖 Python 装饰器、继承机制、类型注解、抽象基类、模块导入等核心概念，适合系统性学习。

---

## 一、Python 模块导入机制

### 1. 导入范围由 `sys.path` 决定

Python 只能导入 **在 `sys.path` 列表中的目录** 下的 `.py` 文件或包（含 `__init__.py` 的目录）。

```python
import sys
print(sys.path)
```

**搜索顺序（优先级从高到低）**：

1. 当前脚本所在目录（或启动目录）
2. `PYTHONPATH` 环境变量指定的路径
3. 标准库路径
4. 第三方包安装路径（如 `site-packages`）

### 2. 如何导入任意位置的模块？

- **临时添加路径**（调试用）：
    
    ```python
    import sys
    sys.path.append('/path/to/your/module')
    import your_module
    ```
    
- **设置环境变量**：
    
    ```bash
    export PYTHONPATH="${PYTHONPATH}:/your/project"
    python script.py
    ```
    
- **推荐：打包安装**（专业项目）：
    
    ```bash
    pip install -e .  # 开发模式安装
    ```
    

> ✅ **总结**：Python 不会自动扫描全盘，必须显式将目标路径加入 `sys.path`。

---

## 二、装饰器（Decorator）详解

### 1. 什么是装饰器？

装饰器是 **修改或增强函数/方法行为** 的语法糖：

```python
@decorator
def func(): ...
# 等价于
def func(): ...
func = decorator(func)
```

### 2. 两类装饰器

|类型|运行时生效？|作用|示例|
|---|---|---|---|
|**功能性装饰器**|✅ 是|改变程序行为|`@staticmethod`, `@property`, `@lru_cache`|
|**元信息装饰器**|❌ 否|仅用于静态检查|`@override`, `@final`, `@overload`|

### 3. 对比示例

#### ✅ `@staticmethod`（运行时生效）

```python
class Math:
    @staticmethod
    def add(a, b): return a + b

Math.add(1, 2)  # ✅ 正常调用
# 若无 @staticmethod，则需 Math().add(1, 2)
```

#### ✅ `@property`（运行时生效）

```python
class Person:
    @property
    def name(self): return self._name

p = Person()
print(p.name)  # 像属性一样访问，无需 p.name()
```

#### ⚠️ `@override`（仅静态检查）

```python
from typing_extensions import override

class Parent:
    def method(self): ...

class Child(Parent):
    @override
    def method(self): ...  # mypy/IDE 检查是否真覆盖
```

- **运行时**：完全忽略，行为不变
- **开发期**：防止拼写错误或 API 变更导致的 bug

#### ✅ 自定义装饰器（运行时生效）

```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_call
def greet(): print("Hi")
greet()  # 输出: Calling greet \n Hi
```

> 💡 **判断法则**：去掉装饰器后，程序运行结果是否改变？
> 
> - 改变 → 功能性装饰器
> - 不变 → 元信息装饰器

---

## 三、类与继承机制

### 1. 方法调用自动传入 `self`

**只要通过实例调用普通方法，Python 就会自动传入 `self`（即实例本身）**，无论函数定义中是否写了 `self`。

#### ❌ 错误示例

```python
class MyClass:
    def greet():  # 没写 self
        print("Hello")

obj = MyClass()
obj.greet()  # TypeError: greet() takes 0 positional arguments but 1 was given
```

> 实际调用：`MyClass.greet(obj)` → 传了 1 个参数，但函数定义接收 0 个。

#### ✅ 正确写法

```python
class MyClass:
    def greet(self):  # 显式声明 self
        print("Hello")
```

#### 🆚 静态方法（不传 self）

```python
class MyClass:
    @staticmethod
    def greet():  # 不需要 self
        print("Hello")

MyClass.greet()  # ✅ 直接调用
```

### 2. 继承行为详解

#### ✅ 继承内容

子类继承父类的：

- 所有方法（实例方法、类方法、静态方法、特殊方法）
- 类属性
- **但不自动继承实例属性**（需调用父类 `__init__`）

#### ⚠️ 关键细节：`__init__` 与实例属性

```python
class Parent:
    def __init__(self):
        self.parent_attr = "Parent"

class Child(Parent):
    def __init__(self):
        self.child_attr = "Child"
        # 未调用 super().__init__()

c = Child()
print(c.child_attr)   # ✅ "Child"
print(c.parent_attr)  # ❌ AttributeError!
c.parent_method()     # ❌ 如果方法依赖 parent_attr
```

> ✅ **正确做法**：
> 
> ```python
> class Child(Parent):
>     def __init__(self):
>         super().__init__()  # 初始化父类属性
>         self.child_attr = "Child"
> ```

#### 📌 总结表

|子类 `__init__` 行为|父类实例属性|父类方法|能否安全调用父类方法？|
|---|---|---|---|
|无 `__init__`|✅ 自动初始化|✅|✅|
|有 `__init__` + `super()`|✅|✅|✅|
|有 `__init__` + 无 `super()`|❌|✅（存在）|❌（若方法依赖属性）|

> 💡 **黄金法则**：只要父类有 `__init__`，子类自定义 `__init__` 时就应调用 `super().__init__()`。

---

## 四、类型注解与现代 Python 函数定义

### 1. 带类型注解的函数定义

```python
def __init__(
    self,
    host: str = "0.0.0.0",
    port: Optional[int] = None,
    api_key: Optional[str] = None
) -> None:
```

#### 语法解析：

- `host: str = "..."`：参数 `host` 期望 `str` 类型，默认值 `"0.0.0.0"`
- `Optional[int]`：等价于 `Union[int, None]`，表示可为 `int` 或 `None`
- `-> None`：返回值类型注解（`__init__` 应返回 `None`）

> 💡 类型注解是 **开发期辅助工具**（mypy/IDE），**运行时无强制**。

#### 🆚 新旧写法对比

```python
# 现代写法（推荐）
def func(x: int = 0) -> str: ...

# 老式写法（功能相同，但无类型提示）
def func(x=0): ...
```

### 2. Python 3.10+ 简化语法

```python
def func(x: int | None = None) -> str | None: ...
# 替代 Optional[int]
```

---

## 五、抽象基类（ABC）与接口设计

### 1. 定义抽象基类

```python
import abc
from typing import Dict

class BasePolicy(abc.ABC):
    @abc.abstractmethod
    def infer(self, obs: Dict) -> Dict:
        """Infer actions from observations."""

    def reset(self) -> None:
        """Reset policy (optional)."""
        pass
```

#### 关键特性：

- **不能直接实例化**：`BasePolicy()` → `TypeError`
- **强制子类实现抽象方法**：若子类未实现 `infer`，实例化时报错
- **普通方法可选重写**：`reset` 有默认空实现

### 2. 子类实现要求

#### ✅ 正确子类

```python
class WebsocketClientPolicy(BasePolicy):
    def infer(self, obs: Dict) -> Dict:
        # 实现具体逻辑
        return {"actions": ...}
    
    # reset 可选重写
```

#### ❌ 错误子类

```python
class BadPolicy(BasePolicy):
    pass

BadPolicy()  # TypeError: Can't instantiate abstract class ... with abstract method infer
```

> 💡 **设计意图**：通过 ABC 定义“接口契约”，确保所有策略类提供统一 API。

---

## 六、综合案例：WebSocket 策略客户端

```python
class WebsocketClientPolicy(_base_policy.BasePolicy):
    def __init__(self, host="0.0.0.0", port=None, api_key=None):
        # 构造 URI
        self._uri = f"ws://{host}:{port}" if port else f"ws://{host}"
        self._api_key = api_key
        self._ws, self._server_metadata = self._wait_for_server()

    def _wait_for_server(self):
        while True:
            try:
                headers = {"Authorization": f"Api-Key {self._api_key}"} if self._api_key else None
                conn = websockets.sync.client.connect(self._uri, additional_headers=headers)
                metadata = msgpack_numpy.unpackb(conn.recv())
                return conn, metadata
            except ConnectionRefusedError:
                time.sleep(5)

    @override
    def infer(self, obs: Dict) -> Dict:
        data = self._packer.pack(obs)      # 序列化（支持 NumPy）
        self._ws.send(data)
        response = self._ws.recv()
        if isinstance(response, str):       # 错误以字符串形式返回
            raise RuntimeError(response)
        return msgpack_numpy.unpackb(response)  # 反序列化

    @override
    def reset(self) -> None:
        pass  # 状态由服务器管理
```

#### 技术亮点：

- **继承抽象基类**：符合 `BasePolicy` 接口规范
- **`@override` 装饰器**：明确标注方法覆盖（开发期检查）
- **自动重连机制**：客户端启动时等待服务器就绪
- **高效序列化**：`msgpack_numpy` 支持 NumPy 数组传输
- **错误反馈**：服务器错误以字符串形式返回，便于调试

---

## 七、最佳实践总结

### 模块导入

- 使用 `pip install -e .` 管理项目依赖
- 避免滥用 `sys.path.append()`（仅限调试）

### 装饰器

- 功能性装饰器（如 `@property`）会改变运行时行为
- 元信息装饰器（如 `@override`）提升代码健壮性

### 继承

- 子类自定义 `__init__` 时务必调用 `super().__init__()`
- 用 `super()` 调用父类方法，保持扩展性

### 类型注解

- 所有函数/方法都应添加类型注解
- 使用 `Optional[T]` 或 `T | None` 表示可空类型

### 抽象基类

- 用 `abc.ABC` + `@abstractmethod` 定义接口契约
- 为可选方法提供默认实现（`pass`）

> 🌟 **核心思想**：Python 的灵活性需要纪律约束——类型注解、抽象基类、装饰器等工具帮助我们在动态语言中构建可靠系统。