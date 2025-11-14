# 小型 Web 服务（Axum）

> **构建 RESTful API** - 如果你熟悉 Go/Gin，Axum 会给你类似的体验，但类型更强

## 项目目标

使用 Axum 框架构建一个简单的 RESTful API 服务，实现用户管理功能。

**功能：**
- 创建用户
- 获取用户列表
- 获取单个用户
- 更新用户
- 删除用户

## 技能点

- Web 框架使用
- 路由处理
- 中间件
- JSON 序列化/反序列化
- 错误处理

## 项目结构

```
web-service/
├── Cargo.toml
└── src/
    └── main.rs
```

## 步骤 1：创建项目

```bash
cargo new web-service
cd web-service
```

## 步骤 2：配置依赖

编辑 `Cargo.toml`：

```toml
[package]
name = "web-service"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.7"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
uuid = { version = "1.0", features = ["v4", "serde"] }
tower = "0.4"
tower-http = { version = "0.5", features = ["cors"] }
```

**依赖说明：**
- `axum` - 现代、快速的 Web 框架
- `tokio` - 异步运行时
- `serde` - 序列化框架
- `uuid` - 生成唯一 ID
- `tower` / `tower-http` - 中间件支持

## 步骤 3：基本实现

### 3.1 定义数据模型

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id: String,
    name: String,
    email: String,
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[derive(Debug, Deserialize)]
struct UpdateUserRequest {
    name: Option<String>,
    email: Option<String>,
}
```

### 3.2 实现内存存储

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

type AppState = Arc<RwLock<Vec<User>>>;

fn create_app_state() -> AppState {
    Arc::new(RwLock::new(Vec::new()))
}
```

### 3.3 实现路由处理函数

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
    routing::{get, post, put, delete},
    Router,
};

// 创建用户
async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    let user = User {
        id: Uuid::new_v4().to_string(),
        name: payload.name,
        email: payload.email,
    };

    state.write().await.push(user.clone());
    Ok(Json(user))
}

// 获取所有用户
async fn get_users(State(state): State<AppState>) -> Json<Vec<User>> {
    let users = state.read().await.clone();
    Json(users)
}

// 获取单个用户
async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Result<Json<User>, StatusCode> {
    let users = state.read().await;
    let user = users.iter().find(|u| u.id == id);

    match user {
        Some(u) => Ok(Json(u.clone())),
        None => Err(StatusCode::NOT_FOUND),
    }
}

// 更新用户
async fn update_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    let mut users = state.write().await;
    let user = users.iter_mut().find(|u| u.id == id);

    match user {
        Some(u) => {
            if let Some(name) = payload.name {
                u.name = name;
            }
            if let Some(email) = payload.email {
                u.email = email;
            }
            Ok(Json(u.clone()))
        }
        None => Err(StatusCode::NOT_FOUND),
    }
}

