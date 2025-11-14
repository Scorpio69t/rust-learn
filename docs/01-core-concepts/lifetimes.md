# Lifetimes（生命周期）

> **生命周期不是"写给 Rust 看"的，是告诉编译器"我保证这个引用不会悬空"**

## 什么是生命周期？

生命周期（Lifetime）是 Rust 中引用的有效期。它确保引用在使用时始终有效，防止悬垂引用。

### 为什么需要生命周期？

```rust
// ❌ 这段代码无法编译（没有生命周期标注）
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

**错误信息：**
```
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
```

**问题：** 编译器不知道返回的引用是来自 `x` 还是 `y`，也不知道这个引用应该活多久。

## 生命周期标注语法

### 基本语法

```rust
&i32        // 一个引用
&'a i32     // 带有显式生命周期的引用
&'a mut i32 // 带有显式生命周期的可变引用
```

### 函数签名中的生命周期

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

**解读：**
- `<'a>` - 声明一个生命周期参数 `'a`
- `x: &'a str` - `x` 的引用至少活 `'a` 那么久
- `y: &'a str` - `y` 的引用至少活 `'a` 那么久
- `-> &'a str` - 返回的引用也至少活 `'a` 那么久

### 使用示例

```rust
fn main() {
    let string1 = String::from("long string is long");

    {
        let string2 = String::from("xyz");
        let result = longest(string1.as_str(), string2.as_str());
        println!("最长的字符串是 {}", result);
    }  // string2 在这里离开作用域
    // result 在这里已经无效，因为它的生命周期不能超过 string2
}
```

## 生命周期标注规则

### 规则 1：每个引用都有生命周期

```rust
// 编译器会自动推断生命周期
fn first_word(s: &str) -> &str {
    // 实际上等价于：
    // fn first_word<'a>(s: &'a str) -> &'a str
    let bytes = s.as_bytes();
    // ...
}
```

### 规则 2：输入生命周期参数

如果函数有多个输入引用，需要明确标注：

```rust
// ❌ 无法编译
fn longest(x: &str, y: &str) -> &str {
    // ...
}

// ✅ 正确
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

### 规则 3：输出生命周期

如果函数返回引用，返回值的生命周期必须与某个输入参数的生命周期相关联：

```rust
// ✅ 返回值的生命周期与输入参数相同
fn first_word<'a>(s: &'a str) -> &'a str {
    // ...
}

// ❌ 无法编译 - 返回值的生命周期不明确
fn bad_function(x: &str, y: &str) -> &str {
    &String::from("hello")  // 返回局部变量的引用
}
```

## 生命周期省略（Lifetime Elision）

在某些情况下，Rust 允许省略生命周期标注：

### 规则 1：每个引用参数都有自己的生命周期

```rust
// 这两个函数签名是等价的：
fn foo(x: &i32)           // 省略形式
fn foo<'a>(x: &'a i32)    // 完整形式
```

### 规则 2：如果只有一个输入生命周期参数，它被赋予所有输出生命周期参数

```rust
// 这两个函数签名是等价的：
fn first_word(s: &str) -> &str
fn first_word<'a>(s: &'a str) -> &'a str
```

### 规则 3：如果方法有多个输入生命周期参数，但其中一个是 `&self` 或 `&mut self`，则 `self` 的生命周期被赋予所有输出生命周期参数

```rust
impl<'a> ImportantExcerpt<'a> {
    // 这两个方法签名是等价的：
    fn level(&self) -> i32
    fn level<'a>(&'a self) -> i32

    // 这个需要显式标注：
    fn announce_and_return_part(&self, announcement: &str) -> &str
    // 等价于：
    fn announce_and_return_part<'b>(&'b self, announcement: &str) -> &'b str
}
```

## 结构体中的生命周期

### 基本用法

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,  // 结构体包含引用，需要生命周期标注
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("找不到 '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
    // i 的生命周期不能超过 novel
}
```

### 多个生命周期参数

```rust
struct TwoRefs<'a, 'b> {
    first: &'a str,
    second: &'b str,
}

