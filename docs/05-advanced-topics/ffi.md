# FFI + 调 Go/C/C++

> **跨语言互操作** - 把 Rust 嵌入其他系统，或调用其他语言的代码

## 什么是 FFI？

FFI（Foreign Function Interface）允许 Rust 与其他语言交互：
- **从 Rust 调用 C/C++** - 使用现有的 C/C++ 库
- **从 C/C++ 调用 Rust** - 将 Rust 代码暴露为 C API
- **从 Go 调用 Rust** - 通过 C ABI

## 从 Rust 调用 C

### 基本示例

```rust
// 在 Cargo.toml 中添加
// [build-dependencies]
// cc = "1.0"

// 创建 C 文件：src/hello.c
/*
#include <stdio.h>

void hello_from_c() {
    printf("Hello from C!\n");
}
*/

// Rust 代码
use std::os::raw::c_int;

extern "C" {
    fn hello_from_c();
}

fn main() {
    unsafe {
        hello_from_c();
    }
}
```

### 使用 bindgen 自动生成绑定

添加依赖：

```toml
[build-dependencies]
bindgen = "0.65"

[dependencies]
libc = "0.2"
```

创建 `build.rs`：

```rust
use std::env;
use std::path::PathBuf;

fn main() {
    println!("cargo:rustc-link-lib=hello");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .generate()
        .expect("无法生成绑定");

    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("无法写入绑定");
}
```

## 从 C 调用 Rust

### 创建 C API

```rust
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_int};

#[no_mangle]
pub extern "C" fn rust_add(a: c_int, b: c_int) -> c_int {
    a + b
}

#[no_mangle]
pub extern "C" fn rust_hello(name: *const c_char) -> *mut c_char {
    let c_str = unsafe { CStr::from_ptr(name) };
    let name_str = c_str.to_str().unwrap();
    let greeting = format!("Hello, {}!", name_str);
    CString::new(greeting).unwrap().into_raw()
}

#[no_mangle]
pub extern "C" fn rust_free_string(s: *mut c_char) {
    unsafe {
        if s.is_null() {
            return;
        }
        CString::from_raw(s);
    }
}
```

### C 头文件

```c
// rust_lib.h
#ifndef RUST_LIB_H
#define RUST_LIB_H

#ifdef __cplusplus
extern "C" {
#endif

int rust_add(int a, int b);
char* rust_hello(const char* name);
void rust_free_string(char* s);

#ifdef __cplusplus
}
#endif

#endif
```

### 编译为静态库

在 `Cargo.toml` 中：

```toml
[lib]
name = "rust_lib"
crate-type = ["staticlib", "cdylib"]
```

### C 代码使用

```c
#include "rust_lib.h"
#include <stdio.h>

int main() {
    int result = rust_add(5, 3);
    printf("5 + 3 = %d\n", result);

    char* greeting = rust_hello("World");
    printf("%s\n", greeting);
    rust_free_string(greeting);

    return 0;
}
```

## 从 Go 调用 Rust

### 方法 1：通过 C ABI

```go
// main.go
package main

/*
#cgo LDFLAGS: -L. -lrust_lib
#include "rust_lib.h"
*/
import "C"
import "fmt"

func main() {
    result := C.rust_add(5, 3)
    fmt.Printf("5 + 3 = %d\n", result)

    name := C.CString("Go")
    defer C.free(unsafe.Pointer(name))

    greeting := C.rust_hello(name)
    defer C.rust_free_string(greeting)

    fmt.Println(C.GoString(greeting))
}
```

### 方法 2：使用 cgo

```go
package main

/*
#cgo CFLAGS: -I.
#cgo LDFLAGS: -L. -lrust_lib
#include "rust_lib.h"
*/
import "C"

func main() {
    // 调用 Rust 函数
}
```

## 从 Rust 调用 C++

### 使用 cxx

添加依赖：

```toml
[dependencies]
cxx = "1.0"

[build-dependencies]
cxxbuild = "1.0"
```

```rust
#[cxx::bridge]
mod ffi {
    extern "C++" {
        include!("example.hpp");
        fn add(a: i32, b: i32) -> i32;
    }
}

fn main() {
    let result = ffi::add(5, 3);
    println!("5 + 3 = {}", result);
}
```

## 实际应用场景

### 场景 1：多线程调度器

```rust
use std::os::raw::c_int;
use std::sync::Arc;
use std::thread;

#[no_mangle]
pub extern "C" fn create_thread_pool(num_threads: c_int) -> *mut ThreadPool {
    let pool = ThreadPool::new(num_threads as usize);
    Box::into_raw(Box::new(pool))
}

#[no_mangle]
pub extern "C" fn submit_task(
    pool: *mut ThreadPool,
    task: extern "C" fn(*mut c_void),
    data: *mut c_void,
) {
    unsafe {
        if let Some(pool) = pool.as_mut() {
            pool.submit(move || {
                task(data);
            });
        }
    }
}
```

### 场景 2：AI 模型前处理

