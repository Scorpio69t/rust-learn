# TCP 回声服务器

> **构建一个异步 TCP 服务器** - 学习 Rust 的异步编程和并发模型

## 项目目标

构建一个 TCP 回声服务器，接收客户端连接，将收到的数据原样返回给客户端。

**功能：**
- 监听指定端口
- 接受多个客户端连接
- 将收到的数据原样返回（echo）
- 使用异步编程提高性能

## 技能点

- `async/.await` 语法
- `tokio::spawn` 并发处理
- TCP 网络编程
- Rust 的异步并发模型

## 项目结构

```
tcp-echo-server/
├── Cargo.toml
└── src/
    └── main.rs
```

## 步骤 1：创建项目

```bash
cargo new tcp-echo-server
cd tcp-echo-server
```

## 步骤 2：配置依赖

编辑 `Cargo.toml`：

```toml
[package]
name = "tcp-echo-server"
version = "0.1.0"
edition = "2021"

[dependencies]
tokio = { version = "1.0", features = ["full"] }
```

**依赖说明：**
- `tokio` - Rust 最流行的异步运行时

## 步骤 3：基本实现

### 3.1 最简单的回声服务器

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("服务器运行在 127.0.0.1:8080");

    loop {
        let (mut socket, addr) = listener.accept().await?;
        println!("新连接: {}", addr);

        tokio::spawn(async move {
            let mut buf = [0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    Ok(0) => {
                        println!("客户端 {} 断开连接", addr);
                        break;
                    }
                    Ok(n) => {
                        if socket.write_all(&buf[0..n]).await.is_err() {
                            break;
                        }
                    }
                    Err(_) => {
                        break;
                    }
                }
            }
        });
    }
}
```

## 完整代码

### 版本 1：基本回声服务器

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;
use std::env;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:8080".to_string());

    let listener = TcpListener::bind(&addr).await?;
    println!("回声服务器运行在 {}", addr);

    loop {
        let (mut socket, client_addr) = listener.accept().await?;
        println!("新客户端连接: {}", client_addr);

        // 为每个连接生成一个异步任务
        tokio::spawn(async move {
            let mut buf = vec![0; 1024];

            loop {
                match socket.read(&mut buf).await {
                    // 读取到 0 字节表示客户端关闭连接
                    Ok(0) => {
                        println!("客户端 {} 断开连接", client_addr);
                        return;
                    }
                    // 成功读取数据
                    Ok(n) => {
                        println!("从 {} 收到 {} 字节", client_addr, n);

                        // 将数据原样写回
                        if let Err(e) = socket.write_all(&buf[0..n]).await {
                            eprintln!("写入错误: {}", e);
                            return;
                        }
                    }
                    // 读取错误
                    Err(e) => {
                        eprintln!("读取错误: {}", e);
                        return;
                    }
                }
            }
        });
    }
}
```

### 版本 2：带日志和错误处理

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpListener;
use std::env;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:8080".to_string());

    let listener = TcpListener::bind(&addr).await?;
    println!("回声服务器运行在 {}", addr);
    println!("等待连接...");

    loop {
        match listener.accept().await {
            Ok((mut socket, client_addr)) => {
                println!("[{}] 新客户端连接", client_addr);

                tokio::spawn(async move {
                    if let Err(e) = handle_client(&mut socket, client_addr).await {
                        eprintln!("[{}] 处理客户端时出错: {}", client_addr, e);
                    }
                });
            }
            Err(e) => {
                eprintln!("接受连接时出错: {}", e);
            }
        }
    }
}

async fn handle_client(
    socket: &mut tokio::net::TcpStream,
    addr: std::net::SocketAddr,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut buf = vec![0; 1024];
    let mut total_bytes = 0;

    loop {
        match socket.read(&mut buf).await? {
            0 => {
                println!("[{}] 客户端关闭连接 (总共处理 {} 字节)", addr, total_bytes);
                return Ok(());
            }
            n => {
                total_bytes += n;
                println!("[{}] 收到 {} 字节 (总计: {})", addr, n, total_bytes);

                socket.write_all(&buf[0..n]).await?;
            }
        }
    }
}
```

### 版本 3：使用 BufReader/BufWriter 优化

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;
use std::env;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let addr = env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:8080".to_string());

    let listener = TcpListener::bind(&addr).await?;
    println!("回声服务器运行在 {}", addr);

    loop {
        let (socket, client_addr) = listener.accept().await?;
        println!("新客户端: {}", client_addr);

        tokio::spawn(async move {
            let (reader, mut writer) = socket.split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) => {
                        println!("[{}] 客户端断开", client_addr);
                        break;
                    }
                    Ok(_) => {
                        print!("[{}] 收到: {}", client_addr, line);
                        if writer.write_all(line.as_bytes()).await.is_err() {
                            break;
                        }
                    }
                    Err(e) => {
                        eprintln!("[{}] 读取错误: {}", client_addr, e);
                        break;
                    }
                }
            }
        });
    }
}
```

## 代码解释

### 1. async/await 语法

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // ...
}
```

- `#[tokio::main]` 宏将函数转换为异步运行时入口
- `async fn` 定义异步函数
- `await` 等待异步操作完成