fn main() {
    let s1 = String::from("first");
    let s2 = String::from("second");
    let two_refs = TwoRefs {
        first: &s1,
        second: &s2,
    };
}
```

## 方法中的生命周期

```rust
impl<'a> ImportantExcerpt<'a> {
    // 方法 1：返回值的生命周期与 self 相同
    fn level(&self) -> i32 {
        3
    }

    // 方法 2：返回值的生命周期与输入参数相同
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("注意！{}", announcement);
        self.part
    }
}
```

## 静态生命周期

`'static` 是一个特殊的生命周期，表示整个程序的持续时间：

```rust
// 字符串字面量有 'static 生命周期
let s: &'static str = "I have a static lifetime.";

// 这个函数返回的引用必须至少活 'static 那么久
fn get_static_str() -> &'static str {
    "hello"
}
```

**注意：** 不要滥用 `'static`，大多数情况下不需要。

## 生命周期与泛型结合

```rust
use std::fmt::Display;

fn longest_with_an_announcement<'a, T>(
    x: &'a str,
    y: &'a str,
    ann: T,
) -> &'a str
where
    T: Display,
{
    println!("公告！{}", ann);
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 没有编译时检查，可能悬空
const std::string& longest(const std::string& x, const std::string& y) {
    return x.length() > y.length() ? x : y;
    // 如果返回的引用指向局部变量，会导致未定义行为
}
```

```rust
// Rust - 编译时检查，保证安全
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
    // 编译器保证返回的引用有效
}
```

### Go 视角

```go
// Go - 运行时检查，可能 panic
func longest(x, y string) *string {
    if len(x) > len(y) {
        return &x  // 可能返回局部变量的引用
    }
    return &y
}
```

```rust
// Rust - 编译时检查，不可能返回悬空引用
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

## 常见错误与解决方案

### 错误 1：忘记生命周期标注

```rust
// ❌ 无法编译
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

**解决方案：** 添加生命周期标注：

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

### 错误 2：返回局部变量的引用

```rust
// ❌ 无法编译
fn bad() -> &str {
    let s = String::from("hello");
    &s  // 返回局部变量的引用
}
```

**解决方案：** 返回所有权：

```rust
fn good() -> String {
    let s = String::from("hello");
    s  // 返回所有权
}
```

### 错误 3：生命周期不匹配

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }  // string2 离开作用域
    println!("最长的字符串是 {}", result);  // ❌ 编译错误
}
```

**解决方案：** 确保引用的生命周期足够长：

```rust
fn main() {
    let string1 = String::from("long string is long");
    let string2 = String::from("xyz");
    let result = longest(string1.as_str(), string2.as_str());
    println!("最长的字符串是 {}", result);  // ✅
}
```

## 实践建议

1. **让编译器推断** - 大多数情况下，编译器可以自动推断生命周期
2. **只在必要时标注** - 当编译器无法推断时才需要显式标注
3. **理解生命周期参数的含义** - `'a` 表示"至少活这么长时间"
4. **阅读错误信息** - Rust 的生命周期错误信息很详细，会告诉你问题所在

## 生命周期标注的思维模型

把生命周期想象成"作用域的标签"：

```rust
fn main() {
    let string1 = String::from("long string");  // 作用域开始
    {
        let string2 = String::from("xyz");      // 作用域开始
        let result = longest(string1.as_str(), string2.as_str());
        // result 的生命周期不能超过 string2
    }  // string2 的作用域结束
    // result 在这里已经无效
}  // string1 的作用域结束
```

## 练习

```rust
// 练习 1：修复这个函数
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

// 练习 2：这个结构体需要生命周期标注吗？
struct Book {
    title: String,
    author: String,
}

// 练习 3：这个方法有什么问题？
impl Book {
    fn get_title(&self) -> &str {
        &self.title
    }
}
```

## 下一步

理解了生命周期后，接下来学习：
- **[Trait（特征）](./traits.md)** - Rust 的多态和抽象机制

---

**记住：生命周期是 Rust 保证引用安全的关键。虽然语法看起来复杂，但它是编译时检查，零运行时开销！** 🦀
