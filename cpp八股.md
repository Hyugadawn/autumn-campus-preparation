# CPP 八股

## 模板实例化

模板实例化怎么回事？如果多种实例化模板，实例化选取顺序是如何的？

## 显式模板实例化（C\+\+98）

在 C\+\+ 中，模板（Template）通常需要将**声明和实现都放在头文件**中，这样编译器才能在使用处（调用点）看到完整定义并自动生成对应类型的代码。

但这种做法存在明显缺点：

- **编译速度慢**：每个 `#include` 该头文件的源文件都会重新实例化一遍模板，造成大量重复编译

- **目标文件膨胀**：链接时同一个模板实例会在多个 `.o` 文件里重复出现（虽然链接器最终会去重，但仍浪费时间）

- **实现细节暴露**：模板的具体实现必须写在头文件里，无法隐藏在 `.cpp`/`.cu` 源文件中

- **无法限制类型**：任何满足语法要求的类型都能实例化模板，难以做到"仅支持特定类型"

**显式模板实例化**正是为解决以上问题而生的机制。其将模板的**声明**和**实现**分离：

- 头文件（`.h`）：只放模板**声明**

- 源文件（`.cpp` / `.cu`）：放模板**定义**（实现），并在文件末尾显式指定需要生成哪些类型的具体代码

示例：

头文件：

```C++
#pragma once

template <typename T>
bool CheckFinite(const T* data, int size);

// C++11 起，可选：显式实例化声明，避免其他文件重复隐式实例化
extern template bool CheckFinite<float>(const float*, int);
extern template bool CheckFinite<double>(const double*, int);
```

源文件

```C++
#include "check_finite.h"
#include <cmath>

template <typename T>
bool CheckFinite(const T* data, int size) {
    for (int i = 0; i < size; ++i) {
        if (!std::isfinite(data[i])) {
            return false;
        }
    }
    return true;
}

// 显式实例化：为 float 和 double 生成具体代码
template bool CheckFinite<float>(const float*, int);
template bool CheckFinite<double>(const double*, int);
```

调用处

```C++
#include "check_finite.h"
#include <iostream>

int main() {
    float arr[] = {1.0f, 2.0f, 3.0f};
    std::cout << CheckFinite(arr, 3) << std::endl;  // ✅ 正常调用 float 版本

    // int arr2[] = {1, 2, 3};
    // CheckFinite(arr2, 3);  // ❌ 链接错误：undefined reference
    //                        // 因为 int 版本没有被显式实例化，找不到对应符号
    return 0;
}
```

## 智能指针和引用（左值）修改

## `static_assert` （C\+\+ 11）

`static_assert` 是 **编译期断言**（C\+\+11 引入）。它在**编译阶段**检查一个常量表达式，如果结果为 `false`，就直接让编译失败并输出你指定的错误信息。

它不是函数，是一个「声明」，运行时**零开销**（编译通过后代码里什么都不剩）。

### 基本语法



```C++
static_assert(常量表达式, "错误信息");   // C++11
static_assert(常量表达式);              // C++17 起消息可省略
```



- 表达式必须是**编译期可求值**的（`constexpr`），会被隐式转换为 `bool`

- C\+\+26 起还允许消息是编译期计算出的字符串（可以拼接类型名等）

### 简单例子

```C++
static_assert(sizeof(int) == 4, "本项目要求 int 必须是 4 字节");
static_assert(sizeof(void*) == 8, "只支持 64 位平台");

constexpr int N = 10;
static_assert(N > 0 && N % 2 == 0, "N 必须是正偶数");
```

如果失败，编译器直接报错：

```Plain Text
error: static assertion failed: 只支持 64 位平台
```

### 可以写在哪里

命名空间作用域、类内、函数内都行：

```C++
static_assert(true);            // 全局

struct Foo {
    int a, b;
    static_assert(sizeof(int) >= 2);   // 类内
};

void f() {
    static_assert(1 + 1 == 2);         // 函数内
}
```

### 最常见的用途：约束模板

配合 `<type_traits>` 检查模板参数：

```C++
#include <type_traits>

template <typename T>
T add(T a, T b) {
    static_assert(std::is_arithmetic_v<T>,
                  "add() 只支持算术类型");
    return a + b;
}

add(1, 2);          // OK
add("a", "b");      // 编译错误，信息清晰
```

好处：错误信息是你自己写的，而不是模板展开后几百行的天书。

### 与其他机制的区别

||检查时机|说明|
|---|---|---|
|`static_assert`|编译期|条件必须是常量表达式|
|`assert`|运行期|`NDEBUG` 下会被去掉|
|`#error`|预处理期|只能配合 `#if` 用宏，看不到类型/`sizeof`|
|`concept` / `requires`|编译期|C\+\+20，参与重载决议，比 `static_assert` 更强|

C\+\+20 有 concepts 后，很多模板约束更推荐用 concept：

```C++
template <std::integral T>   // 比 static_assert 更好：能参与重载决议
T twice(T x) { return x * 2; }
```

但 `static_assert` 仍然适合：平台假设检查、布局检查、以及在函数体内做局部校验。

### 一个经典坑：模板里的 `static_assert(false)`

想做「未支持类型就报错」时，这样写是**错的**：

```C++
template <typename T>
void g(T) {
    static_assert(false, "不支持的类型");  // C++20 及以前：即使从不实例化也会报错
}
```



因为 `false` 与模板参数无关，编译器允许在**不实例化**时就判定它失败。

传统解法——让表达式依赖模板参数：

```C++
template <typename>
inline constexpr bool always_false = false;

template <typename T>
void g(T) {
    static_assert(always_false<T>, "不支持的类型");  // 只有真正实例化时才报错
}
```



> C\+\+23（P2593）已经修正了这个规则：未实例化的模板里写 `static_assert(false)` 不再报错。但为了兼容旧编译器，`always_false` 惯用法仍很常见。
> 
> 



## std::variant\(C\+\+ 17\)

`std::variant` 是 C\+\+17 引入的标准库类型，定义在 `<variant>` 头文件中。

它用于表示“一个值可能是多种类型中的一种”。与传统的 `union` 相比，`std::variant` 会自动记录当前保存的类型，并负责对象的生命周期管理，因此更加安全、易用。

```C++
#include <variant>
#include <string>

std::variant<int, double, std::string> value;

value = 42;                    // 保存 int
value = 3.14;                  // 保存 double
value = std::string("hello");  // 保存 std::string
```

### 使用 `std::visit` 处理不同类型

```C++
int main() {
    std::variant<int, double, std::string> value = "C++";

    std::visit([](const auto& item) {
        using T = std::decay_t<decltype(item)>;

        if constexpr (std::is_same_v<T, int>) {
            std::cout << "整数: " << item << '\n';
        } else if constexpr (std::is_same_v<T, double>) {
            std::cout << "浮点数: " << item << '\n';
        } else if constexpr (std::is_same_v<T, std::string>) {
            std::cout << "字符串: " << item << '\n';
        }
    }, value);
}
```
