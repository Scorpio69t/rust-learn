# JSON 序列化/反序列化

> **使用 serde 处理 JSON** - 学习 Rust 的序列化生态系统

## 项目目标

学习如何使用 `serde` 库进行 JSON 序列化和反序列化，这是 Rust 中最常用的序列化库。

## 技能点

- `derive` 宏的使用
- 类型系统
- `Option`、`Result` 的使用
- 自定义序列化/反序列化

## 项目结构

```
json-demo/
├── Cargo.toml
└── src/
    └── main.rs
```

## 步骤 1：创建项目

```bash
cargo new json-demo
cd json-demo
```

## 步骤 2：配置依赖

编辑 `Cargo.toml`：

```toml
[package]
name = "json-demo"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

**依赖说明：**
- `serde` - 序列化框架
- `serde_json` - JSON 格式支持

## 步骤 3：基本示例

### 3.1 序列化结构体

```rust
use serde::{Serialize, Deserialize};
use serde_json;

#[derive(Serialize, Deserialize, Debug)]
struct Person {
    name: String,
    age: u32,
    email: String,
}

fn main() {
    let person = Person {
        name: String::from("Alice"),
        age: 30,
        email: String::from("alice@example.com"),
    };

    // 序列化为 JSON 字符串
    let json = serde_json::to_string(&person).unwrap();
    println!("{}", json);
    // 输出: {"name":"Alice","age":30,"email":"alice@example.com"}

    // 序列化为格式化的 JSON
    let json_pretty = serde_json::to_string_pretty(&person).unwrap();
    println!("{}", json_pretty);
}
```

### 3.2 反序列化 JSON

```rust
fn main() {
    let json = r#"
        {
            "name": "Bob",
            "age": 25,
            "email": "bob@example.com"
        }
    "#;

    let person: Person = serde_json::from_str(json).unwrap();
    println!("{:?}", person);
}
```

## 完整示例

### 示例 1：用户管理系统

```rust
use serde::{Serialize, Deserialize};
use serde_json;
use std::fs;

#[derive(Serialize, Deserialize, Debug, Clone)]
struct User {
    id: u32,
    name: String,
    email: String,
    active: bool,
}

#[derive(Serialize, Deserialize, Debug)]
struct UserDatabase {
    users: Vec<User>,
}

impl UserDatabase {
    fn new() -> Self {
        UserDatabase { users: Vec::new() }
    }

    fn add_user(&mut self, user: User) {
        self.users.push(user);
    }

    fn save_to_file(&self, filename: &str) -> Result<(), Box<dyn std::error::Error>> {
        let json = serde_json::to_string_pretty(self)?;
        fs::write(filename, json)?;
        Ok(())
    }

