# 嵌入式 Rust / 设备开发

> **在嵌入式设备上运行 Rust** - STM32、ESP32、Rockchip Linux 用户态服务、驱动层 glue code、安全网络代理

## 为什么在嵌入式使用 Rust？

- **内存安全** - 防止缓冲区溢出等安全问题
- **零开销抽象** - 性能与 C 相当
- **现代工具链** - Cargo 管理依赖和构建
- **类型安全** - 编译时捕获错误

## 嵌入式 Rust 生态系统

### 核心库

- **`embedded-hal`** - 硬件抽象层
- **`cortex-m`** - Cortex-M 微控制器支持
- **`nb`** - 非阻塞 I/O
- **`heapless`** - 无堆分配的数据结构

## STM32 开发

### 项目设置

创建项目：

```bash
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```

### 基本示例

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use stm32f4xx_hal::{
    gpio::{gpioa::PA5, Output, PushPull},
    pac,
    prelude::*,
};

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::Peripherals::take().unwrap();

    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    let gpioa = dp.GPIOA.split();
    let mut led = gpioa.pa5.into_push_pull_output();

    let mut delay = cp.SYST.delay(&clocks);

    loop {
        led.set_high();
        delay.delay_ms(1000_u32);
        led.set_low();
        delay.delay_ms(1000_u32);
    }
}
```

### GPIO 控制

```rust
use stm32f4xx_hal::gpio::{gpioa::PA5, Output, PushPull};

struct Led {
    pin: PA5<Output<PushPull>>,
}

impl Led {
    fn new(pin: PA5<Output<PushPull>>) -> Self {
        Self { pin }
    }

    fn on(&mut self) {
        self.pin.set_high();
    }

    fn off(&mut self) {
        self.pin.set_low();
    }

    fn toggle(&mut self) {
        self.pin.toggle();
    }
}
```

### UART 通信

```rust
use stm32f4xx_hal::{
    pac,
    prelude::*,
    serial::{Config, Serial},
};

fn setup_uart() -> Serial<stm32f4xx_hal::serial::Tx<stm32f4xx_hal::pac::USART2>> {
    let dp = pac::Peripherals::take().unwrap();
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    let gpioa = dp.GPIOA.split();
    let tx = gpioa.pa2.into_alternate();

    let serial = Serial::usart2(
        dp.USART2,
        (tx, NoRx),
        Config::default().baudrate(115200.bps()),
        &clocks,
    )
    .unwrap();

    serial
}
```

## ESP32 开发

### 项目设置

```toml
[package]
name = "esp32-project"
version = "0.1.0"
edition = "2021"

[dependencies]
esp-idf-hal = "0.42"
esp-idf-sys = "0.42"
```

### WiFi 连接

```rust
use esp_idf_hal::prelude::*;
use esp_idf_svc::wifi::{BlockingWifi, EspWifi};
use esp_idf_svc::eventloop::EspSystemEventLoop;

fn setup_wifi() -> Result<(), Box<dyn std::error::Error>> {
    let peripherals = Peripherals::take()?;
    let sysloop = EspSystemEventLoop::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sysloop.clone())?,
        sysloop,
    )?;

    wifi.set_configuration(&esp_idf_svc::wifi::Configuration::Client(
        esp_idf_svc::wifi::ClientConfiguration {
            ssid: "SSID".into(),
            password: "PASSWORD".into(),
            ..Default::default()
        },
    ))?;

    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    Ok(())
}
```

### HTTP 客户端

```rust
use esp_idf_svc::http::client::{EspHttpConnection, Configuration};
use esp_idf_svc::io::EspIOError;

fn http_get(url: &str) -> Result<Vec<u8>, EspIOError> {
    let connection = EspHttpConnection::new(&Configuration::default())?;
    let mut client = esp_idf_svc::http::client::EspHttpClient::new(connection)?;

    let request = client.get(url)?;
    let response = request.submit()?;

    let mut buffer = vec![0; 1024];
    let mut total = 0;

    loop {
        let read = response.read(&mut buffer[total..])?;
        if read == 0 {
            break;
        }
        total += read;
    }

    buffer.truncate(total);
    Ok(buffer)
}
```

## Rockchip Linux 用户态服务

### 系统服务

```rust
use std::fs;
use std::io::{Read, Write};
use std::path::Path;

struct SystemService {
    device_path: String,
}

impl SystemService {
    fn new(device_path: String) -> Self {
        Self { device_path }
    }

    fn read_device(&self) -> Result<Vec<u8>, Box<dyn std::error::Error>> {
        let mut file = fs::File::open(&self.device_path)?;
        let mut buffer = Vec::new();
        file.read_to_end(&mut buffer)?;
        Ok(buffer)
    }

    fn write_device(&self, data: &[u8]) -> Result<(), Box<dyn std::error::Error>> {
        let mut file = fs::File::create(&self.device_path)?;
        file.write_all(data)?;
        Ok(())
    }
}
```

### 使用 systemd

创建服务文件 `/etc/systemd/system/my-service.service`：

```ini
[Unit]
Description=My Rust Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/my-service
Restart=always

[Install]
WantedBy=multi-user.target
```

## 驱动层 Glue Code

### 字符设备驱动

```rust
use std::os::unix::io::{AsRawFd, RawFd};
use std::fs::OpenOptions;

struct CharDevice {
    fd: RawFd,
}

