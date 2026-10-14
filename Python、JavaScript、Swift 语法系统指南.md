# Python、JavaScript、Swift 语法系统指南

本文按照“语言模型 → 基础语法 → 数据结构 → 控制流 → 函数 → 类型与对象 → 错误处理 → 模块与异步”的顺序，系统介绍三种语言，并通过相同任务的写法帮助建立对照。

适用范围：Python 3、现代 JavaScript，以及现代 Swift 的常用语法。部分特性对版本或运行环境有要求，会在相关位置说明。各代码块通常独立演示，不应直接把整篇代码拼成一个程序；标注为片段的示例依赖上下文。

## 目录

1. [三种语言的基本模型](#1-三种语言的基本模型)
2. [Python 语法](#2-python-语法)
3. [JavaScript 语法](#3-javascript-语法)
4. [Swift 语法](#4-swift-语法)
5. [跨语言对照与常见误区](#5-跨语言对照与常见误区)
6. [同一个任务的三种实现](#6-同一个任务的三种实现)
7. [学习路线与速查表](#7-学习路线与速查表)

## 1. 三种语言的基本模型

| 维度 | Python | JavaScript | Swift |
| --- | --- | --- | --- |
| 类型检查 | 动态类型；类型注解可供外部检查工具使用 | 动态类型 | 静态类型，支持类型推断 |
| 代码块 | 缩进 | 花括号 `{}` | 花括号 `{}` |
| 变量 | 直接赋值 | `let`、`const`；旧代码常见 `var` | `var`、`let` |
| 空值 | `None` | `null`、`undefined` | Optional 的 `nil` |
| 函数 | `def`、`lambda` | `function`、箭头函数 | `func`、闭包 |
| 集合 | `list`、`tuple`、`dict`、`set` | `Array`、`Object`、`Map`、`Set` | `Array`、`Dictionary`、`Set`、元组 |
| 对象模型 | 类与对象 | 原型链；`class` 提供类语法 | 结构体、枚举、类、协议 |
| 异步 | 协程、`async` / `await` | Promise、`async` / `await` | 并发任务、`async` / `await`、actor |

### 1.1 类型属于谁？

Python 和 JavaScript 中，一个变量可以先引用数字，再引用字符串。Swift 中，变量的类型确定后不能改成不兼容的类型。

```python
value = 42
value = "hello"  # 合法：变量重新绑定到字符串对象
```

```javascript
let value = 42;
value = "hello"; // 合法：动态类型
```

```swift
var value = 42
// value = "hello"  // 编译错误：value 被推断为 Int
```

动态类型不意味着没有类型，静态类型也不意味着每次都必须写类型。Python 的 `1 + "2"` 会报错；Swift 的 `let count = 3` 会自动推断为 `Int`。

### 1.2 语法、标准库与运行环境

需要区分三个层次：

- **语言语法**：`if`、函数声明、类声明、`async` / `await` 等。
- **标准库或内建对象**：Python 的 `len`、JavaScript 的 `Array`、Swift 的 `Array` 等。
- **运行环境或框架 API**：浏览器的 DOM、Node.js 的文件系统模块、Swift 的 Foundation 等。

例如，JavaScript 可以运行在浏览器和服务器，但 `document` 是浏览器提供的对象，并非 JavaScript 语言本身的一部分。

## 2. Python 语法

### 2.1 注释、缩进与基本输出

```python
# 单行注释
name = "Ada"

if name:
    print(f"Hello, {name}")
```

- 代码块由缩进决定，通常使用四个空格。
- `if`、`for`、`while`、`def`、`class` 等语句的头部以冒号结尾。
- 通常一行一条语句，不需要分号。
- 括号内可以自然换行。

```python
numbers = [
    10,
    20,
    30,
]


def add(a, b):
    """返回两个参数的和。"""
    return a + b
```

三引号表示多行字符串。在模块、类或函数开头时，它可以作为文档字符串；它并不是专门的块注释语法。

### 2.2 变量、赋值与解包

```python
age = 20
name = "Ada"
active = True
nothing = None

x, y = 10, 20
x, y = y, x                 # 交换
first, *middle, last = [1, 2, 3, 4]
# first == 1，middle == [2, 3]，last == 4

LIMIT = 100                 # 大写只是“常量”约定，不禁止重新赋值
```

赋值通常是在绑定对象，而不是复制对象。

```python
a = [1, 2]
b = a
b.append(3)
print(a)                   # [1, 2, 3]

c = a.copy()               # 浅拷贝：嵌套对象仍可能共享
```

### 2.3 基础类型与类型转换

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| `int` | `42`、`0xff` | 整数，精度受可用内存限制 |
| `float` | `3.14`、`1e3` | 浮点数，存在二进制表示误差 |
| `bool` | `True`、`False` | 布尔值 |
| `str` | `"hello"` | 不可变 Unicode 字符串 |
| `NoneType` | `None` | 表示没有值 |
| `bytes` | `b"abc"` | 不可变字节序列 |

```python
count = int("42")
price = float("3.5")
label = str(123)
flag = bool([])             # False

print(type(count))         # <class 'int'>
print(isinstance(count, int))  # True
```

`int("abc")` 会抛出 `ValueError`，并不会返回空值。

### 2.4 运算符与真假判断

```python
print(7 / 2)     # 3.5：普通除法
print(7 // 2)    # 3：向下取整除法
print(-7 // 2)   # -4
print(7 % 2)     # 1
print(2 ** 3)    # 8

age = 20
print(18 <= age < 65)       # 支持链式比较
print(age >= 18 and age < 65)
print(not False)
```

常见运算符：

| 类别 | 运算符 |
| --- | --- |
| 算术 | `+ - * / // % **` |
| 比较 | `== != < <= > >=` |
| 逻辑 | `and or not` |
| 成员判断 | `in`、`not in` |
| 对象身份 | `is`、`is not` |
| 位运算 | `& \| ^ ~ << >>` |
| 赋值表达式 | `:=`，Python 3.8 起支持 |

`None`、`False`、数值零、空字符串、空集合通常被视为假。`and` 和 `or` 会短路求值，并返回某个操作数，不保证返回 `bool`。

```python
name = "" or "匿名"         # "匿名"
items = []
if not items:
    print("没有元素")

if name is not None:
    print(name)
```

`==` 比较值，`is` 比较是否为同一个对象。判断空值常用 `value is None`；不要用 `is` 比较数字或字符串内容。

### 2.5 字符串、索引与切片

```python
text = "Python"
print(text[0])              # P
print(text[-1])             # n
print(text[1:4])            # yth，右边界不包含
print(text[:3])             # Pyt
print(text[::2])            # Pto
print(text[::-1])           # nohtyP

name = "Ada"
score = 92.356
print(f"{name} 的分数为 {score:.2f}")

parts = "a,b,c".split(",")
print(" / ".join(parts))    # a / b / c
print(" hello ".strip())    # hello
```

字符串不可变，不能执行 `text[0] = "J"`。Python 字符串索引按 Unicode 码点处理；一个用户感知的字符有时由多个码点组成。

### 2.6 列表、元组、字典与集合

```python
# 列表：有序，可变
numbers = [10, 20, 30]
numbers.append(40)
numbers[0] = 99
last = numbers.pop()

# 元组：有序，其元素引用不能重新赋值
point = (3, 4)
x, y = point
single = (1,)               # 单元素元组需要逗号

# 字典：键值映射，保留插入顺序
user = {"name": "Ada", "age": 20}
user["age"] = 21
city = user.get("city", "未知")
for key, value in user.items():
    print(key, value)

# 集合：元素唯一，不应依赖其遍历顺序
tags = {"python", "swift"}
tags.add("javascript")
empty_set = set()           # {} 是空字典
print(tags & {"python"})    # 交集
print(tags | {"go"})        # 并集
```

字典的键和集合的元素必须可哈希。列表和字典通常不能直接作为键；仅包含可哈希元素的元组可以。

元组不可变不代表其内部引用的对象不可变：`([1, 2],)` 中的列表仍可修改。

### 2.7 条件与循环

```python
score = 85

if score >= 90:
    grade = "A"
elif score >= 60:
    grade = "B"
else:
    grade = "C"

status = "及格" if score >= 60 else "不及格"

for i in range(3):          # 0、1、2
    print(i)

for index, name in enumerate(["Ada", "Lin"], start=1):
    print(index, name)

for name, score in zip(["Ada", "Lin"], [90, 80]):
    print(name, score)

remaining = 3
while remaining > 0:
    remaining -= 1
```

- `break`：退出最近的一层循环。
- `continue`：跳过本次循环的剩余部分。
- `pass`：空语句，用于需要语句但暂时没有实现的位置。
- 循环的 `else`：循环没有被 `break` 中断时执行。

```python
for value in [1, 3, 5]:
    if value % 2 == 0:
        print("找到偶数")
        break
else:
    print("没有偶数")
```

### 2.8 推导式与生成器

```python
squares = [x * x for x in range(6) if x % 2 == 0]
# [0, 4, 16]

lengths = {word: len(word) for word in ["cat", "apple"]}
unique_lengths = {len(word) for word in ["cat", "dog", "apple"]}

lazy_squares = (x * x for x in range(3))
print(next(lazy_squares))   # 0
print(list(lazy_squares))   # [1, 4]，只剩未消费的元素
```

推导式立即创建集合；生成器表达式按需计算。带 `yield` 的函数也会创建生成器。

```python
def countdown(n):
    while n > 0:
        yield n
        n -= 1

print(list(countdown(3)))   # [3, 2, 1]
```

### 2.9 函数、参数与作用域

```python
def greet(name, prefix="Hello"):
    return f"{prefix}, {name}"

print(greet("Ada"))
print(greet(name="Ada", prefix="你好"))


def summarize(*values, scale=1, **options):
    # values 是元组；options 是字典；scale 必须按关键字传入
    return sum(values) * scale, options

print(summarize(1, 2, 3, scale=2, label="demo"))
```

Python 3.8 起，可用 `/` 指定仅限位置参数，用 `*` 指定仅限关键字参数。

```python
def connect(host, /, port=443, *, timeout=5):
    return host, port, timeout

connect("example.com", timeout=10)
# connect(host="example.com")  # 错误：host 仅限位置传参
```

默认参数在函数定义时求值，不能把可变默认值当作“每次新建”。

```python
def append_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

函数也是对象，可以传递、返回和赋值。`lambda` 只能包含一个表达式。

```python
users = [{"name": "B", "age": 20}, {"name": "A", "age": 18}]
users.sort(key=lambda user: user["age"])


def make_counter():
    count = 0

    def increment():
        nonlocal count     # 修改外层函数作用域中的绑定
        count += 1
        return count

    return increment

counter = make_counter()
print(counter())           # 1
```

查找名称通常遵循局部、外层函数、模块全局、内建作用域的顺序。`global` 用于声明修改模块级绑定；`nonlocal` 用于修改外层函数已有的绑定。

### 2.10 类、继承与属性

```python
class Person:
    species = "human"      # 类属性

    def __init__(self, name):
        self.name = name   # 实例属性

    def greet(self):
        return f"我是 {self.name}"

    @property
    def label(self):
        return self.name.upper()


class Student(Person):
    def __init__(self, name, school):
        super().__init__(name)
        self.school = school

    def greet(self):
        return f"{super().greet()}，来自 {self.school}"

student = Student("Ada", "Example School")
print(student.greet())
print(student.label)       # 属性式访问，不加括号
```

- `self` 显式接收实例；它是约定名称。
- `__init__` 初始化实例，通常不返回值。
- `@classmethod` 接收类参数 `cls`；`@staticmethod` 不自动接收实例或类。
- `_name` 表示内部使用的约定；`__name` 触发名称改写，并非绝对私有。
- `__len__`、`__iter__`、`__eq__` 等特殊方法可以接入语言协议。

数据类适合表示以数据为主的对象：

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

print(Point(1.0, 2.0))
```

### 2.11 类型注解与模式匹配

类型注解默认不会由 Python 运行时强制检查。下面的集合类型写法要求 Python 3.9+，`| None` 写法要求 Python 3.10+。

```python
def average(values: list[float]) -> float | None:
    if not values:
        return None
    return sum(values) / len(values)
```

Python 3.10+ 的 `match` 是结构化模式匹配，不只是比较常量：

```python
def describe(event):
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"点击位置：{x}, {y}"
        case [first, second]:
            return f"两个元素：{first}, {second}"
        case _:
            return "其他事件"
```

模式中的普通裸名称通常会捕获值，不是读取同名变量来比较。字典模式可以只匹配部分键。

### 2.12 异常与资源管理

```python
try:
    number = int("42")
except ValueError as error:
    print("输入不是整数", error)
else:
    print("转换成功", number)
finally:
    print("清理阶段")


def require_positive(value):
    if value <= 0:
        raise ValueError("value 必须大于零")
    return value
```

`else` 在 `try` 正常完成后执行，`finally` 在离开该结构时执行。资源通常使用上下文管理器管理：

```python
# 需要当前目录已存在 input.txt
with open("input.txt", "r", encoding="utf-8") as file:
    content = file.read()
# 离开 with 后文件被关闭
```

### 2.13 模块、装饰器与异步

```python
import math
from pathlib import Path

print(math.sqrt(9))
print(Path("data") / "input.txt")

if __name__ == "__main__":
    print("只有作为入口脚本运行时才执行")
```

装饰器把一个函数交给另一个函数处理，再绑定处理结果：`@decorator` 大致相当于 `func = decorator(func)`。

```python
from functools import wraps


def logged(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"调用 {func.__name__}")
        return func(*args, **kwargs)
    return wrapper


@logged
def add(a, b):
    return a + b

print(add(2, 3))
```

异步函数调用后返回协程对象；需要等待或调度它，函数体才会运行。

```python
import asyncio


async def load_value(value):
    await asyncio.sleep(0.01)
    return value * 2


async def main():
    results = await asyncio.gather(load_value(1), load_value(2))
    print(results)          # [2, 4]

asyncio.run(main())         # 普通脚本入口；已有事件循环的环境需另行处理
```

`await` 让当前协程在需要等待时暂停，不等于自动创建线程。阻塞函数也不会因为放进 `async def` 就变成非阻塞函数。

## 3. JavaScript 语法

### 3.1 基本结构与声明

```javascript
// 单行注释
/* 多行注释 */

const name = "Ada";
let age = 20;
age += 1;

if (age >= 18) {
  console.log(`Hello, ${name}`);
}
```

- 用花括号组织代码块。
- 分号在很多位置可以省略，但自动分号插入存在边界情况。
- `let` 和 `const` 有块级作用域。
- `const` 禁止重新赋值，不禁止修改对象内容。
- `var` 具有函数或全局作用域，理解旧代码时需要认识它。

```javascript
const user = { name: "Ada" };
user.name = "Lin";          // 合法
// user = {};              // 错误：不能重新绑定 const

// 不要把 return 与其返回表达式分成这种两行：
function badReturn() {
  return
  { value: 1 };
}
console.log(badReturn());   // undefined
```

`let` / `const` 从块开始到初始化前处于暂时性死区，提前访问会报错。`var` 声明会被提升并初始化为 `undefined`，但赋值不会提前执行。

### 3.2 基础类型

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| `number` | `42`、`3.14` | 一般数值，包含 `NaN`、`Infinity` |
| `bigint` | `123n` | 任意精度整数 |
| `string` | `"hello"` | 不可变字符串 |
| `boolean` | `true`、`false` | 布尔值 |
| `undefined` | `undefined` | 未赋值、缺少属性等情况 |
| `null` | `null` | 通常用于显式表示空值 |
| `symbol` | `Symbol("id")` | 唯一标识值 |
| `object` | `{}`、`[]` | 对象、数组等；函数也是可调用对象 |

```javascript
console.log(typeof 42);         // "number"
console.log(typeof "hello");  // "string"
console.log(typeof null);       // "object"：历史遗留行为
console.log(typeof (() => 1));  // "function"
console.log(Array.isArray([])); // true

console.log(Number("42"));    // 42
console.log(Number("abc"));   // NaN
console.log(String(123));       // "123"
console.log(Number.isNaN(NaN)); // true
```

`number` 无法精确表示所有大整数。需要超过安全整数范围的整数运算时可考虑 `bigint`；不能直接把 `1n` 和 `1` 混合相加。

### 3.3 运算符、相等与默认值

```javascript
console.log(7 / 2);            // 3.5
console.log(Math.trunc(7 / 2)); // 3
console.log(2 ** 3);           // 8

console.log(1 === "1");      // false：严格相等
console.log(1 == "1");       // true：发生类型转换
console.log("5" + 1);        // "51"
console.log("5" - 1);        // 4
```

通常优先使用 `===` 和 `!==`，减少隐式转换带来的歧义。

```javascript
const count = 0;
console.log(count || 10);      // 10：所有假值都会触发默认值
console.log(count ?? 10);      // 0：仅 null、undefined 触发默认值

const user = { profile: null };
console.log(user.profile?.city ?? "未知");
```

- 逻辑运算：`&&`、`||`、`!`，前两者返回某个操作数。
- 空值合并：`??`。
- 可选链：`?.`，用于空值安全的属性访问或调用。
- 三元表达式：`condition ? a : b`。
- `??` 与 `&&` / `||` 直接混用时，需要括号明确分组。

假值包括 `false`、`0`、`-0`、`0n`、`""`、`null`、`undefined`、`NaN`。空数组 `[]` 和空对象 `{}` 都是真值。

### 3.4 字符串与模板字符串

```javascript
const language = "JavaScript";
console.log(language[0]);          // "J"
console.log(language.slice(0, 4));  // "Java"
console.log(`正在学习 ${language}`);

const lines = `第一行
第二行`;
const parts = "a,b,c".split(",");
console.log(parts.join(" / "));
```

JavaScript 字符串的 `length` 和普通索引按 UTF-16 码元计数。某些 emoji 会占两个或更多码元；`for...of` 按码点遍历，但一个可见字符仍可能由多个码点组合而成。

### 3.5 数组、对象、Map 与 Set

```javascript
const numbers = [10, 20, 30];
numbers.push(40);
const last = numbers.pop();
numbers[0] = 99;

const doubled = numbers.map(x => x * 2);
const large = numbers.filter(x => x >= 20);
const total = numbers.reduce((sum, x) => sum + x, 0);
const found = numbers.find(x => x === 20); // 未找到时返回 undefined

const user = { name: "Ada", age: 20 };
user.age = 21;
user["city"] = "Shanghai";
delete user.city;

for (const [key, value] of Object.entries(user)) {
  console.log(key, value);
}

const scores = new Map([
  ["Ada", 90],
  ["Lin", 85],
]);
scores.set("Sam", 88);
console.log(scores.get("Ada"));
console.log(scores.has("Lin"));

const tags = new Set(["js", "js", "swift"]);
console.log(tags.size);            // 2
```

普通对象的属性键主要是字符串或 Symbol；`Map` 可用任意类型作为键，包括对象。`Map`、`Set` 按插入顺序迭代。

`sort()` 默认按字符串顺序比较，而且会修改原数组：

```javascript
const values = [10, 2, 30];
values.sort((a, b) => a - b);       // [2, 10, 30]
const sortedCopy = [...values].sort((a, b) => b - a);
```

### 3.6 解构、展开与剩余参数

```javascript
const [first, second = 0] = [10];
const { name, age: years = 18 } = { name: "Ada", age: 20 };

const original = [1, 2];
const copy = [...original, 3];

const user = { name: "Ada", age: 20 };
const updated = { ...user, age: 21 }; // 后面的同名属性覆盖前面的

function sum(...values) {
  return values.reduce((total, value) => total + value, 0);
}
console.log(sum(...[1, 2, 3]));      // 6
```

解构默认值只在值为 `undefined` 时生效，值为 `null` 时不生效。展开对象或数组得到的是浅拷贝，嵌套对象仍可能共享。

### 3.7 条件、循环与 switch

```javascript
const score = 85;
let grade;
if (score >= 90) {
  grade = "A";
} else if (score >= 60) {
  grade = "B";
} else {
  grade = "C";
}

for (let i = 0; i < 3; i++) {
  console.log(i);
}

for (const value of [10, 20]) {
  console.log(value);             // 值
}

for (const key in { a: 1, b: 2 }) {
  console.log(key);               // 可枚举字符串属性名
}

let remaining = 2;
while (remaining > 0) {
  remaining--;
}

do {
  remaining++;
} while (remaining < 1);
```

`for...in` 可能遍历继承来的可枚举属性，不适合用于按值遍历数组。通常用 `for...of` 遍历数组元素。

```javascript
const command = "start";
switch (command) {
  case "start":
    console.log("开始");
    break;
  case "stop":
    console.log("停止");
    break;
  default:
    console.log("未知命令");
}
```

JavaScript 的 `switch` 分支默认可能贯穿到后面的分支，通常需要 `break` 或 `return` 结束。

### 3.8 函数、闭包与 this

```javascript
function add(a, b = 0) {
  return a + b;
}

const multiply = function (a, b) {
  return a * b;
};

const square = x => x * x;
const makeUser = name => ({ name }); // 返回对象字面量时用括号
```

普通函数的 `this` 通常由调用方式决定；箭头函数捕获外层的 `this`，没有自己的 `this`，也不能用作构造器。

```javascript
const person = {
  name: "Ada",
  greet() {
    return `Hello, ${this.name}`;
  },
};
console.log(person.greet());

const greet = person.greet.bind(person);
console.log(greet());             // 显式绑定接收者
```

闭包可以保存外层变量：

```javascript
function makeCounter() {
  let count = 0;
  return () => ++count;
}
const counter = makeCounter();
console.log(counter());          // 1
console.log(counter());          // 2
```

JavaScript 的默认参数表达式在调用时求值，与 Python 不同。

### 3.9 类、原型与私有字段

```javascript
class Person {
  #name;

  constructor(name) {
    this.#name = name;
  }

  get name() {
    return this.#name;
  }

  greet() {
    return `我是 ${this.#name}`;
  }

  static anonymous() {
    return new Person("匿名");
  }
}

class Student extends Person {
  constructor(name, school) {
    super(name);                   // 派生构造器中，使用 this 前先调用 super
    this.school = school;
  }

  greet() {
    return `${super.greet()}，来自 ${this.school}`;
  }
}

const student = new Student("Ada", "Example School");
console.log(student.greet());
```

`class` 建立在原型模型上。普通实例方法通常定义在类的 `prototype` 上；`#name` 是受语言约束的私有字段，不同于仅靠命名约定的 `_name`。

### 3.10 异常与模块

```javascript
function requirePositive(value) {
  if (value <= 0) {
    throw new Error("value 必须大于零");
  }
  return value;
}

try {
  requirePositive(-1);
} catch (error) {
  console.error(error.message);
} finally {
  console.log("清理阶段");
}
```

ES Modules 使用 `export` 与 `import`：

```javascript
// math.js
export function add(a, b) {
  return a + b;
}
export default class Calculator {}
```

```javascript
// main.js：在启用 ES Modules 的环境中运行
import Calculator, { add } from "./math.js";
console.log(add(1, 2));
const calculator = new Calculator();
```

浏览器通常通过 `<script type="module">` 加载模块。Node.js 是否按 ES Modules 处理文件取决于扩展名及项目配置；旧代码还可能使用 CommonJS 的 `require` / `module.exports`。

### 3.11 Promise 与 async / await

```javascript
async function double(value) {
  return value * 2;
}

async function main() {
  const one = await double(1);
  const both = await Promise.all([double(2), double(3)]);
  console.log(one, both);         // 2 [4, 6]
}

main().catch(console.error);
```

- `async` 函数总是返回 Promise。
- 普通 `return value` 会使返回的 Promise 以该值完成。
- 函数中未处理的异常会使 Promise 拒绝。
- `await` 暂停当前异步函数的后续执行，不阻塞整个 JavaScript 线程。
- `Promise.all` 聚合多个 Promise；其中一个拒绝时，聚合 Promise 会拒绝，但不会自动取消其他操作。

```javascript
// 片段：二者存在先后依赖时这样写
// const user = await loadUser();
// const orders = await loadOrders(user.id);

// 片段：二者互不依赖时可一起启动
// const [user, config] = await Promise.all([loadUser(), loadConfig()]);
```

`array.forEach(async item => ...)` 不会让外层等待所有异步回调。需要逐个等待时用 `for...of`，需要聚合并发结果时用 `Promise.all(array.map(...))`。

## 4. Swift 语法

### 4.1 注释、变量、常量与类型推断

```swift
// 单行注释
/* 多行注释，也支持嵌套块注释 */

let name = "Ada"            // 常量
var age = 20                // 变量
age += 1

let score: Double = 92.5
let active: Bool = true
print("Hello, \(name)")
```

- 用花括号组织代码块，通常不写分号。
- `let` 表示不能重新赋值；`var` 表示可以修改。
- 类型标注放在名称后，例如 `age: Int`。
- 条件必须是 `Bool`；不能把整数或字符串直接用作条件。

```swift
let count = 1
if count > 0 {
    print("有元素")
}
// if count { }             // 不合法
```

### 4.2 基础类型、转换与运算

```swift
let integer: Int = 42
let decimal: Double = 3.14
let flag: Bool = true
let character: Character = "A"
let text: String = "hello"

let total = Double(integer) + decimal
let parsed = Int("42")      // Int?，转换可能失败
let invalid = Int("abc")    // nil

print(7 / 2)                // 3：这里是整数除法
print(7.0 / 2.0)            // 3.5
print(-7 / 2)               // -3：向零截断
```

Swift 通常要求显式数值转换。基本算术运算符包括 `+ - * / %`，逻辑运算符包括 `&& || !`；没有内建的 `**` 幂运算符，也没有 `++` / `--` 运算符。

普通整数运算溢出不会自动扩展精度；需要有意进行回绕运算时可使用 `&+`、`&-`、`&*`。

### 4.3 Optional：显式表达“可能没有值”

`Int?` 表示一个可选整数，可能是某个 `Int`，也可能是 `nil`。

```swift
var nickname: String? = nil
nickname = "Ada"

if let value = nickname {
    print("昵称：\(value)")
} else {
    print("没有昵称")
}

let displayName = nickname ?? "匿名"
let length: Int? = nickname?.count
```

常见用法：

| 语法 | 含义 |
| --- | --- |
| `T?` | Optional 类型 |
| `nil` | 没有值 |
| `if let x = optional` | 有值时解包，绑定在分支内 |
| `guard let x = optional else { ... }` | 失败时提前离开当前作用域 |
| `optional ?? fallback` | 提供默认值 |
| `optional?.member` | 可选链 |
| `optional!` | 强制解包；为 `nil` 时触发运行时错误 |

```swift
func greet(_ name: String?) {
    guard let name = name else {
        print("没有姓名")
        return
    }
    print("Hello, \(name)") // guard 后可继续使用解包后的值
}
```

Optional 是类型系统的一部分，不应把 `!` 当作消除编译错误的常规手段。

### 4.4 字符串与 Unicode

```swift
let name = "Ada"
let message = "Hello, \(name)"
let multiline = """
第一行
第二行
"""

let text = "Swift"
let first = text[text.startIndex]
let secondIndex = text.index(after: text.startIndex)
let second = text[secondIndex]
print(first, second)         // S w

for character in "你好👋" {
    print(character)
}
```

Swift 的 `String` 按扩展字素簇表示 `Character`，不能用 `text[0]` 进行整数索引。不同字符的底层存储长度可能不同，按偏移移动字符串索引也不应假设为常数时间。

字符串切片常得到 `Substring`；需要独立长期保存时，可以使用 `String(substring)` 转换。

### 4.5 数组、字典、集合与元组

```swift
var numbers: [Int] = [10, 20, 30]
numbers.append(40)
numbers[0] = 99
let last: Int? = numbers.popLast()

var ages: [String: Int] = ["Ada": 20, "Lin": 21]
ages["Sam"] = 22
let adaAge: Int? = ages["Ada"]       // 查找可能失败
let unknownAge = ages["Unknown", default: 0]
ages["Lin"] = nil                  // 删除键

var tags: Set<String> = ["swift", "python"]
tags.insert("javascript")
print(tags.contains("swift"))

let point = (x: 3, y: 4)
print(point.x, point.y)
let (x, y) = point
```

数组是有序集合；字典和集合不提供可依赖的排序顺序。字典键和集合元素要求符合 `Hashable`。

数组越界访问会触发运行时错误；`first`、`last`、`popLast()` 等接口使用 Optional 表达可能为空。

### 4.6 条件、循环与区间

```swift
let score = 85
var grade: String
if score >= 90 {
    grade = "A"
} else if score >= 60 {
    grade = "B"
} else {
    grade = "C"
}

let status = score >= 60 ? "及格" : "不及格"

for i in 0..<3 {            // 0、1、2，半开区间
    print(i)
}
for i in 1...3 {            // 1、2、3，闭区间
    print(i)
}
for (index, value) in [10, 20].enumerated() {
    print(index, value)
}
for i in stride(from: 5, through: 1, by: -2) {
    print(i)               // 5、3、1
}

var remaining = 2
while remaining > 0 {
    remaining -= 1
}
repeat {
    remaining += 1
} while remaining < 1
```

`break` 和 `continue` 与另外两种语言的基本用途相同。遍历集合索引时，优先使用 `collection.indices`，避免为可能为空的集合构造错误的闭区间。

### 4.7 switch 与模式匹配

```swift
let point = (2, 2)
switch point {
case (0, 0):
    print("原点")
case (let x, let y) where x == y:
    print("位于 y = x 上，坐标 \(x), \(y)")
case (_, 0):
    print("位于 x 轴")
default:
    print("其他位置")
}
```

- `switch` 必须覆盖所有可能情况；必要时添加 `default`。
- 分支不会默认贯穿，因此通常不需要 `break`。
- 支持元组、区间、枚举关联值、值绑定和 `where` 条件。
- 显式 `fallthrough` 会继续执行下一分支语句，不重新检查下一分支的匹配条件。

### 4.8 函数、参数标签与 inout

```swift
func greet(_ name: String, from city: String = "Shanghai") -> String {
    return "Hello, \(name), from \(city)"
}

print(greet("Ada"))
print(greet("Ada", from: "Beijing"))
```

`from` 是调用处的参数标签，`city` 是函数体内部名称，`_` 表示调用时省略标签。

```swift
func add(a: Int, b: Int) -> Int {
    a + b                          // 单表达式函数可省略 return
}
print(add(a: 1, b: 2))

func sum(_ values: Int...) -> Int {
    values.reduce(0, +)
}
print(sum(1, 2, 3))

func increment(_ value: inout Int) {
    value += 1
}
var count = 1
increment(&count)
```

普通参数在函数体中是常量。`inout` 允许把修改写回调用者变量，调用时使用 `&`；不要把它简单理解为任意可长期保存的裸指针。

### 4.9 闭包与高阶函数

```swift
let square: (Int) -> Int = { value in
    value * value
}

let numbers = [1, 2, 3, 4]
let doubled = numbers.map { $0 * 2 }
let even = numbers.filter { $0 % 2 == 0 }
let total = numbers.reduce(0) { $0 + $1 }
let descending = numbers.sorted { $0 > $1 }
```

完整闭包结构为 `{ (参数) -> 返回类型 in 语句 }`。类型可推断时可以省略标注，`$0`、`$1` 表示匿名参数；最后一个闭包参数常放在调用括号外，称为尾随闭包。

```swift
func makeCounter() -> () -> Int {
    var count = 0
    return {
        count += 1
        return count
    }
}
let counter = makeCounter()
print(counter())                    // 1
```

函数接收的闭包如果会在函数返回后被保存或执行，通常需要 `@escaping` 标注。类实例与其保存的闭包相互强引用时，需要考虑通过 `[weak self]` 等捕获列表避免引用循环。

### 4.10 结构体与值语义

```swift
struct Point {
    var x: Double
    var y: Double

    var distanceSquared: Double {
        x * x + y * y               // 计算属性
    }

    mutating func move(dx: Double, dy: Double) {
        x += dx
        y += dy
    }
}

var a = Point(x: 1, y: 2)
var b = a
b.move(dx: 10, dy: 0)
print(a.x)                         // 1.0
print(b.x)                         // 11.0
```

结构体和枚举具有值语义。赋值后，对其中一个值的修改不会直接改变另一个值；底层实现可能利用写时复制等技术，不能据此推断每次赋值都立即复制所有内存。

修改结构体实例自身的实例方法需要 `mutating`。如果结构体通过 `let` 绑定，不能修改其存储属性。

### 4.11 类、继承、初始化与属性观察

```swift
class Person {
    let name: String
    var age: Int {
        didSet {
            print("年龄从 \(oldValue) 改为 \(age)")
        }
    }

    init(name: String, age: Int) {
        self.name = name
        self.age = age
    }

    func greet() -> String {
        "我是 \(name)"
    }
}

class Student: Person {
    let school: String

    init(name: String, age: Int, school: String) {
        self.school = school
        super.init(name: name, age: age)
    }

    override func greet() -> String {
        "\(super.greet())，来自 \(school)"
    }
}

let person = Person(name: "Ada", age: 20)
let samePerson = person
samePerson.age = 21
print(person.age)                   // 21
print(person === samePerson)        // true：同一实例
```

类具有引用语义。`let person` 固定引用，但仍可以修改实例的 `var` 属性。类支持继承和 `deinit`；结构体不支持类式继承。

`willSet` / `didSet` 用于观察属性赋值。可用 `static` 声明类型成员，用 `final` 限制继承或重写。

### 4.12 枚举、协议、扩展与泛型

Swift 枚举可以携带关联值：

```swift
enum LoadState {
    case idle
    case loading
    case success(String)
    case failure(String)
}

let state = LoadState.success("完成")
switch state {
case .idle:
    print("空闲")
case .loading:
    print("加载中")
case .success(let message):
    print(message)
case .failure(let reason):
    print(reason)
}
```

原始值枚举和关联值枚举用途不同：原始值是每个 case 的固定值，关联值是构造某个枚举值时携带的数据。

```swift
enum Direction: String {
    case north = "N"
    case south = "S"
}
let direction = Direction(rawValue: "N") // Direction?
```

协议声明一组能力，扩展可以增加实现：

```swift
protocol Named {
    var name: String { get }
    func greet() -> String
}

extension Named {
    func greet() -> String {
        "Hello, \(name)"
    }
}

struct User: Named {
    let name: String
}

func describe<T: Named>(_ value: T) -> String {
    value.greet()
}

print(describe(User(name: "Ada")))
```

泛型中的 `T: Named` 是约束。其他常见协议包括 `Equatable`、`Comparable`、`Hashable`、`Codable`。

现代 Swift 中，`some Protocol` 表示对调用方隐藏的某一种确定类型；`any Protocol` 表示可以容纳不同符合协议类型的存在类型。两者不能简单互换。

### 4.13 错误处理与 defer

```swift
enum InputError: Error {
    case notPositive
}

func requirePositive(_ value: Int) throws -> Int {
    guard value > 0 else {
        throw InputError.notPositive
    }
    return value
}

do {
    let value = try requirePositive(-1)
    print(value)
} catch InputError.notPositive {
    print("必须输入正数")
} catch {
    print("其他错误：\(error)")
}

let result = try? requirePositive(-1) // Int?，失败时为 nil
```

- `throws` 标明函数可能抛错。
- `try` 标出可能抛错的调用。
- `try?` 把失败转换为 Optional 的 `nil`。
- `try!` 断言不会抛错，实际抛错时触发运行时错误。
- `defer` 在离开当前作用域时执行，多条 `defer` 按后注册先执行的顺序运行。

```swift
func demonstrateCleanup() {
    defer { print("最后执行清理") }
    print("执行工作")
}
demonstrateCleanup()
```

Optional 常用于表达“没有值”，`throws` 常用于表达“操作失败并携带原因”。

### 4.14 模块、访问控制与异步

```swift
import Foundation

public struct Greeting {
    public let text: String

    public init(text: String) {
        self.text = text
    }
}
```

常见访问级别包括 `private`、`fileprivate`、`internal`（默认）、`public`、`open`。`open` 还允许模块外继承类或重写相应成员。较新的 Swift 版本还支持面向包边界的 `package`。

Swift 5.5 起提供 `async` / `await` 等并发语法：

```swift
func double(_ value: Int) async -> Int {
    value * 2
}

func runExample() async {
    async let first = double(1)
    async let second = double(2)
    let results = await (first, second)
    print(results)                   // (2, 4)
}
// 需要从异步上下文调用 await runExample()；定义函数本身不会启动任务。
```

会抛错的异步调用写作 `try await operation()`。`async let` 创建结构化子任务；`Task { ... }` 可以从同步上下文启动非结构化任务，其生命周期需要调用方考虑。

actor 用于隔离可变状态：

```swift
actor Counter {
    private var value = 0

    func increment() -> Int {
        value += 1
        return value
    }
}

func useCounter() async {
    let counter = Counter()
    let current = await counter.increment()
    print(current)
}
```

跨 actor 隔离边界访问通常需要 `await`。actor 方法在 `await` 暂停期间，其他任务可能进入 actor，因此仍要考虑状态在暂停前后发生变化的情况。具体并发检查严格程度受 Swift 语言模式和编译配置影响。

## 5. 跨语言对照与常见误区

### 5.1 语法速查

| 意图 | Python | JavaScript | Swift |
| --- | --- | --- | --- |
| 输出 | `print(x)` | `console.log(x)` | `print(x)` |
| 布尔值 | `True` / `False` | `true` / `false` | `true` / `false` |
| 无值 | `None` | `null` / `undefined` | `nil`，需 Optional |
| 字符串插值 | `f"Hi {name}"` | `` `Hi ${name}` `` | `"Hi \(name)"` |
| 集合长度 | `len(items)` | `items.length`（数组） | `items.count` |
| 添加数组元素 | `items.append(x)` | `items.push(x)` | `items.append(x)` |
| 遍历元素 | `for x in items:` | `for (const x of items)` | `for x in items` |
| 带索引遍历 | `enumerate(items)` | `items.entries()` | `items.enumerated()` |
| 默认值 | `x if x is not None else y` | `x ?? y` | `x ?? y` |
| 条件表达式 | `a if test else b` | `test ? a : b` | `test ? a : b` |
| 匿名函数 | `lambda x: x * 2` | `x => x * 2` | `{ $0 * 2 }` |
| 异常捕获 | `try` / `except` | `try` / `catch` | `do` / `catch` |
| 资源清理 | `with` / `finally` | `finally` | `defer` |

### 5.2 数值运算容易出现的差异

| 表达式或需求 | Python | JavaScript | Swift |
| --- | --- | --- | --- |
| `7 / 2` | `3.5` | `3.5` | `3`，整数推断情况下 |
| 负整数除法取整 | `-7 // 2` 为 `-4` | `Math.trunc(-7 / 2)` 为 `-3` | `-7 / 2` 为 `-3` |
| `-7 % 2` | `1` | `-1` | `-1` |
| 大整数 | `int` 支持任意精度 | 使用 `bigint` | `Int` 是固定宽度 |
| 数字与字符串相加 | 通常报错 | 可能字符串拼接 | 通常编译失败 |

三种语言常用浮点类型都存在精度误差，不能假设十进制小数运算总能精确表示。

### 5.3 空集合和布尔条件

```python
if []:
    print("不会执行")
```

```javascript
if ([]) {
  console.log("会执行");
}
```

```swift
let values: [Int] = []
if values.isEmpty {
    print("明确判断集合是否为空")
}
```

不要机械地把一种语言的条件表达式搬到另一种语言。

### 5.4 相等、身份与复制

| 问题 | Python | JavaScript | Swift |
| --- | --- | --- | --- |
| 值相等 | `==`，可由类型定制 | 原始值常用 `===`；对象默认比较身份 | 符合 `Equatable` 的类型使用 `==` |
| 同一个对象 | `is` | 对象使用 `===` | 类实例使用 `===` |
| 数组整体比较 | 列表可按元素比较 | 两个不同数组即使内容相同，`===` 也为假 | 元素符合 `Equatable` 时可按元素比较 |
| 集合赋值 | 通常共享同一可变对象 | 数组、对象通常共享引用 | `Array`、`Dictionary`、`Set` 具有值语义 |

Swift 值类型内部仍可保存类引用。复制包含引用的结构体或数组，不会自动把所引用的所有类实例深拷贝。

### 5.5 字符串长度并不统一

对于同一个 emoji，三种语言的长度结果可能不同：

```python
print(len("😀"))            # 1 个 Unicode 码点
```

```javascript
console.log("😀".length);   // 2 个 UTF-16 码元
```

```swift
print("😀".count)           // 1 个 Character
```

处理截断、光标移动和用户可见字符计数时，需要先明确是在数“字节”“码元”“码点”还是“字素簇”。

### 5.6 异步写法相似，执行模型不同

| 方面 | Python | JavaScript | Swift |
| --- | --- | --- | --- |
| 调用异步函数得到 | 协程对象 | Promise | 需在合适上下文通过 `await` 调用；`async` 本身不创建新任务 |
| 启动并发工作的常见方式 | `asyncio.create_task`、`gather` | 调用返回 Promise 的操作后聚合 | `async let`、任务组、`Task` |
| 等待多个结果 | `asyncio.gather` | `Promise.all` | `async let` 或任务组 |
| 状态隔离 | 依赖具体设计与同步机制 | 依赖事件循环、worker 等环境机制 | actor 与隔离检查 |

三种语言中的 `async` 都不意味着“这段计算自动在多个 CPU 核上并行”。也不要把取消任务理解为强制立即终止任意代码。

### 5.7 最值得提前记住的误区

1. **Python 可变默认参数**：默认列表会跨调用复用，通常用 `None` 再初始化。
2. **Python 身份比较**：`is` 不是字符串或数值相等比较。
3. **JavaScript `const`**：限制重新绑定，不让对象自动变成不可变对象。
4. **JavaScript 排序**：数字排序通常要传比较函数，默认排序不是数值排序。
5. **JavaScript 异步遍历**：`forEach` 不等待异步回调完成。
6. **Swift 强制解包**：`!` 不是安全的默认处理方式。
7. **Swift 集合赋值**：值语义与 Python、JavaScript 的可变集合引用行为不同。
8. **三种语言的复制**：浅拷贝不等于深拷贝；共享引用需要单独考虑。

## 6. 同一个任务的三种实现

任务：给定一组学生，筛选分数不低于 60 的学生，按分数降序排列，再按姓名升序处理同分情况，最后输出姓名和平均分。这里使用 ASCII 姓名，避免跨语言字符串排序规则的差异干扰示例。

### 6.1 Python

```python
from dataclasses import dataclass


@dataclass
class Student:
    name: str
    score: int


students = [
    Student("Ada", 90),
    Student("Bob", 55),
    Student("Cara", 80),
    Student("Dan", 90),
]

passed = sorted(
    (student for student in students if student.score >= 60),
    key=lambda student: (-student.score, student.name),
)

names = [student.name for student in passed]
average = (
    sum(student.score for student in passed) / len(passed)
    if passed
    else None
)

print(", ".join(names))
if average is not None:
    print(f"平均分：{average:.2f}")
else:
    print("没有及格学生")
```

关键语法：数据类、生成器表达式、元组排序键、列表推导式、条件表达式、f-string。

### 6.2 JavaScript

```javascript
const students = [
  { name: "Ada", score: 90 },
  { name: "Bob", score: 55 },
  { name: "Cara", score: 80 },
  { name: "Dan", score: 90 },
];

const passed = students
  .filter(student => student.score >= 60)
  .sort((a, b) => {
    if (a.score !== b.score) return b.score - a.score;
    return a.name < b.name ? -1 : a.name > b.name ? 1 : 0;
  });

const names = passed.map(student => student.name);
const average = passed.length > 0
  ? passed.reduce((sum, student) => sum + student.score, 0) / passed.length
  : null;

console.log(names.join(", "));
if (average !== null) {
  console.log(`平均分：${average.toFixed(2)}`);
} else {
  console.log("没有及格学生");
}
```

关键语法：对象字面量、链式调用、箭头函数、排序比较器、`reduce`、模板字符串。`filter` 创建了新数组，因此这里的 `sort` 不修改原来的 `students` 数组顺序。

### 6.3 Swift

```swift
import Foundation

struct Student {
    let name: String
    let score: Int
}

let students = [
    Student(name: "Ada", score: 90),
    Student(name: "Bob", score: 55),
    Student(name: "Cara", score: 80),
    Student(name: "Dan", score: 90),
]

let passed = students
    .filter { $0.score >= 60 }
    .sorted {
        if $0.score != $1.score {
            return $0.score > $1.score
        }
        return $0.name < $1.name
    }

let names = passed.map { $0.name }
let average: Double? = passed.isEmpty
    ? nil
    : Double(passed.reduce(0) { $0 + $1.score }) / Double(passed.count)

print(names.joined(separator: ", "))
if let average = average {
    print(String(format: "平均分：%.2f", average))
} else {
    print("没有及格学生")
}
```

关键语法：结构体、尾随闭包、匿名参数、Optional、显式数值转换、可选绑定。示例中的格式化字符串接口来自 Foundation。

三种实现均输出：

```text
Ada, Dan, Cara
平均分：86.67
```

## 7. 学习路线与速查表

### 7.1 建议学习顺序

1. **变量和类型**：写出数字、字符串、布尔值以及“没有值”的表达方式。
2. **控制流和集合**：实现查找、筛选、统计、排序。
3. **函数**：掌握参数、返回值、闭包、作用域。
4. **对象与数据建模**：Python 类、JavaScript 对象与原型、Swift 结构体与协议。
5. **错误和资源管理**：区分没有结果、输入非法和执行失败。
6. **模块与异步**：先理解调用和等待的关系，再处理并发、取消和状态共享。

练习时，可以把同一段业务逻辑分别用三种语言实现，再检查它们在空集合、非法输入、同分排序和引用共享情况下的行为。

### 7.2 每种语言最需要建立的直觉

| 语言 | 重点 |
| --- | --- |
| Python | 缩进决定结构；名称绑定对象；协议与迭代器贯穿语言；注解不等于运行时检查 |
| JavaScript | 隐式转换会改变结果；对象以引用方式共享；`this` 与调用方式有关；Promise 表达异步结果 |
| Swift | 类型与 Optional 要显式处理；值语义和引用语义并存；协议、泛型和枚举用于建模；并发受隔离规则约束 |

### 7.3 进一步阅读

以下为官方资料入口，适合在学习具体特性时继续查阅：

- [Python 官方教程](https://docs.python.org/zh-cn/3/tutorial/)
- [Python 语言参考](https://docs.python.org/zh-cn/3/reference/)
- [ECMAScript 语言规范](https://tc39.es/ecma262/)
- [The Swift Programming Language](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/)