    fn load_from_file(filename: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let contents = fs::read_to_string(filename)?;
        let db: UserDatabase = serde_json::from_str(&contents)?;
        Ok(db)
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建数据库
    let mut db = UserDatabase::new();

    // 添加用户
    db.add_user(User {
        id: 1,
        name: String::from("Alice"),
        email: String::from("alice@example.com"),
        active: true,
    });

    db.add_user(User {
        id: 2,
        name: String::from("Bob"),
        email: String::from("bob@example.com"),
        active: false,
    });

    // 保存到文件
    db.save_to_file("users.json")?;
    println!("用户数据已保存到 users.json");

    // 从文件加载
    let loaded_db = UserDatabase::load_from_file("users.json")?;
    println!("加载的用户: {:?}", loaded_db);

    Ok(())
}
```

### 示例 2：处理可选字段

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Product {
    id: u32,
    name: String,
    price: f64,
    description: Option<String>,  // 可选字段
    tags: Vec<String>,
}

fn main() {
    // JSON 中有 description
    let json1 = r#"
        {
            "id": 1,
            "name": "Laptop",
            "price": 999.99,
            "description": "高性能笔记本电脑",
            "tags": ["electronics", "computers"]
        }
    "#;

    let product1: Product = serde_json::from_str(json1).unwrap();
    println!("{:?}", product1);

    // JSON 中没有 description（使用 null 或省略）
    let json2 = r#"
        {
            "id": 2,
            "name": "Mouse",
            "price": 29.99,
            "tags": ["electronics", "accessories"]
        }
    "#;

    let product2: Product = serde_json::from_str(json2).unwrap();
    println!("{:?}", product2);
}
```

### 示例 3：自定义字段名

```rust
#[derive(Serialize, Deserialize, Debug)]
struct User {
    #[serde(rename = "user_name")]
    name: String,

    #[serde(rename = "user_age")]
    age: u32,

    #[serde(skip_serializing_if = "Option::is_none")]
    email: Option<String>,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        age: 30,
        email: Some(String::from("alice@example.com")),
    };

    let json = serde_json::to_string_pretty(&user).unwrap();
    println!("{}", json);
    // 输出字段名是 user_name 和 user_age
}
```

### 示例 4：嵌套结构

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Address {
    street: String,
    city: String,
    zip_code: String,
}

#[derive(Serialize, Deserialize, Debug)]
struct Person {
    name: String,
    age: u32,
    address: Address,
    phones: Vec<String>,
}

fn main() {
    let json = r#"
        {
            "name": "Charlie",
            "age": 35,
            "address": {
                "street": "123 Main St",
                "city": "New York",
                "zip_code": "10001"
            },
            "phones": ["123-456-7890", "098-765-4321"]
        }
    "#;

    let person: Person = serde_json::from_str(json).unwrap();
    println!("{:?}", person);
}
```

### 示例 5：枚举序列化

```rust
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
enum Message {
    Text { content: String },
    Image { url: String, width: u32, height: u32 },
    Video { url: String, duration: u32 },
}

fn main() {
    let messages = vec![
        Message::Text {
            content: String::from("Hello!"),
        },
        Message::Image {
            url: String::from("https://example.com/image.jpg"),
            width: 800,
            height: 600,
        },
        Message::Video {
            url: String::from("https://example.com/video.mp4"),
            duration: 120,
        },
    ];

    let json = serde_json::to_string_pretty(&messages).unwrap();
    println!("{}", json);
}
```

## 高级特性

### 自定义序列化

```rust
use serde::{Serializer, Serialize};

fn serialize_bool_as_string<S>(value: &bool, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    serializer.serialize_str(if *value { "yes" } else { "no" })
}

#[derive(Serialize, Debug)]
struct Config {
    #[serde(serialize_with = "serialize_bool_as_string")]
    enabled: bool,
}

fn main() {
    let config = Config { enabled: true };
    let json = serde_json::to_string(&config).unwrap();
    println!("{}", json);  // {"enabled":"yes"}
}
```

### 处理日期时间

添加依赖：

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
chrono = { version = "0.4", features = ["serde"] }
```

```rust
use chrono::{DateTime, Utc};
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Event {
    name: String,
    #[serde(with = "chrono::serde::ts_seconds")]
    timestamp: DateTime<Utc>,
}

fn main() {
    let event = Event {
        name: String::from("Meeting"),
        timestamp: Utc::now(),
    };

    let json = serde_json::to_string(&event).unwrap();
    println!("{}", json);
}
```

### 处理错误

```rust
use serde_json;
use std::fs;

fn load_user(filename: &str) -> Result<User, Box<dyn std::error::Error>> {
    let contents = fs::read_to_string(filename)?;
    let user: User = serde_json::from_str(&contents)?;
    Ok(user)
}

fn main() {
    match load_user("user.json") {
        Ok(user) => println!("加载成功: {:?}", user),
        Err(e) => println!("加载失败: {}", e),
    }
}
```

## 实际应用：配置文件管理

```rust
use serde::{Serialize, Deserialize};
use std::fs;
use std::path::Path;

#[derive(Serialize, Deserialize, Debug)]
struct Config {
    server: ServerConfig,
    database: DatabaseConfig,
    logging: LoggingConfig,
}

#[derive(Serialize, Deserialize, Debug)]
struct ServerConfig {
    host: String,
    port: u16,
    timeout: u64,
}

#[derive(Serialize, Deserialize, Debug)]
struct DatabaseConfig {
    url: String,
    max_connections: u32,
}

#[derive(Serialize, Deserialize, Debug)]
struct LoggingConfig {
    level: String,
    file: Option<String>,
}

impl Config {
    fn load<P: AsRef<Path>>(path: P) -> Result<Self, Box<dyn std::error::Error>> {
        let contents = fs::read_to_string(path)?;
        let config: Config = serde_json::from_str(&contents)?;
        Ok(config)
    }

    fn save<P: AsRef<Path>>(&self, path: P) -> Result<(), Box<dyn std::error::Error>> {
        let json = serde_json::to_string_pretty(self)?;
        fs::write(path, json)?;
        Ok(())
    }
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 创建默认配置
    let config = Config {
        server: ServerConfig {
            host: String::from("localhost"),
            port: 8080,
            timeout: 30,
        },
        database: DatabaseConfig {
            url: String::from("postgresql://localhost/mydb"),
            max_connections: 10,
        },
        logging: LoggingConfig {
            level: String::from("info"),
            file: Some(String::from("app.log")),
        },
    };

    // 保存配置
    config.save("config.json")?;

    // 加载配置
    let loaded_config = Config::load("config.json")?;
    println!("{:?}", loaded_config);

    Ok(())
}
```

## 性能优化

### 使用流式解析（处理大文件）

```rust
use serde_json::Deserializer;
use std::fs::File;
use std::io::BufReader;

fn load_large_json(filename: &str) -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open(filename)?;
    let reader = BufReader::new(file);
    let stream = Deserializer::from_reader(reader).into_iter::<User>();

    for user in stream {
        let user = user?;
        println!("{:?}", user);
    }

    Ok(())
}
```

## 常见问题

### Q: 如何处理 JSON 中的 null？

**A:** 使用 `Option<T>`：

```rust
#[derive(Deserialize)]
struct Data {
    value: Option<String>,  // null 会被解析为 None
}
```

### Q: 如何忽略未知字段？

**A:** 使用 `#[serde(deny_unknown_fields)]` 或允许未知字段：

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]  // 拒绝未知字段
struct StrictData {
    name: String,
}

// 或者允许未知字段（默认行为）
#[derive(Deserialize)]
struct FlexibleData {
    name: String,
    // 其他未知字段会被忽略
}
```

### Q: 如何自定义序列化格式？

**A:** 使用 `#[serde(with = "...")]` 或实现自定义序列化器。

## 扩展练习

1. **实现一个简单的数据库** - 使用 JSON 文件存储数据
2. **配置文件解析器** - 支持多种格式（JSON、TOML、YAML）
3. **API 客户端** - 序列化请求和反序列化响应
4. **数据迁移工具** - 在不同 JSON 格式之间转换

## 下一步

完成了 JSON 序列化后，你已经掌握了：
- `derive` 宏的使用
- 类型系统
- 错误处理
- 文件操作

接下来可以尝试：
- **[小型 Web 服务（Axum）](./web-service.md)** - 构建 RESTful API

---

**记住：serde 是 Rust 生态系统中最强大的序列化库，支持多种格式，性能优秀！** 🦀
