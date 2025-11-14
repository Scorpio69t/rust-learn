# 智能指针

> **Rust 的内存管理工具** - 智能指针提供了比引用更多的功能

## 什么是智能指针？

智能指针是拥有指向数据的指针，并具有额外的元数据和功能的数据结构。在 Rust 中，智能指针通常实现了 `Deref` 和 `Drop` trait。

### 智能指针 vs 引用

- **引用**：只借用数据
- **智能指针**：拥有数据

## Box<T> - 堆分配

### 基本用法

`Box<T>` 允许你将数据存储在堆上，而不是栈上：

```rust
let b = Box::new(5);
println!("b = {}", b);
```

### 为什么需要 Box？

**1. 递归类型**

```rust
// ❌ 无法编译 - 递归类型大小未知
enum List {
    Cons(i32, List),  // List 的大小未知
    Nil,
}

// ✅ 使用 Box
enum List {
    Cons(i32, Box<List>),  // Box<List> 的大小是固定的
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(1, Box::new(Cons(2, Box::new(Cons(3, Box::new(Nil))))));
}
```

**2. 大型数据结构**

```rust
// 将大型数据移到堆上，避免栈溢出
let large_array = Box::new([0; 1000000]);
```

**3. 转移所有权而不复制**

```rust
fn take_ownership(b: Box<i32>) {
    println!("拥有值: {}", b);
}

let b = Box::new(5);
take_ownership(b);
// b 在这里已经无效
```

### Box 的 Deref trait

```rust
let x = 5;
let y = Box::new(x);

assert_eq!(5, x);
assert_eq!(5, *y);  // 解引用 Box
```

**自定义 Deref：**

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y);  // 解引用
}
```

### 解引用强制多态（Deref Coercion）

```rust
fn hello(name: &str) {
    println!("你好, {}!", name);
}

fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);  // &MyBox<String> 自动转换为 &str
}
```

## Rc<T> - 引用计数

### 为什么需要 Rc？

`Rc<T>` 允许数据有多个所有者（只读）：

```rust
use std::rc::Rc;

enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));  // 共享 a
    let c = Cons(4, Rc::clone(&a));  // 共享 a
}
```

### Rc::clone vs 深拷贝

```rust
let a = Rc::new(String::from("hello"));
let b = Rc::clone(&a);  // 只增加引用计数，不复制数据

// 等价于
let b = a.clone();  // 但 Rc::clone 更明确
```

### 查看引用计数

```rust
use std::rc::Rc;

let a = Rc::new(String::from("hello"));
println!("引用计数: {}", Rc::strong_count(&a));  // 1

{
    let b = Rc::clone(&a);
    println!("引用计数: {}", Rc::strong_count(&a));  // 2
}

println!("引用计数: {}", Rc::strong_count(&a));  // 1
```

### Rc 的限制

**Rc<T> 只允许不可变引用：**

```rust
let a = Rc::new(5);
// *a = 10;  // ❌ 无法编译 - Rc 不允许可变引用
```

## RefCell<T> - 内部可变性

### 什么是内部可变性？

内部可变性（Interior Mutability）允许你在不可变引用的情况下修改数据。

### 基本用法

```rust
use std::cell::RefCell;

let data = RefCell::new(5);

{
    let mut r = data.borrow_mut();  // 获取可变借用
    *r = 10;
}  // r 在这里离开作用域，借用结束

println!("data = {}", data.borrow());  // 输出：10
```

### 借用规则（运行时检查）

`RefCell<T>` 在运行时检查借用规则：

```rust
let data = RefCell::new(5);

let r1 = data.borrow_mut();
let r2 = data.borrow_mut();  // ❌ 运行时 panic！不能有两个可变借用
```

**编译时 vs 运行时：**
- **引用（&T, &mut T）**：编译时检查借用规则
- **RefCell<T>**：运行时检查借用规则

### 结合 Rc 和 RefCell

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List {
    Cons(Rc<RefCell<i32>>, Rc<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let value = Rc::new(RefCell::new(5));

    let a = Rc::new(Cons(Rc::clone(&value), Rc::new(Nil)));
    let b = Cons(Rc::new(RefCell::new(3)), Rc::clone(&a));
    let c = Cons(Rc::new(RefCell::new(4)), Rc::clone(&a));

    *value.borrow_mut() += 10;

    println!("a after = {:?}", a);
    println!("b after = {:?}", b);
    println!("c after = {:?}", c);
}
```

## Arc<T> - 原子引用计数

### 为什么需要 Arc？

`Arc<T>` 是 `Rc<T>` 的线程安全版本，用于多线程环境：

```rust
use std::sync::Arc;
use std::thread;

let data = Arc::new(5);

for i in 0..10 {
    let data = Arc::clone(&data);
    thread::spawn(move || {
        println!("线程 {}: {}", i, data);
    });
}
```

### Rc vs Arc

- **Rc<T>**：单线程使用，性能更好
- **Arc<T>**：多线程使用，线程安全