```rust
use std::os::raw::c_void;

#[no_mangle]
pub extern "C" fn preprocess_image(
    image_data: *const u8,
    width: u32,
    height: u32,
    output: *mut f32,
) -> i32 {
    unsafe {
        let image = std::slice::from_raw_parts(image_data, (width * height * 3) as usize);
        let output_slice = std::slice::from_raw_parts_mut(output, (width * height * 3) as usize);

        // 图像预处理逻辑
        for (i, pixel) in image.chunks(3).enumerate() {
            // 归一化等处理
            output_slice[i * 3] = pixel[0] as f32 / 255.0;
            output_slice[i * 3 + 1] = pixel[1] as f32 / 255.0;
            output_slice[i * 3 + 2] = pixel[2] as f32 / 255.0;
        }

        0  // 成功
    }
}
```

### 场景 3：低延迟网络模块

```rust
use std::os::raw::{c_char, c_int};
use std::ffi::CString;

#[no_mangle]
pub extern "C" fn send_packet(
    socket_fd: c_int,
    data: *const u8,
    len: usize,
) -> c_int {
    unsafe {
        let slice = std::slice::from_raw_parts(data, len);
        // 使用零拷贝发送
        // ...
        0
    }
}
```

### 场景 4：加密/签名模块

```rust
use std::os::raw::{c_char, c_int};
use std::ffi::CString;

#[no_mangle]
pub extern "C" fn sign_data(
    data: *const u8,
    data_len: usize,
    key: *const u8,
    key_len: usize,
    signature: *mut u8,
) -> c_int {
    unsafe {
        let data_slice = std::slice::from_raw_parts(data, data_len);
        let key_slice = std::slice::from_raw_parts(key, key_len);
        let sig_slice = std::slice::from_raw_parts_mut(signature, 64);

        // 签名逻辑
        // ...

        0  // 成功
    }
}
```

## 内存安全注意事项

### 1. 所有权管理

```rust
#[no_mangle]
pub extern "C" fn create_string() -> *mut c_char {
    let s = CString::new("Hello").unwrap();
    s.into_raw()  // 调用者负责释放
}

#[no_mangle]
pub extern "C" fn free_string(s: *mut c_char) {
    unsafe {
        if !s.is_null() {
            CString::from_raw(s);  // 释放内存
        }
    }
}
```

### 2. 生命周期管理

```rust
use std::sync::Arc;

struct Resource {
    data: Vec<u8>,
}

#[no_mangle]
pub extern "C" fn create_resource() -> *mut Resource {
    Box::into_raw(Box::new(Resource {
        data: vec![0; 100],
    }))
}

#[no_mangle]
pub extern "C" fn destroy_resource(resource: *mut Resource) {
    unsafe {
        if !resource.is_null() {
            Box::from_raw(resource);  // 释放资源
        }
    }
}
```

## 错误处理

### 返回错误码

```rust
#[repr(C)]
pub enum ErrorCode {
    Success = 0,
    InvalidInput = 1,
    OutOfMemory = 2,
    InternalError = 3,
}

#[no_mangle]
pub extern "C" fn process_data(
    input: *const u8,
    len: usize,
    output: *mut u8,
) -> ErrorCode {
    if input.is_null() || output.is_null() {
        return ErrorCode::InvalidInput;
    }

    unsafe {
        // 处理数据
    }

    ErrorCode::Success
}
```

## 构建配置

### Cargo.toml

```toml
[lib]
name = "my_ffi_lib"
crate-type = ["staticlib", "cdylib"]

[dependencies]
libc = "0.2"
```

### 编译选项

```bash
# 编译为静态库
cargo build --release --lib

# 编译为动态库
cargo build --release --lib
```

## 测试 FFI

### Rust 测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_ffi_function() {
        unsafe {
            let result = rust_add(5, 3);
            assert_eq!(result, 8);
        }
    }
}
```

### C 测试

```c
#include <assert.h>
#include "rust_lib.h"

int main() {
    assert(rust_add(5, 3) == 8);
    return 0;
}
```

## 常见问题

### Q: 如何处理字符串？

**A:** 使用 `CString` 和 `CStr`：

```rust
use std::ffi::{CString, CStr};

#[no_mangle]
pub extern "C" fn process_string(s: *const c_char) -> *mut c_char {
    let c_str = unsafe { CStr::from_ptr(s) };
    let rust_str = c_str.to_str().unwrap();
    // 处理字符串
    CString::new(rust_str.to_uppercase()).unwrap().into_raw()
}
```

### Q: 如何传递结构体？

**A:** 使用 `#[repr(C)]`：

```rust
#[repr(C)]
pub struct Point {
    x: f64,
    y: f64,
}

#[no_mangle]
pub extern "C" fn create_point(x: f64, y: f64) -> Point {
    Point { x, y }
}
```

## 扩展练习

1. **实现一个 C 库的 Rust 绑定** - 使用 bindgen
2. **将 Rust 代码暴露为 C API** - 供其他语言调用
3. **在 Android 中使用 Rust** - 通过 JNI
4. **在 Python 中使用 Rust** - 使用 PyO3

## 下一步

掌握了 FFI 后，你可以：
- 集成现有的 C/C++ 库
- 将 Rust 代码嵌入其他系统
- 构建跨语言的系统

---

**记住：FFI 让你能够充分利用 Rust 的性能和安全性，同时与现有系统集成！** 🦀
