# 错误处理

> **Rust 的错误处理哲学** - 使用类型系统保证错误被处理

## Rust 的错误处理哲学

Rust 没有异常，而是使用类型系统来处理错误：
- **`Result<T, E>`** - 表示可能失败的操作
- **`Option<T>`** - 表示可能为空的值
- **编译时检查** - 强制处理所有错误情况

## Result<T, E>

### 基本用法

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### 示例：文件操作

```rust
use std::fs::File;
use std::io::Error;

fn read_file(filename: &str) -> Result<String, Error> {
    let mut file = File::open(filename)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

### 处理 Result

```rust
match read_file("hello.txt") {
    Ok(contents) => println!("文件内容: {}", contents),
    Err(e) => println!("错误: {}", e),
}
```

## ? 运算符

### 基本用法

`?` 运算符是错误传播的简写：

```rust
// 使用 match
fn read_file(filename: &str) -> Result<String, Error> {
    match File::open(filename) {
        Ok(mut file) => {
            let mut contents = String::new();
            match file.read_to_string(&mut contents) {
                Ok(_) => Ok(contents),
                Err(e) => Err(e),
            }
        },
        Err(e) => Err(e),
    }
}

// 使用 ? 运算符（更简洁）
fn read_file(filename: &str) -> Result<String, Error> {
    let mut file = File::open(filename)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    Ok(contents)
}
```

### ? 运算符的工作原理

```rust
// 如果值是 Ok，解包并继续
// 如果值是 Err，立即返回错误
let file = File::open("hello.txt")?;
// 等价于：
let file = match File::open("hello.txt") {
    Ok(file) => file,
    Err(e) => return Err(e),
};
```

### 链式调用

```rust
use std::fs;
use std::io;

fn read_config() -> Result<String, io::Error> {
    let contents = fs::read_to_string("config.txt")?;
    Ok(contents)
}

fn process_config() -> Result<(), io::Error> {
    let config = read_config()?;
    println!("配置: {}", config);
    Ok(())
}
```

## 自定义错误类型

### 使用 thiserror

添加依赖：

```toml
[dependencies]
thiserror = "1.0"
```

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum MyError {
    #[error("IO 错误: {0}")]
    Io(#[from] std::io::Error),

    #[error("解析错误: {0}")]
    Parse(#[from] std::num::ParseIntError),

    #[error("自定义错误: {message}")]
    Custom { message: String },
}

fn might_fail() -> Result<(), MyError> {
    let file = std::fs::File::open("file.txt")?;  // 自动转换为 MyError::Io
    let number: i32 = "not a number".parse()?;     // 自动转换为 MyError::Parse
    Ok(())
}
```

### 手动实现

```rust
use std::fmt;

#[derive(Debug)]
enum MyError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
    Custom(String),
}

impl fmt::Display for MyError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MyError::Io(e) => write!(f, "IO 错误: {}", e),
            MyError::Parse(e) => write!(f, "解析错误: {}", e),
            MyError::Custom(msg) => write!(f, "自定义错误: {}", msg),
        }
    }
}

impl std::error::Error for MyError {}
```

## anyhow - 快速错误处理

### 基本用法

添加依赖：

```toml
[dependencies]
anyhow = "1.0"
```

```rust
use anyhow::{Context, Result};

fn read_file(filename: &str) -> Result<String> {
    let contents = std::fs::read_to_string(filename)
        .with_context(|| format!("无法读取文件: {}", filename))?;
    Ok(contents)
}

fn main() -> Result<()> {
    let contents = read_file("config.txt")?;
    println!("{}", contents);
    Ok(())
}
```

### 添加上下文

```rust
use anyhow::{Context, Result};

fn process_config() -> Result<()> {
    let config = std::fs::read_to_string("config.txt")
        .context("读取配置文件失败")?;

    let value: i32 = config.trim().parse()
        .context("解析配置值失败")?;

    Ok(())
}
```

### 错误链

```rust
use anyhow::{Context, Result, Error};

fn step1() -> Result<()> {
    step2().context("步骤 1 失败")?;
    Ok(())
}

fn step2() -> Result<()> {
    step3().context("步骤 2 失败")?;
    Ok(())
}

fn step3() -> Result<()> {
    Err(anyhow::anyhow!("原始错误"))
}
```

## 错误处理模式

### 模式 1：传播错误

```rust
fn read_and_process(filename: &str) -> Result<String, std::io::Error> {
    let contents = std::fs::read_to_string(filename)?;
    // 处理内容
    Ok(contents)
}
```

### 模式 2：转换错误类型

```rust
use std::num::ParseIntError;

fn parse_number(s: &str) -> Result<i32, ParseIntError> {
    s.parse()
}

fn process_number(s: &str) -> Result<i32, String> {
    parse_number(s).map_err(|e| format!("解析失败: {}", e))
}
```

### 模式 3：提供默认值

```rust
fn get_config_value(key: &str) -> String {
    std::env::var(key).unwrap_or_else(|_| "default".to_string())
}
```

### 模式 4：组合多个错误

```rust
fn process_files(filenames: &[&str]) -> Result<Vec<String>, Vec<String>> {
    let mut results = Vec::new();
    let mut errors = Vec::new();

    for filename in filenames {
        match std::fs::read_to_string(filename) {
            Ok(contents) => results.push(contents),
            Err(e) => errors.push(format!("{}: {}", filename, e)),
        }
    }

    if errors.is_empty() {
        Ok(results)
    } else {
        Err(errors)
    }
}
```

## Option<T> 错误处理

### 基本用法

```rust
fn find_user(id: u32) -> Option<String> {
    if id == 1 {
        Some("Alice".to_string())
    } else {
        None
    }
}

match find_user(1) {
    Some(name) => println!("找到用户: {}", name),
    None => println!("用户不存在"),
}
```

