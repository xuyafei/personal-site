---
title: "C++ 异常处理完全指南：从基础到高级实践详解"
description: "深入理解C++异常处理机制，包含try-catch语法、栈展开、RAII原则、noexcept关键字和最佳实践的完整教程"
keywords: ["C++", "异常处理", "try-catch", "栈展开", "RAII", "noexcept", "std::exception", "函数try块"]
author: "徐亚飞"
date: "2025-08-10"
categories: ["C++"]
tags: ["C++", "异常处理", "错误处理", "RAII", "内存管理", "最佳实践"]
difficulty: "中级到高级"
reading_time: "30分钟"
prerequisites: ["C++基础", "面向对象编程", "内存管理基础"]
---

# C++ 异常处理完全指南：从基础到高级实践

## 核心结论

C++ 的异常机制主要由**抛出异常（throw）**和**捕获异常（catch）**组成。异常可以用来处理非预期错误，让代码从错误现场跳到安全的地方继续运行。

## 1. 基本语法与流程

### 1.1 核心语法结构

```cpp
try {
    // 可能抛异常的代码
} catch (const Derived& e) {
    // 先写具体/派生类型
} catch (const Base& e) {
    // 再写基类
} catch (...) {
    // 兜底：未知类型
}
```

**关键要点**：
- `throw 表达式;` 用来抛出异常（任何类型的对象都能被抛出，推荐抛标准/自定义异常类）
- 捕获顺序很重要：派生类在前，基类在后，否则会被基类抢先匹配
- 按 `const` 引用捕获（如 `const std::exception&`）避免切片与拷贝
- 重新抛出：在 catch 内用 `throw;`（不要写 `throw e;`，后者会复制，可能改变动态类型）

### 1.2 基本示例

```cpp
#include <iostream>
#include <stdexcept>

void mayFail(int x) {
    if (x == 0)
        throw std::runtime_error("Division by zero!");
    std::cout << "10 / " << x << " = " << 10 / x << "\n";
}

int main() {
    try {
        mayFail(2);
        mayFail(0); // 这里会抛异常
        mayFail(5); // 不会执行到这里
    }
    catch (const std::runtime_error& e) {
        std::cerr << "Caught exception: " << e.what() << "\n";
    }
    catch (...) {
        std::cerr << "Caught unknown exception\n";
    }

    std::cout << "Program continues...\n";
}
```

**要点**：
- `catch (const std::runtime_error& e)` 按类型捕获异常
- `catch (...)` 捕获所有异常
- 一旦 `throw` 被触发，会跳过当前函数剩余的代码，进入最近的 catch

## 2. 栈展开（Stack Unwinding）与对象生命周期

### 2.1 什么是栈展开？

栈展开是指当异常被抛出时，C++运行时系统会沿着函数调用链（调用栈）从当前执行点开始向上回溯，依次退出（析构）栈上的局部对象，直到找到能够处理该异常的catch块为止的过程。

### 2.2 栈展开的具体行为

1. **局部对象析构**：在栈展开过程中，所有已经构造的局部对象（在离开其作用域时）会被自动析构
2. **清理栈帧**：每个函数的栈帧会被依次清理
3. **查找处理程序**：系统会查找匹配的catch块

### 2.3 详细示例说明

```cpp
#include <iostream>
#include <stdexcept>

class Test {
public:
    Test(int id) : id(id) { std::cout << "Constructing " << id << "\n"; }
    ~Test() { std::cout << "Destructing " << id << "\n"; }
private:
    int id;
};

void func3() {
    Test t3(3);
    throw std::runtime_error("Exception from func3");
    // t3会被析构 - 栈展开时执行
}

void func2() {
    Test t2(2);
    func3();
    // t2会被析构 - 栈展开时执行
}

void func1() {
    Test t1(1);
    func2();
    // t1会被析构 - 正常情况下执行
}

int main() {
    try {
        func1();
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << "\n";
    }
    return 0;
}
```

