+++
title = "AI Infra Guide 验收：前置知识-编程基础"
date = 2026-11-18
description = "Python；Cpp；Python 与 Cpp 互操作；Linux 开发；环境隔离与软件栈；Git；问题定位"

[taxonomies]
tags = ["ai-infra-learning"]
+++

# 验收

## 解释 is 与 == 、浅拷贝与深拷贝的区别

a is b 代表 a 和 b 指向同一个对象（内存），a == b 则仅代表二者的值相等，例如：

```
a = [1, 2]
b = a
b.append(3)

print(a)  # [1, 2, 3]
print(a is b) # True: 指向同一个对象
print(a == b) # True: 值相等
```

浅拷贝仅复制外层，深拷贝则递归复制，复制即意味着开辟了新的内存，不再指向原来的对象，但是存储着相同的值，例如：

```
import copy

src = [[3], [9]]
shallow = src.copy()
deep = copy.deepcopy(src)

shallow[0].append(9)
print(src)  # [[3, 9], [9]]  
print(deep)  # [[3], [9]]
```

## 说明默认可变参数为什么会跨调用共享

以下面的例子说明：

```
def collect_bad(x: int, result: list[int] = []) -> list[int]:
 result.append(x)
 return result
 
def collect_good(x: int, result: list[int] | None = None) - > list[int]:
 if result is None:
  result = []
 result.append(x)
 return result
```

接下来，调用它们：

```
collect(3)
collect(9)
collect(39)

print(result)
```

对于 `collect_bad` 和 `collect_good` 两个函数，结果分别是：

```
[3, 9, 39] # 这不是我们期望的结果
[39] # 这是我们期望的结果
```

我们期望每次调用函数，list 都是空的，一次只有一个 x 被 collect，但是 `collect_bad` 函数没有出现期望的结果，这是因为：**默认参数在函数定义时求值，而不是每次调用时求值**。

默认参数：即带有默认值的参数，在上面的例子中就是 `result` 这个参数。Python 中，默认参数在函数定义时（即解释器执行 `def` 语句时）求值。具体而言，解释器会将这些求值后的对象绑定到函数对象的 `__defaults__` 属性中。因此，后续每次调用该函数且未显示传入对应实参时，均会复用该属性存储的同一对象引用。如果这个默认参数是可变参数，对其进行原地修改会引发**跨调用共享**的状态累加。

因此，采用 `None` 这个不可变参数，每次在函数内部重新动态初始化可变对象，避免**跨调用共享**带来的状态污染。

## 写一个保留函数元信息的装饰器和一个上下文管理器

函数元信息：`__name__`、`__doc__`、`__module__`、`__defaults__` 等，用于描述函数的信息。

一个保留原函数的计时器装饰器代码如下（使用 functools 库中的 @wraps 装饰器）：

```
from functools import wraps
from time import perf_counter
from collections.abc import Callable
from typing import ParamSpec,TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(fn： Callable[P， R]) -> Callable[P, R]:
 @wraps(fn)
 def wrapper(*args: P.args, **kwargs:P.kwargs) -> R:
  start = perf_counter()
  try:
   return fn(*args, **kwargs)
  finally:
   elapsed_ms = (perf_counter() - start) * 1_000
   print(f"{fn.__name__}:{elapsed_ms:.3f} ms")
 return wrapper
 
@timed
def preprocess(values: list[float]) -> list[float]:
 return [value * 2 for vaule in values]
```

`@timed` 相当于 `preprocess = timed(preprocess)` ，由于 timed 返回一个 `wrapper` 函数，因此实际上 `preprocess` 函数被 `wrapper` 函数所“代理”。

具体流程是这样的：`preprocess = timed(preprocess)`，`timed` 传入 `preprocess` 这个函数，`@wraps(fn)` 保留 `preprocess` 的元信息，接着，`timed` 内部定义的 `wrapper`，由于其参数是 `*agrs` 和 `**kwargs`，因此，会直接将 `preporcess` 的参数传入，然后，由于闭包，`wrapper` 在 `return fn(...)` 的时候能找到外层的 `fn`，执行 `preprocess` 原内部的逻辑。

一个上下文管理器（使用 @contextmanager 装饰器将函数对象变成可被 with 操作的对象）：

```
from contextlib import contextmanager
from collections.abc import Iterator
from time import perf_counter

@contextmanager
def timer(name: str) -> Iterator[None]:
 start = perf_counter
 try:
  yield
 finally:
        elapsed_ms = (perf_counter() - start) * 1_000
        print(f"{name}: {elapsed_ms:.3f} ms")

with timer("load data"):
 data = [value for value in range(100_000)]
```

`@contextmanager` 函数可以实现一个生成器函数（有 `yield` 的函数）的上下文管理协议（`__enter__` 和 `__exit__` 方法），从而被 `with` 操作，使用 `with` 能把资源的获取和释放放在同一个语法词块内，即使发生异常，也能执行清理。

