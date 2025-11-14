# 第 2 章：入门路线

基于 [Rust Book](https://doc.rust-lang.org/book/) 的重点章节，快速掌握 Rust 的基础语法和常用特性。

## 重点章节

### 1. [结构体与枚举](./structs-enums.md)
- 结构体的定义和使用
- 枚举和模式匹配（Rust 核心特性）
- `Option` 和 `Result` 枚举

### 2. [泛型与特征](./generics-traits.md)
- 泛型函数和结构体
- Trait 定义和实现
- Trait bound 和 where 子句

### 3. [智能指针](./smart-pointers.md)
- `Box<T>` - 堆分配
- `Rc<T>` / `Arc<T>` - 引用计数
- `RefCell<T>` / `Mutex<T>` / `RwLock<T>` - 内部可变性

### 4. [并发编程](./concurrency.md)
- 线程创建和管理
- 消息传递（channel）
- 共享状态（Mutex、Arc）
- 异步编程基础

## 学习建议

- **看 Rust Book 40% 就够了**，重点看上述章节
- **边看边写代码**，不要只看不练
- **遇到不懂的概念，回到第 1 章复习**

## 参考资源

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)

---

**这一章是打基础的阶段，扎实的基础会让你在后续学习中事半功倍！**
