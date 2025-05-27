---
title: "C++访问控制详解"
date: 2025-05-27
draft: false
description: "深入解析C++访问控制机制，从基础概念到高级应用，从原理到实践"
tags: ["C++", "面向对象", "访问控制", "封装", "继承"]
categories: ["编程语言"]
author: "徐亚飞"
showToc: true
TocOpen: true
weight: 19
math: true
---

# C++访问控制详解

## 一、基本概念

### 1. 访问控制修饰符

C++ 中的类成员默认是 private，结构体成员默认是 public。

三种访问控制修饰符：

| 修饰符 | 类内访问 | 派生类访问 | 类外访问 |
|--------|----------|------------|----------|
| public | ✅ | ✅ | ✅ |
| protected | ✅ | ✅ | ❌ |
| private | ✅ | ❌ | ❌ |

### 2. 访问控制与继承

派生类是否可以访问基类的成员，还受到**继承方式（public/protected/private）**的影响。

| 继承方式 | 基类 public 成员在派生类中变为 | 基类 protected 成员变为 | 基类 private 成员变为 |
|----------|-------------------------------|------------------------|----------------------|
| public | public | protected | 不可访问 |
| protected | protected | protected | 不可访问 |
| private | private | private | 不可访问 |

示例：
```cpp
class Base {
public:    int a;
protected: int b;
private:   int c;
};

class Derived1 : public Base {
    // a 是 public，b 是 protected，c 无法访问
};

class Derived2 : protected Base {
    // a 和 b 都变为 protected，c 无法访问
};

class Derived3 : private Base {
    // a 和 b 都变为 private，c 无法访问
};
```

## 二、访问控制详解

### 1. public 公有成员

#### 特点
- 表示这个成员对所有人都可见：类自身、派生类、外部代码
- 常用于类的接口函数（如 getX()、setX()），是对外的"公共入口"

#### 应用场景
1. 接口类中的函数：
```cpp
class Animal {
public:
    virtual void speak() = 0; // 公共接口
};
```

2. 用户调用方法或访问数据时：
```cpp
obj.print();
```

### 2. protected 受保护成员

#### 基本定义
protected 成员只能被：
- 当前类访问
- 其派生类访问
- 类外无法访问

它介于 private（更封闭）和 public（完全开放）之间。

#### 访问权限详解

1. protected 成员的访问权限如下：

| 访问者 | private | protected | public |
|--------|---------|-----------|--------|
| 本类 | ✅ | ✅ | ✅ |
| 子类 | ❌ | ✅ | ✅ |
| 外部类或对象 | ❌ | ❌ | ✅ |

所以 protected 成员：
- ✅ 子类可以访问
- ❌ 外部不能访问（包括 main 函数、别的类、用户代码）
- ✅ 本类自然可以访问

#### 使用场景

1. 让子类复用父类实现（但不暴露给外部）：
```cpp
class Base {
protected:
    int internalCounter = 0;

public:
    void reset() { internalCounter = 0; }
};

class Derived : public Base {
public:
    void increment() { internalCounter++; }  // 可以访问
};
```

2. 在模板方法模式中保护基础逻辑：
```cpp
class Game {
public:
    void run() {
        init();    // 固定的初始化流程
        play();    // 留给子类决定怎么实现
        cleanup(); // 固定的收尾流程
    }

protected:
    virtual void init() { std::cout << "Base init\n"; }
    virtual void play() = 0;
    virtual void cleanup() { std::cout << "Base cleanup\n"; }
};

class Chess : public Game {
protected:
    void play() override {
        std::cout << "Playing chess\n";
        // init(); // ✅ 可以写，但设计上不推荐这么做
    }
};
```

#### 受控开放（Controlled Extensibility）

"子类内部可以访问 init()，为什么说不能调用？"

✅ 答案是：语法上可以调用，但设计上"不建议"调用

这正是面向对象设计中的一个重要思想："受控开放（Controlled Extensibility）"。

🚧 为什么说是"受控开放"？
- run() 是父类控制的流程，父类负责"何时"调用 init()
- 虽然子类可以访问 init()（因为是 protected），但子类不应该改变流程的调用时机，否则就打破了框架设计的封装性

🧠 举个现实例子理解：

你有一个电饭锅（父类），内部流程是：
1. 加热（init）
2. 蒸煮（play）
3. 保温（cleanup）

你继承它做了一个"智能电饭锅"（子类）——你可以重写"蒸煮"的逻辑，但你不能插手何时开始加热、保温。

如果你在 play() 里面又调用一次 init()，你其实在"偷偷重新加热"，破坏了流程，结果可能"煮坏饭"。