## 根据 I/O 密集、CPU 密集选择线程、进程或异步 I/O

| 密集型 | 选择 |
| ------ | ------------ |
| I/O 密集 | 线程 或 asyncio |
| CPU 密集 | 多进程 或 原生扩展 |

## GPU 异步执行为什么会让普通 CPU 计时失真

Guide 中是这样描述的：

> 对于 GPU，Kernel 启动通常是异步的。CPU 计时器只量到“提交任务”的时间，需要在边界同步，或使用框架提供的 GPU Event。同步会改变流水线行为，因此端到端吞吐和单算子延迟应分别测量。

### 异步执行

具体的说，GPU 是异步的，意味着 CPU 向 GPU 下发指令，将任务写入 GPU 硬件队列——完成这一轻微开销后，CPU 会继续做其他工作，二者可以并行计算。而 `Python` 的 `time.time()`，`time.perf_counter()` 等 `time` 库中的函数用来计算时间时，是在 CPU 设备上执行的，因此其测得的时间只是 CPU 下发指令的轻微开销，而与此同时，GPU 可能刚开始计算任务。这就是**GPU 异步执行会让普通 CPU 计时失真**的原因。

### 预热

此外，现代 GPU 设备可以处于几种不同的功耗状态之一。当 GPU 未被用于任何目的，且未开启持久模式时，GPU 会自动将其功耗状态降至极低水平，有时甚至完全关闭。在较低的功耗状态下，GPU 会关闭不同的硬件组件，包括内存子系统、内部子系统，甚至是计算核心和缓存。因此需要**预热**。

### 实践

参考[Correct Way To Measure Time - PyTorch Best Practices](https://github.com/Paxoo/PyTorch-Best_Practices/wiki/Correct-Way-To-Measure-Time))，可以得到正确测量 GPU 上任务执行时间的方法。

对于端到端吞吐和单算子延迟，由于二者的目标不同，测量方法也不同。具体的说，单算子延迟由于要求精确，因此 `torch.cuda.synchronize()` 应放在循环内，而端到端的目标是压榨极限，因此必须满载，`torch.cuda.synchronize()` 要放到循环外一次处理。

但是 pytorch 文档中给出的示例确是相同的，这可能是工程和理论的差距。

```
# GPU-Warmup
for _ in range(10):
    _ = model(input)

for rep in range(repetitions):
 starter.record()
 _ = model(input)
 ender.record()

 # wait for GPU sync
 torch.cuda.synchronize() # 示例中都放到了循环内
```

## 区分 C++ 指针、引用、值和 `const` 引用的接口语义

### 指针、引用、值

```
#include <iostream>

int main() {
 int value = 10;

 int* ptr = &value;
 int& ref = value;  // 引用，指向同一块内存，ref 是 value 的别名

 *ptr = 9;  
 ref += 1; 

 std::cout << value << '\n';  // value = 10
}

```

### const 引用的接口语义

```
void consume(T value);  // 按值：函数获得自己的变量，适合小型标量
void update(T& value);  // 可变借用：函数会修改调用方
void inspect(const T& value);  // 只读借用：避免复制 <- 这里，const 引用的接口语义
void optional(T* value);  // 指针可能为空，通常不用有对象
```

## 说明 RAII、`unique_ptr`、`shared_ptr` 各自解决什么问题

RAII：Resource Acqusition Is Initialization，资源获取即初始化，即在构造时获取资源，在析构时释放资源。它解决的问题是：资源生命周期必须有清晰的边界。

具体来说，就是确保程序在正常退出或者抛出异常的时候，资源（如内存、文件句柄、网络 Socket，互斥锁等）都能够被确定性地释放。如果资源没有被释放，就会造成资源泄露，长期出现资源泄露，可能会造成内存枯竭以至于崩溃等问题。

`unique_ptr`：唯一拥有，可移动不可复制。解决了所有权模糊问题，避免了 `double free` 问题。

`shared_ptr`：共享所有权，引用计数共享所有。解决的是当一块动态分配内存被多个模块、对象、线程使用的时候，且无法确定最后谁使用完的问题。

## 解释预处理、编译、汇编、链接和动态装载阶段

假设文件名字叫做 `example.cpp`

预处理：展开头文件和宏，得到 `example.ii`
编译：生成汇编文件 `example.s`
汇编：处理汇编代码，生成可重定位的目标文件 `example.o`
链接：扫描所有的 `.o` 和库文件（静态库`.a`，动态库`.so`），把各个模块中未解析的符号（全局变量、函数等）与实际定义关联起来，并修正代码段和数据段中的相对地址，生成可执行 `ELF` 文件
动态装载：操作系统将可执行文件及依赖的动态库映射到内存，完成运行时符号解析（`GOT/PLT`），最终移交控制权启动程序

## 用 CMake 构建一个最小 C++ 库
