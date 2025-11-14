# 命令行工具（CLI）

> **构建一个类似 grep 的命令行工具** - 学习文件操作、错误处理和迭代器

## 项目目标

构建一个命令行工具 `mygrep`，功能类似于 `grep`，可以在文件中搜索指定的模式。

**使用示例：**
```bash
mygrep pattern file.txt
```

## 技能点

- 文件读写操作
- 错误处理（`?` 运算符）
- Trait 和迭代器
- 命令行参数解析

## 项目结构

```
mygrep/
├── Cargo.toml
└── src/
    └── main.rs
```

## 步骤 1：创建项目

```bash
cargo new mygrep
cd mygrep
```

## 步骤 2：配置依赖

编辑 `Cargo.toml`：

```toml
[package]
name = "mygrep"
version = "0.1.0"
edition = "2021"

[dependencies]
clap = { version = "4.0", features = ["derive"] }
anyhow = "1.0"
```

**依赖说明：**
- `clap` - 强大的命令行参数解析库
- `anyhow` - 简化的错误处理

## 步骤 3：实现基本功能

### 3.1 定义命令行参数

```rust
use clap::Parser;

/// 在文件中搜索模式
#[derive(Parser, Debug)]
#[command(name = "mygrep")]
#[command(about = "在文件中搜索指定的模式", long_about = None)]
struct Args {
    /// 要搜索的模式
    pattern: String,

    /// 要搜索的文件路径
    file_path: String,

    /// 是否忽略大小写
    #[arg(short, long)]
    ignore_case: bool,
}
```

### 3.2 实现搜索功能

```rust
use std::fs;
use anyhow::{Context, Result};

fn search_in_file(pattern: &str, file_path: &str, ignore_case: bool) -> Result<()> {
    let contents = fs::read_to_string(file_path)
        .with_context(|| format!("无法读取文件: {}", file_path))?;

    let search_pattern = if ignore_case {
        pattern.to_lowercase()
    } else {
        pattern.to_string()
    };

    for (line_num, line) in contents.lines().enumerate() {
        let search_line = if ignore_case {
            line.to_lowercase()
        } else {
            line.to_string()
        };

        if search_line.contains(&search_pattern) {
            println!("{}:{}: {}", file_path, line_num + 1, line);
        }
    }

    Ok(())
}
```

### 3.3 主函数

```rust
use clap::Parser;
use anyhow::Result;

fn main() -> Result<()> {
    let args = Args::parse();

    search_in_file(&args.pattern, &args.file_path, args.ignore_case)?;

    Ok(())
}
```

## 完整代码

```rust
use clap::Parser;
use anyhow::{Context, Result};
use std::fs;

/// 在文件中搜索模式
#[derive(Parser, Debug)]
#[command(name = "mygrep")]
#[command(about = "在文件中搜索指定的模式", long_about = None)]
struct Args {
    /// 要搜索的模式
    pattern: String,

    /// 要搜索的文件路径
    file_path: String,

    /// 是否忽略大小写
    #[arg(short, long)]
    ignore_case: bool,
}

fn search_in_file(pattern: &str, file_path: &str, ignore_case: bool) -> Result<()> {
    let contents = fs::read_to_string(file_path)
        .with_context(|| format!("无法读取文件: {}", file_path))?;

    let search_pattern = if ignore_case {
        pattern.to_lowercase()
    } else {
        pattern.to_string()
    };

    for (line_num, line) in contents.lines().enumerate() {
        let search_line = if ignore_case {
            line.to_lowercase()
        } else {
            line.to_string()
        };

        if search_line.contains(&search_pattern) {
            println!("{}:{}: {}", file_path, line_num + 1, line);
        }
    }

    Ok(())
}

fn main() -> Result<()> {
    let args = Args::parse();

    search_in_file(&args.pattern, &args.file_path, args.ignore_case)?;

    Ok(())
}
```

## 代码解释

### 1. 命令行参数解析

```rust
#[derive(Parser, Debug)]
struct Args {
    pattern: String,
    file_path: String,
    #[arg(short, long)]
    ignore_case: bool,
}
```

- `Parser` derive 宏自动生成参数解析代码
- `#[arg(short, long)]` 表示可以使用 `-i` 或 `--ignore-case`

### 2. 错误处理

```rust
let contents = fs::read_to_string(file_path)
    .with_context(|| format!("无法读取文件: {}", file_path))?;
```

- `?` 运算符自动传播错误
- `with_context` 添加错误上下文信息
- `Result<()>` 表示可能失败但不返回值的操作

### 3. 迭代器使用

```rust
for (line_num, line) in contents.lines().enumerate() {
    // ...
}
```

