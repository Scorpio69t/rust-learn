# Trait（特征）

> **Rust 的多态机制** - 类似 C++ 的概念类 + 接口 + 类型约束

## 什么是 Trait？

Trait 定义了类型必须实现的功能。它类似于其他语言中的接口（Interface），但更强大。

### 基本概念

```rust
// 定义一个 trait
trait Summary {
    fn summarize(&self) -> String;
}

// 为类型实现 trait
struct NewsArticle {
    headline: String,
    location: String,
    author: String,
    content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
```

## Trait 的定义

### 基本语法

```rust
trait TraitName {
    fn method1(&self);
    fn method2(&self) -> String;
    fn method3(&self, param: i32) -> bool;
}
```

### 带默认实现

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(阅读更多...)")
    }

    fn summarize_author(&self) -> String;  // 没有默认实现
}
```

### 默认实现调用其他方法

```rust
trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(阅读更多来自 {}...)", self.summarize_author())
    }
}
```

## 实现 Trait

### 为类型实现 Trait

```rust
struct Tweet {
    username: String,
    content: String,
    reply: bool,
    retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

### 使用 Trait

```rust
fn main() {
    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from("当然，你知道，就像..."),
        reply: false,
        retweet: false,
    };

    println!("1 条新推文: {}", tweet.summarize());
}
```

## Trait 作为参数

### Trait Bound 语法

```rust
// 方式 1：使用 trait bound
pub fn notify<T: Summary>(item: &T) {
    println!("突发新闻! {}", item.summarize());
}

// 方式 2：使用 impl Trait 语法（更简洁）
pub fn notify(item: &impl Summary) {
    println!("突发新闻! {}", item.summarize());
}
```

### 多个 Trait Bound

```rust
// 方式 1：使用 + 语法
pub fn notify<T: Summary + Display>(item: &T) {
    // ...
}

// 方式 2：使用 where 子句（更清晰）
pub fn notify<T>(item: &T)
where
    T: Summary + Display,
{
    // ...
}
```

## Trait 作为返回值

### 返回实现 Trait 的类型

```rust
fn returns_summarizable() -> impl Summary {
    Tweet {
        username: String::from("horse_ebooks"),
        content: String::from("当然，你知道，就像..."),
        reply: false,
        retweet: false,
    }
}
```

**限制：** 只能返回单一的具体类型：

```rust
// ❌ 无法编译 - 返回不同类型
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        NewsArticle { /* ... */ }
    } else {
        Tweet { /* ... */ }
    }
}
```

## 常用标准库 Trait

### `Display` - 格式化输出

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{}", p);  // 输出：(3, 4)
}
```

### `Debug` - 调试输出

```rust
#[derive(Debug)]  // 使用 derive 宏自动实现
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{:?}", p);  // 输出：Point { x: 3, y: 4 }
}
```

### `Clone` - 克隆

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 3, y: 4 };
    let p2 = p1.clone();  // 克隆
    println!("p1: {:?}, p2: {:?}", p1, p2);
}
```

### `Copy` - 复制语义

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 3, y: 4 };
    let p2 = p1;  // 复制，不是移动
    println!("p1: {:?}, p2: {:?}", p1, p2);
}
```

### `PartialEq` 和 `Eq` - 相等比较

```rust
#[derive(PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 3, y: 4 };
    let p2 = Point { x: 3, y: 4 };
    println!("{}", p1 == p2);  // 输出：true
}
```

### `PartialOrd` 和 `Ord` - 排序

```rust
#[derive(PartialOrd, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 3, y: 4 };
    let p2 = Point { x: 5, y: 6 };
    println!("{}", p1 < p2);  // 输出：true
}
```

## Derive 宏

使用 `#[derive(...)]` 可以自动实现某些 Trait：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

**常用的 derive：**
- `Debug` - 调试输出
- `Clone` - 克隆
- `Copy` - 复制语义
- `PartialEq` - 部分相等
- `Eq` - 完全相等
- `PartialOrd` - 部分排序
- `Ord` - 完全排序
- `Hash` - 哈希
- `Default` - 默认值

## Trait Bound 与泛型

### 泛型函数

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}
```

### 泛型结构体

```rust
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("最大的成员是 x = {}", self.x);
        } else {
            println!("最大的成员是 y = {}", self.y);
        }
    }
}
```

## 关联类型（Associated Types）

```rust
trait Iterator {
    type Item;  // 关联类型

    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count < 6 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 虚函数表（运行时多态）
class Summary {
public:
    virtual std::string summarize() = 0;
};

class NewsArticle : public Summary {
    std::string summarize() override {
        return "...";
    }
};
```

```rust
// Rust - 零开销抽象（编译时多态）
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle { /* ... */ }

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        // ...
    }
}
```

### Go 视角

```go
// Go - 接口（隐式实现）
type Summary interface {
    Summarize() string
}

type NewsArticle struct { /* ... */ }

func (n NewsArticle) Summarize() string {
    return "..."
}
```

```rust
// Rust - Trait（显式实现）
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle { /* ... */ }

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        // ...
    }
}
```

## 常见错误与解决方案

### 错误 1：忘记实现必需的方法

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Tweet { /* ... */ }

impl Summary for Tweet {
    // ❌ 忘记实现 summarize
}
```

**解决方案：** 实现所有必需的方法：

```rust
impl Summary for Tweet {
    fn summarize(&self) -> String {
        // 实现
    }
}
```

### 错误 2：Trait Bound 不满足

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {  // ❌ T 没有实现 PartialOrd
            largest = item;
        }
    }
    largest
}
```

**解决方案：** 添加 Trait Bound：

```rust
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    // ...
}
```

### 错误 3：孤儿规则（Orphan Rule）

```rust
// ❌ 无法编译 - 不能为外部类型实现外部 Trait
impl Display for Vec<String> {
    // ...
}
```

**解决方案：** 使用 newtype 模式：

```rust
struct Wrapper(Vec<String>);

impl Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
```

## 实践建议

1. **优先使用标准库 Trait** - `Display`, `Debug`, `Clone` 等
2. **使用 derive 宏** - 让编译器自动实现
3. **理解 Trait Bound** - 明确类型需要什么能力
4. **使用 `impl Trait` 语法** - 更简洁易读

## 高级特性

### 关联常量

```rust
trait MyTrait {
    const CONSTANT: i32 = 42;

    fn method(&self);
}
```

### 关联函数

```rust
trait MyTrait {
    fn new() -> Self;  // 关联函数（类似静态方法）
    fn method(&self);
}
```

### Trait 对象（动态分发）

```rust
// 使用 Box<dyn Trait> 实现动态分发
fn returns_summarizable() -> Box<dyn Summary> {
    Box::new(Tweet { /* ... */ })
}
```

## 练习

```rust
// 练习 1：为这个结构体实现 Display trait
struct Point {
    x: i32,
    y: i32,
}

// 练习 2：创建一个泛型函数，接受实现了 Summary 的类型
fn print_summary<T: Summary>(item: &T) {
    // ...
}

// 练习 3：使用 derive 宏为这个结构体添加必要的 trait
struct Person {
    name: String,
    age: u32,
}
```

## 下一步

理解了 Trait 后，你已经掌握了 Rust 的四大核心概念！接下来可以：
- 进入 **[第 2 章：入门路线](../02-getting-started/)** - 学习 Rust 的基础语法和常用特性

---

**记住：Trait 是 Rust 实现多态和抽象的核心机制。它提供了零开销的抽象，让你写出既安全又高效的代码！** 🦀