**输出结果**：
```
Constructing 1
Constructing 2
Constructing 3
Destructing 3  // 异常抛出后首先析构t3
Destructing 2  // 然后析构t2
Caught exception: Exception from func3
Destructing 1  // t1在try块正常退出时析构
```

### 2.4 详细解释为什么 Destructing 2 出现在 Caught exception 之前

**调用栈和异常处理流程**：

1. **函数调用顺序**：
   - `main()` 调用 `func1()`
   - `func1()` 调用 `func2()`
   - `func2()` 调用 `func3()`
   - `func3()` 抛出异常

2. **异常抛出时的栈展开顺序**：
   - 异常首先在 `func3()` 中抛出
   - 运行时系统开始从 `func3()` 向上查找匹配的 catch 块
   - 第一步：离开 `func3()` 的作用域，析构 t3（输出 Destructing 3）
   - 第二步：离开 `func2()` 的作用域，析构 t2（输出 Destructing 2）
   - 第三步：离开 `func1()` 的作用域，但在 `func1()` 之前就已经进入了 try 块（在 `main()` 中），所以 `func1()` 的栈帧被跳过（因为异常已经被 `main()` 的 catch 捕获）
   - 第四步：在 `main()` 中捕获异常（输出 Caught exception）
   - 最后：try 块正常结束，`func1()` 返回，t1 被析构（输出 Destructing 1）

**关键点解释**：
- t2 在 catch 之前析构是因为：
  - `func3()` 抛出异常后，控制权要返回到 `main()` 的 catch 块
  - 必须首先退出 `func2()` 和 `func3()` 的作用域（栈展开）
  - 因此 t2 和 t3 必须在异常被捕获前析构
- t1 在 catch 之后析构是因为：
  - `func1()` 是在 try 块内调用的
  - 当异常被捕获后，try 块已经结束，控制流继续执行 catch 块之后的代码
  - `func1()` 的栈帧此时才被正常清理，t1 被析构

### 2.5 栈展开与RAII

**重要注意事项**：
1. **资源管理**：栈展开确保了即使发生异常，局部资源也能被正确释放
2. **RAII原则**：这是为什么C++推荐使用RAII（Resource Acquisition Is Initialization）模式管理资源
3. **异常安全**：栈展开是实现异常安全的基础
4. **不可捕获异常**：如果没有找到匹配的catch块，`std::terminate()`会被调用

```cpp
struct File {
    FILE* f;
    File(const char* path) : f(std::fopen(path, "rb")) {
        if (!f) throw std::runtime_error("open failed");
    }
    ~File() { if (f) std::fclose(f); } // 有异常时也会执行
};
```

## 3. 标准异常体系

### 3.1 标准异常层次结构

都继承自 `std::exception`，可用 `what()` 获得简短错误信息：

- **逻辑错误**：`std::logic_error`（`invalid_argument`, `out_of_range`…）
- **运行期错误**：`std::runtime_error`（`system_error`, `filesystem_error`…）
- **内存**：`std::bad_alloc`（`new` 失败时抛出）
- **其他**：`std::bad_cast`, `std::bad_function_call` …

**建议**：抛/捕都围绕 `std::exception` 家族，避免抛 `int`、字符串字面量这类"非异常对象"。

### 3.2 常用标准异常示例

```cpp
#include <stdexcept>
#include <string>

void processString(const std::string& str) {
    if (str.empty()) {
        throw std::invalid_argument("String cannot be empty");
    }
    
    if (str.length() > 100) {
        throw std::length_error("String too long");
    }
    
    // 处理字符串...
}

void accessArray(const std::vector<int>& vec, size_t index) {
    if (index >= vec.size()) {
        throw std::out_of_range("Index out of bounds");
    }
    
    // 访问数组元素...
}
```

## 4. 自定义异常

### 4.1 自定义异常的写法

```cpp
struct MediaOpenError : std::runtime_error {
    using std::runtime_error::runtime_error; // 继承构造
};

// 使用
throw MediaOpenError("device=cam0, reason=busy");
```