- `lines()` 返回行的迭代器
- `enumerate()` 添加行号索引

## 运行和测试

### 编译

```bash
cargo build --release
```

### 运行

```bash
# 基本用法
cargo run -- pattern file.txt

# 忽略大小写
cargo run -- -i pattern file.txt

# 或者使用编译后的二进制
./target/release/mygrep pattern file.txt
```

### 测试文件

创建一个测试文件 `test.txt`：

```
这是第一行
这是第二行
这是第三行
包含 pattern 的行
```

运行：

```bash
cargo run -- pattern test.txt
```

输出：

```
test.txt:4: 包含 pattern 的行
```

## 进阶功能

### 功能 1：支持多个文件

```rust
#[derive(Parser, Debug)]
struct Args {
    pattern: String,

    /// 要搜索的文件路径（可以指定多个）
    file_paths: Vec<String>,

    #[arg(short, long)]
    ignore_case: bool,
}

fn main() -> Result<()> {
    let args = Args::parse();

    for file_path in &args.file_paths {
        search_in_file(&args.pattern, file_path, args.ignore_case)?;
    }

    Ok(())
}
```

### 功能 2：支持正则表达式

添加依赖：

```toml
[dependencies]
regex = "1.10"
```

修改搜索函数：

```rust
use regex::Regex;

fn search_in_file(pattern: &str, file_path: &str, ignore_case: bool) -> Result<()> {
    let contents = fs::read_to_string(file_path)
        .with_context(|| format!("无法读取文件: {}", file_path))?;

    let regex = if ignore_case {
        Regex::new(&format!("(?i){}", pattern))?
    } else {
        Regex::new(pattern)?
    };

    for (line_num, line) in contents.lines().enumerate() {
        if regex.is_match(line) {
            println!("{}:{}: {}", file_path, line_num + 1, line);
        }
    }

    Ok(())
}
```

### 功能 3：彩色输出

添加依赖：

```toml
[dependencies]
colored = "2.0"
```

修改输出：

```rust
use colored::*;

if search_line.contains(&search_pattern) {
    println!(
        "{}:{}: {}",
        file_path.blue(),
        (line_num + 1).to_string().green(),
        line
    );
}
```

## 错误处理最佳实践

### 使用 anyhow

```rust
use anyhow::{Context, Result};

fn read_file(path: &str) -> Result<String> {
    fs::read_to_string(path)
        .with_context(|| format!("无法读取文件: {}", path))
}
```

### 使用 thiserror（更精确的错误类型）

```toml
[dependencies]
thiserror = "1.0"
```

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum MyGrepError {
    #[error("文件读取错误: {0}")]
    IoError(#[from] std::io::Error),

    #[error("文件未找到: {0}")]
    FileNotFound(String),
}
```

## 测试

创建 `tests/integration_test.rs`：

```rust
use std::process::Command;

#[test]
fn test_basic_search() {
    let output = Command::new("cargo")
        .args(&["run", "--", "pattern", "test.txt"])
        .output()
        .expect("执行失败");

    let stdout = String::from_utf8(output.stdout).unwrap();
    assert!(stdout.contains("pattern"));
}
```

运行测试：

```bash
cargo test
```

## 常见问题

### Q: 如何处理大文件？

**A:** 使用流式读取：

```rust
use std::io::{BufRead, BufReader};
use std::fs::File;

fn search_in_file_large(pattern: &str, file_path: &str) -> Result<()> {
    let file = File::open(file_path)?;
    let reader = BufReader::new(file);

    for (line_num, line) in reader.lines().enumerate() {
        let line = line?;
        if line.contains(pattern) {
            println!("{}:{}: {}", file_path, line_num + 1, line);
        }
    }

    Ok(())
}
```

### Q: 如何提高性能？

**A:**
- 使用并行处理（`rayon` 库）
- 使用更高效的字符串搜索算法（Boyer-Moore）
- 使用内存映射文件（`memmap` 库）

## 扩展练习

1. **添加行号选项** - 允许用户选择是否显示行号
2. **添加计数功能** - 只显示匹配的行数
3. **支持递归搜索** - 在目录中递归搜索
4. **支持排除文件** - 忽略某些文件类型
5. **添加配置文件** - 支持配置文件自定义行为

## 下一步

完成了 CLI 工具后，你已经掌握了：
- 文件操作
- 错误处理
- 命令行参数解析
- 迭代器使用

接下来可以尝试：
- **[JSON 序列化/反序列化](./json-serde.md)** - 学习使用 serde

---

**记住：CLI 工具是 Rust 的强项之一，类型安全和错误处理让 CLI 工具既安全又易用！** 🦀