✅ 总结你的疑问：
| 问题 | 回答 |
|------|------|
| 子类能不能调用 init()？ | ✅ 语法上可以调用，因为是 protected，对子类可见 |
| 为什么说"子类不能调用 init()"？ | ❌ 不是语法限制，而是设计上的约束，避免子类破坏父类设计好的流程 |
| 谁能调用 init()？ | Game 类自己可以，Chess 子类也可以——但设计上只允许 run() 控制它 |
| 什么是"受控开放"？ | 父类允许子类扩展部分行为（如 play()），但保留流程控制权（如 init()） |

🔍 如果你硬要让 init() 不能被子类调用？
那就把它设为 private，但这样子类连 override 都做不了，这就牺牲了灵活性。因此我们用 protected，是一种对子类开放、但约定俗成要遵守使用边界的方式。

这种设计模式（称为模板方法模式）正好是面向对象三大特性——封装、继承、多态的经典应用体现之一。下面我们来深入讲清楚这个设计模式，并结合 OOP 三大特性逐步解析：

🎯 一、什么是模板方法模式？

模板方法模式定义：
在基类中定义一个算法的骨架（固定的执行流程），而将某些步骤延迟到子类中实现。
基类控制整体流程，子类负责实现细节，流程不可更改，细节可扩展。

🔧 二、结构原型代码（经典案例）

```cpp
class Game {
public:
    void run() {
        init();       // 第一步：初始化
        play();       // 第二步：进行游戏（延迟到子类）
        cleanup();    // 第三步：清理资源
    }

protected:
    virtual void init()    { std::cout << "Game init\n"; }
    virtual void play() = 0;  // 抽象方法，强制子类实现
    virtual void cleanup() { std::cout << "Game cleanup\n"; }
};

class ChessGame : public Game {
protected:
    void play() override {
        std::cout << "Playing Chess\n";
    }
};

// 使用：
int main() {
    Game* g = new ChessGame();
    g->run(); // 只暴露一个 run()，调用流程由父类控制
    delete g;
}
```

🧱 三、和 OOP 三大特性关系详解

1. ✅ 封装（Encapsulation）
   - 封装的是**"整体流程"**，不允许外部和子类干扰 run() 方法中固定的执行顺序
   - init()、cleanup() 甚至 run() 都可以设置为 protected / private，暴露最小的接口（如 run() 为 public，其余为 protected），隐藏实现细节、对外暴露统一入口

🟩 封装体现：
   - 对外只暴露 run()
   - 内部怎么初始化和清理，外部无权干涉

2. ✅ 继承（Inheritance）
   - 子类继承父类 Game
   - 利用继承来扩展具体的 play() 实现，支持多种不同玩法，如 ChessGame、FootballGame 等

🟩 继承体现：
   - 复用父类代码结构
   - 子类扩展定制化逻辑

3. ✅ 多态（Polymorphism）
   - play() 是 virtual 的（甚至纯虚函数），父类指针 Game* 可以指向任何具体子类
   - 运行时根据对象类型动态调用 play()，达到多态效果

🟩 多态体现：
   - Game* g = new ChessGame();
   - 调用 g->run() 时，内部实际调用的是子类 ChessGame 的 play() 实现

📦 四、模板方法模式的优点总结

| 优点 | 解释 |
|------|------|
| ✅ 统一流程控制 | 父类封装整个算法结构，子类不能乱改顺序，确保流程一致性 |
| ✅ 扩展灵活 | 子类可以自由定制 play() 的细节 |
| ✅ 避免重复代码 | 公共的 init() 和 cleanup() 由父类统一实现 |
| ✅ 支持开闭原则 | 添加新玩法只需新建子类，不动原有父类代码 |

🧠 五、类比现实世界帮助理解

想象一个"点外卖"的流程（父类控制）：
```cpp
class OrderFood {
public:
    void order() {
        selectRestaurant();
        chooseFood();      // 不同人点的菜不同
        pay();
    }

protected:
    void selectRestaurant() { std::cout << "Choose restaurant\n"; }
    virtual void chooseFood() = 0;
    void pay() { std::cout << "Pay for food\n"; }
};

// 不同人继承这个流程定制 chooseFood()：
class AliceOrder : public OrderFood {
    void chooseFood() override { std::cout << "Alice orders sushi\n"; }
};

class BobOrder : public OrderFood {
    void chooseFood() override { std::cout << "Bob orders burger\n"; }
};
```

你只管调用 order()，点餐流程就全跑完了，外部不需要知道细节，这就叫模板方法模式的魅力。

🧭 六、模板方法模式应用场景
- 游戏开发：通用流程，扩展玩法
- 网络请求：模板封装请求流程、回调子类处理响应
- 框架设计：如 Qt、Java 的 GUI 框架中 paintEvent() 就是典型的模板方法模式

🔚 总结一张表：OOP 特性如何在模板方法中体现

| 特性 | 模板方法中的体现 |
|------|-----------------|
| 封装 | run() 封装整体流程，对外隐藏细节 |
| 继承 | 子类继承父类结构，自定义部分逻辑 |
| 多态 | 子类实现虚函数，运行时动态决定调用哪一版 |

