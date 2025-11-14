# Borrow & Reference（借用与引用）

> **所有权的补充** - 如何在不获取所有权的情况下使用值

## 什么是借用？

借用（Borrowing）允许你使用值，但不获取所有权。通过引用（Reference）来实现。

```rust
let s = String::from("hello");
let len = calculate_length(&s);  // &s 是引用，不获取所有权
println!("{} 的长度是 {}", s, len);  // ✅ s 仍然有效
```

## 引用 vs 所有权

### 使用所有权（会移动）

```rust
fn take_ownership(s: String) -> usize {
    s.len()  // s 的所有权被函数获取
}

fn main() {
    let s = String::from("hello");
    let len = take_ownership(s);  // s 被移动
    // println!("{}", s);  // ❌ 编译错误
}
```

### 使用引用（不移动）

```rust
fn calculate_length(s: &String) -> usize {
    s.len()  // s 是引用，不获取所有权
}

fn main() {
    let s = String::from("hello");
    let len = calculate_length(&s);  // 传递引用
    println!("{} 的长度是 {}", s, len);  // ✅ s 仍然有效
}
```

## 不可变引用（Immutable Reference）

### 基本用法

```rust
fn main() {
    let s = String::from("hello");
    let r1 = &s;  // 不可变引用
    let r2 = &s;  // 可以有多个不可变引用
    let r3 = &s;  // 可以有多个不可变引用

    println!("{}, {}, {}", r1, r2, r3);  // ✅ 都可以使用
}
```

### 规则：可以有多个不可变引用

```rust
fn main() {
    let s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    let r3 = &s;

    // 可以有任意多个不可变引用
    println!("r1: {}, r2: {}, r3: {}", r1, r2, r3);
}
```

## 可变引用（Mutable Reference）

### 基本用法

```rust
fn change(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    change(&mut s);  // 传递可变引用
    println!("{}", s);  // 输出：hello, world
}
```

### 规则 1：可变引用必须唯一

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &mut s;  // ✅ 第一个可变引用
    // let r2 = &mut s;  // ❌ 编译错误！不能有第二个可变引用

    println!("{}", r1);
}
```

**为什么？** 防止数据竞争（Data Race）。如果两个可变引用同时修改数据，会导致未定义行为。

### 规则 2：可变引用和不可变引用不能同时存在

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;      // 不可变引用
    let r2 = &s;      // 另一个不可变引用
    // let r3 = &mut s;  // ❌ 编译错误！不能在有不可变引用时创建可变引用

    println!("{}, {}", r1, r2);
}
```

**为什么？** 不可变引用假设数据不会被改变，如果同时有可变引用，这个假设就不成立了。

### 作用域分离

```rust
fn main() {
    let mut s = String::from("hello");

    {
        let r1 = &s;  // 不可变引用
        println!("{}", r1);
    }  // r1 在这里离开作用域

    let r2 = &mut s;  // ✅ 现在可以创建可变引用了
    println!("{}", r2);
}
```

## 借用规则总结

1. **任意时刻，只能满足以下之一：**
   - 一个可变引用
   - 任意数量的不可变引用

2. **引用必须总是有效的**（生命周期，下一章会讲）

3. **不能同时有可变和不可变引用**

## 悬垂引用（Dangling References）

Rust 编译器会防止悬垂引用：

```rust
// ❌ 这段代码无法编译
fn dangle() -> &String {
    let s = String::from("hello");
    &s  // 返回 s 的引用
}  // s 在这里离开作用域，被 drop
// 返回的引用指向无效内存
```

**解决方案：** 返回所有权而不是引用：

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // 返回所有权
}
```

## 切片（Slice）作为引用

切片是引用的一种特殊形式：

```rust
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];  // 返回字符串切片
        }
    }

    &s[..]  // 返回整个字符串
}

fn main() {
    let s = String::from("hello world");
    let word = first_word(&s);
    println!("第一个单词: {}", word);
}
```

## 与 C++/Go 的对比

### C++ 视角

```cpp
// C++ - 需要手动管理引用和指针
void modify(std::string& s) {
    s += ", world";
}

std::string s = "hello";
modify(s);  // 可能有多处引用，需要程序员保证安全
```

```rust
// Rust - 编译器保证安全
fn modify(s: &mut String) {
    s.push_str(", world");
}

let mut s = String::from("hello");
modify(&mut s);  // 编译器保证只有一个可变引用
```

### Go 视角

```go
// Go - 运行时检查
func modify(s *string) {
    *s += ", world"
}

s := "hello"
modify(&s)  // Go 的指针可能悬空，需要运行时检查
```

```rust
// Rust - 编译时检查
fn modify(s: &mut String) {
    s.push_str(", world")
}

let mut s = String::from("hello");
modify(&mut s);  // 编译时保证引用有效
```

## 常见错误与解决方案

### 错误 1：同时有可变和不可变引用

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &mut s;  // ❌ 编译错误
println!("{}", r1);
```

**解决方案：** 分离作用域或改变顺序：

```rust
let mut s = String::from("hello");
{
    let r1 = &s;
    println!("{}", r1);
}  // r1 离开作用域
let r2 = &mut s;  // ✅ 现在可以了
```

### 错误 2：在循环中借用

```rust
let mut vec = vec![String::from("a")];
for s in &vec {
    vec.push(String::from("b"));  // ❌ 编译错误：不能同时借用
}
```

**解决方案：** 先收集需要添加的元素：

```rust
let mut vec = vec![String::from("a")];
let mut to_add = Vec::new();
for s in &vec {
    to_add.push(String::from("b"));
}
vec.extend(to_add);  // ✅
```

### 错误 3：返回局部变量的引用

```rust
fn bad() -> &String {
    let s = String::from("hello");
    &s  // ❌ 编译错误：返回悬垂引用
}
```

**解决方案：** 返回所有权：

```rust
fn good() -> String {
    let s = String::from("hello");
    s  // ✅ 返回所有权
}
```

## 实践建议

1. **优先使用不可变引用** - 更安全，可以同时有多个
2. **可变引用要谨慎** - 确保同一时间只有一个
3. **理解作用域** - 引用的生命周期不能超过被引用的值
4. **使用切片** - 字符串切片 `&str` 比 `&String` 更灵活

## 借用检查器的帮助

Rust 的借用检查器会在编译时检查所有借用规则：

```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &s;
    let r2 = &mut s;  // ❌ 编译错误

    // 错误信息会告诉你：
    // - 在哪里创建了不可变引用
    // - 在哪里尝试创建可变引用
    // - 如何修复
}
```

**错误信息示例：**
```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:18
  |
3 |     let r1 = &s;
  |              -- immutable borrow occurs here
4 |     let r2 = &mut s;
  |              ^^^^^^ mutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- immutable borrow later used here
```

## 练习

```rust
// 练习 1：修复这段代码
fn main() {
    let mut s = String::from("hello");
    let r1 = &s;
    let r2 = &mut s;
    println!("{}, {}", r1, r2);
}

// 练习 2：这个函数有什么问题？
fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

// 练习 3：为什么这段代码可以编译？
fn main() {
    let mut s = String::from("hello");
    let r1 = &mut s;
    let r2 = &mut s;
    println!("{}, {}", r1, r2);
}
```

## 下一步

理解了借用后，接下来学习：
- **[Lifetimes（生命周期）](./lifetimes.md)** - 理解引用的有效期

---

**记住：借用是 Rust 在保证内存安全的同时，允许你灵活使用值的关键机制。虽然规则严格，但这是值得的！** 🦀
