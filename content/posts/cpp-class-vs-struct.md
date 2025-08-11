---
title: "全面解析C++中类(class)与结构体(struct)的区别"
date: 2025-04-26
author: "徐亚飞"
categories: ["C++", "编程语言", "教程"]
tags: ["C++"]
description: "深入探讨C++中类(class)与结构体(struct)的区别"
toc: true
toc_sticky: true
comments: true
---

# 全面解析C++中类(class)与结构体(struct)的区别

## 一、最核心区别：默认访问控制

在C++中，`class`和`struct`的唯一语法区别在于默认访问权限：

```cpp
// 结构体示例
struct MyStruct {
    int x;       // 默认public访问权限
    void foo() {} // 默认public
};

// 类示例
class MyClass {
    int x;       // 默认private访问权限
    void bar() {} // 默认private
};
```

### 继承时的默认权限
```cpp
struct D1 : Base {};   // 默认public继承
class D2 : Base {};    // 默认private继承
```

## 二、历史起源与设计哲学

| 特性 | struct (结构体) | class (类) |
|------|----------------|------------|
| 诞生时间 | 源自C语言 | C++新增概念 |
| 设计初衷 | 数据打包聚合 | 面向对象封装 |
| 核心理念 | "这是一个数据集合" | "这是一个具有行为的对象" |

## 三、实际开发中的惯用准则

### 应该使用struct的场景

1. 纯数据集合
```cpp
struct Color {
    uint8_t r, g, b, a;  // 全部公有
};
```

2. 简单值类型
```cpp
struct Point {
    double x, y;
    // 可以包含简单方法
    double distance() const { return sqrt(x*x + y*y); }
};
```

3. 接口配置参数
```cpp
struct Config {
    string title;
    int width;
    int height;
};
```

### 应该使用class的场景

1. 需要封装的业务对象
```cpp
class BankAccount {
private:
    string owner_;
    double balance_;
public:
    void deposit(double amount) { /*...*/ }
    bool withdraw(double amount) { /*...*/ }
};
```

2. 需要复杂生命周期的资源管理
```cpp
class DatabaseConnection {
    Connection* conn_;
public:
    explicit DatabaseConnection(string url) { /*...*/ }
    ~DatabaseConnection() { /* 自动释放资源 */ }
};
```

3. 需要多态继承的体系
```cpp
class Shape {
public:
    virtual double area() const = 0;
};
```

## 四、技术能力完全对比

| 语言特性 | struct支持情况 | class支持情况 | 示例代码 |
|----------|----------------|---------------|----------|
| 成员变量 | ✓ | ✓ | `int x;` |
| 成员函数 | ✓ | ✓ | `void f() {}` |
| 访问控制 | ✓ | ✓ | `public:` |
| 构造函数/析构函数 | ✓ | ✓ | `~T() {}` |
| 运算符重载 | ✓ | ✓ | `T operator+()` |
| 继承 | ✓ | ✓ | `struct D : B {};` |
| 虚函数 | ✓ | ✓ | `virtual void f() = 0;` |
| 友元 | ✓ | ✓ | `friend class F;` |
| 模板 | ✓ | ✓ | `template<typename T>` |

## 五、模板元编程中的差异实践

### struct在元编程中的优势
```cpp
// 类型特征检查通常用struct实现
template<typename T>
struct is_pointer {
    static constexpr bool value = false;
};

template<typename T>
struct is_pointer<T*> {
    static constexpr bool value = true;
};

// 使用示例
static_assert(is_pointer<int*>::value, "必须是指针类型");
```

### 原因分析
1. 元编程通常需要公开所有成员
2. 避免频繁写public关键字
3. 符合"数据即接口"的元编程哲学

## 六、内存布局完全一致
```cpp
struct S {
    int a;
    double b;
};

class C {
    int a;
    double b;
};

// 验证内存布局相同
static_assert(sizeof(S) == sizeof(C));
static_assert(offsetof(S, b) == offsetof(C, b));
```

### 继承时的特殊情况
```cpp
struct A { int x; };
class B : A { int y; };  // 私有继承可能影响空基类优化
```

## 七、与C语言的兼容性细节

| 特性 | C struct | C++ struct |
|------|----------|------------|
| 类型声明 | 必须带struct关键字 | 可直接作为类型名 |
| 成员函数 | 不支持 | 支持 |
| 访问控制 | 无 | 支持 |
| 静态成员 | 不支持 | 支持 |

### C/C++混合编程注意事项：
```cpp
#ifdef __cplusplus
extern "C" {
#endif

// 确保C兼容的布局
struct CCompatStruct {
    int x;
    float y;
};

#ifdef __cplusplus
}
#endif
```

## 八、现代C++中的最佳实践

1. 结构化绑定(struct适用)
```cpp
struct Employee {
    string name;
    int id;
    double salary;
};

auto [name, id, salary] = getEmployee(); // C++17结构化绑定
```

2. 类的不变量维护(class适用)
```cpp
class Temperature {
    double kelvin_;
public:
    void setCelsius(double c) {
        kelvin_ = c + 273.15;
        assert(kelvin_ > 0 && "绝对温度不能为负");
    }
};
```

3. 移动语义支持(两者均可)
```cpp
struct Buffer {
    vector<uint8_t> data;
    Buffer(Buffer&& other) noexcept : data(std::move(other.data)) {}
};

class FileHandle {
    FILE* handle_;
public:
    FileHandle(FileHandle&& other) : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
};
```

## 九、完整特性对比表格

| 对比维度 | struct | class |
|----------|--------|-------|
| **基本性质** | | |
| 关键字 | struct | class |
| 默认访问权限 | public | private |
| 默认继承方式 | public | private |
| **设计用途** | | |
| 数据聚合 | 首选 | 可用但不惯用 |
| 对象封装 | 可用但不惯用 | 首选 |
| 接口定义 | 适合POD接口 | 适合抽象接口 |
| **语法特性** | | |
| 成员函数 | 支持 | 支持 |
| 虚函数 | 支持 | 支持 |
| 友元声明 | 支持 | 支持 |
| **其他特性** | | |
| 模板元编程 | 更常用 | 较少使用 |
| C兼容性 | 部分兼容 | 不兼容 |
| 内存布局 | 与class相同 | 与struct相同 |
| 结构化绑定 | 天然适合 | 需要显式tuple接口 |

## 十、经典面试题解析

### Q1：以下代码有何问题？
```cpp
class Circle {
    double radius;
public:
    double area() const { return 3.14 * radius * radius; }
};

struct Square {
    double side;
    double area() const { return side * side; }
};
```

**答案：**
* 从技术上讲没有问题
* 但从设计角度看：
    * Circle将数据隐藏是合理的
    * Square作为简单的几何图形，使用struct更合适

### Q2：为什么STL中用struct实现迭代器特性？
```cpp
template<class Iterator>
struct iterator_traits {
    using value_type = typename Iterator::value_type;
    // ...
};
```

**答案：**
1. 特性类需要所有成员公开
2. 避免频繁写public关键字
3. 符合"特性即数据"的元编程哲学

## 总结选择策略

1. 默认选择class当：
    * 需要维护不变量的类型
    * 需要复杂生命周期的对象
    * 需要多态继承的体系

2. 默认选择struct当：
    * 纯数据集合
    * 简单的值类型
    * 需要与C交互的数据结构
    * 模板元编程场景

3. 永远保持一致：
    * 同一个项目中保持统一风格
    * 在混合使用时明确标注原因
