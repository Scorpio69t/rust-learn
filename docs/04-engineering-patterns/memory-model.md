# 内存模型

> **深入理解 Rust 的内存管理** - 从栈到堆，从所有权到智能指针

## Rust 的内存布局

### 栈（Stack）

栈是后进先出（LIFO）的数据结构，用于存储：
- 局部变量
- 函数参数
- 返回值

**特点：**
- 快速分配和释放
- 大小固定
- 自动管理

```rust
fn main() {
    let x = 5;        // 存储在栈上
    let y = x;        // 复制到栈上
    println!("{}", x); // x 仍然有效
}
```

### 堆（Heap）

堆用于存储：
- 动态大小的数据
- 生命周期不确定的数据

**特点：**
- 需要手动分配（通过智能指针）
- 大小可变
- 需要管理

```rust
let boxed = Box::new(5);  // 存储在堆上
```

## 所有权与内存

### 栈上的移动

```rust
let s1 = String::from("hello");  // s1 拥有数据
let s2 = s1;                     // 移动所有权
// s1 不再有效
```

### 堆上的移动

```rust
let v1 = vec![1, 2, 3];  // v1 拥有堆上的数据
let v2 = v1;             // 移动所有权
// v1 不再有效
```

## 智能指针详解

### Box<T> - 堆分配

**用途：**
- 在堆上分配数据
- 递归类型
- 大型数据结构

```rust
let boxed = Box::new(5);
println!("{}", boxed);  // 自动解引用
```

**内存布局：**
```
栈: [ptr] ──┐
            │
堆:         └─> [5]
```

### Rc<T> - 引用计数（单线程）

**用途：**
- 多个所有者共享数据
- 只读访问

```rust
use std::rc::Rc;

let data = Rc::new(5);
let data1 = Rc::clone(&data);  // 增加引用计数
let data2 = Rc::clone(&data);  // 增加引用计数

// 引用计数 = 3
```

**内存布局：**
```
栈: [Rc] ──┐
           │
堆:        └─> [count: 3, data: 5]
```

**限制：**
- 只允许不可变引用
- 不是线程安全的

### Arc<T> - 原子引用计数（多线程）

**用途：**
- 多线程环境下的共享数据
- 只读访问

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(5);

for _ in 0..10 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("{}", data);
    });
}
```

**与 Rc 的区别：**
- `Arc` 使用原子操作，线程安全
- `Rc` 使用普通整数，不是线程安全的

### RefCell<T> - 内部可变性（单线程）

**用途：**
- 在不可变引用下修改数据
- 运行时借用检查

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

{
    let mut r = data.borrow_mut();
    *r = 10;
}

println!("{}", data.borrow());
```

**内存布局：**
```
栈: [RefCell] ──┐
                │
堆:             └─> [borrow_count, data: 10]
```

**限制：**
- 运行时检查借用规则
- 违反规则会 panic
- 不是线程安全的

### Mutex<T> - 互斥锁（多线程）

**用途：**
- 多线程环境下的可变数据
- 互斥访问

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let data = Arc::new(Mutex::new(0));

for _ in 0..10 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        let mut num = data.lock().unwrap();
        *num += 1;
    });
}
```

**内存布局：**
```
栈: [Arc] ──┐
            │
堆:         └─> [Mutex] ──> [data: 0]
```

### RwLock<T> - 读写锁（多线程）

**用途：**
- 多线程环境下的可变数据
- 允许多个读操作

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let data = Arc::new(RwLock::new(0));

// 多个读操作可以同时进行
let read1 = Arc::clone(&data);
thread::spawn(move || {
    let r = read1.read().unwrap();
    println!("{}", *r);
});

// 写操作需要独占
let write = Arc::clone(&data);
thread::spawn(move || {
    let mut w = write.write().unwrap();
    *w += 1;
});
```

## 常见组合模式

### Rc<RefCell<T>> - 单线程多所有者可变

```rust
use std::rc::Rc;
use std::cell::RefCell;

let data = Rc::new(RefCell::new(5));
let data1 = Rc::clone(&data);

*data.borrow_mut() = 10;
println!("{}", data1.borrow());  // 输出: 10
```

**使用场景：**
- 树形结构
- 图结构
- 单线程环境下的共享可变状态

### Arc<Mutex<T>> - 多线程多所有者可变

```rust
use std::sync::{Arc, Mutex};
use std::thread;

let data = Arc::new(Mutex::new(0));

let handles: Vec<_> = (0..10).map(|_| {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        let mut num = data.lock().unwrap();
        *num += 1;
    })
}).collect();

for handle in handles {
    handle.join().unwrap();
}
```

