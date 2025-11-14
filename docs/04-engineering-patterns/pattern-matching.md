# 模式匹配 & Enum

> **Rust 的灵魂** - 大量 C++/Go 哲学将在此被颠覆

## 为什么模式匹配是 Rust 的灵魂？

模式匹配是 Rust 最强大的特性之一，它让你能够：
- **安全地处理所有可能的情况** - 编译时保证完整性
- **优雅地解构数据** - 从复杂结构中提取值
- **表达业务逻辑** - 代码即文档

## match 表达式

### 基本用法

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

### 匹配 Result<T, E>

```rust
use std::fs::File;
use std::io::Error;

fn read_file(filename: &str) -> Result<String, Error> {
    match File::open(filename) {
        Ok(mut file) => {
            // 读取文件
            Ok(String::from("文件内容"))
        },
        Err(error) => Err(error),
    }
}
```

## 模式匹配的威力

### 1. 穷尽性检查

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

fn move_player(direction: Direction) {
    match direction {
        Direction::Up => println!("向上"),
        Direction::Down => println!("向下"),
        Direction::Left => println!("向左"),
        Direction::Right => println!("向右"),
        // 如果添加新的方向，编译器会提醒你处理它
    }
}
```

### 2. 解构结构体

```rust
struct Point {
    x: i32,
    y: i32,
}

fn process_point(p: Point) {
    match p {
        Point { x: 0, y: 0 } => println!("原点"),
        Point { x, y: 0 } => println!("在 x 轴上，x = {}", x),
        Point { x: 0, y } => println!("在 y 轴上，y = {}", y),
        Point { x, y } => println!("不在轴上: ({}, {})", x, y),
    }
}
```

### 3. 解构元组

```rust
fn process_tuple(tuple: (i32, i32, i32)) {
    match tuple {
        (0, 0, 0) => println!("原点"),
        (x, 0, 0) => println!("x 轴上的点: {}", x),
        (0, y, 0) => println!("y 轴上的点: {}", y),
        (0, 0, z) => println!("z 轴上的点: {}", z),
        (x, y, z) => println!("空间中的点: ({}, {}, {})", x, y, z),
    }
}
```

### 4. 解构引用

```rust
let points = vec![
    Point { x: 0, y: 0 },
    Point { x: 1, y: 2 },
    Point { x: 3, y: 4 },
];

for point in &points {
    match point {
        Point { x: 0, y: 0 } => println!("原点"),
        Point { x, y } => println!("点: ({}, {})", x, y),
    }
}
```

## 匹配守卫（Match Guards）

匹配守卫允许在模式匹配中添加额外的条件：

```rust
let num = Some(4);

match num {
    Some(x) if x < 5 => println!("小于 5: {}", x),
    Some(x) => println!("{}", x),
    None => (),
}
```

### 复杂示例

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id_variable @ 3..=7 } => {
        println!("找到范围内的 id: {}", id_variable)
    },
    Message::Hello { id: 10..=12 } => {
        println!("找到另一个范围内的 id")
    },
    Message::Hello { id } => {
        println!("找到其他 id: {}", id)
    },
}
```

## if let 语法糖

当只需要匹配一个模式时，`if let` 比 `match` 更简洁：

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

### if let 与 else

```rust
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("来自 {:?} 的 25 美分硬币!", state);
} else {
    count += 1;
}
```

## while let 条件循环

```rust
let mut stack = Vec::new();

stack.push(1);
stack.push(2);
stack.push(3);

while let Some(top) = stack.pop() {
    println!("{}", top);
}
```

## for 循环中的模式匹配

```rust
let v = vec!['a', 'b', 'c'];

for (index, value) in v.iter().enumerate() {
    println!("{} 在索引 {}", value, index);
}
```

## let 语句中的模式匹配

```rust
let (x, y, z) = (1, 2, 3);
let (x, y) = (1, 2);
let (x, ..) = (1, 2, 3, 4, 5);
```

## 函数参数中的模式匹配

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("当前位置: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```

## 枚举的威力

### 状态机

```rust
enum State {
    Idle,
    Loading { progress: u32 },
    Loaded { data: Vec<u8> },
    Error { message: String },
}

fn process_state(state: State) {
    match state {
        State::Idle => println!("等待中..."),
        State::Loading { progress } => {
            println!("加载中: {}%", progress);
        },
        State::Loaded { data } => {
            println!("已加载 {} 字节", data.len());
        },
        State::Error { message } => {
            println!("错误: {}", message);
        },
    }
}
```

### 错误处理

```rust
enum MathError {
    DivisionByZero,
    NegativeSquareRoot,
    Overflow,
}

enum MathResult {
    Ok(f64),
    Err(MathError),
}

fn divide(a: f64, b: f64) -> MathResult {
    if b == 0.0 {
        MathResult::Err(MathError::DivisionByZero)
    } else {
        MathResult::Ok(a / b)
    }
}
```

### 可选值

```rust
enum Option<T> {
    Some(T),
    None,
}

// 使用 Option 而不是 null
fn find_user(id: u32) -> Option<String> {
    if id == 1 {
        Some(String::from("Alice"))
    } else {
        None
    }
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - switch 语句（较弱）
enum Color { Red, Green, Blue };

void process_color(Color c) {
    switch (c) {
        case Red: /* ... */ break;
        case Green: /* ... */ break;
        case Blue: /* ... */ break;
        // 可能忘记处理某个 case
    }
}
```

```rust
// Rust - match 表达式（强大）
enum Color {
    Red,
    Green,
    Blue,
}

fn process_color(c: Color) {
    match c {
        Color::Red => { /* ... */ },
        Color::Green => { /* ... */ },
        Color::Blue => { /* ... */ },
        // 编译器保证所有情况都被处理
    }
}
```

### Go 视角

```go
// Go - switch 语句
type Color int

const (
    Red Color = iota
    Green
    Blue
)

func processColor(c Color) {
    switch c {
    case Red:
        // ...
    case Green:
        // ...
    case Blue:
        // ...
    default:
        // 需要手动处理默认情况
    }
}
```

```rust
// Rust - match 表达式（更安全）
enum Color {
    Red,
    Green,
    Blue,
}

fn process_color(c: Color) {
    match c {
        Color::Red => { /* ... */ },
        Color::Green => { /* ... */ },
        Color::Blue => { /* ... */ },
        // 不需要 default，编译器保证完整性
    }
}
```

## 高级模式

### @ 绑定

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };

match msg {
    Message::Hello { id: id_variable @ 3..=7 } => {
        println!("找到范围内的 id: {}", id_variable)
    },
    Message::Hello { id: 10..=12 } => {
        println!("找到另一个范围内的 id")
    },
    Message::Hello { id } => {
        println!("找到其他 id: {}", id)
    },
}
```

### 多重模式

```rust
let x = 1;

match x {
    1 | 2 => println!("一或二"),
    3 => println!("三"),
    _ => println!("其他"),
}
```

### 范围模式

```rust
let x = 5;

match x {
    1..=5 => println!("一到五"),
    6..=10 => println!("六到十"),
    _ => println!("其他"),
}
```

### 忽略模式

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    (first, _, third, _, fifth) => {
        println!("一些数字: {}, {}, {}", first, third, fifth)
    },
}
```

### 使用 .. 忽略剩余部分

```rust
struct Point {
    x: i32,
    y: i32,
    z: i32,
}

let origin = Point { x: 0, y: 0, z: 0 };

match origin {
    Point { x, .. } => println!("x 是 {}", x),
}
```

## 实际应用示例

### 示例 1：解析器

```rust
enum Token {
    Number(i32),
    Plus,
    Minus,
    Multiply,
    Divide,
    LParen,
    RParen,
}

fn parse_expression(tokens: Vec<Token>) -> i32 {
    // 使用模式匹配解析表达式
    // ...
    0
}
```

### 示例 2：状态机

```rust
enum ConnectionState {
    Disconnected,
    Connecting,
    Connected { socket: TcpStream },
    Error { error: String },
}

fn handle_connection(state: ConnectionState) -> ConnectionState {
    match state {
        ConnectionState::Disconnected => {
            ConnectionState::Connecting
        },
        ConnectionState::Connecting => {
            // 尝试连接
            ConnectionState::Connected { socket: /* ... */ }
        },
        ConnectionState::Connected { socket } => {
            // 处理连接
            state
        },
        ConnectionState::Error { error } => {
            println!("连接错误: {}", error);
            ConnectionState::Disconnected
        },
    }
}
```

### 示例 3：配置解析

```rust
enum ConfigValue {
    String(String),
    Number(i64),
    Boolean(bool),
    Array(Vec<ConfigValue>),
    Object(HashMap<String, ConfigValue>),
}

fn process_config(value: ConfigValue) {
    match value {
        ConfigValue::String(s) => println!("字符串: {}", s),
        ConfigValue::Number(n) => println!("数字: {}", n),
        ConfigValue::Boolean(b) => println!("布尔: {}", b),
        ConfigValue::Array(arr) => {
            for item in arr {
                process_config(item);
            }
        },
        ConfigValue::Object(map) => {
            for (key, value) in map {
                println!("键: {}", key);
                process_config(value);
            }
        },
    }
}
```

## 常见错误与解决方案

### 错误 1：忘记处理所有情况

```rust
enum Color {
    Red,
    Green,
    Blue,
}

fn process_color(c: Color) {
    match c {
        Color::Red => println!("红色"),
        Color::Green => println!("绿色"),
        // ❌ 缺少 Blue 的处理
    }
}
```

**解决方案：** 添加所有分支或使用 `_`：

```rust
match c {
    Color::Red => println!("红色"),
    Color::Green => println!("绿色"),
    Color::Blue => println!("蓝色"),
    // 或者
    _ => println!("其他颜色"),
}
```

### 错误 2：模式匹配中的所有权

```rust
let option = Some(String::from("hello"));

match option {
    Some(s) => println!("{}", s),
    None => (),
}

println!("{}", option);  // ❌ option 已被移动
```

**解决方案：** 使用引用：

```rust
match &option {
    Some(s) => println!("{}", s),
    None => (),
}
```

## 实践建议

1. **优先使用 match** - 比 if-else 更安全、更清晰
2. **利用穷尽性检查** - 让编译器帮你发现遗漏的情况
3. **使用枚举表示状态** - 比布尔值更清晰
4. **解构复杂数据** - 使用模式匹配提取值
5. **使用 if let 简化代码** - 当只需要匹配一个模式时

## 扩展练习

1. **实现一个简单的计算器** - 使用枚举表示操作符
2. **实现一个状态机** - 使用枚举表示状态
3. **实现一个配置解析器** - 使用枚举表示不同类型的值
4. **实现一个 AST 遍历器** - 使用模式匹配遍历抽象语法树

## 下一步

掌握了模式匹配后，接下来学习：
- **[迭代器链](./iterators.md)** - 函数式编程思想

---

**记住：模式匹配是 Rust 的灵魂，它让你写出既安全又优雅的代码。充分利用它，你的代码质量会大幅提升！** 🦀
