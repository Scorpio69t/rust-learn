# 第 5 章：进阶方向

针对有经验的开发者（特别是 C++/Go 背景），深入探索 Rust 的高级应用场景。

## 进阶方向

### ① [异步系统 / 高性能服务端](./async-tokio.md) ⭐⭐⭐
**构建：**
- Actor 服务器
- WebSocket 服务
- 多路复用 IO 服务（kqueue/epoll）
- 零拷贝

**技术栈：** Tokio、Axum、Tower

### ② [FFI + 调 Go/C/C++](./ffi.md) ⭐⭐
**你可以：**
- 把 Rust 嵌入 Android 服务
- Rust 写高性能核心、暴露 C API

**适合做：**
- 多线程调度器
- AI 模型前处理
- 低延迟网络模块
- 加密/签名模块

### ③ [中间件开发](./middleware.md) ⭐⭐
**Rust 非常适合：**
- 事件系统
- 调度
- 连接池
- 线程池
- Actor 模型
- Protobuf 编解码器

### ④ [嵌入式 Rust / 设备开发](./embedded.md) ⭐⭐
**你有大量嵌入式经验，可以用 Rust 做：**
- STM32 / ESP32
- Rockchip Linux 用户态服务
- 驱动层 glue code
- 安全网络代理

## 学习建议

- **选择你感兴趣的方向深入学习**
- **不需要全部掌握**，根据项目需求选择
- **参考优秀的开源项目**，学习最佳实践

## 参考资源

- [Tokio 文档](https://tokio.rs/)
- [Rust 异步编程](https://rust-lang.github.io/async-book/)
- [嵌入式 Rust](https://embedded.rs/)

---

**这些进阶方向能让你充分发挥 Rust 的优势，构建高性能、安全的系统！** 🚀