**设计原则**：
- 继承自 `std::runtime_error`（或合适的基类）
- 仅保存必要上下文信息（如设备名、错误码）

### 4.2 更复杂的自定义异常

```cpp
class DatabaseError : public std::runtime_error {
public:
    enum class ErrorCode {
        ConnectionFailed,
        QueryFailed,
        TransactionFailed
    };
    
    DatabaseError(ErrorCode code, const std::string& message)
        : std::runtime_error(message), error_code_(code) {}
    
    ErrorCode getErrorCode() const { return error_code_; }
    
private:
    ErrorCode error_code_;
};

// 使用示例
void connectDatabase() {
    // 模拟连接失败
    throw DatabaseError(DatabaseError::ErrorCode::ConnectionFailed, 
                       "Failed to connect to database server");
}
```

## 5. 函数try块（Function-try-block）

### 5.1 作用与语法

**作用**：
- 在构造函数的初始化列表执行之前捕获异常
- 用来统一处理构造期间抛出的异常（包括基类构造、成员初始化）
- 常见于类的构造函数（尤其是成员初始化依赖外部资源时）

**语法**：
```cpp
ClassName::ClassName(args) try : member(init), ...
{
    // 构造体内代码
} catch(...) {
    // 捕获任何构造期异常
}
```

### 5.2 为什么需要函数try块？

**普通 try 捕不到初始化列表里的异常**

假设你这样写：
```cpp
struct Decoder {
    Buffer buf; // 构造函数里可能抛异常
    Decoder() : buf(4096) { // <-- 如果这里抛异常
        try {
            // 这里的 try 只能包住这部分
        } catch (...) {
            // 捕不到 buf 构造里的异常
        }
    }
};
```

- `buf(4096)` 在构造函数体运行之前就执行了
- 如果 `buf` 构造函数抛异常，这个异常直接沿调用栈向上抛，不会进入你在构造函数体里的 try 块

### 5.3 函数try块的用法

函数try块语法能把初始化列表和构造体代码一起包进同一个 try 作用域：

```cpp
struct Decoder {
    Buffer buf;
    Decoder() try : buf(4096) { // buf 构造 & 构造体代码都包在 try 里
        // 构造体逻辑
    } catch (const std::exception& e) {
        throw std::runtime_error(std::string("Decoder init failed: ") + e.what());
    }
};
```

这样即使 `buf(4096)` 抛出异常，也会进入 catch，我们就有机会加上下文（比如 "Decoder init failed: ..."）再抛出去。

### 5.4 实际运行示例

```cpp
#include <iostream>
#include <stdexcept>
#include <string>

struct Buffer {
    explicit Buffer(size_t size) {
        if (size > 1024) {
            throw std::runtime_error("Buffer size too large");
        }
    }
};

struct Decoder {
    Buffer buf;
    Decoder() try : buf(4096) { // 这里会抛异常
        std::cout << "Decoder constructed\n";
    } catch (const std::exception& e) {
        throw std::runtime_error(std::string("Decoder init failed: ") + e.what());
    }
};

int main() {
    try {
        Decoder d;
    } catch (const std::exception& e) {
        std::cerr << e.what() << '\n';
    }
}
```

**输出**：
```
Decoder init failed: Buffer size too large
```

### 5.5 为什么要在catch中重新抛出？

**问题**：既然 function-try-block 已经捕获到了异常，为什么不直接在 catch 里处理完，而是又要 throw 一次？

**答案**：跟异常上下文包装、调用方可见性有关。

#### 1. 如果直接捕获但不抛出

```cpp
Decoder() try : buf(4096) {
    std::cout << "Decoder constructed\n";
} catch (const std::exception& e) {
    std::cerr << "Error: " << e.what() << "\n";
    // 不再抛出
}
```

