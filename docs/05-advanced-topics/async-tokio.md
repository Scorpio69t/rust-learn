# 异步系统 / 高性能服务端

> **构建高性能异步服务** - Actor 服务器、WebSocket、多路复用 IO、零拷贝

## 为什么选择异步？

异步编程允许你在等待 I/O 操作时执行其他任务，大大提高并发性能：

- **高并发** - 单线程处理数千个连接
- **低延迟** - 无线程切换开销
- **资源高效** - 比线程更轻量

## Tokio 基础

### 基本概念

Tokio 是 Rust 最流行的异步运行时：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("开始");

    tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        println!("异步任务完成");
    });

    println!("主函数继续执行");
    sleep(Duration::from_secs(2)).await;
}
```

### 异步函数

```rust
async fn fetch_data(url: &str) -> Result<String, Box<dyn std::error::Error>> {
    let response = reqwest::get(url).await?;
    let body = response.text().await?;
    Ok(body)
}
```

### 并发执行

```rust
use tokio::time::{sleep, Duration};

async fn task(name: &str, duration: u64) {
    println!("任务 {} 开始", name);
    sleep(Duration::from_secs(duration)).await;
    println!("任务 {} 完成", name);
}

#[tokio::main]
async fn main() {
    // 并发执行多个任务
    tokio::join!(
        task("A", 2),
        task("B", 1),
        task("C", 3),
    );
}
```

## Actor 模型服务器

### 什么是 Actor？

Actor 是一个独立的计算单元，通过消息传递通信：

```rust
use tokio::sync::mpsc;
use std::collections::HashMap;

// Actor 消息类型
enum Message {
    Get { key: String, respond_to: mpsc::Sender<Option<String>> },
    Set { key: String, value: String },
}

// Actor 结构
struct KeyValueActor {
    receiver: mpsc::Receiver<Message>,
    data: HashMap<String, String>,
}

impl KeyValueActor {
    fn new(receiver: mpsc::Receiver<Message>) -> Self {
        Self {
            receiver,
            data: HashMap::new(),
        }
    }

    async fn run(&mut self) {
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                Message::Get { key, respond_to } => {
                    let value = self.data.get(&key).cloned();
                    let _ = respond_to.send(value).await;
                }
                Message::Set { key, value } => {
                    self.data.insert(key, value);
                }
            }
        }
    }
}

// Actor 句柄
#[derive(Clone)]
struct KeyValueHandle {
    sender: mpsc::Sender<Message>,
}

impl KeyValueHandle {
    fn new() -> Self {
        let (sender, receiver) = mpsc::channel(100);
        let mut actor = KeyValueActor::new(receiver);

        tokio::spawn(async move {
            actor.run().await;
        });

        Self { sender }
    }

    async fn get(&self, key: String) -> Option<String> {
        let (send, mut recv) = mpsc::channel(1);
        let msg = Message::Get {
            key,
            respond_to: send,
        };

        let _ = self.sender.send(msg).await;
        recv.recv().await.unwrap()
    }

    async fn set(&self, key: String, value: String) {
        let msg = Message::Set { key, value };
        let _ = self.sender.send(msg).await;
    }
}

