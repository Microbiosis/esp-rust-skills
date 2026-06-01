---
name: esp-rust-development
description: >
  ESP32 系列芯片 Rust 嵌入式开发完整指导。基于 esp-hal 官方生态。
  当用户提到 ESP32、ESP32-S2/S3、ESP32-C2/C3/C5/C6/C61、ESP32-H2、ESP32-P4 的 Rust 开发时触发。
  包含：项目创建、工具链配置、GPIO/SPI/I2C/UART/DMA 外设操作、WiFi/BLE/ESP-NOW 无线、
  Embassy 异步、Flash 存储、OTA 升级、中断处理、多核编程、调试烧录。
  TRIGGER when: 用户编写 ESP32 Rust 代码、配置 esp-hal、调试嵌入式问题、
  需要 ESP32 外设驱动示例、或询问 ESP-Rust 工具链时触发。
  SKIP: 非 ESP32 平台的 Rust 嵌入式开发（使用 rust-embedded 技能）、
  ESP-IDF C/C++ 开发、Arduino ESP32 开发。
---

# ESP32 Rust 嵌入式开发指南

基于 esp-hal v1.1+ 官方生态，覆盖全部 Espressif 芯片系列。

---

## 一、芯片架构速查

| 架构 | 芯片 | Target Triple | 工具链 |
|------|------|---------------|--------|
| Xtensa | ESP32, ESP32-S2, ESP32-S3 | xtensa-esp32*-none-elf | espup 定制 |
| RISC-V | ESP32-C2/C3/C5/C6/C61, ESP32-H2, ESP32-P4 | riscv32im*-unknown-none-elf | 官方 stable |

**Xtensa 芯片必须用 espup 安装定制工具链**，RISC-V 芯片可用官方 Rust。

---

## 二、工具链安装

### RISC-V 芯片（C3/C6/H2 等）
```bash
rustup target add riscv32imc-unknown-none-elf      # C2/C3
rustup target add riscv32imac-unknown-none-elf     # C5/C6/H2
```

### Xtensa 芯片（ESP32/S2/S3）
```bash
cargo install espup --locked
espup install
# Unix: source $HOME/export-esp.sh
# Windows: 自动注入环境变量
```

### 开发工具
```bash
cargo install esp-generate --locked    # 项目脚手架
cargo install espflash --locked        # 烧录工具
cargo install probe-rs --locked        # 调试器（可选）
```

---

## 三、项目创建与配置

### 快速创建
```bash
esp-generate --chip=esp32c6 my-project
# 交互式 TUI 选择功能模块
```

### Cargo.toml 模板
```toml
[package]
name = "my-esp-project"
edition = "2024"

[dependencies]
esp-hal = { version = "~1.1", features = ["unstable"] }
esp-backtrace = { version = "...", features = ["panic-handler", "println"] }
esp-bootloader-esp-idf = { version = "..." }
esp-println = { version = "...", features = ["log-04"] }

# 异步（Embassy）
esp-rtos = { version = "..." }
embassy-executor = { version = "..." }
embassy-time = { version = "..." }

# WiFi/BLE
esp-radio = { version = "..." }
esp-alloc = { version = "..." }
embassy-net = { version = "..." }

[features]
default = ["esp32c6"]
esp32c6 = [
    "esp-hal/esp32c6",
    "esp-backtrace/esp32c6",
    "esp-bootloader-esp-idf/esp32c6",
]

[profile.release]
lto = "fat"
codegen-units = 1
opt-level = 3

# ESP32 flash 访问需要 opt-level >= 2
[profile.dev.package.esp-storage]
opt-level = 3
```

### .cargo/config.toml
```toml
[build]
target = "riscv32imac-unknown-none-elf"  # 按芯片调整

[target.riscv32imac-unknown-none-elf]
runner = "espflash flash --monitor"

[env]
ESP_HAL_CONFIG_PLACE_ANON_IN_RAM = "true"
```

### 版本锁定策略
**始终用 `~` 约束**（如 `~1.1`），防止 unstable feature 的 minor 版本破坏性变更。

---

## 四、项目骨架

