# C++ `explicit` 关键字

## 一、`explicit` 是什么？

`explicit` 用于限制构造函数或类型转换运算符被隐式使用，避免编译器在不明确的情况下自动进行类型转换。

核心作用：**让类型转换更加明确，减少意外的隐式转换。**

## 二、`explicit` 修饰构造函数

### 1. 不使用 `explicit`

```
class MyString {
public:
    MyString(const char* str) {}
};

void SendData(MyString str) {}

SendData("hello");  // 合法
```

原因：编译器可以通过 `MyString(const char*)` 将 `"hello"` 隐式转换成 `MyString` 对象，再传给 `SendData()`。

### 2. 使用 `explicit`

```
class MyString {
public:
    explicit MyString(const char* str) {}
};

void SendData(MyString str) {}

SendData("hello");             // 编译报错
SendData(MyString("hello"));   // 合法
```

加上 `explicit` 后，构造函数不能用于这种普通隐式转换，但仍然可以显式调用。

### 3. 为什么需要它？

- 防止意外的隐式类型转换。
- 让函数调用的参数类型更加明确。
- 避免某些不必要的临时对象构造。
- 提高代码的可读性，减少意外匹配到重载函数的情况。

注意：`explicit` 本身不是性能优化开关。显式构造对象仍然可能产生相同的构造开销。

## 三、`explicit` 修饰类型转换运算符

类型转换运算符允许类对象转换为指定类型。

```
class SmartPtr {
public:
    int* ptr_ = nullptr;

    explicit operator bool() const {
        return ptr_ != nullptr;
    }
};
```

其中：

- `operator bool()`：定义对象转换成 `bool` 的方式。
- `const`：表示该转换函数不会修改当前对象。
- `return ptr_ != nullptr`：指针非空时返回 `true`，否则返回 `false`。

使用示例：

```
SmartPtr p;

if (p) {
    // 合法，条件判断允许显式布尔转换
}

int x = p + 5;  // 编译报错
```

### 为什么 `if (p)` 合法？

C++ 存在一种特殊规则，叫作**语境转换（Contextual Conversion）**。

在 `if`、`while` 等需要布尔条件的语境中，允许使用 `explicit operator bool()`。

但在 `p + 5` 这种普通算术表达式中，不能将该显式转换运算符作为普通隐式转换使用。

这样既能判断智能指针是否有效，又能避免它意外参与数字运算。

### 其他类型转换运算符

```
operator int() const;
operator double() const;
operator bool() const;
```

这些运算符分别定义对象转换为 `int`、`double`、`bool` 的方式。

它们也可以加上 `explicit`：

```
explicit operator int() const;
explicit operator double() const;
```

加上后，普通隐式转换会受到限制；需要时可以使用显式转换，例如：

```
int x = static_cast<int>(obj);
```

## 四、C++20：条件 `explicit`

C++20 允许根据编译期条件决定是否启用 `explicit`。

基本语法：

```
explicit(布尔常量表达式)
```

- 条件为 `true`：相当于使用 `explicit`。
- 条件为 `false`：相当于不使用 `explicit`。

示例：

```
#include <type_traits>

template<typename T>
struct Wrapper {
    template<typename U>
    explicit(!std::is_convertible_v<U, T>)
    Wrapper(U&& u);
};
```

其中：

`std::is_convertible_v` 用于判断 `U` 是否能够隐式转换成 `T`。

因此：

```
explicit(!std::is_convertible_v<U, T>)
```

表示：

- `U` 能隐式转换成 `T`：条件为 `false`，允许 `Wrapper` 隐式构造。
- `U` 不能隐式转换成 `T`：条件为 `true`，禁止 `Wrapper` 隐式构造。

注意：这里的 `U` 必须通过 `template` 声明。条件 `explicit` 只决定当前构造函数是否允许隐式调用，并不保证构造函数内部的类型转换一定合法。

## 五、总结对比

| 使用位置                               | 作用                                   |
| -------------------------------------- | -------------------------------------- |
| `explicit MyClass(int)`                | 防止构造函数被普通隐式转换使用         |
| `explicit operator bool()`             | 允许语境布尔转换，阻止普通隐式布尔转换 |
| `explicit operator int()`              | 限制对象被普通隐式转换为整数           |
| `explicit(condition) Constructor(...)` | 根据编译期条件决定是否限制隐式转换     |

## 六、记忆要点

1. `explicit` 的核心作用是限制隐式转换，不是禁止所有类型转换。
2. 修饰构造函数时，防止其他类型自动转换成当前类对象。
3. 修饰类型转换运算符时，防止当前类对象被普通隐式转换成目标类型。
4. `explicit operator bool()` 是常见用法，允许 `if (obj)`，但阻止普通隐式布尔转换。
5. C++20 的条件 `explicit` 可以根据编译期条件自动决定是否限制隐式转换。
6. `explicit` 不保证消除临时对象或性能开销，它主要用于控制类型转换规则。