# 并发编程

> **Rust 的并发模型** - 通过所有权和类型系统保证线程安全

## Rust 的并发哲学

Rust 的并发模型基于以下原则：

1. **编译时保证线程安全** - 不需要运行时检查
2. **零开销抽象** - 并发原语没有运行时开销
3. **类型系统防止数据竞争** - 借用检查器保证安全

## 线程（Threads）

### 创建线程

```rust
use std::thread;
use std::time::Duration;

thread::spawn(|| {
    for i in 1..10 {
        println!("线程中的数字: {}", i);
        thread::sleep(Duration::from_millis(1));
    }
});

for i in 1..5 {
    println!("主线程中的数字: {}", i);
    thread::sleep(Duration::from_millis(1));
}
```

### 等待线程完成

```rust
use std::thread;

let handle = thread::spawn(|| {
    "Hello from a thread!"
});

let result = handle.join().unwrap();
println!("{}", result);
```

### 使用 move 闭包

```rust
use std::thread;

let v = vec![1, 2, 3];

let handle = thread::spawn(move || {
    println!("这是向量: {:?}", v);
});

handle.join().unwrap();
```

## 消息传递（Message Passing）

### Channel - 线程间通信

Rust 的并发模型基于"不要通过共享内存来通信，而是通过通信来共享内存"：

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

thread::spawn(move || {
    let val = String::from("hi");
    tx.send(val).unwrap();
});

let received = rx.recv().unwrap();
println!("收到: {}", received);
```

### 多个发送者

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();
let tx1 = mpsc::Sender::clone(&tx);

thread::spawn(move || {
    let vals = vec![
        String::from("hi"),
        String::from("from"),
        String::from("the"),
        String::from("thread"),
    ];

    for val in vals {
        tx.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

thread::spawn(move || {
    let vals = vec![
        String::from("more"),
        String::from("messages"),
        String::from("for"),
        String::from("you"),
    ];

    for val in vals {
        tx1.send(val).unwrap();
        thread::sleep(Duration::from_secs(1));
    }
});

for received in rx {
    println!("收到: {}", received);
}
```

## 共享状态（Shared State）

### Mutex<T> - 互斥锁

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

for handle in handles {
    handle.join().unwrap();
}

println!("结果: {}", *counter.lock().unwrap());
```

### RwLock<T> - 读写锁

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let data = Arc::new(RwLock::new(5));

// 多个读操作可以同时进行
let read1 = Arc::clone(&data);
thread::spawn(move || {
    let r = read1.read().unwrap();
    println!("读取: {}", *r);
});

// 写操作需要独占
let write = Arc::clone(&data);
thread::spawn(move || {
    let mut w = write.write().unwrap();
    *w += 1;
});
```

### Mutex vs RwLock

| 特性 | Mutex<T> | RwLock<T> |
|------|----------|-----------|
| 读操作 | 独占 | 共享 |
| 写操作 | 独占 | 独占 |
| 性能 | 更快 | 稍慢 |
| 使用场景 | 读写频率相近 | 读多写少 |

## Send 和 Sync Trait

### Send Trait

`Send` 标记 trait 表示类型的所有权可以在线程间传递：

```rust
use std::thread;

// 大多数类型都实现了 Send
let handle = thread::spawn(move || {
    let x = 5;  // i32 实现了 Send
    println!("{}", x);
});

handle.join().unwrap();
```

### Sync Trait

`Sync` 标记 trait 表示类型可以安全地在多个线程中共享引用：

```rust
// Arc<T> 实现了 Sync（如果 T 实现了 Sync）
use std::sync::Arc;
use std::thread;

let data = Arc::new(5);

for i in 0..10 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("{}", data);  // Arc<i32> 实现了 Sync
    });
}
```

### 自动实现

大多数类型自动实现了 `Send` 和 `Sync`：

- **实现了 Send 的类型：** 大多数类型，除了 `Rc<T>`
- **实现了 Sync 的类型：** 大多数类型，除了 `Rc<T>` 和 `RefCell<T>`

## 常见并发模式

### 模式 1：工作池（Worker Pool）

```rust
use std::sync::mpsc;
use std::thread;

type Job = Box<dyn FnOnce() + Send + 'static>;

struct Worker {
    id: usize,
    thread: thread::JoinHandle<()>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let job = receiver.lock().unwrap().recv().unwrap();
            println!("Worker {} 执行任务", id);
            job();
        });

        Worker { id, thread }
    }
}
```

### 模式 2：生产者-消费者

```rust
use std::sync::mpsc;
use std::thread;

let (tx, rx) = mpsc::channel();

// 生产者
thread::spawn(move || {
    for i in 0..10 {
        tx.send(i).unwrap();
    }
});

// 消费者
for received in rx {
    println!("收到: {}", received);
}
```

### 模式 3：屏障（Barrier）

```rust
use std::sync::{Arc, Barrier};
use std::thread;

let barrier = Arc::new(Barrier::new(3));

for i in 0..3 {
    let barrier = Arc::clone(&barrier);
    thread::spawn(move || {
        println!("线程 {} 到达屏障前", i);
        barrier.wait();
        println!("线程 {} 通过屏障", i);
    });
}
```

