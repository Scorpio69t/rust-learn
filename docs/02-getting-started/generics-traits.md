# 泛型与特征

> **编写可复用的代码** - 泛型提供类型抽象，Trait 提供行为抽象

## 泛型（Generics）

### 为什么需要泛型？

**不使用泛型（代码重复）：**

```rust
fn largest_i32(list: &[i32]) -> i32 {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn largest_char(list: &[char]) -> char {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

**使用泛型（代码复用）：**

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

### 函数定义中的泛型

```rust
// 单个泛型参数
fn largest<T>(list: &[T]) -> T {
    // ...
}

// 多个泛型参数
fn swap<T, U>(x: T, y: U) -> (U, T) {
    (y, x)
}
```

### 结构体定义中的泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

let integer_point = Point { x: 5, y: 10 };
let float_point = Point { x: 1.0, y: 4.0 };
```

### 多个泛型参数

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

let both_integer = Point { x: 5, y: 10 };
let integer_and_float = Point { x: 5, y: 4.0 };
let both_float = Point { x: 1.0, y: 4.0 };
```

### 枚举定义中的泛型

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### 方法定义中的泛型

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// 为特定类型实现方法
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

### 泛型的性能

**重要：Rust 的泛型是零开销的！**

```rust
// 编译时，Rust 会为每个使用的类型生成特化版本
let integer = Some(5);      // 生成 Option<i32>
let float = Some(5.0);       // 生成 Option<f64>
let string = Some("hello");  // 生成 Option<&str>

// 这叫做单态化（Monomorphization）
// 运行时性能和手写代码一样
```

## Trait（特征）

### Trait 定义

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

### 实现 Trait

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
```

### 默认实现

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(阅读更多...)")
    }
}

// 实现时可以覆盖默认实现，也可以不实现
impl Summary for NewsArticle {
    // 使用默认实现
}
```

### 默认实现可以调用其他方法

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(阅读更多来自 {}...)", self.summarize_author())
    }
}
```

## Trait Bound（特征约束）

### 基本语法

```rust
// 方式 1：使用 trait bound
pub fn notify<T: Summary>(item: &T) {
    println!("突发新闻! {}", item.summarize());
}

// 方式 2：使用 impl Trait（更简洁）
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

**限制：** 只能返回单一的具体类型。

### 使用 Trait Bound 修复 largest 函数

```rust
// ❌ 无法编译
fn largest<T>(list: &[T]) -> T {
    let mut largest = list[0];
    for &item in list.iter() {
        if item > largest {  // T 没有实现比较操作
            largest = item;
        }
    }
    largest
}

// ✅ 正确
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

## 常用标准库 Trait

### Display - 格式化输出

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

### Debug - 调试输出

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{:?}", p);  // 输出：Point { x: 3, y: 4 }
}
```

### Clone 和 Copy

```rust
#[derive(Clone, Copy)]
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

### PartialEq 和 Eq

```rust
#[derive(PartialEq, Eq)]
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

### PartialOrd 和 Ord

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
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
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

## 高级 Trait 特性

### 关联类型（Associated Types）

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

### 关联函数

```rust
trait Animal {
    fn new(name: String) -> Self;  // 关联函数（类似静态方法）
    fn name(&self) -> &str;
}

struct Dog {
    name: String,
}

impl Animal for Dog {
    fn new(name: String) -> Self {
        Dog { name }
    }

    fn name(&self) -> &str {
        &self.name
    }
}
```

### 完全限定语法（Fully Qualified Syntax）

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("这是机长在说话。");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("起来！");
    }
}

impl Human {
    fn fly(&self) {
        println!("*挥舞手臂*");
    }
}

fn main() {
    let person = Human;
    person.fly();                    // 调用 Human 的方法
    Pilot::fly(&person);             // 调用 Pilot 的方法
    Wizard::fly(&person);            // 调用 Wizard 的方法
    <Human as Pilot>::fly(&person);  // 完全限定语法
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 模板
template<typename T>
T largest(const std::vector<T>& list) {
    // ...
}

// C++ - 虚函数（运行时多态）
class Summary {
public:
    virtual std::string summarize() = 0;
};
```

```rust
// Rust - 泛型（编译时多态）
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    // ...
}

// Rust - Trait（编译时多态，零开销）
trait Summary {
    fn summarize(&self) -> String;
}
```

### Go 视角

```go
// Go - 接口（隐式实现）
type Summary interface {
    Summarize() string
}

// Go - 没有泛型（Go 1.18 之前）
func largest(list []int) int {
    // ...
}
```

```rust
// Rust - Trait（显式实现）
trait Summary {
    fn summarize(&self) -> String;
}

// Rust - 泛型（编译时特化）
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    // ...
}
```

## 常见错误与解决方案

### 错误 1：缺少 Trait Bound

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

### 错误 2：孤儿规则（Orphan Rule）

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

### 错误 3：泛型类型不匹配

```rust
struct Point<T> {
    x: T,
    y: T,
}

let p = Point { x: 5, y: 4.0 };  // ❌ 类型不匹配
```

**解决方案：** 使用多个泛型参数：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

let p = Point { x: 5, y: 4.0 };  // ✅
```

## 实践建议

1. **优先使用泛型** - 避免代码重复，提高复用性
2. **合理使用 Trait Bound** - 明确类型需要什么能力
3. **使用 derive 宏** - 让编译器自动实现常用 Trait
4. **理解零开销抽象** - Rust 的泛型在编译时特化，没有运行时开销
5. **使用 where 子句** - 当 Trait Bound 很多时，提高可读性

## 实际应用示例

### 示例 1：通用排序函数

```rust
fn sort<T: Ord>(list: &mut [T]) {
    list.sort();
}

fn main() {
    let mut numbers = vec![3, 1, 4, 1, 5, 9, 2, 6];
    sort(&mut numbers);
    println!("{:?}", numbers);
}
```

### 示例 2：通用查找函数

```rust
fn find<T: PartialEq>(list: &[T], target: &T) -> Option<usize> {
    for (index, item) in list.iter().enumerate() {
        if item == target {
            return Some(index);
        }
    }
    None
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    if let Some(index) = find(&numbers, &3) {
        println!("找到 3，索引: {}", index);
    }
}
```

### 示例 3：泛型结构体

```rust
struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack { items: Vec::new() }
    }

    fn push(&mut self, item: T) {
        self.items.push(item);
    }

    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    fn peek(&self) -> Option<&T> {
        self.items.last()
    }
}
```

## 练习

```rust
// 练习 1：实现一个泛型的 Pair 结构体
struct Pair<T, U> {
    first: T,
    second: U,
}

// 为 Pair 实现方法
impl<T, U> Pair<T, U> {
    // 实现 new 方法
    // 实现 swap 方法，交换 first 和 second
}

// 练习 2：实现一个泛型的 find 函数
fn find<T: PartialEq>(list: &[T], target: &T) -> Option<usize> {
    // 实现查找逻辑
}

// 练习 3：为自定义类型实现 Display trait
struct Point {
    x: i32,
    y: i32,
}

// 实现 Display，输出格式为 "(x, y)"
```

## 下一步

掌握了泛型和 Trait 后，接下来学习：
- **[智能指针](./smart-pointers.md)** - 理解 Rust 的内存管理机制

---

**记住：泛型和 Trait 是 Rust 实现代码复用的核心机制。它们提供了零开销的抽象，让你写出既灵活又高效的代码！** 🦀