这样做的后果：
- 异常被吃掉（swallow），外层调用者就不会知道构造失败
- 构造函数结束后，Decoder 对象会被认为构造成功，但实际上它可能是不完整的（构造中途失败的对象不能算"已构造成功"——实际上标准禁止在 catch 中"完成"一个失败的构造）
- 一般情况下，构造失败就应该让异常继续往上抛，让调用方决定如何处理

#### 2. 为什么要重新抛出（throw）

重新抛的目的通常有两个：

**(1) 异常转换（Exception Translation）**

底层抛出的可能是个通用异常，比如：
```cpp
throw std::runtime_error("Buffer size too large");
```

调用方只知道"Buffer太大"，但不知道这是在 Decoder 初始化时发生的，也不知道发生在哪个设备、哪个模块。

于是我们在 catch 里加上业务上下文：
```cpp
throw std::runtime_error("Decoder init failed: " + std::string(e.what()));
```

这样调用方看到的异常信息就更完整：
```
Decoder init failed: Buffer size too large
```

**(2) 异常类型升级**

底层可能抛 `std::runtime_error`，但业务层希望抛自定义异常（方便分类处理）：
```cpp
throw MediaOpenError(device_name, ERR_INIT_FAIL, e.what());
```

这样调用方就能用：
```cpp
catch (const MediaOpenError& e) { ... }
```

而不是只能用笼统的：
```cpp
catch (const std::exception& e) { ... }
```

#### 3. 为什么不是 throw; 而是构造新异常？

- `throw;` 会原封不动地把当前异常重新抛出，不会改变类型和信息
- 我们要在这里做的是加上下文，所以需要构造一个新的异常对象并 throw

**总结**：function-try-block 里再抛一次，是为了在捕获构造期异常的同时，附加更多上下文或转换成业务可识别的异常类型，方便外层定位和处理，而不是吃掉异常。

## 6. noexcept 关键字

### 6.1 基本概念

`noexcept` 声明一个函数不会抛出异常（编译器可做优化），如果抛出了异常，程序会直接调用 `std::terminate` 终止。

### 6.2 基本示例

```cpp
#include <iostream>
#include <stdexcept>

void safeFunc() noexcept {
    std::cout << "This function won't throw.\n";
}

void unsafeFunc() noexcept {
    throw std::runtime_error("Oops!"); // 会直接终止程序
}

int main() {
    safeFunc();
    try {
        unsafeFunc(); // 不会被 catch 捕获
    }
    catch (...) {
        std::cout << "Won't reach here.\n";
    }
}
```

**要点**：
- `noexcept` 是承诺：我不会抛异常
- 如果承诺不兑现，C++ 会直接终止程序
- 在性能要求高的代码（比如移动构造）中，`noexcept` 可以减少不必要的异常安全检查

### 6.3 容器优化

移动构造/移动赋值若是 `noexcept`，`std::vector` 等容器扩容时会优先走"无抛移动"，性能更好。

```cpp
struct Packet {
    Packet(Packet&&) noexcept;  // 强烈建议声明为 noexcept
    Packet& operator=(Packet&&) noexcept;
};
```

### 6.4 析构函数与异常

- 析构函数里最好不要抛异常。如果析构在异常传播阶段再次抛异常，会 `std::terminate()`
- 如果必须汇报错误：在析构里吞掉异常+记录日志，或提供显式 `close()` 返回 Status

```cpp
~Cleaner() noexcept {
    try { 
        stop(); 
    } catch (...) { 
        log("Cleaner::stop failed"); 
    }
}
```

### 6.5 跨线程与异步传播

- `std::thread` 的线程函数里抛出的异常不会自动穿越线程边界，线程会 `std::terminate()`
- 方案 A：用 `std::async`，在 `future.get()` 处自动重新抛出
- 方案 B：手动用 `std::exception_ptr` 传递

```cpp
std::exception_ptr ep;
std::thread t([&]{
    try { 
        work(); 
    }
    catch (...) { 
        ep = std::current_exception(); 
    }
});
t.join();
if (ep) std::rethrow_exception(ep);
```

## 7. 异常链（Nested Exceptions）

