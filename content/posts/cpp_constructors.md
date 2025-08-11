---
title: "C++构造函数详解：从基础到高级"
date: 2025-05-23
draft: false
description: "深入解析C++中各种构造函数的类型、使用场景和最佳实践，包括默认构造、有参构造、拷贝构造、移动构造和委托构造"
tags: ["C++", "构造函数", "移动语义", "拷贝构造", "委托构造"]
categories: ["C++"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 17
math: true
---

# C++构造函数详解：从基础到高级

## 一、构造函数的种类

### 1. 默认构造函数（Default Constructor）

默认构造函数是无参数或所有参数都有默认值的构造函数。

```cpp
class A {
public:
    A() { cout << "A default"; }  // 默认构造函数
};
```

⚠️ **重要提示**：如果没有定义任何构造函数，编译器会隐式提供一个默认构造函数。

### 2. 有参构造函数（Parameterized Constructor）

有参构造函数接收参数，允许初始化成员变量。

```cpp
class A {
public:
    A(int x) { cout << x; }
};
```

⚠️ **重要提示**：一旦定义了有参构造函数，默认构造函数不会再自动生成！需要手动补上：

```cpp
A() = default;  // 显式声明生成默认构造函数
```

### 3. 拷贝构造函数（Copy Constructor）

拷贝构造函数用于用已有对象初始化新对象。

```cpp
class A {
public:
    A(const A& other) { cout << "copy"; }
};
```

### 4. 移动构造函数（C++11）

移动构造函数接收右值引用的构造函数，用于资源转移。

```cpp
A(A&& other) { cout << "move"; }
```

### 5. 委托构造函数（C++11）

委托构造函数允许在一个构造函数中调用另一个构造函数。

```cpp
A(int x) : A() { cout << x; }  // 委托给默认构造函数
```

## 二、默认构造函数的生成规则

### 1. 自动生成情况

| 情况 | 默认构造函数是否自动生成 |
|------|------------------------|
| 没写任何构造函数 | ✅ 自动生成 |
| 写了带参数的构造函数 | ❌ 不自动生成 |
| 写了拷贝构造函数 | ❌ 不自动生成 |
| 使用 = default 显式声明 | ✅ 手动生成 |
| 使用 = delete 禁用默认构造 | ❌ 编译器不会生成 |

### 2. 示例代码

```cpp
class A {
public:
    A(int) {}
};

int main() {
    A a;  // ❌ 错误：A() 不存在
}
```

## 三、构造函数初始化列表

### 1. 基本用法

```cpp
class B {
    int x;
public:
    B(int val) : x(val) { }  // 推荐写法
};
```

### 2. 初始化列表 VS 构造体内赋值

| 方式 | 说明 |
|------|------|
| 初始化列表 | 在构造函数调用基类/成员构造函数前初始化成员变量 |
| 构造体内赋值 | 在构造函数体中，对成员进行赋值（可能引发额外构造 + 赋值） |

⚠️ **重要提示**：成员变量/基类的构造只能在初始化列表中调用！

## 四、构造顺序与细节规则

### 1. 构造顺序

```cpp
class A { };
class B { };
class C : public B {
    A a;
public:
    C() {}
};
```

构造顺序：
1. 基类（B）
2. 成员变量（A）
3. 派生类（C）

### 2. 成员变量构造顺序

成员变量构造顺序 = 它们的声明顺序，与初始化列表书写顺序无关！

## 五、禁止构造与显式构造

### 1. 禁止默认构造

```cpp
class A {
    A() = delete;
};
```

### 2. 防止隐式转换（explicit）

```cpp
class A {
public:
    explicit A(int);  // 防止 A a = 1; 这种隐式构造
};
```

## 六、构造函数调用案例分析

### 示例 1：只有有参构造

```cpp
class A {
public:
    A(int) {}
};

int main() {
    A a;      // ❌ 错误：没有默认构造函数
    A b(10);  // ✅ 正确
}
```

### 示例 2：强制显式构造

```cpp
class A {
public:
    explicit A(int) {}
};

A a = 10;  // ❌ 错误，不能隐式转换
A b(10);   // ✅ 正确
```

## 七、继承中的构造函数

子类必须显式调用基类无默认构造函数的构造器：

```cpp
class Base {
public:
    Base(int x) {}
};

class Derived : public Base {
public:
    Derived() : Base(42) {}
};
```

❌ 如果 Base 没有默认构造函数，而你忘记初始化，就会报错。

## 八、左值与右值详解

### 1. 基本概念

#### 左值（Lvalue）
- 有名字、有内存地址、可以取地址（&）
- 可以出现在赋值语句的左边或右边
- 有持久性（生命周期）

```cpp
int x = 10;
x = 20;  // x 是左值
int& lref = x;  // 左值引用
```

#### 右值（Rvalue）
- 没有名字、没有持久地址（临时的值）
- 只能出现在赋值语句的右边，不能取地址
- 临时性（用完即销毁）

```cpp
int y = x + 5;     // x + 5 是右值（表达式结果）
int z = 42;        // 42 是右值（字面量）
int&& rref = 3 + 4;  // 右值引用
```

### 2. C++11 引入的值类别扩展

| 类型 | 示例 | 特点 |
|------|------|------|
| 左值（Lvalue） | int a; a = 3; | 有名字，有生命周期 |
| 纯右值（Prvalue） | 3, a + b, "abc" | 临时值，不能取地址 |
| 将亡值（Xvalue） | std::move(a) | 即将被销毁的资源，可以"窃取" |

### 3. 为什么移动构造函数只接受右值？

移动构造函数的本质是"资源窃取"：

```cpp
MyString(MyString&& other);  // 接受右值引用
```

这个构造函数的意义是："我知道你是个即将销毁的对象（右值），我可以放心地偷走你的资源。"

```cpp
MyString a("hello");
MyString b = a;           // 这是左值，只能调用拷贝构造函数
MyString c = std::move(a); // std::move(a) 是将亡值，可以触发移动构造
```

### 4. std::move 的本质

std::move 实际上只是一个类型转换，它把左值强制转换为右值引用：

```cpp
MyString b = std::move(a);  // 以为 std::move() 在"移动"
// 实际效果：
std::move(a) --> (MyString&&)a
```

真正移动资源的，是你的移动构造函数内部逻辑（比如：把指针转移、置空）。

### 5. 值类别识别练习

```cpp
int x = 5;
int& lref = x;          // 左值引用
int&& rref = 3 + 4;     // 右值引用

int y = std::move(x);   // move 生成将亡值，x 仍是左值
```

- x 是左值
- 3 + 4 是纯右值
- std::move(x) 是将亡值（x 本身是左值，move 后转为右值）

## 九、移动构造函数详解

### 1. 定义与使用场景

移动构造函数是 C++11 引入的一种构造函数，用于"窃取"资源而不是复制资源。

```cpp
class MyString {
    char* data;
public:
    // 移动构造函数
    MyString(MyString&& other) noexcept {
        data = other.data;     // 直接"窃取"指针
        other.data = nullptr;  // 原对象不能 delete 了
        cout << "移动构造\n";
    }
};
```

### 2. noexcept 关键字

noexcept 表示函数不会抛出异常，这对于移动构造函数特别重要：

- 如果移动构造函数有 noexcept，STL 容器会优先使用移动操作
- 如果没有 noexcept，STL 可能会退而使用拷贝构造
- 很多标准库实现要求移动构造/移动赋值操作必须是 noexcept 的

### 3. 移动语义的本质

移动操作会破坏原对象的内容（资源），但不会让对象失效：

```cpp
MyString a("hello");
MyString b = std::move(a); // a 的资源移动到 b

// 此时：
a.data == nullptr;         // a 变成空壳
b.data -> 指向 "hello"

// 但 a 还是一个合法对象，可以被析构，可以再赋值
a = MyString("new content"); // a 可以被重新赋值
```

## 十、Rule of Five（五法则）

对于管理动态分配资源的类，需要实现以下五个特殊成员函数：

1. 析构函数
2. 拷贝构造函数
3. 拷贝赋值运算符
4. 移动构造函数
5. 移动赋值运算符

### 完整示例：MyString 类

```cpp
class MyString {
private:
    char* data;

public:
    // 构造函数
    MyString(const char* str = "") {
        data = new char[strlen(str) + 1];
        strcpy(data, str);
    }

    // 析构函数
    ~MyString() {
        delete[] data;
    }

    // 拷贝构造函数
    MyString(const MyString& other) {
        data = new char[strlen(other.data) + 1];
        strcpy(data, other.data);
    }

    // 拷贝赋值运算符
    MyString& operator=(const MyString& other) {
        if (this == &other) return *this;
        delete[] data;
        data = new char[strlen(other.data) + 1];
        strcpy(data, other.data);
        return *this;
    }

    // 移动构造函数
    MyString(MyString&& other) noexcept {
        data = other.data;
        other.data = nullptr;
    }

    // 移动赋值运算符
    MyString& operator=(MyString&& other) noexcept {
        if (this == &other) return *this;
        delete[] data;
        data = other.data;
        other.data = nullptr;
        return *this;
    }
};
```

## 十一、总结

1. 构造函数是类对象初始化的关键
2. 默认构造函数的生成规则需要特别注意
3. 初始化列表比构造体内赋值更高效
4. 移动语义可以显著提升性能
5. 资源管理类需要实现完整的五法则

---

*参考文献：*
1. "The C++ Programming Language" by Bjarne Stroustrup
2. "Effective Modern C++" by Scott Meyers
3. "C++ Primer" by Stanley Lippman
4. "C++ Templates: The Complete Guide" by David Vandevoorde
5. "C++ Concurrency in Action" by Anthony Williams 
