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

``` Python
a = [1, 2]
b = a
b.append(3)

print(a)  # [1, 2, 3]
print(a is b) # True: 指向同一个对象
print(a == b) # True: 值相等
```

浅拷贝仅复制外层，深拷贝则递归复制，复制即意味着开辟了新的内存，不再指向原来的对象，但是存储着相同的值，例如：

``` Python
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

``` Python
def collect_bad(x: int, result: list[int] = []) -> list[int]:
    result.append(x)
    return result
 
def collect_good(x: int, result: list[int] | None = None) -> list[int]:
    if result is None:
        result = []
    result.append(x)
    return result
```

接下来，调用它们：

``` Python
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

``` Python
from functools import wraps
from time import perf_counter
from collections.abc import Callable
from typing import ParamSpec,TypeVar

P = ParamSpec("P")
R = TypeVar("R")

def timed(fn: Callable[P， R]) -> Callable[P, R]:
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
    return [value * 2 for value in values]
```

`@timed` 相当于 `preprocess = timed(preprocess)` ，由于 timed 返回一个 `wrapper` 函数，因此实际上 `preprocess` 函数被 `wrapper` 函数所“代理”。

具体流程是这样的：`preprocess = timed(preprocess)`，`timed` 传入 `preprocess` 这个函数，`@wraps(fn)` 保留 `preprocess` 的元信息，接着，`timed` 内部定义的 `wrapper`，由于其参数是 `*args` 和 `**kwargs`，因此，会直接将 `preprocess` 的参数传入，然后，由于闭包，`wrapper` 在 `return fn(...)` 的时候能找到外层的 `fn`，执行 `preprocess` 原内部的逻辑。

一个上下文管理器（使用 @contextmanager 装饰器将函数对象变成可被 with 操作的对象）：

``` Python
from contextlib import contextmanager
from collections.abc import Iterator
from time import perf_counter

@contextmanager
def timer(name: str) -> Iterator[None]:
    start = perf_counter()
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

``` Python
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

``` Cpp
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

``` Cpp
void consume(T value);  // 按值：函数获得自己的变量，适合小型标量
void update(T& value);  // 可变借用：函数会修改调用方
void inspect(const T& value);  // 只读借用：避免复制 <- 这里，const 引用的接口语义
void optional(T* value);  // 指针可能为空，通常不用有对象
```

## 说明 RAII、`unique_ptr`、`shared_ptr` 各自解决什么问题

RAII：Resource Acquisition Is Initialization，资源获取即初始化，即在构造时获取资源，在析构时释放资源。它解决的问题是：资源生命周期必须有清晰的边界。

具体来说，就是确保程序在正常退出或者抛出异常的时候，资源（如内存、文件句柄、网络 Socket，互斥锁等）都能够被确定性地释放。如果资源没有被释放，就会造成资源泄露，长期出现资源泄露，可能会造成内存枯竭以至于崩溃等问题。

`unique_ptr`：唯一拥有，可移动不可复制。解决了所有权模糊问题，避免了 `double free` 问题。

`shared_ptr`：共享所有权，引用计数共享所有。解决的是当一块动态分配内存被多个模块、对象、线程使用的时候，且无法确定最后谁使用完的问题。

## 解释预处理、编译、汇编、链接和动态装载阶段

假设文件名字叫做 `example.cpp`

预处理：展开头文件和宏，得到 `example.ii`；
编译：生成汇编文件 `example.s`；
汇编：处理汇编代码，生成可重定位的目标文件 `example.o`；
链接：扫描所有的 `.o` 和库文件（静态库`.a`，动态库`.so`），把各个模块中未解析的符号（全局变量、函数等）与实际定义关联起来，并修正代码段和数据段中的相对地址，生成可执行 `ELF` 文件；
动态装载：操作系统将可执行文件及依赖的动态库映射到内存，完成运行时符号解析（`GOT/PLT`），最终移交控制权启动程序。

## 用 CMake 构建一个最小 C++ 库

一个可用 `Cpp` 库应该包含暴露的 `.h` 头文件和生成的 `.a` 静态库或 `.so` 动态库，具体实现上如下：

目录：

```
vector_ops/
├── CMakeLists.txt
├── include/vector_ops.h
└── src/vector_ops.cpp
```

`CmakeLists.txt`:

``` CMake
cmake_minimum_required(VERSION 3.20)
project(vector_ops LANGUAGES CXX)

add_library(vector_ops src/vector_ops.cpp)
target_include_directories(vector_ops PUBLIC include)
target_compile_features(vector_ops PUBLIC cxx_std_20)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(vector_ops PRIVATE -Wall -Wextra -Wpedantic)
endif()
```

构建：

``` Bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

## 解释 shape、stride、dtype、device 和 lifetime 为什么属于跨语言接口