3. 避免"朋友外人"乱改状态：
```cpp
class BankAccount {
protected:
    double balance;

public:
    BankAccount(double initial) : balance(initial) {}
    virtual void deposit(double amount) { balance += amount; }
};

class PremiumAccount : public BankAccount {
public:
    PremiumAccount(double initial) : BankAccount(initial) {}
    void bonusInterest() { balance += balance * 0.05; }  // 访问受保护的 balance
};
```

### 3. private 私有成员

#### 特点
- 表示成员只能被当前类访问，外部和派生类都不能访问
- 是最严格的访问控制，通常用于保护内部数据，防止误用

#### 应用场景
1. 成员变量几乎总是设为 private，以实现封装：
```cpp
class User {
private:
    std::string password;

public:
    void setPassword(const std::string& pwd);
};
```

2. 阻止派生类访问内部实现细节：
```cpp
class Connection {
private:
    void openSocket(); // 外部和子类都不能调用
};
```

## 三、友元机制

### 1. 基本概念

C++ 中可以通过 friend 声明，使某些函数或类可以访问另一个类的私有/受保护成员。

```cpp
class A {
private:
    int secret = 42;

    friend void reveal(A&);
};

void reveal(A& a) {
    std::cout << a.secret << std::endl;  // 合法
}
```

### 2. 应用场景

#### 1. 操作符重载（非成员）
```cpp
class Vector {
private:
    int x, y;

public:
    Vector(int x, int y) : x(x), y(y) {}

    friend Vector operator+(const Vector& a, const Vector& b);
};

Vector operator+(const Vector& a, const Vector& b) {
    return Vector(a.x + b.x, a.y + b.y);  // 可以访问 private 成员
}
```

#### 2. 辅助类（Builder、Iterator）访问内部状态
```cpp
class House;

class HouseBuilder {
public:
    House create();
};

class House {
private:
    std::string wallType;
    int windows;

    friend class HouseBuilder;
};
```

### 3. 友元的注意点

| 特点 | 说明 |
|------|------|
| 不受继承限制 | friend 不会被继承给子类 |
| 单向关系 | A 是 B 的 friend，不代表 B 是 A 的 friend |
| 会破坏封装 | 滥用会导致类之间高度耦合，失去模块边界 |

### 4. friend vs protected 区别

| 特性 | protected | friend |
|------|-----------|--------|
| 可访问权限 | 类和子类 | 被授权的类/函数 |
| 是否继承可见 | ✅ | ❌ |
| 是否破坏封装 | 否（较弱） | 是（较强） |
| 使用典型场景 | 继承设计 / 模板方法 / 抽象类 | 操作符重载 / 辅助类 / 高效协作类设计 |

## 四、设计建议与最佳实践

### 1. 访问控制的设计建议

| 类成员 | 建议使用的访问控制 | 原因 |
|--------|-------------------|------|
| 成员变量 | private（或 protected） | 封装、控制访问 |
| 公共 API 接口 | public | 暴露外部接口 |
| 辅助函数 | private | 仅限类内部调用 |
| 可被子类重用的基础功能 | protected | 子类共享但不对外暴露 |

### 2. 常见设计陷阱与误区

1. 误将数据成员设为 public
   - 会破坏封装性，使对象数据暴露在外部，易于出错
   - 改进方法：使用 getter/setter

2. 将所有函数设为 public
   - 会暴露太多无关内部逻辑，导致接口复杂、难维护

3. 滥用 protected
   - 只有当你确信子类需要访问时才使用，否则就应当 private

### 3. 实战建议

#### 什么时候用 protected？
- 你希望子类扩展/控制行为，但又不希望外部使用
- 比如状态、模板方法的步骤、受保护工具函数等

#### 什么时候用 friend？
- 函数需要跨类访问私有数据，但不希望改变数据的所有者
- 比如双向关联类、操作符重载、构建器类、调试工具等

## 五、总结

### 核心概念回顾
- private：最严格的访问控制，仅类内可访问
- protected：类内和派生类可访问，外部不可访问
- public：完全开放，所有地方都可访问
- friend：特殊机制，允许指定外部访问私有成员

### 最佳实践
1. 优先使用 private 保护数据
2. 谨慎使用 protected，只在确实需要时使用
3. 合理使用 public 暴露接口
4. 谨慎使用 friend，避免破坏封装

### 一句话总结
- private 是"最安全"
- protected 是"给子类留口子"
- public 是"对外的窗口"
- friend 是"把钥匙交给别人"

---

*参考文献：*
1. "The C++ Programming Language" by Bjarne Stroustrup
2. "Effective C++" by Scott Meyers
3. "C++ Primer" by Stanley Lippman
4. "Design Patterns" by Erich Gamma
5. "Clean Code" by Robert C. Martin 
