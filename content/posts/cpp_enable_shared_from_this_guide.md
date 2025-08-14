---
title: "C++ std::enable_shared_from_this 完全指南：原理、使用与实战"
date: 2025-08-01
description: "深入解析C++中std::enable_shared_from_this的原理、作用、使用场景及注意事项，包含详细示例和错误分析"
tags: ["C++", "智能指针", "shared_ptr", "enable_shared_from_this", "内存管理", "异步编程"]
categories: ["C++"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 20
math: true
---

# C++ std::enable_shared_from_this 完全指南：原理、使用与实战

下面我来深入全面讲解 `std::enable_shared_from_this` 的原理、作用、使用场景及注意事项，并结合你提供的代码进行详细分析。

---

## 一、std::enable_shared_from_this 是什么？

`std::enable_shared_from_this<T>` 是一个标准库模板类，允许类**安全地在自身内部获取一个指向自己的 `shared_ptr<T>` 实例**。

### 为什么需要它？

我们知道，`shared_ptr<T>` 用于管理对象的生命周期，而有时你可能希望在类的成员函数中返回 `shared_ptr<this>`，但如果直接用 `shared_ptr<T>(this)`，会造成严重的错误，因为这样会导致**引用计数混乱甚至双重释放**。

**正确方式：**继承 `enable_shared_from_this<T>`，然后使用 `shared_from_this()` 获取当前对象的 `shared_ptr`。

## 二、示例解析

```cpp
#include <memory>
#include <iostream>

class MyClass : public std::enable_shared_from_this<MyClass> {
public:
    std::shared_ptr<MyClass> getSelf() {
        return shared_from_this();  //  获取一个共享指针，指向自身
    }
};

int main() {
    auto obj = std::make_shared<MyClass>();   // 创建对象，use_count = 1
    auto self = obj->getSelf();               // 再获得一个 shared_ptr，use_count = 2

    std::cout << "use_count: " << self.use_count() << std::endl;  // 输出 2
}
```

**解释：**
- `MyClass` 继承了 `std::enable_shared_from_this<MyClass>`，这使得类内部可以安全调用 `shared_from_this()`。
- `shared_from_this()` 实际上返回的是一个新的 `shared_ptr`，但它和原来的共享的是同一个控制块，不会导致重复析构。

## 错误用法对比

如果你没有通过 `std::make_shared` 来创建对象，而是在栈上创建，或者直接 `new` 一个对象然后 `shared_ptr` 管理，就会出错：

```cpp
int main() {
    MyClass* raw = new MyClass();        //  没有构建控制块
    auto self = raw->getSelf();          //  undefined behavior，程序崩溃
}
```

**错误原因：**
- `shared_from_this()` 只有在这个对象最初就是通过 `shared_ptr` 管理（比如 `make_shared`）创建时才合法。
- 否则，`shared_from_this()` 内部找不到共享控制块，调用会抛出 `std::bad_weak_ptr` 异常或者直接崩溃。

## 原理小科普：enable_shared_from_this 内部结构

```cpp
template<class T>
class enable_shared_from_this {
protected:
    mutable std::weak_ptr<T> weak_this;
public:
    std::shared_ptr<T> shared_from_this() const {
        return std::shared_ptr<T>(weak_this);
    }
};
```

- 当你用 `make_shared<MyClass>()` 创建对象时，系统会自动设置 `weak_this`。
- `shared_from_this()` 实际上是用 `weak_ptr` 来构造一个新的 `shared_ptr`。

## 典型使用场景

| 场景 | 说明 |
|------|------|
| 回调/观察者模式 | 成员函数中需要传出 `shared_ptr<this>` |
| 异步操作 | 对象在执行异步任务过程中不想被提前释放（保活） |
| 工厂模式中 this 引用传递 | 工厂返回对象时需要返回 `shared_ptr<this>` |
| 类作为某个资源池成员，需要自保引用 | 例如音视频中的 Frame 缓存池 |

## 实战举例：防止异步中对象被提前销毁

```cpp
class Task : public std::enable_shared_from_this<Task> {
public:
    void runAsync() {
        std::shared_ptr<Task> self = shared_from_this();
        std::thread([self]() {
            std::this_thread::sleep_for(std::chrono::seconds(1));
            std::cout << "Task still alive!\n";
        }).detach();
    }
};

int main() {
    auto task = std::make_shared<Task>();
    task->runAsync();

    std::this_thread::sleep_for(std::chrono::seconds(2));
}
```

- 在异步线程中使用 `shared_from_this()` 保证对象不会过早析构。
- 非常适合你的音视频开发场景（例如视频帧在后台异步处理时）。

## 总结

| 特性 | 说明 |
|------|------|
| 安全地在类内部生成 `shared_ptr<this>` | 使用 `shared_from_this()` |
| 需要继承 `enable_shared_from_this<T>` | 否则会抛异常或崩溃 |
| 只能在由 `shared_ptr` 管理的对象上使用 | 不能栈上、不能裸指针 |
| 原理基于内部 `weak_ptr` | 避免循环引用 |