#[tokio::main]
async fn main() {
    let kv = KeyValueHandle::new();

    kv.set("hello".to_string(), "world".to_string()).await;
    let value = kv.get("hello".to_string()).await;
    println!("值: {:?}", value);
}
```

## WebSocket 服务

### 使用 tokio-tungstenite

添加依赖：

```toml
[dependencies]
tokio = { version = "1.0", features = ["full"] }
tokio-tungstenite = "0.20"
futures-util = "0.3"
```

```rust
use futures_util::{SinkExt, StreamExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_tungstenite::{accept_async, tungstenite::Message};

async fn handle_client(stream: TcpStream, addr: std::net::SocketAddr) {
    println!("新 WebSocket 连接: {}", addr);

    let ws_stream = match accept_async(stream).await {
        Ok(ws) => ws,
        Err(e) => {
            eprintln!("WebSocket 握手失败: {}", e);
            return;
        }
    };

    let (mut ws_sender, mut ws_receiver) = ws_stream.split();

    // 发送欢迎消息
    let _ = ws_sender.send(Message::Text("欢迎!".to_string())).await;

    // 处理消息
    while let Some(msg) = ws_receiver.next().await {
        match msg {
            Ok(Message::Text(text)) => {
                println!("收到: {}", text);
                // 回声消息
                let _ = ws_sender.send(Message::Text(text)).await;
            }
            Ok(Message::Close(_)) => {
                println!("客户端关闭连接");
                break;
            }
            Err(e) => {
                eprintln!("错误: {}", e);
                break;
            }
            _ => {}
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("WebSocket 服务器运行在 ws://127.0.0.1:8080");

    while let Ok((stream, addr)) = listener.accept().await {
        tokio::spawn(handle_client(stream, addr));
    }

    Ok(())
}
```

## 多路复用 IO 服务

### 使用 epoll/kqueue

Tokio 底层使用 epoll（Linux）或 kqueue（macOS/BSD）：

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn handle_connection(mut stream: TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    let mut buf = [0; 1024];

    loop {
        match stream.read(&mut buf).await? {
            0 => return Ok(()),
            n => {
                stream.write_all(&buf[0..n]).await?;
            }
        }
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("服务器运行在 127.0.0.1:8080");

    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle_connection(stream));
    }
}
```

### 自定义事件循环

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(100);

    // 生产者
    for i in 0..10 {
        let tx = tx.clone();
        tokio::spawn(async move {
            tx.send(i).await.unwrap();
        });
    }
    drop(tx);

    // 消费者
    while let Some(msg) = rx.recv().await {
        println!("收到: {}", msg);
    }
}
```

## 零拷贝技术

### 使用 sendfile

```rust
use tokio::fs::File;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

async fn send_file(stream: &mut TcpStream, file_path: &str) -> Result<(), Box<dyn std::error::Error>> {
    let mut file = File::open(file_path).await?;
    let mut buffer = vec![0; 8192];

    loop {
        let n = file.read(&mut buffer).await?;
        if n == 0 {
            break;
        }
        stream.write_all(&buffer[0..n]).await?;
    }

    Ok(())
}
```

### 使用内存映射

```rust
use memmap2::MmapOptions;
use std::fs::File;

fn send_mmap_file(file: File) -> Result<(), Box<dyn std::error::Error>> {
    let mmap = unsafe { MmapOptions::new().map(&file)? };
    // 直接使用 mmap 数据，无需复制
    Ok(())
}
```

## 高性能 HTTP 服务器

### 使用 Axum

```rust
use axum::{
    routing::get,
    Router,
    extract::State,
    response::Json,
};
use std::sync::Arc;
use tokio::sync::RwLock;

type AppState = Arc<RwLock<u64>>;

async fn get_counter(State(state): State<AppState>) -> Json<u64> {
    let count = state.read().await;
    Json(*count)
}

async fn increment_counter(State(state): State<AppState>) -> Json<u64> {
    let mut count = state.write().await;
    *count += 1;
    Json(*count)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(RwLock::new(0));

    let app = Router::new()
        .route("/", get(get_counter))
        .route("/increment", get(increment_counter))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("服务器运行在 http://0.0.0.0:3000");

    axum::serve(listener, app).await.unwrap();
}
```

## 连接池

### 实现连接池

```rust
use std::sync::Arc;
use tokio::sync::{Semaphore, Mutex};
use tokio::net::TcpStream;

struct ConnectionPool {
    semaphore: Arc<Semaphore>,
    connections: Arc<Mutex<Vec<TcpStream>>>,
}

impl ConnectionPool {
    fn new(max_connections: usize) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_connections)),
            connections: Arc::new(Mutex::new(Vec::new())),
        }
    }

    async fn get_connection(&self) -> Result<TcpStream, Box<dyn std::error::Error>> {
        let _permit = self.semaphore.acquire().await?;

        // 尝试从池中获取连接
        let mut connections = self.connections.lock().await;
        if let Some(conn) = connections.pop() {
            return Ok(conn);
        }

        // 创建新连接
        TcpStream::connect("127.0.0.1:8080").await
            .map_err(|e| e.into())
    }

    async fn return_connection(&self, conn: TcpStream) {
        let mut connections = self.connections.lock().await;
        connections.push(conn);
    }
}
```

## 性能优化技巧

### 1. 使用异步 I/O

```rust
// ❌ 同步 I/O（阻塞）
let data = std::fs::read_to_string("file.txt")?;

