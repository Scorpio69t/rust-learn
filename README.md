# 🦀 Rust 学习之路 - 从入门到进阶

> 一本系统性的 Rust 学习指南，适合有 C++/Go 经验的开发者快速掌握 Rust 的核心概念和工程实践

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/rust-1.70+-orange.svg)](https://www.rust-lang.org)

## 📖 关于本书

本仓库是一本系统性的 Rust 学习指南，旨在帮助有经验的开发者（特别是 C++/Go 背景）快速掌握 Rust 的核心概念和工程实践。本书采用理论与实践相结合的方式，通过大量示例代码和实战项目，让你真正理解 Rust 的设计哲学和最佳实践。

### 🎯 适合人群

- ✅ 有 C++/Go/Java 等系统编程语言经验的开发者
- ✅ 希望学习 Rust 但觉得官方文档过于冗长的开发者
- ✅ 想要深入理解 Rust 所有权、借用、生命周期等核心概念的开发者
- ✅ 准备将 Rust 应用到实际项目中的工程师

### 📚 学习路径

本书按照以下路径组织，建议按顺序学习：

1. **[第 0 章：理解 Rust 的核心哲学](./docs/00-rust-philosophy.md)** - 理解 Rust 的设计目标和核心理念
2. **[第 1 章：四大核心概念](./docs/01-core-concepts/)** - Ownership、Borrow、Lifetimes、Trait
3. **[第 2 章：入门路线](./docs/02-getting-started/)** - 基于 Rust Book 的重点章节学习
4. **[第 3 章：实战项目](./docs/03-practical-projects/)** - 通过小项目巩固知识
5. **[第 4 章：工程模式](./docs/04-engineering-patterns/)** - 从"能写"到"写得专业"
6. **[第 5 章：进阶方向](./docs/05-advanced-topics/)** - 异步系统、FFI、嵌入式等高级主题

## 🚀 快速开始

### 安装 Rust

```bash
# Linux/macOS
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Windows
# 下载并运行 https://rustup.rs/

# 验证安装
rustc --version
cargo --version
```

### 使用本书

1. **克隆仓库**
   ```bash
   git clone https://github.com/yourusername/rust-learn.git
   cd rust-learn
   ```

2. **阅读文档**
   - 按照章节顺序阅读 `docs/` 目录下的文档
   - 每个章节都包含理论讲解和代码示例

3. **运行示例**
   ```bash
   cd examples/Chapter01/guessing_game
   cargo run
   ```

4. **实践项目**
   - 在 `projects/` 目录下创建自己的项目
   - 参考 `docs/03-practical-projects/` 中的项目指导

## 📁 仓库结构

```
rust-learn/
├── README.md                 # 本文件 - 开篇介绍
├── LICENSE                   # MIT 许可证
├── docs/                     # 学习文档目录
│   ├── 00-rust-philosophy.md           # Rust 核心哲学
│   ├── 01-core-concepts/               # 四大核心概念
│   │   ├── README.md
│   │   ├── ownership.md                # 所有权
│   │   ├── borrowing.md                # 借用与引用
│   │   ├── lifetimes.md                # 生命周期
│   │   └── traits.md                   # 特征
│   ├── 02-getting-started/             # 入门路线
│   │   ├── README.md
│   │   ├── structs-enums.md            # 结构体与枚举
│   │   ├── generics-traits.md          # 泛型与特征
│   │   ├── smart-pointers.md           # 智能指针
│   │   └── concurrency.md              # 并发编程
│   ├── 03-practical-projects/          # 实战项目
│   │   ├── README.md
│   │   ├── tcp-echo-server.md          # TCP 回声服务器
│   │   ├── cli-tool.md                 # 命令行工具
│   │   ├── json-serde.md               # JSON 序列化
│   │   └── web-service.md              # Web 服务
│   ├── 04-engineering-patterns/        # 工程模式
│   │   ├── README.md
│   │   ├── pattern-matching.md         # 模式匹配
│   │   ├── iterators.md                # 迭代器
│   │   ├── error-handling.md           # 错误处理
│   │   └── memory-model.md             # 内存模型
│   └── 05-advanced-topics/             # 进阶主题
│       ├── README.md
│       ├── async-tokio.md              # 异步编程
│       ├── ffi.md                      # FFI 调用
│       ├── embedded.md                 # 嵌入式开发
│       └── middleware.md               # 中间件开发
├── examples/                # 代码示例
│   └── Chapter01/           # 第一章示例代码
│       ├── hello_cargo/     # Hello Cargo
│       ├── guessing_game/   # 猜数字游戏
│       └── minigrep/        # 迷你 grep
└── projects/                # 完整项目
    └── README.md            # 项目说明
```

## 🎓 学习建议

### ⚠️ 重要提示：Rust 的学习方法与其他语言完全不同

Rust 难不是因为语法，而是因为：

- **Borrow Checker（借用检查器）** - 编译时保证内存安全
- **Ownership（所有权）** - 独特的内存管理模型
- **Lifetime（生命周期）** - 确保引用不会悬空
- **零开销抽象 + 高级类型系统** - 在保证安全的同时获得 C++ 级别的性能

### 📝 学习策略

1. **从"Rust 设计理念"入手** → 不要急着写业务代码
2. **先用小项目打通概念** → 理解所有权和借用
3. **再做实战项目优化** → 掌握工程模式

### ❌ 常见误区

- **不要试图马上用 Rust 重写整个大项目**
- **不要跳过所有权和借用检查器的学习**
- **不要期望用 C++/Go 的思维直接写 Rust**

### ✅ 正确做法

- 先从一个小模块开始
- 再做一个 CLI 工具
- 最后接入生产环境

## 🔥 Rust 的核心优势

### 1. 内存安全（Memory Safety）

- ✅ 编译时防止数据竞争（Data Race）
- ✅ 防止空指针解引用（Null Pointer Dereference）
- ✅ 防止使用后释放（Use After Free）
- ✅ 防止缓冲区溢出（Buffer Overflow）

### 2. 零开销抽象（Zero-Cost Abstractions）

- ✅ 泛型、trait、闭包、迭代器都是 0 开销
- ✅ 性能与 C/C++ 持平
- ✅ 高级抽象不带来运行时成本

### 3. 优秀的工程化工具

- ✅ **Cargo** - 强大的包管理和构建工具
- ✅ **rustfmt** - 统一的代码格式化工具
- ✅ **clippy** - 智能的代码检查工具
- ✅ **rustdoc** - 自动生成文档工具

## 📖 推荐资源

### 官方资源

- [The Rust Book](https://doc.rust-lang.org/book/) - 官方入门教程
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) - 通过示例学习
- [Rustlings](https://github.com/rust-lang/rustlings) - 交互式练习

### 社区资源

- [Rust 中文社区](https://rustcc.cn/)
- [Rust 语言中文网](https://kaisery.github.io/trpl-zh-cn/)
- [Rust 异步编程](https://rust-lang.github.io/async-book/)

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

- 发现错误？请提交 Issue
- 有改进建议？欢迎提交 PR
- 想添加新内容？非常欢迎！

## 📄 许可证

本项目采用 [MIT 许可证](LICENSE)。

## 🙏 致谢

感谢 Rust 社区提供的优秀资源和工具，以及所有为 Rust 发展做出贡献的开发者。

---

**开始你的 Rust 学习之旅吧！** 🦀

> 记住：Rust 学习曲线陡峭，但一旦过了所有权这关，你的 C++/Go 经验会爆炸式加乘。