### 7.1 场景背景

很多时候，底层函数抛出了异常，但中间层想加上下文信息，又不想完全丢掉原始异常的细节。Java/Python 有"异常堆栈链"的机制，C++ 用 `std::throw_with_nested` 来实现类似效果。

### 7.2 代码分层解析

**底层 low()**：
```cpp
void low() { 
    throw std::runtime_error("device lost"); 
}
```
模拟最底层操作（比如硬件驱动），直接抛一个异常：device lost。

**中间层 mid()**：
```cpp
void mid() {
    try { 
        low(); 
    }
    catch (...) {
        std::throw_with_nested(std::runtime_error("open camera failed"));
    }
}
```

- `mid()` 调用 `low()`
- 如果 `low()` 抛异常（这里是 device lost），`catch (...)` 会捕获到它
- `std::throw_with_nested` 会新抛出一个异常，但是它会把当前捕获的异常作为"嵌套异常"保存起来，形成异常链：

```
"open camera failed" (外层)
    -> "device lost" (内层)
```

**顶层 top()**：
```cpp
void top() {
    try { 
        mid(); 
    }
    catch (const std::exception& e) {
        auto print = [](const std::exception& ex, auto&& self, int depth)->void {
            std::cerr << std::string(depth*2, ' ') << ex.what() << '\n';
            try { 
                std::rethrow_if_nested(ex); 
            }
            catch (const std::exception& inner) { 
                self(inner, self, depth+1); 
            }
            catch (...) {}
        };
        print(e, print, 0);
    }
}
```

- 调用 `mid()`，会捕获到外层异常 "open camera failed"
- 定义了一个 print lambda，用递归的方式打印异常链
- `std::rethrow_if_nested(ex)` 会检查 ex 有没有嵌套的异常：
  - 如果有，就重新抛出那个嵌套的异常
  - 再由 `catch (const std::exception& inner)` 捕获，并递归打印
- depth 用来控制缩进，形成层级结构

### 7.3 整体调用流程

1. `top()` 调用 `mid()`
2. `mid()` 调用 `low()` → `low()` 抛出 "device lost"
3. `mid()` 捕获到 "device lost" → 用 `std::throw_with_nested` 包装成 "open camera failed"
4. `top()` 捕获到 "open camera failed" → 打印它
5. 递归进入嵌套异常 → 打印 "device lost"

**运行结果**：
```
open camera failed
  device lost
```

### 7.4 核心概念总结

| 机制 | 作用 |
|------|------|
| `std::throw_with_nested(e)` | 抛出新的异常对象 e，并把当前捕获的异常作为它的"嵌套异常"保存 |
| `std::rethrow_if_nested(e)` | 如果 e 有嵌套异常，就重新抛出它，否则什么也不做 |
| 递归打印 | 不断 rethrow_if_nested → catch → 打印，实现异常链输出 |

**结合业务的意义**：
- 底层 → 只关心技术原因（例如设备丢失、网络中断）
- 中间层 → 在不丢失底层原因的前提下，加上业务上下文（例如"打开相机失败"）
- 最外层 → 打印出完整的异常链，方便调试

## 8. 双通道错误处理接口

### 8.1 接口设计模式

这个例子演示一种"双通道错误处理"接口设计：
- 内部实现用 C++ 异常来写（方便结构化错误传递、加上下文、减少 if 嵌套）
- 外部调用方既可以不关心异常、直接用错误码判断，也可以通过 msg 拿到文字信息

```cpp
enum class Err { Ok, Busy, NotFound, Unknown };

Err open_device(std::string id, std::string* msg) noexcept {
    try {
        do_open(id);
        return Err::Ok;
    } catch (const std::exception& e) {
        if (msg) *msg = e.what();
        return translate(e); // 把异常映射成枚举
    } catch (...) {
        return Err::Unknown;
    }
}
```

### 8.2 关键点解释

