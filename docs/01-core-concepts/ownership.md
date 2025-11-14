# Ownership（所有权）

> **Rust 最核心的概念，没有之一！** 理解所有权是学习 Rust 的第一步。

## 什么是所有权？

所有权（Ownership）是 Rust 独特的内存管理机制。简单来说：

- **每个值都有一个所有者（owner）**
- **同一时间只能有一个所有者**
- **当所有者离开作用域时，值会被自动释放（drop）**

## 所有权的三条规则

1. Rust 中的每个值都有一个所有者
2. 同一时间只能有一个所有者
3. 当所有者离开作用域时，值会被丢弃

## 基本示例

### 示例 1：简单类型（栈上数据）

```rust
fn main() {
    let x = 5;        // x 拥有值 5
    let y = x;        // 复制（copy），因为 i32 实现了 Copy trait
    println!("x = {}, y = {}", x, y);  // ✅ 都可以使用
}
```

**为什么可以？** 因为 `i32` 是简单类型，存储在栈上，实现了 `Copy` trait，所以 `y = x` 是**复制**，不是移动。

### 示例 2：堆上数据（String）

```rust
fn main() {
    let s1 = String::from("hello");  // s1 拥有这个 String
    let s2 = s1;                     // ❌ 移动（move），s1 不再有效
    // println!("{}", s1);           // 编译错误！s1 已经被移动
    println!("{}", s2);              // ✅ s2 现在拥有这个 String
}
```

**为什么不行？** `String` 存储在堆上，包含指向堆内存的指针。当 `s2 = s1` 时，发生了**移动（move）**，`s1` 的所有权转移给了 `s2`，`s1` 不再有效。

## 移动（Move）语义

### 什么是移动？

移动是 Rust 的默认行为（对于没有实现 `Copy` trait 的类型）：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;  // s1 的所有权移动给了 s2

    // s1 在这里已经无效了
    // println!("{}", s1);  // ❌ 编译错误：value borrowed after move
}
```

### 移动 vs 复制

```rust
// 复制（Copy）- 栈上的简单类型
let x = 5;
let y = x;  // 复制，x 和 y 都有效

// 移动（Move）- 堆上的复杂类型
let s1 = String::from("hello");
let s2 = s1;  // 移动，s1 不再有效
```

**实现了 `Copy` trait 的类型：**
- 所有整数类型（`i32`, `u32`, `i64` 等）
- 布尔类型（`bool`）
- 浮点类型（`f32`, `f64`）
- 字符类型（`char`）
- 元组（如果所有元素都实现了 `Copy`）

**没有实现 `Copy` trait 的类型：**
- `String`
- `Vec<T>`
- 自定义结构体（除非显式实现 `Copy`）

## 函数与所有权

### 传递所有权

```rust
fn take_ownership(s: String) {
    println!("{}", s);
    // s 在这里离开作用域，被 drop
}

fn main() {
    let s = String::from("hello");
    take_ownership(s);  // s 的所有权移动给了函数
    // println!("{}", s);  // ❌ 编译错误！s 已经被移动
}
```

### 返回值转移所有权

```rust
fn give_ownership() -> String {
    let s = String::from("hello");
    s  // 返回 s，所有权转移给调用者
}

fn take_and_give_back(s: String) -> String {
    s  // 接收所有权，然后返回
}

fn main() {
    let s1 = give_ownership();
    let s2 = String::from("world");
    let s3 = take_and_give_back(s2);
    // s1 和 s3 有效，s2 无效
}
```

## 作用域与 Drop

### 作用域规则

```rust
fn main() {
    {  // 新的作用域
        let s = String::from("hello");
        // s 在这里有效
    }  // s 在这里离开作用域，被自动 drop

    // s 在这里已经无效
}
```

### Drop 机制

Rust 在值离开作用域时自动调用 `drop` 函数：

```rust
struct CustomStruct {
    data: String,
}

impl Drop for CustomStruct {
    fn drop(&mut self) {
        println!("Dropping {}", self.data);
    }
}

fn main() {
    let c = CustomStruct {
        data: String::from("my data"),
    };
    // c 在这里离开作用域，会调用 drop
}
// 输出：Dropping my data
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 需要手动管理内存
std::string* s = new std::string("hello");
// ... 使用 s
delete s;  // 必须手动释放，否则内存泄漏
```

```rust
// Rust - 自动管理
let s = String::from("hello");
// s 离开作用域时自动释放，无需手动 delete
```

### Go 视角

```go
// Go - 垃圾回收
s := "hello"
// Go 的 GC 会在某个时刻回收，但不确定何时
```

```rust
// Rust - 编译时确定
let s = String::from("hello");
// Rust 在编译时就知道何时释放，零运行时开销
```

## 常见错误与解决方案

### 错误 1：使用已移动的值

```rust
let s1 = String::from("hello");
let s2 = s1;
println!("{}", s1);  // ❌ 编译错误
```

**解决方案：** 使用引用（下一章会讲）或克隆：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // 克隆，s1 和 s2 都有效
println!("{}", s1);   // ✅
```

### 错误 2：在循环中移动

```rust
let vec = vec![String::from("a"), String::from("b")];
for s in vec {  // vec 的所有权被移动
    println!("{}", s);
}
// println!("{:?}", vec);  // ❌ 编译错误
```

**解决方案：** 使用引用：

```rust
let vec = vec![String::from("a"), String::from("b")];
for s in &vec {  // 借用引用
    println!("{}", s);
}
println!("{:?}", vec);  // ✅
```

## 实践建议

1. **理解移动是默认行为** - 对于堆上的数据，赋值就是移动
2. **使用 `clone()` 要谨慎** - 克隆有性能开销，优先考虑引用
3. **注意函数参数** - 函数参数会获取所有权，除非使用引用
4. **阅读编译错误** - Rust 的错误信息会告诉你所有权在哪里出了问题

## 练习

尝试以下代码，预测哪些会编译通过，哪些会失败：

```rust
// 练习 1
let x = 5;
let y = x;
println!("x = {}, y = {}", x, y);

// 练习 2
let s1 = String::from("hello");
let s2 = s1;
println!("s1 = {}, s2 = {}", s1, s2);

// 练习 3
let vec1 = vec![1, 2, 3];
let vec2 = vec1;
println!("vec1 = {:?}", vec1);
```

## 下一步

理解了所有权后，接下来学习：
- **[Borrow & Reference（借用与引用）](./borrowing.md)** - 如何在不获取所有权的情况下使用值

---

**记住：所有权是 Rust 内存安全的基础。虽然一开始会觉得限制很多，但这是 Rust 能在编译时保证内存安全的关键！** 🦀
