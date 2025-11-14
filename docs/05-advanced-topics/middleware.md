# 中间件开发

> **构建可复用的中间件组件** - 事件系统、调度、连接池、线程池、Actor 模型、Protobuf 编解码器

## 什么是中间件？

中间件是位于应用程序和底层系统之间的软件层，提供：
- **可复用功能** - 连接池、线程池等
- **抽象层** - 简化复杂操作
- **横切关注点** - 日志、监控、认证等

## 事件系统

### 基本事件系统

```rust
use std::sync::Arc;
use tokio::sync::broadcast;

type EventSender = broadcast::Sender<Event>;
type EventReceiver = broadcast::Receiver<Event>;

#[derive(Clone, Debug)]
enum Event {
    UserLogin { user_id: u32 },
    UserLogout { user_id: u32 },
    MessageSent { from: u32, to: u32, content: String },
}

struct EventBus {
    sender: EventSender,
}

impl EventBus {
    fn new() -> Self {
        let (sender, _) = broadcast::channel(100);
        Self { sender }
    }

    fn subscribe(&self) -> EventReceiver {
        self.sender.subscribe()
    }

    fn publish(&self, event: Event) -> Result<usize, broadcast::error::SendError<Event>> {
        self.sender.send(event)
    }
}

#[tokio::main]
async fn main() {
    let bus = Arc::new(EventBus::new());

    // 订阅者 1
    let bus1 = bus.clone();
    tokio::spawn(async move {
        let mut rx = bus1.subscribe();
        while let Ok(event) = rx.recv().await {
            println!("订阅者 1 收到: {:?}", event);
        }
    });

    // 订阅者 2
    let bus2 = bus.clone();
    tokio::spawn(async move {
        let mut rx = bus2.subscribe();
        while let Ok(event) = rx.recv().await {
            println!("订阅者 2 收到: {:?}", event);
        }
    });

    // 发布事件
    bus.publish(Event::UserLogin { user_id: 1 }).unwrap();
    bus.publish(Event::MessageSent {
        from: 1,
        to: 2,
        content: "Hello".to_string(),
    }).unwrap();

    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

### 类型安全的事件系统

```rust
use std::any::Any;
use std::sync::Arc;
use tokio::sync::mpsc;

trait EventHandler: Send + Sync {
    fn handle(&self, event: &dyn Any);
}

struct EventDispatcher {
    handlers: Vec<Arc<dyn EventHandler>>,
}

impl EventDispatcher {
    fn new() -> Self {
        Self {
            handlers: Vec::new(),
        }
    }

    fn register(&mut self, handler: Arc<dyn EventHandler>) {
        self.handlers.push(handler);
    }

    fn dispatch(&self, event: &dyn Any) {
        for handler in &self.handlers {
            handler.handle(event);
        }
    }
}
```

## 任务调度器

### 简单的任务调度器

```rust
use std::sync::Arc;
use tokio::sync::mpsc;
use tokio::time::{Duration, Instant};

type Task = Box<dyn Fn() + Send + 'static>;

struct Scheduler {
    task_sender: mpsc::Sender<(Instant, Task)>,
}

impl Scheduler {
    fn new() -> Self {
        let (tx, mut rx) = mpsc::channel(100);

        tokio::spawn(async move {
            let mut tasks = Vec::new();

            loop {
                // 接收新任务
                tokio::select! {
                    Some((time, task)) = rx.recv() => {
                        tasks.push((time, task));
                        tasks.sort_by_key(|(time, _)| *time);
                    }
                    _ = tokio::time::sleep(Duration::from_millis(100)) => {
                        // 检查到期的任务
                        let now = Instant::now();
                        while let Some((time, task)) = tasks.first() {
                            if *time <= now {
                                let (_, task) = tasks.remove(0);
                                task();
                            } else {
                                break;
                            }
                        }
                    }
                }
            }
        });

        Self { task_sender: tx }
    }

    fn schedule(&self, delay: Duration, task: Task) {
        let execute_at = Instant::now() + delay;
        let _ = self.task_sender.try_send((execute_at, task));
    }
}