### Option 的方法

```rust
let value = Some(5);

// unwrap - 如果是 None 会 panic
let x = value.unwrap();

// unwrap_or - 提供默认值
let x = value.unwrap_or(0);

// unwrap_or_else - 使用闭包计算默认值
let x = value.unwrap_or_else(|| 0);

// map - 转换值
let doubled = value.map(|x| x * 2);

// and_then - 链式操作
let result = value.and_then(|x| if x > 0 { Some(x * 2) } else { None });
```

### Option 转 Result

```rust
let option = Some(5);
let result: Result<i32, String> = option.ok_or_else(|| "值为空".to_string());
```

## 错误处理最佳实践

### 1. 使用 Result 而不是 panic

```rust
// ❌ 不好
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("除以零");
    }
    a / b
}

// ✅ 好
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("除以零".to_string())
    } else {
        Ok(a / b)
    }
}
```

### 2. 提供有意义的错误信息

```rust
use anyhow::Context;

fn read_config() -> Result<String> {
    std::fs::read_to_string("config.txt")
        .context("无法读取配置文件")?;
    Ok(contents)
}
```

### 3. 使用 thiserror 定义错误类型

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("IO 错误: {0}")]
    Io(#[from] std::io::Error),

    #[error("配置错误: {0}")]
    Config(String),
}
```

### 4. 使用 anyhow 快速原型

```rust
use anyhow::Result;

fn quick_prototype() -> Result<()> {
    let data = std::fs::read_to_string("data.txt")?;
    // 处理数据
    Ok(())
}
```

## 实际应用示例

### 示例 1：配置文件解析

```rust
use anyhow::{Context, Result};
use serde::Deserialize;

#[derive(Deserialize)]
struct Config {
    host: String,
    port: u16,
}

fn load_config() -> Result<Config> {
    let contents = std::fs::read_to_string("config.toml")
        .context("无法读取配置文件")?;

    let config: Config = toml::from_str(&contents)
        .context("无法解析配置文件")?;

    Ok(config)
}
```

### 示例 2：网络请求

```rust
use anyhow::{Context, Result};

async fn fetch_url(url: &str) -> Result<String> {
    let response = reqwest::get(url)
        .await
        .context("网络请求失败")?;

    let body = response.text()
        .await
        .context("读取响应失败")?;

    Ok(body)
}
```

### 示例 3：数据库操作

```rust
use anyhow::{Context, Result};

fn get_user(id: u32) -> Result<User> {
    let conn = establish_connection()
        .context("无法连接数据库")?;

    let user = query_user(&conn, id)
        .context("查询用户失败")?;

    Ok(user)
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 异常
std::string read_file(const std::string& filename) {
    std::ifstream file(filename);
    if (!file) {
        throw std::runtime_error("无法打开文件");
    }
    // ...
}
```

```rust
// Rust - Result（编译时检查）
fn read_file(filename: &str) -> Result<String, Error> {
    let mut file = File::open(filename)?;
    // ...
    Ok(contents)
}
```

### Go 视角

```go
// Go - 多返回值
func readFile(filename string) (string, error) {
    data, err := ioutil.ReadFile(filename)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

```rust
// Rust - Result（类型安全）
fn read_file(filename: &str) -> Result<String, Error> {
    let contents = std::fs::read_to_string(filename)?;
    Ok(contents)
}
```

## 常见错误与解决方案

### 错误 1：忘记处理错误

```rust
let file = File::open("hello.txt");  // ❌ 返回 Result，需要处理
```

**解决方案：** 使用 `?` 或 `match`：

```rust
let file = File::open("hello.txt")?;  // ✅
// 或
let file = match File::open("hello.txt") {
    Ok(f) => f,
    Err(e) => return Err(e),
};
```

### 错误 2：过度使用 unwrap()

```rust
let value = some_result.unwrap();  // ❌ 可能 panic
```

**解决方案：** 使用 `?` 或 `match`：

```rust
let value = some_result?;  // ✅
// 或
let value = match some_result {
    Ok(v) => v,
    Err(e) => return Err(e),
};
```

### 错误 3：错误类型不匹配

```rust
fn func1() -> Result<(), Error1> { /* ... */ }
fn func2() -> Result<(), Error2> { /* ... */ }

fn combined() -> Result<(), Error1> {
    func1()?;
    func2()?;  // ❌ 类型不匹配
    Ok(())
}
```

**解决方案：** 使用 `map_err` 或定义统一的错误类型：

```rust
fn combined() -> Result<(), Error1> {
    func1()?;
    func2().map_err(|e| Error1::from(e))?;  // ✅
    Ok(())
}
```

## 实践建议

1. **优先使用 Result** - 不要使用 panic 处理可恢复的错误
2. **使用 ? 运算符** - 简化错误传播
3. **提供上下文** - 使用 `context()` 或 `with_context()` 添加错误信息
4. **定义错误类型** - 使用 `thiserror` 定义清晰的错误类型
5. **使用 anyhow 快速原型** - 在需要时再细化错误类型

## 扩展练习

1. **实现一个配置解析器** - 处理多种错误类型
2. **实现一个 HTTP 客户端** - 处理网络错误
3. **实现一个数据库包装器** - 处理数据库错误
4. **实现一个文件处理器** - 处理文件操作错误

## 下一步

掌握了错误处理后，接下来学习：
- **[内存模型](./memory-model.md)** - 深入理解 Rust 的内存管理

---

**记住：Rust 的错误处理通过类型系统在编译时保证错误被处理，这是 Rust 安全性的重要组成部分！** 🦀
