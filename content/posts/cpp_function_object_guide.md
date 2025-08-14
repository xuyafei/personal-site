---
title: "C++ 函数对象（Function Object）完全指南：从基础到实战"
date: 2025-08-01
description: "深入解析C++函数对象（仿函数）的概念、原理、使用方法和实战应用，包含大量示例代码和详细说明"
tags: ["C++", "函数对象", "仿函数", "STL", "模板编程", "lambda表达式"]
categories: ["C++"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 19
math: true
---

# C++ 函数对象（Function Object）完全指南：从基础到实战

C++ 的函数对象（Function Object）是一个非常强大又实用的特性，是现代 C++ 编程中不可或缺的一部分。下面我会从基础到进阶、从原理到实战、配合大量示例完整、细致地讲解函数对象（仿函数）。

---

## 什么是函数对象？

在 C++ 中：

- **函数对象（Function Object）**是一个可以像函数一样使用的对象。
- 它本质上是一个定义了 `operator()` 的类或结构体的对象。
- 也称为**仿函数（Functor）**。

## 1. 函数对象的基本定义

### 示例 1：最基础的函数对象

```cpp
#include <iostream>

struct Adder {
    int operator()(int a, int b) const {
        return a + b;
    }
};

int main() {
    Adder add;
    std::cout << "3 + 4 = " << add(3, 4) << std::endl;  // 像函数一样调用
}
```

**输出：**
```
3 + 4 = 7
```

**说明：**
- `Adder` 是一个函数对象类
- 定义了 `operator()(int, int)`，所以对象 `add` 可以像函数一样调用：`add(3, 4)`

## 2. 函数对象 vs 普通函数

| 对比项 | 函数对象 | 普通函数 |
|--------|----------|----------|
| 状态 | 可以保存状态（成员变量） | 无状态 |
| 灵活性 | 可以重载 `operator()`，适应不同场景 | 不可重载 |
| 类型 | 类型明确，可作为模板参数 | 函数名退化为函数指针 |
| 性能 | 编译器可内联优化 | 一般性能好，但不如内联 |
| 适用场景 | STL 算法、策略模式、lambda 底层实现 | 通用函数调用 |

## 3. STL 中的函数对象使用

### 示例 2：配合 STL 算法

```cpp
#include <vector>
#include <algorithm>
#include <iostream>

struct Greater {
    bool operator()(int a, int b) const {
        return a > b;
    }
};

int main() {
    std::vector<int> v = {5, 2, 9, 1, 3};

    std::sort(v.begin(), v.end(), Greater{});  // 降序排序

    for (int n : v)
        std::cout << n << " ";
}
```

**输出：**
```
9 5 3 2 1
```

**说明：**
- STL 的很多算法，如 `std::sort`、`std::find_if`、`std::transform`，都接受函数对象作为参数。
- 可以通过函数对象传入自定义逻辑。

## 4. 函数对象可以携带状态

### 示例 3：闭包功能（捕获状态）

```cpp
#include <iostream>

class Accumulator {
    int total = 0;
public:
    int operator()(int x) {
        total += x;
        return total;
    }
};

int main() {
    Accumulator acc;
    std::cout << acc(5) << std::endl;  // 5
    std::cout << acc(10) << std::endl; // 15
    std::cout << acc(7) << std::endl;  // 22
}
```

函数对象可以像 lambda 一样持有状态（数据），这是普通函数做不到的。

## 5. 函数对象可用于模板参数

### 示例 4：策略模式

```cpp
#include <iostream>

template <typename Strategy>
class Context {
public:
    void execute(int a, int b) {
        Strategy strategy;
        std::cout << strategy(a, b) << std::endl;
    }
};

struct Add {
    int operator()(int a, int b) const {
        return a + b;
    }
};

struct Multiply {
    int operator()(int a, int b) const {
        return a * b;
    }
};

int main() {
    Context<Add> ctx1;
    ctx1.execute(2, 3);  // 输出 5

    Context<Multiply> ctx2;
    ctx2.execute(2, 3);  // 输出 6
}
```

使用函数对象作为模板参数，可以实现高性能、灵活的策略选择（无虚函数开销）。

## 6. 标准库中的函数对象

C++ STL 提供了很多内建的函数对象，定义在 `<functional>` 中：

| 名称 | 含义 |
|------|------|
| `std::plus<T>` | 加法 |
| `std::minus<T>` | 减法 |
| `std::multiplies<T>` | 乘法 |
| `std::divides<T>` | 除法 |
| `std::less<T>` | 小于 |
| `std::greater<T>` | 大于 |
| `std::equal_to<T>` | 相等 |

### 示例 5：std::greater

```cpp
#include <set>
#include <iostream>

int main() {
    std::set<int, std::greater<int>> s = {5, 3, 8, 1};  // 降序排列

    for (int n : s)
        std::cout << n << " ";
}
```

### 以 std::greater<T> 为例子：从定义、实现原理、使用方式等角度，完整剖析 std::greater<T> 为什么是函数对象

#### 一、std::greater<T> 是什么？

它是一个类模板，定义在头文件 `<functional>` 中，属于标准库的函数对象（Function Object）。

**重点：** 它是一个结构体，内部实现了 `operator()`，所以它就是函数对象！

#### 二、std::greater<T> 的源码简化（C++17 前后差不多）

```cpp
namespace std {
    template <typename T = void>
    struct greater {
        constexpr bool operator()(const T& lhs, const T& rhs) const {
            return lhs > rhs;
        }
    };
}
```

**解读：**
- `std::greater<T>` 是一个模板结构体；
- 它重载了函数调用运算符 `operator()`；
- 所以它是一个可以像函数一样使用的对象，也就是函数对象。

**背后的机制：**
- `std::greater<int>()` 创建一个临时对象；
- 这个对象重载了 `operator()(a, b)`；
- 所以 `std::sort` 内部就像调用函数一样使用它：

```cpp
if (comp(a, b)) // comp 就是 std::greater<int>()
```

## 7. 与 Lambda 的关系

lambda 本质上就是一个匿名函数对象！

```cpp
auto add = [](int a, int b) { return a + b; };

// 等价于：
struct LambdaEquivalent {
    int operator()(int a, int b) const {
        return a + b;
    }
};
```

Lambda 就是函数对象的语法糖！现代 C++ 推荐优先使用 lambda，因为它更简洁。

## 8. 与 std::function 结合

函数对象 + `std::function` 可以构建非常灵活的接口：

```cpp
#include <functional>
#include <iostream>

void run(std::function<void()> task) {
    task();  // 统一接口调用
}

int main() {
    run([] { std::cout << "Lambda run!\n"; });

    struct Hello {
        void operator()() const { std::cout << "Hello from functor!\n"; }
    };

    run(Hello{});
}
```

## 9. 捕获状态的 lambda 替代函数对象

```cpp
auto make_adder(int base) {
    return [base](int x) {
        return base + x;
    };
}

int main() {
    auto add5 = make_adder(5);
    std::cout << add5(10) << std::endl;  // 输出 15
}
```

## 总结：函数对象的特点

| 特点 | 描述 |
|------|------|
| 核心特性 | 重载 `operator()`，实现"对象行为函数" |
| 可携带状态 | 可持有数据（如 lambda 捕获变量） |
| 可组合 | 可作为 STL、模板、策略参数 |
| 性能优越 | 可内联、无虚函数开销 |
| 替代方式 | C++11 后推荐使用 lambda 表达式 |