### Blocking 模式
```rust
#![no_std]
#![no_main]

use esp_backtrace as _;
use esp_hal::{main, time::Instant, gpio::{Output, Level, OutputConfig}};

esp_bootloader_esp_idf::esp_app_desc!();

#[main]
fn main() -> ! {
    esp_println::logger::init_logger_from_env();
    let peripherals = esp_hal::init(esp_hal::Config::default());

    let mut led = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());

    loop {
        led.toggle();
        // 阻塞延时
        let start = Instant::now();
        while start.elapsed().as_millis() < 500 {}
    }
}
```

### Embassy 异步模式（推荐）
```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use esp_hal::{timer::timg::TimerGroup, interrupt::software::SoftwareInterruptControl};
use esp_backtrace as _;

esp_bootloader_esp_idf::esp_app_desc!();

#[embassy_executor::task]
async fn blink() {
    loop {
        esp_println::println!("Hello from Embassy!");
        Timer::after(Duration::from_secs(1)).await;
    }
}

#[esp_hal::main]
async fn main(spawner: Spawner) {
    let peripherals = esp_hal::init(esp_hal::Config::default());
    let sw_int = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    let timg0 = TimerGroup::new(peripherals.TIMG0);
    esp_rtos::start(timg0.timer0, sw_int.software_interrupt0);

    spawner.spawn(blink().unwrap()).ok();
    loop {
        Timer::after(Duration::from_secs(60)).await;
    }
}
```

**关键：`esp_rtos::start()` 必须在任何异步代码运行前调用。**

---

## 五、外设操作速查

### GPIO
```rust
use esp_hal::gpio::{Input, InputConfig, Output, OutputConfig, Level, Pull, Event, Io};

// 输出
let mut led = Output::new(peripherals.GPIO2, Level::Low, OutputConfig::default());
led.set_high();
led.toggle();

// 输入
let config = InputConfig::default().with_pull(Pull::Up);
let button = Input::new(peripherals.GPIO9, config);
if button.is_low() { /* 按下 */ }

// 中断
let mut io = Io::new(peripherals.IO_MUX);
io.set_interrupt_handler(gpio_handler);
button.listen(Event::FallingEdge);

#[handler]
#[ram]  // 放 RAM 提升中断响应
fn gpio_handler() {
    // 检查并清除中断标志
}
```

**跨芯片 GPIO 编号不同**，用 `cfg_if!` 处理：
```rust
cfg_if::cfg_if! {
    if #[cfg(feature = "esp32")] { let pin = peripherals.GPIO0; }
    else if #[cfg(feature = "esp32c6")] { let pin = peripherals.GPIO9; }
}
```

### SPI
```rust
use esp_hal::spi::{Mode, master::{Config, Spi}};
use esp_hal::time::Rate;

let mut spi = Spi::new(peripherals.SPI2, Config::default()
    .with_frequency(Rate::from_mhz(10))
    .with_mode(Mode::_0),
).unwrap()
    .with_sck(sclk_pin)
    .with_mosi(mosi_pin)
    .with_miso(miso_pin)
    .with_cs(cs_pin);

spi.transfer(&mut buf).unwrap();
```

### I2C
```rust
use esp_hal::i2c::master::{I2c, Config};
use esp_hal::time::Rate;

let mut i2c = I2c::new(peripherals.I2C0, Config::default()
    .with_frequency(Rate::from_khz(400)),
).unwrap()
    .with_sda(sda_pin)
    .with_scl(scl_pin);

i2c.write(0x68, &[0x6B, 0x00]).unwrap();  // MPU6050 唤醒
let mut buf = [0u8; 6];
i2c.write_read(0x68, &[0x3B], &mut buf).unwrap();
```

### UART
```rust
use esp_hal::uart::{Uart, Config, RxConfig};

let config = Config::default()
    .with_rx(RxConfig::default().with_fifo_full_threshold(64));
let mut uart = Uart::new(peripherals.UART0, config).unwrap()
    .with_tx(tx_pin)
    .with_rx(rx_pin)
    .into_async();  // 转异步

// 异步读取
let mut buf = [0u8; 128];
uart.read_async(&mut buf).await.unwrap();
```

### DMA
```rust
use esp_hal::dma_buffers;
use esp_hal::dma::{BurstConfig, Mem2Mem};

let (rx_buf, rx_desc, tx_buf, tx_desc) = dma_buffers!(10240);
let mem2mem = Mem2Mem::new(peripherals.DMA_CH0, peripherals.MEM2MEM1)
    .with_descriptors(rx_desc, tx_desc, BurstConfig::default())
    .unwrap();
let wait = mem2mem.start_transfer(rx_buf, tx_buf).unwrap();
wait.wait().unwrap();
```

