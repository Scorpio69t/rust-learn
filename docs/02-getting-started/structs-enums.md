# 结构体与枚举

> **Rust 的核心数据结构** - 结构体组织数据，枚举表示选择，模式匹配处理它们

## 结构体（Struct）

### 定义和实例化

```rust
// 定义结构体
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

// 创建实例
let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};

// 访问字段
println!("用户邮箱: {}", user1.email);
```

### 可变结构体

```rust
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};

user1.email = String::from("anotheremail@example.com");
```

### 字段初始化简写

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,        // 字段名和变量名相同时可以简写
        username,     // 等价于 email: email
        active: true,
        sign_in_count: 1,
    }
}
```

### 结构体更新语法

```rust
let user2 = User {
    email: String::from("another@example.com"),
    username: String::from("anotherusername567"),
    ..user1  // 使用 user1 的其他字段
};
```

### 元组结构体（Tuple Structs）

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

### 类单元结构体（Unit-like Structs）

```rust
struct AlwaysEqual;

let subject = AlwaysEqual;
```

### 方法（Methods）

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // 关联函数（类似静态方法）
    fn new(width: u32, height: u32) -> Rectangle {
        Rectangle { width, height }
    }

    // 方法（第一个参数是 &self）
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // 可变方法
    fn set_width(&mut self, width: u32) {
        self.width = width;
    }

    // 获取所有权的方法
    fn take(self) -> (u32, u32) {
        (self.width, self.height)
    }

    // 方法可以调用其他方法
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}

fn main() {
    let rect = Rectangle::new(30, 50);
    println!("面积: {}", rect.area());
}
```

### 多个 impl 块

```rust
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

impl Rectangle {
    fn perimeter(&self) -> u32 {
        2 * (self.width + self.height)
    }
}
```

## 枚举（Enum）

### 基本定义

```rust
enum IpAddrKind {
    V4,
    V6,
}

let four = IpAddrKind::V4;
let six = IpAddrKind::V6;
```

### 带数据的枚举

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));
let loopback = IpAddr::V6(String::from("::1"));
```

### 更复杂的枚举

```rust
enum Message {
    Quit,                                    // 没有关联数据
    Move { x: i32, y: i32 },                // 有命名字段的结构体
    Write(String),                           // 包含一个 String
    ChangeColor(i32, i32, i32),             // 包含三个 i32
}

impl Message {
    fn call(&self) {
        // 方法实现
    }
}

let m = Message::Write(String::from("hello"));
m.call();
```

### Option 枚举

`Option<T>` 是 Rust 标准库中最重要的枚举之一：

```rust
enum Option<T> {
    Some(T),
    None,
}
```

**使用示例：**

```rust
let some_number = Some(5);
let some_string = Some("a string");
let absent_number: Option<i32> = None;

// Option 必须显式处理
match some_number {
    Some(value) => println!("值是: {}", value),
    None => println!("没有值"),
}
```

**为什么需要 Option？**

```rust
// ❌ 其他语言可能返回 null
fn get_value() -> i32 {
    // 可能返回 null，导致运行时错误
}

// ✅ Rust 使用 Option
fn get_value() -> Option<i32> {
    Some(5)  // 或者 None
    // 必须显式处理 None 的情况
}
```

### Result 枚举

`Result<T, E>` 用于错误处理：

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

**使用示例：**

```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => {
            panic!("打开文件时出错: {:?}", error)
        },
    };
}
```

## 模式匹配（Pattern Matching）

### match 表达式

`match` 是 Rust 最强大的控制流结构：

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

### 绑定值的模式

```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska,
    // ...
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState),
}

fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("来自 {:?} 的 25 美分硬币!", state);
            25
        },
    }
}
```

### 匹配 Option<T>

```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        None => None,
        Some(i) => Some(i + 1),
    }
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```

### 通配模式和 _ 占位符

```rust
let dice_roll = 9;
match dice_roll {
    3 => add_fancy_hat(),
    7 => remove_fancy_hat(),
    other => move_player(other),  // 匹配其他所有值
}

// 或者使用 _ 忽略值
match dice_roll {
    3 => add_fancy_hat(),
    7 => remove_fancy_hat(),
    _ => reroll(),  // 忽略具体值
}
```

### if let 语法糖

```rust
let some_value = Some(3);

// 使用 match
match some_value {
    Some(3) => println!("三"),
    _ => (),
}

// 使用 if let（更简洁）
if let Some(3) = some_value {
    println!("三");
}
```

### while let 条件循环

```rust
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

### 解构结构体和枚举

```rust
struct Point {
    x: i32,
    y: i32,
}

let p = Point { x: 0, y: 7 };

match p {
    Point { x, y: 0 } => println!("在 x 轴上，x = {}", x),
    Point { x: 0, y } => println!("在 y 轴上，y = {}", y),
    Point { x, y } => println!("不在轴上: ({}, {})", x, y),
}
```