## Mutex<T> 和 RwLock<T> - 线程安全

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

- **Mutex<T>**：一次只允许一个访问（读或写）
- **RwLock<T>**：允许多个读操作，但写操作需要独占

## 智能指针对比

| 类型 | 所有者数量 | 可变性 | 线程安全 | 检查时机 |
|------|-----------|--------|---------|---------|
| `Box<T>` | 1 | 可变 | 是 | 编译时 |
| `Rc<T>` | 多个 | 不可变 | 否 | 编译时 |
| `RefCell<T>` | 1 | 可变 | 否 | 运行时 |
| `Arc<T>` | 多个 | 不可变 | 是 | 编译时 |
| `Mutex<T>` | 多个 | 可变 | 是 | 运行时 |
| `RwLock<T>` | 多个 | 可变 | 是 | 运行时 |

## 常见组合

### Rc<RefCell<T>> - 单线程多所有者可变

```rust
use std::rc::Rc;
use std::cell::RefCell;

let data = Rc::new(RefCell::new(5));
let data2 = Rc::clone(&data);

*data.borrow_mut() = 10;
println!("data2 = {}", data2.borrow());  // 输出：10
```

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

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 手动管理
std::shared_ptr<int> ptr = std::make_shared<int>(5);
// 需要手动管理，可能内存泄漏
```

```rust
// Rust - 自动管理
let ptr = Rc::new(5);
// 自动管理，编译时保证安全
```

### Go 视角

```go
// Go - 垃圾回收
type Data struct {
    value int
}

d := &Data{value: 5}
// Go 的 GC 管理，但不确定何时回收
```

```rust
// Rust - 编译时确定
let d = Box::new(Data { value: 5 });
// 编译时确定生命周期，零运行时开销
```

## 常见错误与解决方案

### 错误 1：Rc 循环引用导致内存泄漏

```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
enum List {
    Cons(i32, RefCell<Rc<List>>),
    Nil,
}

// 可能创建循环引用
```

**解决方案：** 使用 `Weak<T>` 打破循环：

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}
```

### 错误 2：RefCell 运行时 panic

```rust
let data = RefCell::new(5);
let r1 = data.borrow_mut();
let r2 = data.borrow_mut();  // ❌ 运行时 panic
```

**解决方案：** 确保借用不重叠：

```rust
let data = RefCell::new(5);
{
    let r1 = data.borrow_mut();
    // 使用 r1
}  // r1 离开作用域
let r2 = data.borrow_mut();  // ✅ 现在可以了
```

### 错误 3：Mutex 死锁

```rust
let m1 = Arc::new(Mutex::new(0));
let m2 = Arc::new(Mutex::new(0));

// 可能死锁
let t1 = thread::spawn(move || {
    let _a = m1.lock().unwrap();
    let _b = m2.lock().unwrap();
});

let t2 = thread::spawn(move || {
    let _b = m2.lock().unwrap();
    let _a = m1.lock().unwrap();  // 可能死锁
});
```

**解决方案：** 总是以相同顺序获取锁。

## 实践建议

1. **优先使用 Box** - 大多数情况下 Box 就足够了
2. **需要多所有者时用 Rc** - 单线程环境
3. **需要多线程时用 Arc** - 多线程环境
4. **需要内部可变性时用 RefCell** - 单线程
5. **需要线程安全的内部可变性时用 Mutex/RwLock** - 多线程
6. **避免循环引用** - 使用 Weak 打破循环

## 实际应用示例

### 示例 1：树形结构

```rust
use std::rc::{Rc, Weak};
use std::cell::RefCell;

#[derive(Debug)]
struct Node {
    value: i32,
    parent: RefCell<Weak<Node>>,
    children: RefCell<Vec<Rc<Node>>>,
}

fn main() {
    let leaf = Rc::new(Node {
        value: 3,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![]),
    });

    let branch = Rc::new(Node {
        value: 5,
        parent: RefCell::new(Weak::new()),
        children: RefCell::new(vec![Rc::clone(&leaf)]),
    });

    *leaf.parent.borrow_mut() = Rc::downgrade(&branch);
}
```

### 示例 2：线程安全的计数器

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
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
}
```

## 练习

```rust
// 练习 1：使用 Box 实现一个简单的链表
enum List {
    Cons(i32, Box<List>),
    Nil,
}

// 练习 2：使用 Rc 和 RefCell 实现一个可以修改的共享列表
use std::rc::Rc;
use std::cell::RefCell;

// 练习 3：使用 Arc 和 Mutex 实现一个线程安全的计数器
use std::sync::{Arc, Mutex};
use std::thread;
```

## 下一步

掌握了智能指针后，接下来学习：
- **[并发编程](./concurrency.md)** - 使用 Rust 进行多线程编程

---

**记住：智能指针是 Rust 内存管理的重要组成部分。理解它们的使用场景，能让你写出更灵活、更安全的代码！** 🦀