#[tokio::main]
async fn main() {
    let scheduler = Arc::new(Scheduler::new());

    scheduler.schedule(
        Duration::from_secs(2),
        Box::new(|| println!("任务 1 执行")),
    );

    scheduler.schedule(
        Duration::from_secs(1),
        Box::new(|| println!("任务 2 执行")),
    );

    tokio::time::sleep(Duration::from_secs(3)).await;
}
```

## 连接池

### 通用连接池

```rust
use std::sync::Arc;
use tokio::sync::{Semaphore, Mutex};
use tokio::net::TcpStream;

struct Connection {
    stream: TcpStream,
    created_at: std::time::Instant,
}

struct ConnectionPool {
    semaphore: Arc<Semaphore>,
    connections: Arc<Mutex<Vec<Connection>>>,
    max_connections: usize,
    max_idle_time: std::time::Duration,
}

impl ConnectionPool {
    fn new(max_connections: usize, max_idle_time: std::time::Duration) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_connections)),
            connections: Arc::new(Mutex::new(Vec::new())),
            max_connections,
            max_idle_time,
        }
    }

    async fn get_connection(&self, addr: &str) -> Result<PooledConnection, Box<dyn std::error::Error>> {
        let _permit = self.semaphore.acquire().await?;

        // 尝试从池中获取连接
        let mut connections = self.connections.lock().await;
        let now = std::time::Instant::now();

        // 清理过期连接
        connections.retain(|conn| now.duration_since(conn.created_at) < self.max_idle_time);

        if let Some(conn) = connections.pop() {
            return Ok(PooledConnection {
                connection: conn,
                pool: self.clone(),
            });
        }

        // 创建新连接
        let stream = TcpStream::connect(addr).await?;
        let connection = Connection {
            stream,
            created_at: now,
        };

        Ok(PooledConnection {
            connection,
            pool: self.clone(),
        })
    }

    async fn return_connection(&self, conn: Connection) {
        let mut connections = self.connections.lock().await;
        if connections.len() < self.max_connections {
            connections.push(conn);
        }
    }
}

struct PooledConnection {
    connection: Connection,
    pool: Arc<ConnectionPool>,
}

impl Drop for PooledConnection {
    fn drop(&mut self) {
        let pool = self.pool.clone();
        let conn = std::mem::replace(&mut self.connection, Connection {
            stream: unsafe { std::mem::zeroed() },
            created_at: std::time::Instant::now(),
        });

        tokio::spawn(async move {
            pool.return_connection(conn).await;
        });
    }
}
```

## 线程池

### 异步任务池

```rust
use std::sync::Arc;
use tokio::sync::mpsc;
use tokio::task::JoinHandle;

type Task = Box<dyn FnOnce() + Send + 'static>;

struct ThreadPool {
    sender: mpsc::Sender<Task>,
    handles: Vec<JoinHandle<()>>,
}

impl ThreadPool {
    fn new(size: usize) -> Self {
        let (sender, mut receiver) = mpsc::channel(100);
        let mut handles = Vec::new();

        for _ in 0..size {
            let mut receiver = receiver.clone();
            let handle = tokio::spawn(async move {
                while let Some(task) = receiver.recv().await {
                    task();
                }
            });
            handles.push(handle);
        }

        Self { sender, handles }
    }

    async fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let _ = self.sender.send(Box::new(f)).await;
    }
}

#[tokio::main]
async fn main() {
    let pool = Arc::new(ThreadPool::new(4));

    for i in 0..10 {
        let pool = pool.clone();
        pool.execute(move || {
            println!("任务 {} 执行", i);
        }).await;
    }

    tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;
}
```

## Actor 模型中间件

### Actor 系统

```rust
use std::sync::Arc;
use tokio::sync::mpsc;

enum ActorMessage {
    Process { data: Vec<u8> },
    Shutdown,
}

struct Actor {
    receiver: mpsc::Receiver<ActorMessage>,
    state: ActorState,
}

struct ActorState {
    processed_count: u64,
}

impl Actor {
    fn new(receiver: mpsc::Receiver<ActorMessage>) -> Self {
        Self {
            receiver,
            state: ActorState {
                processed_count: 0,
            },
        }
    }

    async fn run(&mut self) {
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                ActorMessage::Process { data } => {
                    self.handle_process(data).await;
                }
                ActorMessage::Shutdown => {
                    break;
                }
            }
        }
    }

    async fn handle_process(&mut self, data: Vec<u8>) {
        // 处理数据
        self.state.processed_count += 1;
        println!("处理了 {} 个消息", self.state.processed_count);
    }
}