**使用场景：**
- 多线程环境下的共享可变状态
- 并发计数器
- 线程安全的缓存

### Arc<RwLock<T>> - 多线程读多写少

```rust
use std::sync::{Arc, RwLock};
use std::thread;

let data = Arc::new(RwLock::new(0));

// 多个读操作
for _ in 0..5 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        let r = data.read().unwrap();
        println!("读取: {}", *r);
    });
}

// 写操作
let data = Arc::clone(&data);
thread::spawn(move || {
    let mut w = data.write().unwrap();
    *w += 1;
});
```

**使用场景：**
- 读多写少的场景
- 配置数据
- 缓存系统

## Pin<T> - 固定数据

### 为什么需要 Pin？

`Pin` 用于固定数据在内存中的位置，主要用于：
- 自引用结构
- 异步编程中的 Future

```rust
use std::pin::Pin;

struct SelfReferential {
    data: String,
    pointer: *const String,  // 指向 data
}

impl SelfReferential {
    fn new(data: String) -> Pin<Box<Self>> {
        let mut boxed = Box::new(SelfReferential {
            data,
            pointer: std::ptr::null(),
        });

        boxed.pointer = &boxed.data as *const String;

        Box::pin(boxed)
    }
}
```

## 内存安全保证

### 编译时检查

```rust
let mut data = 5;
let r1 = &data;
let r2 = &mut data;  // ❌ 编译错误：不能同时有可变和不可变引用
```

### 运行时检查（RefCell）

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
let r1 = data.borrow();
let r2 = data.borrow_mut();  // ❌ 运行时 panic
```

## 性能考虑

### 栈 vs 堆

| 特性 | 栈 | 堆 |
|------|----|----|
| 分配速度 | 快 | 慢 |
| 访问速度 | 快 | 慢 |
| 大小限制 | 小 | 大 |
| 管理方式 | 自动 | 手动 |

### 智能指针开销

| 类型 | 开销 |
|------|------|
| `Box<T>` | 最小（只是一个指针） |
| `Rc<T>` | 引用计数（原子操作） |
| `Arc<T>` | 原子引用计数（更慢） |
| `RefCell<T>` | 运行时检查 |
| `Mutex<T>` | 锁操作 |

## 内存泄漏

### Rust 可以内存泄漏吗？

**可以！** 但 Rust 保证不会出现：
- 使用后释放（Use After Free）
- 空指针解引用（Null Pointer Dereference）
- 数据竞争（Data Race）

### 循环引用导致的内存泄漏

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

use List::{Cons, Nil};

impl List {
    fn tail(&self) -> Option<&RefCell<Rc<List>>> {
        match self {
            Cons(_, item) => Some(item),
            Nil => None,
        }
    }
}

fn main() {
    let a = Rc::new(Cons(5, RefCell::new(Rc::new(Nil))));
    let b = Rc::new(Cons(10, RefCell::new(Rc::clone(&a))));

    if let Some(link) = a.tail() {
        *link.borrow_mut() = Rc::clone(&b);  // 创建循环引用
    }

    // a 和 b 的引用计数永远不会变为 0
    // 导致内存泄漏
}
```

**解决方案：** 使用 `Weak<T>` 打破循环：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 手动管理
int* ptr = new int(5);
// 必须手动 delete
delete ptr;
```

```rust
// Rust - 自动管理
let ptr = Box::new(5);
// 自动释放
```

### Go 视角

```go
// Go - 垃圾回收
data := make([]int, 1000)
// GC 在某个时刻回收，不确定何时
```

```rust
// Rust - 编译时确定
let data = vec![0; 1000];
// 编译时确定何时释放
```

## 实践建议

1. **优先使用栈** - 栈分配更快
2. **需要时才用堆** - 使用 `Box`、`Vec` 等
3. **理解所有权** - 这是 Rust 内存安全的基础
4. **选择合适的智能指针** - 根据场景选择
5. **避免循环引用** - 使用 `Weak` 打破循环

## 扩展练习

1. **实现一个简单的内存分配器** - 理解堆分配
2. **分析不同智能指针的性能** - 基准测试
3. **实现一个无锁数据结构** - 使用原子操作
4. **优化内存使用** - 减少不必要的堆分配

## 下一步

掌握了内存模型后，你已经完成了第 4 章的学习！接下来可以：
- 进入 **[第 5 章：进阶方向](../05-advanced-topics/)** - 探索 Rust 的高级应用

---

**记住：理解 Rust 的内存模型是写出高效、安全代码的关键。虽然一开始可能觉得复杂，但这是 Rust 强大性能和安全性的基础！** 🦀