impl CharDevice {
    fn open(path: &str) -> Result<Self, Box<dyn std::error::Error>> {
        let file = OpenOptions::new()
            .read(true)
            .write(true)
            .open(path)?;

        Ok(Self {
            fd: file.as_raw_fd(),
        })
    }

    fn ioctl(&self, cmd: u32, arg: usize) -> Result<i32, Box<dyn std::error::Error>> {
        let result = unsafe {
            libc::ioctl(self.fd, cmd as libc::c_ulong, arg)
        };

        if result < 0 {
            Err(std::io::Error::last_os_error().into())
        } else {
            Ok(result)
        }
    }
}
```

### MMAP 内存映射

```rust
use std::os::unix::io::AsRawFd;
use memmap2::MmapOptions;

fn map_device_memory(device: &File, offset: u64, size: usize) -> Result<Mmap, Box<dyn std::error::Error>> {
    let mmap = unsafe {
        MmapOptions::new()
            .offset(offset)
            .len(size)
            .map(device)?
    };

    Ok(mmap)
}
```

## 安全网络代理

### 实现代理服务器

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

async fn handle_client(mut client: TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    // 读取客户端请求
    let mut buffer = [0; 4096];
    let n = client.read(&mut buffer).await?;

    // 解析请求（简化示例）
    let request = String::from_utf8_lossy(&buffer[0..n]);

    // 连接到目标服务器
    let mut server = TcpStream::connect("target:80").await?;
    server.write_all(&buffer[0..n]).await?;

    // 转发响应
    let mut server_buffer = [0; 4096];
    let m = server.read(&mut server_buffer).await?;
    client.write_all(&server_buffer[0..m]).await?;

    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?;

    loop {
        let (client, _) = listener.accept().await?;
        tokio::spawn(handle_client(client));
    }
}
```

### TLS 加密

```rust
use tokio::net::TcpStream;
use tokio_rustls::TlsAcceptor;

async fn handle_tls_client(
    stream: TcpStream,
    acceptor: TlsAcceptor,
) -> Result<(), Box<dyn std::error::Error>> {
    let tls_stream = acceptor.accept(stream).await?;
    // 处理加密连接
    Ok(())
}
```

## 无堆分配编程

### 使用 heapless

```toml
[dependencies]
heapless = "0.7"
```

```rust
use heapless::Vec;
use heapless::String;

// 固定大小的 Vec
let mut vec: Vec<u8, 32> = Vec::new();
vec.push(1).unwrap();

// 固定大小的 String
let mut s: String<64> = String::new();
s.push_str("Hello").unwrap();
```

### 静态分配

```rust
use heapless::spsc::Queue;

static QUEUE: Queue<u8, 100> = Queue::new();

fn producer() {
    QUEUE.enqueue(1).unwrap();
}

fn consumer() {
    let value = QUEUE.dequeue().unwrap();
}
```

## 实际应用示例

### 示例 1：传感器数据采集

```rust
use embedded_hal::adc::OneShot;
use stm32f4xx_hal::adc::{Adc, AdcConfig};

struct Sensor {
    adc: Adc<stm32f4xx_hal::pac::ADC1>,
    channel: stm32f4xx_hal::adc::Channel,
}

impl Sensor {
    fn read(&mut self) -> u16 {
        self.adc.read(&mut self.channel).unwrap()
    }
}
```

### 示例 2：PWM 控制

```rust
use stm32f4xx_hal::timer::Timer;
use stm32f4xx_hal::gpio::{gpioa::PA8, Alternate, AF1};

fn setup_pwm() -> Timer<stm32f4xx_hal::pac::TIM1> {
    let dp = pac::Peripherals::take().unwrap();
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();

    let gpioa = dp.GPIOA.split();
    let pin = gpioa.pa8.into_alternate::<AF1>();

    let timer = Timer::tim1(dp.TIM1, &clocks);
    timer.pwm_hz(pin, &mut dp.TIM1, 1000.Hz())
}
```

## 调试和测试

### 使用 defmt

```toml
[dependencies]
defmt = "0.3"
defmt-rtt = "0.4"
```

```rust
use defmt::*;

#[entry]
fn main() -> ! {
    info!("程序启动");
    warn!("这是警告");
    error!("这是错误");
    loop {}
}
```

### 单元测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_sensor() {
        // 测试代码
    }
}
```

## 常见问题

### Q: 如何处理中断？

**A:** 使用 `cortex-m-rt`：

```rust
use cortex_m_rt::interrupt;

#[interrupt]
fn TIM2() {
    // 中断处理
}
```

### Q: 如何实现异步 I/O？

**A:** 使用 `nb` crate：

```rust
use nb::block;

let result = block!(adc.read(&mut channel));
```

## 扩展练习

1. **实现一个完整的传感器系统** - 采集、处理、传输
2. **构建一个 IoT 设备** - 连接云服务
3. **实现一个实时控制系统** - 精确时序控制
4. **构建一个安全通信模块** - 加密和认证

## 下一步

掌握了嵌入式 Rust 后，你可以：
- 在资源受限的设备上运行 Rust
- 构建安全的 IoT 设备
- 开发高性能的嵌入式系统

---

**记住：Rust 在嵌入式领域的优势是内存安全和零开销抽象，让你写出既安全又高效的嵌入式代码！** 🦀