## 异步编程基础

### async/await 语法

```rust
use std::future::Future;

async fn hello() -> String {
    String::from("Hello")
}

async fn world() -> String {
    String::from("World")
}

async fn hello_world() {
    let h = hello().await;
    let w = world().await;
    println!("{} {}", h, w);
}
```

### 使用 Tokio

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

### 并发执行多个异步任务

```rust
use tokio::time::{sleep, Duration};

async fn task(name: &str, duration: u64) {
    println!("任务 {} 开始", name);
    sleep(Duration::from_secs(duration)).await;
    println!("任务 {} 完成", name);
}

#[tokio::main]
async fn main() {
    tokio::join!(
        task("A", 2),
        task("B", 1),
        task("C", 3),
    );
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 手动管理线程和锁
#include <thread>
#include <mutex>

std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;
}

std::thread t1(increment);
std::thread t2(increment);
// 可能数据竞争，需要手动保证安全
```

```rust
// Rust - 编译时保证安全
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));

let c1 = Arc::clone(&counter);
let t1 = thread::spawn(move || {
    let mut num = c1.lock().unwrap();
    *num += 1;
});
// 编译时保证线程安全
```

### Go 视角

```go
// Go - 使用 goroutine 和 channel
var counter int
var mu sync.Mutex

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}

go increment()
go increment()
// 运行时检查，可能死锁
```

```rust
// Rust - 使用线程和 channel
use std::sync::mpsc;

let (tx, rx) = mpsc::channel();
// 编译时检查，不可能死锁（在 channel 层面）
```

## 常见错误与解决方案

### 错误 1：数据竞争

```rust
let mut counter = 0;

for _ in 0..10 {
    thread::spawn(move || {
        counter += 1;  // ❌ 编译错误：不能同时有多个可变引用
    });
}
```

**解决方案：** 使用 `Arc<Mutex<T>>`：

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
}
```

### 错误 2：死锁

```rust
let m1 = Arc::new(Mutex::new(0));
let m2 = Arc::new(Mutex::new(0));

let m1_clone = Arc::clone(&m1);
let m2_clone = Arc::clone(&m2);

thread::spawn(move || {
    let _a = m1_clone.lock().unwrap();
    let _b = m2_clone.lock().unwrap();
});

let _b = m2.lock().unwrap();
let _a = m1.lock().unwrap();  // 可能死锁
```

**解决方案：** 总是以相同顺序获取锁。

### 错误 3：忘记 join

```rust
thread::spawn(|| {
    // 长时间运行的任务
});

// 主线程可能先退出
```

**解决方案：** 保存 handle 并 join：

```rust
let handle = thread::spawn(|| {
    // 长时间运行的任务
});

handle.join().unwrap();
```

## 实践建议

1. **优先使用消息传递** - Channel 比共享状态更安全
2. **需要共享状态时用 Mutex** - 简单直接
3. **读多写少用 RwLock** - 性能更好
4. **使用 Arc 共享数据** - 多线程环境
5. **避免死锁** - 总是以相同顺序获取锁
6. **考虑异步编程** - 对于 I/O 密集型任务

## 实际应用示例

### 示例 1：并行计算

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn parallel_sum(numbers: Vec<i32>) -> i32 {
    let num_threads = 4;
    let chunk_size = numbers.len() / num_threads;
    let sum = Arc::new(Mutex::new(0));
    let mut handles = vec![];

    for chunk in numbers.chunks(chunk_size) {
        let chunk = chunk.to_vec();
        let sum = Arc::clone(&sum);

        let handle = thread::spawn(move || {
            let chunk_sum: i32 = chunk.iter().sum();
            let mut total = sum.lock().unwrap();
            *total += chunk_sum;
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    *sum.lock().unwrap()
}
```

### 示例 2：异步 HTTP 请求

```rust
use tokio::time::{sleep, Duration};

async fn fetch_url(url: &str) -> String {
    sleep(Duration::from_millis(100)).await;
    format!("响应来自 {}", url)
}

#[tokio::main]
async fn main() {
    let urls = vec!["url1", "url2", "url3"];

    let mut handles = vec![];
    for url in urls {
        let handle = tokio::spawn(async move {
            fetch_url(url).await
        });
        handles.push(handle);
    }

    for handle in handles {
        let result = handle.await.unwrap();
        println!("{}", result);
    }
}
```

## 练习

```rust
// 练习 1：实现一个线程安全的计数器
use std::sync::{Arc, Mutex};
use std::thread;

// 练习 2：使用 channel 实现生产者-消费者模式
use std::sync::mpsc;

// 练习 3：使用 async/await 实现并发下载
use tokio::time::{sleep, Duration};
```

## 下一步

掌握了并发编程后，你已经完成了 Rust 的基础学习！接下来可以：
- 进入 **[第 3 章：实战项目](../03-practical-projects/)** - 通过实际项目巩固知识

---

**记住：Rust 的并发模型通过类型系统在编译时保证线程安全，这是 Rust 最强大的特性之一！** 🦀