**(1) 接口是 noexcept**
- 说明对外保证不会抛异常
- 所有异常都在函数内部捕获、转换为错误码返回
- 这样外部调用方不需要写 try-catch，更适合 C 接口或跨语言调用（比如 C++ → C → Python）

**(2) msg 用来输出可选的错误信息**
- 这是典型的输出参数设计：
  - 如果 `msg != nullptr`，就把异常的 `what()` 填进去
  - 如果调用方不关心信息，可以传 `nullptr`
- 好处：返回值保留了枚举错误码，错误详情用字符串返回，互不干扰

**(3) translate(e)**
- 把异常映射成错误码，例如：
```cpp
Err translate(const std::exception& e) {
    if (dynamic_cast<const BusyError*>(&e)) return Err::Busy;
    if (dynamic_cast<const NotFoundError*>(&e)) return Err::NotFound;
    return Err::Unknown;
}
```
- 这样中间层和底层都可以用异常，但对外统一成一个固定的枚举集合

**(4) catch (...)**
- 捕获所有未知类型异常（可能是 `throw int;` 或其他非 `std::exception`）
- 防止 `noexcept` 接口因异常传播而触发 `std::terminate()`

### 8.3 调用方的用法

```cpp
std::string err_msg;
Err err = open_device("cam0", &err_msg);

if (err != Err::Ok) {
    std::cerr << "Open failed (" << static_cast<int>(err) 
              << "): " << err_msg << "\n";
}
```

- 调用方不用写 try-catch
- 既有结构化的错误码（方便逻辑判断），又有详细文字信息（方便日志输出）

### 8.4 设计的优点

| 特点 | 说明 |
|------|------|
| 双通道 | 错误码 + 错误信息并存，调用方可以按需选择 |
| 异常封装 | 内部用异常写逻辑（少 if/else，清晰），外部看到的是错误码 |
| noexcept 安全 | 对外接口绝不抛异常，适合跨语言/系统边界 |
| 可扩展 | 新增异常类型 → 扩展 translate 即可，无需改接口 |

### 8.5 常见变体

**1. 不需要错误信息**
```cpp
Err open_device(std::string id) noexcept;
```
适合非常底层的 C 接口。

**2. 用 std::expected / tl::expected**
```cpp
std::expected<void, Err> open_device(std::string id);
```
现代 C++ 推荐，用类型安全替代 msg 输出参数。

**3. 异常链 + 错误码**
- 内部用 `std::throw_with_nested` 保留完整异常链，msg 输出拼接后的信息

### 8.6 对比纯异常接口

| 接口风格 | 优点 | 缺点 |
|----------|------|------|
| 纯异常 | 内部逻辑清晰；能传递丰富上下文 | 调用方必须 try-catch，异常传播可能跨语言失效 |
| 纯错误码 | 调用方简单；易跨语言 | 内部逻辑啰嗦（大量 if/else），容易丢上下文 |
| 异常转错误码 | 兼顾内部清晰与外部简单；保留上下文 | 要写 translate()，有点维护成本 |

## 9. catch (...) 的正确用法

### 9.1 使用原则

- 用作边界兜底（线程入口、main()、任务调度器），记录日志、做有限清理，然后通常重新抛出或转换为状态码
- 不要在业务深处"吞掉异常"，否则问题被静默

### 9.2 正确示例

```cpp
int main() {
    try {
        run_app();
    } catch (const std::exception& e) {
        log(std::string("Fatal: ") + e.what());
        return EXIT_FAILURE;
    } catch (...) {
        log("Fatal: unknown exception");
        return EXIT_FAILURE;
    }
}
```

## 10. 与错误码的取舍

### 10.1 异常 vs 错误码对比

| 特性 | 异常 | 错误码 |
|------|------|--------|
| 适用场景 | 非预期、非常态的失败 | 高频、可预期的分支 |
| 优点 | 错误从数据通路剥离，保持主干逻辑清晰 | 性能开销小，跨边界成本低 |
| 缺点 | 跨边界成本、调试心智负担 | 内部逻辑啰嗦，容易丢上下文 |
| 典型应用 | 资源耗尽、协议破坏、系统调用失败 | try_lock 失败、ABI 边界、实时/嵌入环境 |