### 2. 并发处理

```rust
tokio::spawn(async move {
    // 处理客户端连接
});
```

- `tokio::spawn` 创建新的异步任务
- `async move` 移动所有权到新任务
- 每个连接在独立的任务中处理

### 3. 异步 I/O

```rust
socket.read(&mut buf).await
socket.write_all(&buf[0..n]).await
```

- `read` 和 `write_all` 是异步操作
- `await` 等待操作完成，不阻塞其他任务

## 测试服务器

### 使用 telnet 测试

```bash
# 启动服务器
cargo run

# 在另一个终端
telnet 127.0.0.1 8080
```

### 使用 nc (netcat) 测试

```bash
# 启动服务器
cargo run

# 在另一个终端
echo "Hello, Rust!" | nc 127.0.0.1 8080
```

### 使用 Rust 客户端测试

创建 `client.rs`：

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut stream = TcpStream::connect("127.0.0.1:8080").await?;

    let message = "Hello, Echo Server!\n";
    stream.write_all(message.as_bytes()).await?;

    let mut buf = vec![0; 1024];
    let n = stream.read(&mut buf).await?;
    println!("收到: {}", String::from_utf8_lossy(&buf[0..n]));

    Ok(())
}
```

## 进阶功能

### 功能 1：添加超时

```rust
use tokio::time::{timeout, Duration};

async fn handle_client_with_timeout(
    socket: &mut tokio::net::TcpStream,
    addr: std::net::SocketAddr,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut buf = vec![0; 1024];

    loop {
        match timeout(Duration::from_secs(30), socket.read(&mut buf)).await {
            Ok(Ok(0)) => {
                println!("[{}] 客户端关闭", addr);
                return Ok(());
            }
            Ok(Ok(n)) => {
                socket.write_all(&buf[0..n]).await?;
            }
            Ok(Err(e)) => {
                return Err(e.into());
            }
            Err(_) => {
                println!("[{}] 超时，关闭连接", addr);
                return Ok(());
            }
        }
    }
}
```

### 功能 2：限制连接数

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    let max_connections = 100;
    let semaphore = Arc::new(Semaphore::new(max_connections));

    loop {
        let (socket, addr) = listener.accept().await?;
        let permit = semaphore.clone().acquire_owned().await.unwrap();

        tokio::spawn(async move {
            // 处理连接
            drop(permit);  // 释放许可
        });
    }
}
```

### 功能 3：添加统计信息

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Default)]
struct Stats {
    total_connections: u64,
    total_bytes: u64,
    active_connections: u64,
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let stats = Arc::new(RwLock::new(Stats::default()));
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    // 定期打印统计信息
    let stats_clone = stats.clone();
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_secs(5));
        loop {
            interval.tick().await;
            let s = stats_clone.read().await;
            println!("统计: 总连接={}, 总字节={}, 活跃={}",
                     s.total_connections, s.total_bytes, s.active_connections);
        }
    });

    loop {
        let (socket, _) = listener.accept().await?;
        let stats = stats.clone();

        {
            let mut s = stats.write().await;
            s.total_connections += 1;
            s.active_connections += 1;
        }

        tokio::spawn(async move {
            // 处理连接
            {
                let mut s = stats.write().await;
                s.active_connections -= 1;
            }
        });
    }
}
```

## 性能优化

### 1. 使用连接池

```rust
use tokio::net::TcpListener;
use std::sync::Arc;
use tokio::sync::Semaphore;

let semaphore = Arc::new(Semaphore::new(1000));
```

### 2. 零拷贝优化

```rust
use tokio::io::copy;

// 使用 copy 进行零拷贝传输
tokio::spawn(async move {
    let (mut reader, mut writer) = socket.split();
    let _ = copy(&mut reader, &mut writer).await;
});
```

## 常见问题

### Q: 如何处理大量并发连接？

**A:**
- 使用异步 I/O（已经实现）
- 调整系统限制：`ulimit -n 10000`
- 使用连接池限制资源

### Q: 如何优雅关闭服务器？

**A:** 使用信号处理：

```rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    tokio::select! {
        _ = async {
            loop {
                // 处理连接
            }
        } => {}
        _ = signal::ctrl_c() => {
            println!("收到关闭信号，优雅关闭...");
        }
    }

    Ok(())
}
```

## 扩展练习

1. **添加命令支持** - 支持不同的命令（如 `/help`, `/quit`）
2. **实现聊天室** - 多个客户端可以互相通信
3. **添加认证** - 要求客户端提供密码
4. **实现文件传输** - 支持文件上传和下载
5. **添加 WebSocket 支持** - 升级为 WebSocket 服务器

## 下一步

完成了 TCP 服务器后，你已经掌握了：
- 异步编程
- 并发处理
- 网络编程
- Tokio 的使用

接下来可以尝试：
- **[小型 Web 服务（Axum）](./web-service.md)** - 构建 HTTP 服务器

---

**记住：Rust 的异步编程模型既安全又高效，Tokio 提供了强大的异步运行时！** 🦀