### 匹配守卫（Match Guards）

```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("小于 5: {}", x),
    Some(x) => println!("{}", x),
    None => (),
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 结构体
struct User {
    std::string username;
    std::string email;
    bool active;
};

// C++ - 枚举（较弱）
enum Color { Red, Green, Blue };
```

```rust
// Rust - 结构体（更强大）
struct User {
    username: String,
    email: String,
    active: bool,
}

// Rust - 枚举（可以带数据）
enum Color {
    Red,
    Green,
    Blue,
    Rgb(u8, u8, u8),
    Hsv { h: u8, s: u8, v: u8 },
}
```

### Go 视角

```go
// Go - 结构体
type User struct {
    Username string
    Email    string
    Active   bool
}

// Go - 没有真正的枚举
const (
    Red = iota
    Green
    Blue
)
```

```rust
// Rust - 结构体（有方法）
struct User {
    username: String,
    email: String,
    active: bool,
}

impl User {
    fn is_active(&self) -> bool {
        self.active
    }
}

// Rust - 强大的枚举
enum Color {
    Red,
    Green,
    Blue,
}
```

## 常见错误与解决方案

### 错误 1：忘记处理 None

```rust
let x: Option<i32> = Some(5);
let y = x + 1;  // ❌ 编译错误
```

**解决方案：** 使用 match 或 unwrap：

```rust
let x: Option<i32> = Some(5);
let y = match x {
    Some(value) => value + 1,
    None => 0,
};
```

### 错误 2：match 必须穷尽所有可能

```rust
match some_option {
    Some(x) => println!("{}", x),
    // ❌ 缺少 None 分支
}
```

**解决方案：** 添加所有分支：

```rust
match some_option {
    Some(x) => println!("{}", x),
    None => (),
}
```

### 错误 3：结构体字段所有权

```rust
struct User {
    username: String,
}

fn create_user() -> User {
    let username = String::from("user");
    User { username }  // ✅ 移动所有权
}

fn bad_example() {
    let username = String::from("user");
    let user = User { username };
    println!("{}", username);  // ❌ username 已被移动
}
```

**解决方案：** 使用引用或克隆：

```rust
struct User {
    username: String,
}

fn main() {
    let username = String::from("user");
    let user = User { username: username.clone() };
    println!("{}", username);  // ✅
}
```

## 实践建议

1. **优先使用枚举** - Rust 的枚举比 C++/Go 强大得多
2. **使用 Option 而不是 null** - 编译时保证安全
3. **使用 Result 处理错误** - 强制错误处理
4. **充分利用模式匹配** - match 是 Rust 的核心特性
5. **使用 if let 简化代码** - 当只需要匹配一个模式时

## 实际应用示例

### 示例 1：Web 请求处理

```rust
enum HttpMethod {
    GET,
    POST { body: String },
    PUT { body: String },
    DELETE,
}

fn handle_request(method: HttpMethod) {
    match method {
        HttpMethod::GET => println!("处理 GET 请求"),
        HttpMethod::POST { body } => {
            println!("处理 POST 请求，body: {}", body);
        },
        HttpMethod::PUT { body } => {
            println!("处理 PUT 请求，body: {}", body);
        },
        HttpMethod::DELETE => println!("处理 DELETE 请求"),
    }
}
```

### 示例 2：文件操作

```rust
use std::fs::File;
use std::io::Error;

fn read_file(filename: &str) -> Result<String, Error> {
    match File::open(filename) {
        Ok(mut file) => {
            // 读取文件内容
            Ok(String::from("文件内容"))
        },
        Err(e) => Err(e),
    }
}

fn main() {
    match read_file("hello.txt") {
        Ok(content) => println!("内容: {}", content),
        Err(e) => println!("错误: {:?}", e),
    }
}
```

## 练习

```rust
// 练习 1：定义一个表示几何形状的枚举
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle { a: f64, b: f64, c: f64 },
}

// 为 Shape 实现 area 方法
impl Shape {
    fn area(&self) -> f64 {
        // 实现计算面积的逻辑
    }
}

// 练习 2：使用 Option 处理可能为空的值
fn find_user(id: u32) -> Option<String> {
    // 如果找到用户返回 Some(username)，否则返回 None
}

// 练习 3：使用 Result 处理文件操作
fn read_config() -> Result<String, String> {
    // 读取配置文件，成功返回内容，失败返回错误信息
}
```

## 下一步

掌握了结构体和枚举后，接下来学习：
- **[泛型与特征](./generics-traits.md)** - 编写可复用的代码

---

**记住：结构体和枚举是 Rust 组织数据的基础，模式匹配是处理它们的强大工具。这是 Rust 最优雅的特性之一！** 🦀