首先解释一下这几个概念：`shape` 是对 N 维张量的一种描述，`stride` 是步长，例如，在一个 2 行 3 列的矩阵中，`shape` 等于 `(2, 3)`，`stride` 等于 `(3, 1)`，即 `offset = 3i + j`，可见，`shape` 和 `stride` 共同将多维张量的索引映射成成一维物理内存的索引；`dtype` 即 `data type`，数据类型，规定了内存位宽和对齐方式，`device` 是设备，如 `CPU`、`GPU` 等，`lifetime` 是生命周期，涉及所有权的问题。

然后理解跨语言：不同语言的设计理念不同，因此各自擅长的事也不同，例如 `Python` 开发快，生态丰富，适合写编排层，而 `Cpp` 凭借出色的性能适合写系统层，因此在开发中，为了兼顾开发效率和性能，同时使用两种语言是必要的；但是两种语言的内存管理模式、对象等显然是不同的，`Python` 中对象是动态的，内存分配比较灵活，而 `Cpp` 中则依赖手动分配内存或智能指针等，需要明确的内存地址和字节偏移，因此当二者同时操作一块内存对象的时候，便会产生冲突。

跨语言接口就是为了解决这个问题，通过 `shape` 等概念，设计一套统一的接口，协助两种语言写作开发。

为了提升跨语言的性能，还需要引入零拷贝的概念。这里不详细展开。

## 在 Linux 上查进程、资源占用、端口和动态库依赖

查进程：

``` Bash
ps -ef    # e：显示所有进程，f：全格式显示
htop      # top 的增强版
```

资源占用（CPU、内存、磁盘和网络）、端口：

| 问题 | 常用命令 |
| :--- | :--- |
| CPU/负载 | `top`、`uptime`、`mpstat` |
| 进程内存 | `ps`、`top`、`pmap -x PID` |
| 系统内存 | `free -h`、`vmstat` |
| 磁盘容量 | `df -h` |
| 目录占用 | `du -sh PATH`、`du -h --max-depth=1` |
| 文件 I/O | `iostat`、`iotop` |
| 监听端口 | `ss -lntp` |
| 连接测试 | `curl -v URL`、`nc -vz HOST PORT` |

动态库依赖：

| 工具 | 底层机制 | 核心命令 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **readelf** | 静态解析 ELF 的 `.dynamic` 节 | `readelf -d ./app \| grep NEEDED` | 安全审计、交叉编译、查看直接依赖 |
| **objdump** | 静态解析 ELF 私有头信息 | `objdump -p ./app \| grep NEEDED` | 替代 readelf 的静态分析工具 |
| **ldd** | 调用动态链接器模拟加载 | `ldd ./app` | 查看完整依赖树及绝对路径，排查 not found |
| **ldconfig** | 管理动态链接器缓存 | `ldconfig -p \| grep lib_name` | 查找系统已安装的动态库及其路径 |

## 区分 NVIDIA Driver、CUDA Toolkit、CUDA Runtime 与 cuDNN

原文中是这样说明的：

> NVIDIA Driver：操作系统与 GPU 通信的基础；nvidia-smi 主要反映驱动状态；
CUDA Toolkit：开发工具集合，包含 nvcc、头文件、调试/分析工具和库；
CUDA Runtime/数学库：应用运行时加载的用户态组件；
cuDNN：面向深度学习算子的独立加速库，不等于整个 CUDA。

## 创建小而清晰的 Git 分支和提交，并在提交前检查 diff

分支相关命令：

``` Bash
git branch                  # 查看全部本地分支，并标记当前分支
git branch -v               # 查看全部本地分支，标记当前分支，并列出最新 commit messege
git branch -a               # 查看全部本地和远程关联分支
git branch --show-current   # 仅查看当前分支

git branch <branch-name>    # 仅创建分支，不移动指针
git switch -c <branch-name> # 创建分支并移动指针指向该分支

git switch <branch-name>    # 切换分支

git branch -d <branch-name> # 删除分支
git branch -D <branch-name> # 强制删除分支，无视合并状态
```

检查 `diff`：

``` Bash
git diff                # 工作区 vs 暂存区
git diff --staged       # 暂存区 vs HEAD
git diff HEAD~1 HEAD    # HEAD~1 vs HEAD
```

## 能用“最小复现 + 环境快照 + 单变量实验”定位问题

原文中给出的定位框架：
> 遇到问题时按以下顺序记录：
    期望行为：应该发生什么；
    实际行为：完整报错、退出码、错误输出；
    最小复现：最短命令、最小输入；
    环境：代码提交、依赖版本、系统、硬件；
    边界定位：数据、Python、C++、CUDA、驱动、网络中的哪一层；
    单变量实验：每次只改变一项并记录结果。

需实践检验。