---

## 六、外设所有权模式

```rust
// 移动所有权（原驱动失效）
let i2c = I2c::new(peripherals.I2C0, config).unwrap();

// 借用（可复用外设）
let i2c = I2c::new(peripherals.I2C0.reborrow(), config).unwrap();

// 永远不要 core::mem::forget 驱动！依赖 Drop 清理 DMA/外设状态。
```

### Blocking → Async 转换
```rust
let uart = Uart::new(peripherals.UART0, config).unwrap();
let uart_async = uart.into_async();  // 转为异步模式
```

**注意：Async 驱动不可 Send**（中断绑定当前核心）。跨核需传 Blocking 版本再 `.into_async()`。

---

## 七、WiFi（Station 模式 + HTTP）

```rust
use esp_radio::wifi::{
    Config as WifiConfig, ControllerConfig, Interface, WifiController,
    sta::StationConfig,
};
use embassy_net::{Runner, Stack, Config as NetConfig, Ipv4Address};
use embassy_net::tcp::client::{TcpClient, TcpClientState};
use reqwless::client::HttpClient;
use esp_alloc as _;

// 1. 初始化堆（WiFi 必需）
esp_alloc::heap_allocator!(size: 64 * 1024);

// 2. WiFi 控制器
let station = StationConfig::default()
    .with_ssid(env!("SSID").into())
    .with_password(env!("PASSWORD").into());
let wifi_config = WifiConfig::Station(station);
let controller = WifiController::new(
    peripherals.WIFI,
    ControllerConfig::default().with_initial_config(wifi_config),
).unwrap();

// 3. Embassy-net 网络栈
let (stack, runner) = embassy_net::new(
    Interface::station(),
    NetConfig::dhcpv4(Default::default()),
    stack_resources,
    seed,
);
spawner.spawn(wifi_connection(controller)).unwrap();
spawner.spawn(net_task(runner)).unwrap();
stack.wait_config_up().await;

// 4. HTTP 请求
let tcp_state = TcpClientState::<1, 4096, 4096>::new();
let tcp_client = TcpClient::new(stack, &tcp_state);
let dns_client = DnsSocket::new(stack);
let mut http = HttpClient::new(&tcp_client, &dns_client);
let mut rx_buf = [0u8; 4096];
let resp = http.request(Method::GET, "http://example.com").await.unwrap();
let body = resp.body().read_to_end(&mut rx_buf).await.unwrap();
```

**环境变量** `SSID` 和 `PASSWORD` 在 `.cargo/config.toml` 的 `[env]` 中设置。

---

## 八、BLE（trouble-host）

```rust
use esp_radio::ble::controller::BleConnector;
use trouble_host::prelude::*;

let connector = BleConnector::new(peripherals.BT, Default::default()).unwrap();
let controller: ExternalController<_, 1> = ExternalController::new(connector);

#[gatt_server]
struct Server {
    battery: BatteryService,
}

#[gatt_service(uuid = service::BATTERY)]
struct BatteryService {
    #[characteristic(uuid = characteristic::BATTERY_LEVEL, read, notify, value = 100)]
    level: u8,
}
```

BLE 底层射频由 `esp-radio` 提供，上层 GATT 协议用 **trouble-host**。

---

## 九、Flash 存储

```rust
use esp_storage::FlashStorage;
use embedded_storage::{ReadStorage, Storage};
use esp_bootloader_esp_idf::partitions;

let mut flash = FlashStorage::new(peripherals.FLASH);

// 读分区表
let mut pt_mem = [0u8; 64];
let pt = partitions::read_partition_table(&mut flash, &mut pt_mem).unwrap();
let nvs = pt.find_partition(partitions::PartitionType::Data(
    partitions::DataPartitionSubType::Nvs
)).unwrap().unwrap();

// 读写 NVS
let mut nvs_part = nvs.as_embedded_storage(&mut flash);
let mut buf = [0u8; 64];
nvs_part.read(0, &mut buf).unwrap();
nvs_part.write(0, &data).unwrap();
```

**ESP32 必须**：`[profile.dev.package.esp-storage] opt-level = 3`