#[derive(Clone)]
struct ActorRef {
    sender: mpsc::Sender<ActorMessage>,
}

impl ActorRef {
    fn new() -> Self {
        let (sender, receiver) = mpsc::channel(100);
        let mut actor = Actor::new(receiver);

        tokio::spawn(async move {
            actor.run().await;
        });

        Self { sender }
    }

    async fn process(&self, data: Vec<u8>) {
        let _ = self.sender.send(ActorMessage::Process { data }).await;
    }

    async fn shutdown(&self) {
        let _ = self.sender.send(ActorMessage::Shutdown).await;
    }
}
```

## Protobuf 编解码器

### 使用 prost

添加依赖：

```toml
[dependencies]
prost = "0.12"
tokio = { version = "1.0", features = ["full"] }
bytes = "1.0"
```

定义 Protobuf 消息：

```protobuf
// message.proto
syntax = "proto3";

package example;

message User {
    uint32 id = 1;
    string name = 2;
    string email = 3;
}

message UserList {
    repeated User users = 1;
}
```

Rust 代码：

```rust
use prost::Message;
use bytes::Bytes;

// 编码
fn encode_user(user: &User) -> Vec<u8> {
    let mut buf = Vec::new();
    user.encode(&mut buf).unwrap();
    buf
}

// 解码
fn decode_user(data: &[u8]) -> Result<User, prost::DecodeError> {
    User::decode(data)
}

// 使用示例
fn main() {
    let user = User {
        id: 1,
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
    };

    let encoded = encode_user(&user);
    let decoded = decode_user(&encoded).unwrap();

    println!("原始: {:?}", user);
    println!("解码: {:?}", decoded);
}
```

### 流式编解码

```rust
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;

async fn send_protobuf_message(
    stream: &mut TcpStream,
    message: &impl Message,
) -> Result<(), Box<dyn std::error::Error>> {
    let mut buf = Vec::new();
    message.encode(&mut buf)?;

    // 发送长度前缀
    let len = buf.len() as u32;
    stream.write_u32(len).await?;

    // 发送消息
    stream.write_all(&buf).await?;
    Ok(())
}

async fn receive_protobuf_message<T: Message + Default>(
    stream: &mut TcpStream,
) -> Result<T, Box<dyn std::error::Error>> {
    // 读取长度前缀
    let len = stream.read_u32().await?;

    // 读取消息
    let mut buf = vec![0; len as usize];
    stream.read_exact(&mut buf).await?;

    // 解码
    let message = T::decode(&buf[..])?;
    Ok(message)
}
```

## 实际应用示例

### 示例 1：HTTP 中间件链

```rust
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};

async fn logging_middleware(req: Request, next: Next) -> Response {
    println!("请求: {} {}", req.method(), req.uri());
    next.run(req).await
}

async fn auth_middleware(req: Request, next: Next) -> Response {
    // 检查认证
    next.run(req).await
}

// 使用
let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(logging_middleware))
    .layer(middleware::from_fn(auth_middleware));
```

### 示例 2：消息队列中间件

```rust
use std::sync::Arc;
use tokio::sync::mpsc;

struct MessageQueue {
    sender: mpsc::Sender<Message>,
}

impl MessageQueue {
    fn new() -> (Self, MessageQueueConsumer) {
        let (sender, receiver) = mpsc::channel(1000);
        (
            Self { sender },
            MessageQueueConsumer { receiver },
        )
    }

    async fn publish(&self, message: Message) -> Result<(), mpsc::error::SendError<Message>> {
        self.sender.send(message).await
    }
}

struct MessageQueueConsumer {
    receiver: mpsc::Receiver<Message>,
}

impl MessageQueueConsumer {
    async fn consume(&mut self) -> Option<Message> {
        self.receiver.recv().await
    }
}
```

## 扩展练习

1. **实现一个完整的中间件框架** - 支持插件系统
2. **构建一个高性能连接池** - 支持多种后端
3. **实现一个分布式任务调度器** - 跨机器调度
4. **构建一个消息总线** - 支持多种消息类型

## 下一步

掌握了中间件开发后，你可以：
- 构建可复用的组件
- 简化复杂系统
- 提高代码复用性

---

**记住：中间件是构建大型系统的基础，Rust 的类型系统让中间件既安全又高效！** 🦀