// ✅ 异步 I/O（非阻塞）
let data = tokio::fs::read_to_string("file.txt").await?;
```

### 2. 批量处理

```rust
use tokio::time::{sleep, Duration};

async fn process_batch(items: Vec<String>) {
    // 批量处理，减少系统调用
    for item in items {
        // 处理项目
    }
}
```

### 3. 使用缓冲

```rust
use tokio::io::{BufReader, BufWriter};

let reader = BufReader::new(stream);
let writer = BufWriter::new(stream);
```

### 4. 避免不必要的克隆

```rust
// ❌ 不必要的克隆
let data = large_data.clone();

// ✅ 使用引用
let data = &large_data;
```

## 实际应用示例

### 示例 1：高并发 HTTP 代理

```rust
use axum::{
    extract::Request,
    http::StatusCode,
    response::Response,
};
use tower::ServiceBuilder;
use tower_http::trace::TraceLayer;

async fn proxy_handler(req: Request) -> Result<Response, StatusCode> {
    // 转发请求到后端服务器
    // ...
    Ok(Response::new(/* ... */))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/*path", get(proxy_handler))
        .layer(
            ServiceBuilder::new()
                .layer(TraceLayer::new_for_http())
                .into_inner()
        );

    // ...
}
```

### 示例 2：实时数据流处理

```rust
use tokio::sync::broadcast;

#[tokio::main]
async fn main() {
    let (tx, _rx) = broadcast::channel(100);

    // 生产者
    tokio::spawn(async move {
        for i in 0..100 {
            let _ = tx.send(i);
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    });

    // 多个消费者
    for _ in 0..5 {
        let mut rx = tx.subscribe();
        tokio::spawn(async move {
            while let Ok(msg) = rx.recv().await {
                println!("收到: {}", msg);
            }
        });
    }
}
```

## 监控和调试

### 使用 tracing

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = "0.3"
```

```rust
use tracing::{info, error, warn};

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    info!("服务器启动");

    // 你的代码
}
```

## 常见问题

### Q: 如何处理阻塞操作？

**A:** 使用 `spawn_blocking`：

```rust
use tokio::task;

let result = task::spawn_blocking(|| {
    // CPU 密集型或阻塞操作
    heavy_computation()
}).await?;
```

### Q: 如何限制并发数？

**A:** 使用 `Semaphore`：

```rust
use tokio::sync::Semaphore;

let semaphore = Arc::new(Semaphore::new(10));

for _ in 0..100 {
    let sem = semaphore.clone();
    tokio::spawn(async move {
        let _permit = sem.acquire().await.unwrap();
        // 执行任务
    });
}
```

## 扩展练习

1. **实现一个高性能 HTTP 服务器** - 支持百万并发连接
2. **实现一个 WebSocket 聊天室** - 实时消息传递
3. **实现一个异步任务队列** - 使用 Actor 模型
4. **优化现有服务** - 使用零拷贝和连接池

## 下一步

掌握了异步编程后，你可以：
- 构建高性能 Web 服务
- 实现实时系统
- 处理大规模并发

---

**记住：异步编程是 Rust 高性能的关键，Tokio 提供了强大的异步运行时！** 🦀
