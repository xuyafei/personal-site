---
title: "C++ 多线程核心工具：std::atomic 与 compare_exchange_strong 完全指南"
date: 2025-08-11
draft: false
description: "深入理解 C++ 多线程编程中的原子操作和无锁编程技术，从原理到实战的完整教程"
tags: ["C++", "模板", "泛型编程", "模板元编程", "函数模板", "类模板", "模板特化", "编程技巧"]
categories: ["C++"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 10
---

# C++ 多线程核心工具：std::atomic 与 compare_exchange_strong 完全指南

`std::atomic` 和 `compare_exchange_strong` 是 C++ 多线程中的核心工具，尤其用于无锁（lock-free）编程。我们来从原理、用法、陷阱，到实战代码，系统性地讲清楚这两个概念。

## 一、什么是 std::atomic？

### 定义：

`std::atomic<T>` 是 C++ 提供的 **原子类型模板**，用于保证多个线程之间访问某个变量时不会发生竞态条件。

### 原子操作意味着：

**"不可中断"的操作**：要么全做完，要么完全不做。

### 举个例子：

```cpp
#include <atomic>
std::atomic<int> counter = 0;

void increment() {
    counter++;
}
```

这段代码中，`counter++` 就是一个 **原子操作**，多个线程可以安全地同时执行它，而不需要加锁。

### 问题一

**counter++ 是一个原子操作，那是不是一个线程执行 increment()，另一个线程必须等它执行完？**

**答案：不是"阻塞等待"，而是"并发但安全"。**

- `std::atomic<int> counter` 保证对 `counter++` 这种操作是 **线程安全的**。
- 多个线程可以并发地同时执行 `counter++`，它们不会互相"等待"或"排队"。
- 只是每次更新 `counter` 的操作是 **原子性的**，不会被打断，不会出现数据错乱（不会出现++操作丢失）。

🔧 **你可以理解为**：多个线程都在"抢执行权"，但 CPU 会确保每次只有一个成功修改值。

### 问题二

**如果 counter++ 之后再加一句 counter += 2，这两句加起来是原子操作吗？**

**答案：不是**

```cpp
void increment() {
    counter++;       // 1. 原子操作 
    counter += 2;    // 2. 原子操作 
}
```

虽然这两行各自都是原子操作，但是：

**"这两行加起来"不是一个原子操作！**

举个例子：

假设有两个线程执行 `increment()`，可能发生如下顺序：

| 线程 | 操作 | counter 值 |
|------|------|------------|
| Thread A | counter++ | counter 从 0 变为 1 |
| Thread B | counter++ | counter 从 1 变为 2 |
| Thread A | counter += 2 | counter 从 2 变为 4 |
| Thread B | counter += 2 | counter 从 4 变为 6 |

**结果是：counter = 6**

但你以为的是：
每个线程 counter += 3，总共两个线程，结果应该是 6 
然而如果插入时序不同，你可能得不到预期结果。

### 正确的原子加操作方式是：使用 fetch_add

C++ 提供了真正的原子加操作，如下：

```cpp
counter.fetch_add(3);  // 相当于 counter += 3，原子操作
```

这就避免了你的问题，整块是一个原子动作，线程之间不会交错执行。

### 实验例子：对比两种写法

你可以对比两种写法在多线程下的运行结果，差异非常明显。

| 写法 | 原子性 | 多线程安全 | 推荐用法 |
|------|--------|------------|----------|
| counter++ | 是 | 是 | 可以 |
| counter += 2 | 是 | 是 | 可以 |
| counter++; counter += 2; | 不是整体原子 | 有问题 | 不推荐 |
| counter.fetch_add(3) | 是 | 是 | 推荐 |

### 1. 没用 std::atomic<int> 时会出什么问题？

假设你这么写：

```cpp
int counter = 0;

void increment() {
    counter++;
}
```

然后你启动多个线程并发执行 `increment()`。你可能以为每个线程都能把 counter 增加 1，但实际结果 **小于你预期的加法总数**！

这是为什么？

因为 `counter++` 实际上不是一个简单的机器指令，而是三个步骤：

1. 读取 counter 的值
2. 把值加 1
3. 把新值写回 counter

多个线程执行 `counter++` 时，可能发生指令交错，导致结果丢失！

举个极端例子：

假设：
`counter = 0;`

两个线程 A 和 B 准备同时执行 `counter++`。

它们可能发生如下指令交错：

| 线程 | 操作 | counter 值 |
|------|------|------------|
| A | 读取 0 | 0 |
| B | 读取 0 | 0 |
| A | 加 1 | 1（暂存） |
| B | 加 1 | 1（暂存） |
| A | 写入 1 | 1 ✅ |
| B | 写入 1 | 1 ❌ |

### 问题：

虽然两个线程都加了 1，最终结果只加了 1 次！

这就叫做 **数据竞争（race condition）**，是线程安全的大敌。

---

### 有了 std::atomic<int> 之后：

```cpp
std::atomic<int> counter = 0;

void increment() {
    counter++;
}
```

现在 `counter++` 就是一条原子指令，不会被拆成"读+改+写"三段。

- 多个线程同时执行 `counter++`
- 背后其实是用硬件层面提供的 **原子加法指令**（如 LOCK XADD）
- 操作是不可分割的，不会丢失写入

### 总结：std::atomic vs 普通变量

| 特性 | 普通 int | std::atomic<int> |
|------|----------|------------------|
| 多线程并发安全？ | 否 | 是 |
| counter++ 是否安全？ | 不安全（可能丢增） | 安全 |
| 是否需要加锁？ | 需要手动加锁 | 不需要 |
| 效率 | 快，但有问题 | 快且安全（比加锁快） |

**补充建议：**

- 如果你只是在单线程中使用普通变量，不需要用 `std::atomic`
- 如果你在多个线程中共享读写一个变量，必须使用 `std::atomic` 或者加锁

### 问题

**加锁和使用std::atomic 的效果是一致的吧 那我什么时候选择加锁 什么时候选择std::atomic 呢**

这是个非常重要也很实用的问题！我们来 **对比分析** `std::atomic` 和加锁（如 `std::mutex`）的适用场景，并结合你的"简单例子"给出选择建议。

### 在这个例子中 —— counter++：

你可以用两种方式：

#### 方式一：使用 std::atomic

```cpp
std::atomic<int> counter = 0;

void increment() {
    counter++;  // 原子操作，线程安全
}
```

#### 方式二：使用互斥锁 std::mutex

```cpp
int counter = 0;
std::mutex mtx;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;  // 线程安全（靠锁）
}
```

### 效果是一样的吗？

**功能上是一致的**：都能让 `counter++` 变成线程安全操作。

**性能上不同！**

| 特性 | std::atomic | std::mutex |
|------|-------------|------------|
| 开销 | 非常小（底层是硬件指令） | 较大（涉及内核上下文切换） |
| 性能 | 极快 | 相对较慢 |
| 可组合性 | 差 | 强（适合多个变量操作） |
| 死锁风险 | 无 | 有（用错容易死锁） |

### 总结：选择建议

| 场景 | 推荐方式 | 理由 |
|------|----------|------|
| 只操作一个变量（如：counter++、布尔标志位） | std::atomic | 更快、更轻量 |
| 多个变量需要一起修改 | std::mutex | 需要复合操作，atomic 无法同步多个变量 |
| 希望使用条件变量（condition_variable） | std::mutex | atomic 不支持 wait-notify 机制 |
| 操作有复杂逻辑，需异常安全或条件判断 | std::mutex | lock 更易管理流程 |
| 对性能要求极高（如高频计数器、lock-free 队列） | std::atomic | 无锁性能优越 |

### 结论简洁版

- **能用 atomic 就用 atomic**，它比加锁更轻、更快。
- **多个变量/复杂逻辑就用 mutex**，不要试图用多个 atomic 硬搞同步，容易出错。
- `std::atomic` 适用于：
  - 计数器
  - 状态标志（如 is_ready, is_done）
  - lock-free 数据结构

### 举例：什么时候 atomic 不够用？

假设你想更新两个变量一起完成一个状态迁移：

```cpp
int balance;
int version;

void update() {
    balance += 100;
    version++;
}
```

你想让这个操作具有原子性（不能被打断）。用两个 atomic 是不行的！

#### 正确做法是加锁：

```cpp
std::mutex mtx;
void update() {
    std::lock_guard<std::mutex> lock(mtx);
    balance += 100;
    version++;
}
```

## 二、什么是 compare_exchange_strong？

### 核心思想：

**"如果变量当前的值是我预期的值，就修改它；否则不动。"**

这就是 CAS：Compare And Swap（比较并交换） 的原理。

### 函数签名：

```cpp
bool compare_exchange_strong(T& expected, T desired,
                             std::memory_order success = std::memory_order_seq_cst,
                             std::memory_order failure = std::memory_order_seq_cst);
```

**参数说明：**

| 参数 | 含义 |
|------|------|
| expected | 输入：你认为变量当前是这个值<br>输出：如果失败了，这个变量会被改成当前的实际值 |
| desired | 你想把变量改成的新值 |
| 返回值 | true：交换成功，值被更新了<br>false：交换失败，值没变，expected 被修改 |

## 三、完整示例代码

我们写个小例子来演示 `compare_exchange_strong` 的使用：

```cpp
#include <iostream>
#include <thread>
#include <atomic>

std::atomic<int> sharedValue(100);

void tryUpdate(int expected, int desired) {
    if (sharedValue.compare_exchange_strong(expected, desired)) {
        std::cout << "Thread " << std::this_thread::get_id()
                  << " updated value to " << desired << std::endl;
    } else {
        std::cout << "Thread " << std::this_thread::get_id()
                  << " failed to update. Current value: " << sharedValue
                  << ", expected was: " << expected << std::endl;
    }
}

int main() {
    std::thread t1(tryUpdate, 100, 200);
    std::thread t2(tryUpdate, 100, 300);

    t1.join();
    t2.join();

    std::cout << "Final value: " << sharedValue << std::endl;
    return 0;
}
```

**运行结果可能是：**

```
Thread 140024265680640 updated value to 200
Thread 140024257287936 failed to update. Current value: 200, expected was: 100
Final value: 200
```

说明只有一个线程更新成功，另一个线程因为预期不一致而失败了。

## 四、compare_exchange_strong vs compare_exchange_weak？

| 名称 | 区别 |
|------|------|
| strong | 要求"比较失败"时是真失败，不允许伪失败 |
| weak | 允许"伪失败"，适合循环重试的场景（性能更好） |

**通常建议：**

- 用 `strong` 如果你只打算尝试一次
- 用 `weak` 如果你用 while 循环去反复尝试

```cpp
while (!atomic_var.compare_exchange_weak(expected, desired)) {
    expected = atomic_var.load(); // retry with latest
}
```

## 五、compare_exchange 和 ABA 问题？

正如你前面问的，CAS 本身不能避免 ABA 问题：

如果：

1. `sharedValue == A`
2. 其他线程改成了 B，然后又改回了 A
3. 你执行 CAS 仍然会成功（因为你只比较了值）

### 解决办法是：使用 "版本号" 或者 std::atomic<std::uintptr_t> 包含指针+版本号

## 六、常见使用场景

- 实现自旋锁、自定义锁
- 实现无锁栈、无锁队列
- 实现线程安全状态机、资源池
- 实现简单的引用计数器（如智能指针底层）

## 总结回顾

| 工具 | 作用 |
|------|------|
| std::atomic<T> | 多线程安全访问变量，无需加锁 |
| compare_exchange_strong | CAS 操作，保证原子修改 |
| expected 参数 | 双向：预期输入 + 失败时输出 |
| strong vs weak | 强比较 vs 弱比较（可伪失败） |

## 核心问题：

**"既然 std::atomic<T> 解决不了 ABA 问题，还得加版本号或加锁，那它的意义到底是什么？"**

从本质、用途、限制、以及应用层面彻底讲清楚这个问题。

### 1. std::atomic<T> 的本质是什么？

`std::atomic<T>` 提供的是 **"原子性读写与基本操作"**，比如：

- 原子读写
- 原子自增自减
- 原子交换（exchange）
- 原子比较并交换（compare_exchange_strong/weak）

**它的目的是** 提供"无锁线程安全"的原语，让你在并发程序中可以写出轻量、高性能的代码，而不需要频繁用 mutex 加锁。

---

### 2. 但它有一个非常关键的局限：

`std::atomic<T>` 并不能识别值"变化过但又变回原值"的情况——也就是ABA问题。

**ABA 问题复现流程：**

1. 线程 A 读取了 x = A
2. 线程 B 把 x 改为 B，又改回 A
3. 线程 A 执行 compare_exchange(x, A, C)，成功 （但实际上状态已经变过！）

`std::atomic` 无法判断中间是否发生了状态变动。

---

### 那 std::atomic<T> 到底有什么用？

尽管存在 ABA 问题，`std::atomic` 仍然是非常有用的工具，在很多场景下都完全够用。

**它适用于：**

| 场景 | 是否有 ABA 风险 | 是否适合用 std::atomic |
|------|-----------------|------------------------|
| 计数器、自增值 | 无 | 非常适合 |
| 状态标志位（ready/done） | 无 | 非常适合 |
| 单一资源的锁竞争 | 低 | 适合（如 spinlock） |
| 简单 CAS 替换场景 | 有可能 | 但要小心 ABA |
| Lock-free 栈/队列等结构 | 高风险 | 不建议直接使用 |

### 3. 如果你需要防止 ABA 问题怎么办？

#### 方法 1：带版本号的 CAS（最经典方式）

```cpp
struct PtrWithVer {
    void* ptr;
    int version;
};

std::atomic<PtrWithVer> atomic_ptr;
```

在 compare_exchange 时，连同 ptr 和 version 一起比较，这样就能识别出"值虽然相同，但版本不同"的 ABA 问题。

**原理简单、跨平台、效果好**

---

#### 方法 2：使用更高级的原子类型（平台相关）

某些底层架构（如 ARM、x86）有自己的"tagged pointers" 或带时间戳的 CAS 支持。但标准 C++ 没有直接封装这些。

**移植性差，不推荐通用开发中使用。**

---

#### 方法 3：干脆不用 atomic，直接加锁

最简单、最通用、最保险的方法：`std::mutex`

```cpp
std::mutex mtx;
void safe_operation() {
    std::lock_guard<std::mutex> lock(mtx);
    // 操作不会有 ABA 问题
}
```

缺点是有锁的性能开销，但逻辑上稳定，适合大部分业务场景。

### 总结：std::atomic 的定位与价值

| 特性 | 说明 |
|------|------|
| 是否线程安全？ | 是的（对于单个操作） |
| 是否可以替代锁？ | 一定程度可以，但不是万能的 |
| 是否会有 ABA 问题？ | 可能有，需要注意 |
| 是否适用于复杂结构？ | 不适合（如无锁栈/队列） |
| 是否适合高性能并发？ | 非常适合（前提：无 ABA 风险） |

### 实战建议

| 需求场景 | 推荐方式 |
|----------|----------|
| 多线程计数、自增、自减、布尔标志等轻量操作 | 使用 std::atomic |
| 比较复杂的操作或多变量状态同步 | 使用 std::mutex |
| 想写 lock-free 栈/队列并且规避 ABA | 模拟版本号 CAS / Hazard Pointer |
| 线程池、任务调度等 | mutex + condition_variable |

---

## 总结

`std::atomic` 和 `compare_exchange_strong` 是 C++ 多线程编程中的强大工具，它们提供了无锁编程的基础。虽然存在 ABA 问题等限制，但在合适的场景下，它们能够提供比传统锁机制更好的性能。理解这些工具的原理和适用场景，对于编写高性能的并发程序至关重要。

记住：**选择合适的工具比使用最强大的工具更重要**。在简单场景下使用 `std::atomic`，在复杂场景下使用适当的锁机制，这样才能写出既高效又正确的并发代码。