// 删除用户
async fn delete_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> StatusCode {
    let mut users = state.write().await;
    let index = users.iter().position(|u| u.id == id);

    match index {
        Some(i) => {
            users.remove(i);
            StatusCode::NO_CONTENT
        }
        None => StatusCode::NOT_FOUND,
    }
}
```

### 3.4 创建路由和应用

```rust
fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user).get(get_users))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state = create_app_state();
    let app = create_router(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("服务器运行在 http://0.0.0.0:3000");

    axum::serve(listener, app).await.unwrap();
}
```

## 完整代码

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    response::Json,
    routing::{get, post, put, delete},
    Router,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;
use uuid::Uuid;

// 数据模型
#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id: String,
    name: String,
    email: String,
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    name: String,
    email: String,
}

#[derive(Debug, Deserialize)]
struct UpdateUserRequest {
    name: Option<String>,
    email: Option<String>,
}

// 应用状态
type AppState = Arc<RwLock<Vec<User>>>;

fn create_app_state() -> AppState {
    Arc::new(RwLock::new(Vec::new()))
}

// 路由处理函数
async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    let user = User {
        id: Uuid::new_v4().to_string(),
        name: payload.name,
        email: payload.email,
    };

    state.write().await.push(user.clone());
    Ok(Json(user))
}

async fn get_users(State(state): State<AppState>) -> Json<Vec<User>> {
    let users = state.read().await.clone();
    Json(users)
}

async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Result<Json<User>, StatusCode> {
    let users = state.read().await;
    let user = users.iter().find(|u| u.id == id);

    match user {
        Some(u) => Ok(Json(u.clone())),
        None => Err(StatusCode::NOT_FOUND),
    }
}

async fn update_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    let mut users = state.write().await;
    let user = users.iter_mut().find(|u| u.id == id);

    match user {
        Some(u) => {
            if let Some(name) = payload.name {
                u.name = name;
            }
            if let Some(email) = payload.email {
                u.email = email;
            }
            Ok(Json(u.clone()))
        }
        None => Err(StatusCode::NOT_FOUND),
    }
}

async fn delete_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> StatusCode {
    let mut users = state.write().await;
    let index = users.iter().position(|u| u.id == id);

    match index {
        Some(i) => {
            users.remove(i);
            StatusCode::NO_CONTENT
        }
        None => StatusCode::NOT_FOUND,
    }
}

// 创建路由
fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user).get(get_users))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let state = create_app_state();
    let app = create_router(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .unwrap();

    println!("服务器运行在 http://0.0.0.0:3000");
    println!("API 端点:");
    println!("  POST   /users      - 创建用户");
    println!("  GET    /users      - 获取所有用户");
    println!("  GET    /users/:id  - 获取单个用户");
    println!("  PUT    /users/:id  - 更新用户");
    println!("  DELETE /users/:id  - 删除用户");

    axum::serve(listener, app).await.unwrap();
}
```

## 代码解释

### 1. 状态管理

```rust
type AppState = Arc<RwLock<Vec<User>>>;
```

- `Arc` 允许多个处理器共享状态
- `RwLock` 提供线程安全的读写访问

### 2. 提取器（Extractors）

```rust
State(state): State<AppState>
Path(id): Path<String>
Json(payload): Json<CreateUserRequest>
```

- `State` - 提取应用状态
- `Path` - 提取路径参数
- `Json` - 解析 JSON 请求体

### 3. 响应类型

```rust
Json<User>           // JSON 响应
StatusCode           // HTTP 状态码
Result<T, StatusCode> // 成功或错误
```

## 测试 API

### 使用 curl

```bash
# 创建用户
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Alice","email":"alice@example.com"}'

# 获取所有用户
curl http://localhost:3000/users

# 获取单个用户
curl http://localhost:3000/users/{id}

# 更新用户
curl -X PUT http://localhost:3000/users/{id} \
  -H "Content-Type: application/json" \
  -d '{"name":"Bob"}'

# 删除用户
curl -X DELETE http://localhost:3000/users/{id}
```

### 使用 httpie

```bash
# 创建用户
http POST localhost:3000/users name=Alice email=alice@example.com

# 获取所有用户
http GET localhost:3000/users
```

## 添加中间件

### CORS 支持

```rust
use tower_http::cors::CorsLayer;

fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user).get(get_users))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .layer(CorsLayer::permissive())
        .with_state(state)
}
```

### 日志中间件

```rust
use tower_http::trace::TraceLayer;

fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user).get(get_users))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .layer(TraceLayer::new_for_http())
        .with_state(state)
}
```

### 自定义中间件

```rust
use axum::middleware::Next;
use axum::response::Response;
use axum::http::Request;

async fn logging_middleware<B>(
    req: Request<B>,
    next: Next<B>,
) -> Response {
    println!("请求: {} {}", req.method(), req.uri());
    next.run(req).await
}

fn create_router(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user).get(get_users))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .layer(axum::middleware::from_fn(logging_middleware))
        .with_state(state)
}
```

## 错误处理

### 自定义错误类型

```rust
use axum::response::{IntoResponse, Response};
use axum::http::StatusCode;

#[derive(Debug)]
enum AppError {
    NotFound,
    InvalidInput(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "资源未找到"),
            AppError::InvalidInput(msg) => (StatusCode::BAD_REQUEST, msg.as_str()),
        };

        (status, Json(serde_json::json!({"error": message}))).into_response()
    }
}
```