### 10.2 选择建议

- **异常**：适合"非预期、非常态"的失败（资源耗尽、协议破坏、系统调用失败）。优点是能将错误从数据通路剥离，保持主干逻辑清晰；缺点是跨边界成本、调试心智负担
- **错误码**：适合高频、可预期分支（如 try_lock 失败）、ABI 边界、实时/嵌入等可能禁用异常的环境
- **对外 API** 若要稳定 ABI，很多库选择对外"错误码/Result"+内部用异常

## 11. 与资源管理（RAII / 智能指针）

### 11.1 统一规则

拿到资源的地方就立刻包成对象（`unique_ptr`、`shared_ptr`、`lock_guard`、自定义 Guard）。这样即使中途抛异常，析构会自动释放，避免泄漏/死锁。

```cpp
std::mutex m;
void f() {
    std::lock_guard<std::mutex> lk(m); // 异常也会解锁
    // ...
}
```

### 11.2 RAII 示例

```cpp
class ResourceManager {
private:
    std::unique_ptr<Resource> resource_;
    
public:
    ResourceManager() : resource_(std::make_unique<Resource>()) {
        // 如果构造失败会抛异常，但 unique_ptr 会自动清理
    }
    
    // 析构函数自动清理资源，即使有异常也会执行
    ~ResourceManager() = default;
    
    void doWork() {
        // 即使这里抛异常，资源也会被正确释放
        if (someCondition()) {
            throw std::runtime_error("Work failed");
        }
        resource_->process();
    }
};
```

## 12. 常见坑与最佳实践清单

### 12.1 最佳实践（✅）

- ✅ 捕获 `const std::exception&`，输出 `what()`；兜底 `catch (...)` 仅在边界
- ✅ 自定义异常继承标准异常；保存轻量上下文；支持 `what()`
- ✅ 构造/移动/析构尽量 `noexcept`；析构不抛
- ✅ 边修改边可能抛时，用拷贝-交换达到强保证
- ✅ 线程入口处有 try–catch；用 `std::async` 或 `exception_ptr` 传播
- ✅ 使用函数-try-块给构造/初始化添加上下文
- ✅ 使用 RAII 管理资源，确保异常安全

### 12.2 避免的做法（❌）

- ❌ 不要用异常做正常分支控制（影响可读性与性能）
- ❌ 不要到处 catch 再"吃掉"；排查困难
- ❌ 不要 throw 非异常类型（int、字符串字面量）
- ❌ 不要从 `noexcept` 函数泄出异常
- ❌ 不要在析构中抛异常；必须时请吞掉并记录
- ❌ 不要忽略异常，要记录日志或重新抛出

### 12.3 异常安全保证

| 保证级别 | 描述 | 示例 |
|----------|------|------|
| 基本保证 | 对象保持有效状态，但内容可能改变 | 大部分操作 |
| 强保证 | 操作要么成功，要么对象状态不变 | 拷贝-交换模式 |
| 不抛保证 | 操作不会抛出异常 | `noexcept` 函数 |

```cpp
// 强保证示例：拷贝-交换模式
class Widget {
private:
    std::vector<int> data_;
    
public:
    void updateData(const std::vector<int>& newData) {
        Widget temp(*this);           // 创建副本
        temp.data_ = newData;         // 修改副本
        swap(*this, temp);            // 交换（不抛异常）
    }                                 // 如果前面失败，原对象不变
};
```

## 13. 性能考虑

### 13.1 异常处理的性能开销

- **正常路径**：现代编译器对异常处理进行了优化，正常执行路径几乎没有开销
- **异常路径**：栈展开和异常处理有一定开销，但通常只在错误情况下发生
- **代码大小**：异常处理会增加代码大小，因为需要生成栈展开表

### 13.2 何时避免使用异常