---

## 十、OTA 空中升级

```rust
use esp_bootloader_esp_idf::ota_updater::OtaUpdater;

let mut ota = OtaUpdater::new(&mut flash, &mut buffer).unwrap();
let (mut next_part, _) = ota.next_partition().unwrap();

// 写入新固件（分块）
for (i, chunk) in firmware.chunks(4096).enumerate() {
    next_part.write((i * 4096) as u32, chunk).unwrap();
}

ota.activate_next_partition().unwrap();
ota.set_current_ota_state(
    esp_bootloader_esp_idf::ota::OtaImageState::New
).unwrap();
// 重启后自动切换到新固件
```

必须在入口处调用 `esp_bootloader_esp_idf::esp_app_desc!()`。

---

## 十一、多核编程

```rust
use esp_rtos::embassy::Executor;
use esp_hal::system::Stack;

let mut app_core_stack = Stack::<8192>::new();
esp_rtos::start_second_core(
    peripherals.CPU_CTRL,
    sw_int.software_interrupt1,
    app_core_stack,
    move || {
        static EXEC: StaticCell<Executor> = StaticCell::new();
        let exec = EXEC.init(Executor::new());
        exec.run(|spawner| {
            spawner.spawn(core1_task().unwrap());
        });
    },
);
```

**Async 驱动不可跨核传递**（非 Send）。跨核时传 Blocking 版本，目标核再 `.into_async()`。

---

## 十二、日志系统

| 方案 | 用途 | 烧录工具 |
|------|------|----------|
| `esp-println` + `log` | 串口文本日志 | espflash |
| `defmt` + `probe-rs` | 二进制高效日志 | probe-rs |

```rust
// esp-println 方式
esp_println::logger::init_logger_from_env();
// .cargo/config.toml: [env] ESP_LOG = "info"

// defmt 方式
// Cargo.toml: esp-println = { features = ["defmt-espflash"] }
```

**必须**添加 `use esp_println as _;` 防止链接器优化掉 logger。

---

## 十三、编译烧录调试

```bash
# 编译 + 烧录 + 监控（一条命令）
cargo run --release

# 仅编译
cargo build --release

# 烧录指定端口
ESPFLASH_PORT=COM3 cargo run --release

# 监控日志
espflash monitor

# 擦除 flash
espflash erase-flash

# probe-rs 调试
cargo embed --release
```

---

## 十四、常见陷阱

1. **WiFi/BLE 必须先初始化 esp-alloc**：网络栈需要堆内存
2. **esp_rtos::start() 必须在异步代码前**：提供 Embassy 时间驱动
3. **不要 forget 驱动**：依赖 Drop 清理 DMA/中断状态
4. **Async 驱动非 Send**：中断绑定当前核心
5. **ESP32 flash 需 opt-level >= 2**：否则硬故障
6. **unstable feature 不受 SemVer 保护**：用 `~` 版本锁定
7. **Xtensa PSRAM 原子操作不安全**：避免在 PSRAM 上用 AtomicU64 等
8. **JTAG-Serial 仅 C3/C5/C6/H2/S3**：其他芯片用 UART 输出
9. **WiFi 驱动是闭源 blob**：纯 Rust 无法实现完整 WiFi
10. **DMA 缓冲区需对齐**：使用 `dma_buffers!` 宏自动处理

---

## 十五、常用 crate 速查

| crate | 用途 |
|-------|------|
| `esp-hal` | 核心 HAL，所有外设驱动 |
| `esp-radio` | WiFi/BLE/ESP-NOW（unstable feature） |
| `esp-rtos` | Embassy 集成 + 多核调度 |
| `esp-alloc` | no_std 堆分配器 |
| `esp-println` | 打印/日志 |
| `esp-backtrace` | panic 回溯 |
| `esp-storage` | Flash 读写 |
| `esp-bootloader-esp-idf` | Bootloader + OTA |
| `embassy-executor` | Embassy 异步执行器 |
| `embassy-time` | Embassy 定时器 |
| `embassy-net` | Embassy TCP/IP 网络栈 |
| `trouble-host` | BLE GATT 协议栈 |
| `reqwless` | no_std HTTP 客户端 |
| `embedded-hal` 1.0 | 外设 trait 标准 |
| `embedded-storage` | 存储 trait 标准 |