### 使用 Result

```rust
async fn get_user(
    State(state): State<AppState>,
    Path(id): Path<String>,
) -> Result<Json<User>, AppError> {
    let users = state.read().await;
    let user = users.iter().find(|u| u.id == id);

    match user {
        Some(u) => Ok(Json(u.clone())),
        None => Err(AppError::NotFound),
    }
}
```

## 进阶功能

### 功能 1：添加数据库支持

使用 SQLx：

```toml
[dependencies]
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres"] }
```

```rust
use sqlx::PgPool;

type AppState = Arc<PgPool>;

async fn create_user(
    State(pool): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *",
        payload.name,
        payload.email
    )
    .fetch_one(&*pool)
    .await
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    Ok(Json(user))
}
```

### 功能 2：添加认证

```rust
use axum::extract::Request;
use axum::middleware::Next;

async fn auth_middleware(
    req: Request,
    next: Next,
) -> Response {
    // 检查认证令牌
    let auth_header = req.headers().get("Authorization");
    // ... 验证逻辑

    next.run(req).await
}
```

### 功能 3：添加分页

```rust
#[derive(Debug, Deserialize)]
struct Pagination {
    page: Option<usize>,
    per_page: Option<usize>,
}

async fn get_users(
    State(state): State<AppState>,
    Query(pagination): Query<Pagination>,
) -> Json<Vec<User>> {
    let users = state.read().await;
    let page = pagination.page.unwrap_or(1);
    let per_page = pagination.per_page.unwrap_or(10);

    let start = (page - 1) * per_page;
    let end = start + per_page;

    Json(users[start..end.min(users.len())].to_vec())
}
```

## 与 Go/Gin 的对比

### Go/Gin 版本

```go
func createUser(c *gin.Context) {
    var req CreateUserRequest
    c.BindJSON(&req)

    user := User{
        ID: uuid.New().String(),
        Name: req.Name,
        Email: req.Email,
    }

    users = append(users, user)
    c.JSON(200, user)
}
```

### Rust/Axum 版本

```rust
async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, StatusCode> {
    // 类型安全，编译时检查
    let user = User {
        id: Uuid::new_v4().to_string(),
        name: payload.name,
        email: payload.email,
    };

    state.write().await.push(user.clone());
    Ok(Json(user))
}
```

**优势：**
- 类型安全 - 编译时检查
- 零开销抽象 - 运行时性能优秀
- 内存安全 - 无数据竞争

## 常见问题

### Q: 如何处理文件上传？

**A:** 使用 `multipart`：

```rust
use axum::extract::Multipart;

async fn upload_file(mut multipart: Multipart) -> Result<String, StatusCode> {
    while let Some(field) = multipart.next_field().await.unwrap() {
        let name = field.name().unwrap().to_string();
        let data = field.bytes().await.unwrap();
        // 处理文件数据
    }
    Ok("上传成功".to_string())
}
```

### Q: 如何添加 WebSocket 支持？

**A:** 使用 `axum::extract::ws`：

```rust
use axum::extract::ws::{WebSocket, Message};

async fn websocket_handler(ws: WebSocket) {
    // 处理 WebSocket 连接
}
```

## 扩展练习

1. **添加验证** - 使用 `validator` 库验证输入
2. **添加缓存** - 使用 Redis 缓存数据
3. **添加限流** - 使用 `tower` 中间件限制请求频率
4. **添加 GraphQL** - 使用 `async-graphql` 实现 GraphQL API
5. **添加 gRPC** - 使用 `tonic` 实现 gRPC 服务

## 下一步

完成了 Web 服务后，你已经掌握了：
- Web 框架使用
- RESTful API 设计
- 异步编程
- 中间件使用

恭喜！你已经完成了所有实战项目！接下来可以：
- 进入 **[第 4 章：工程模式](../04-engineering-patterns/)** - 学习 Rust 的工程实践

---

**记住：Axum 提供了类型安全、高性能的 Web 开发体验，是 Rust Web 开发的首选框架！** 🦀