- 在性能关键的实时系统中
- 在嵌入式系统中（可能禁用异常）
- 在跨语言边界（C 接口）
- 在频繁调用的函数中（如果错误是预期的）

### 13.3 异常 vs 错误码性能对比

```cpp
// 异常方式
void processWithException() {
    try {
        riskyOperation();
    } catch (const std::exception& e) {
        handleError(e);
    }
}

// 错误码方式
ErrorCode processWithErrorCode() {
    ErrorCode result = riskyOperation();
    if (result != ErrorCode::Success) {
        handleError(result);
    }
    return result;
}
```

在正常路径上，两者性能相近；在错误路径上，异常处理开销更大，但提供了更好的错误信息。

## 14. 调试与测试

### 14.1 异常调试技巧

```cpp
// 设置断点或日志来跟踪异常传播
void debugFunction() {
    try {
        riskyOperation();
    } catch (const std::exception& e) {
        std::cerr << "Exception in debugFunction: " << e.what() << std::endl;
        std::cerr << "Stack trace: " << std::endl;
        // 可以在这里添加堆栈跟踪
        throw; // 重新抛出
    }
}
```

### 14.2 异常测试

```cpp
// 使用 Google Test 测试异常
#include <gtest/gtest.h>

TEST(ExceptionTest, ThrowsExpectedException) {
    EXPECT_THROW({
        throw std::runtime_error("test");
    }, std::runtime_error);
    
    EXPECT_THROW_MSG({
        throw std::runtime_error("specific message");
    }, std::runtime_error, "specific message");
}
```

## 15. 总结

C++ 异常处理是一个强大的错误处理机制，但需要正确理解和使用。关键要点包括：

### 15.1 核心概念

1. **异常传播**：异常会沿着调用栈向上传播，直到被捕获
2. **栈展开**：异常传播过程中会自动析构局部对象
3. **RAII**：利用栈展开实现自动资源管理
4. **异常安全**：确保异常不会破坏对象状态

### 15.2 最佳实践

1. **选择合适的错误处理方式**：
   - 异常：非预期、非常态错误
   - 错误码：高频、可预期分支

2. **正确使用异常**：
   - 继承标准异常类
   - 提供有意义的错误信息
   - 在适当的地方捕获和处理

3. **确保异常安全**：
   - 使用 RAII 管理资源
   - 提供适当的异常安全保证
   - 避免在析构函数中抛出异常

4. **性能考虑**：
   - 在性能关键路径上谨慎使用异常
   - 考虑使用 `noexcept` 优化
   - 在跨语言边界使用错误码

### 15.3 常见陷阱

1. **误用异常**：用异常做正常流程控制
2. **忽略异常**：捕获异常但不处理
3. **异常泄漏**：从 `noexcept` 函数抛出异常
4. **资源泄漏**：不使用 RAII 管理资源

记住：异常处理是 C++ 中处理错误的重要工具，但需要正确理解其机制和限制，在合适的场景下使用，并遵循最佳实践。

```cpp
// 总结示例：完整的异常处理模式
class RobustClass {
private:
    std::unique_ptr<Resource> resource_;
    
public:
    RobustClass() try : resource_(std::make_unique<Resource>()) {
        // 构造逻辑
    } catch (const std::exception& e) {
        throw std::runtime_error("RobustClass init failed: " + std::string(e.what()));
    }
    
    void doWork() {
        if (!resource_) {
            throw std::logic_error("Resource not initialized");
        }
        
        try {
            resource_->process();
        } catch (const std::exception& e) {
            // 记录日志
            log("Work failed: " + std::string(e.what()));
            // 重新抛出或返回错误码
            throw;
        }
    }
    
    ~RobustClass() noexcept {
        try {
            if (resource_) {
                resource_->cleanup();
            }
        } catch (...) {
            // 析构函数中吞掉异常
            log("Cleanup failed");
        }
    }
};
```

通过正确使用异常处理，可以写出更加健壮、可维护的 C++ 代码。
