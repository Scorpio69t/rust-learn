# 第 3 章：实战项目

通过 4 个 mini-project，每个 1–2 小时能写完，快速感受 Rust 的强大。

## 项目列表

### ① [TCP 回声服务器](./tcp-echo-server.md) ⭐⭐⭐
**技能点：**
- `async/.await`
- `tokio::spawn`
- `read/write`
- Rust 的并发模型

**适合人群：** 有网络编程经验的开发者

### ② [命令行工具（CLI）](./cli-tool.md) ⭐⭐
**推荐库：** `clap` + `anyhow` + `color-eyre`

**例子：** `mygrep pattern file.txt`

**技能：**
- 文件读写
- 错误处理（`?` 运算符）
- trait + iterator

**适合人群：** 所有学习者

### ③ [JSON 序列化/反序列化](./json-serde.md) ⭐
**用 `serde`**

**技能：**
- `derive` 宏
- 类型系统
- `Option`、`Result`

**适合人群：** 所有学习者

### ④ [小型 Web 服务（Axum）](./web-service.md) ⭐⭐⭐
**如果你熟悉 Go/Gin → Axum 是一模一样的体验，但类型更强**

**技能：**
- Web 框架使用
- 路由处理
- 中间件

**适合人群：** 有 Web 开发经验的开发者

## 学习建议

- **按顺序完成**，每个项目都巩固不同的知识点
- **不要复制粘贴**，自己手写一遍
- **遇到问题先看错误信息**，Rust 的错误信息很详细

## 预计时间

- 每个项目：1-2 小时
- 总计：4-8 小时

---

**通过实战项目，你会真正感受到 Rust 的强大和优雅！** 🚀
