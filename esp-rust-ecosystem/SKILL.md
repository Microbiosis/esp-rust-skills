---
name: esp-rust-ecosystem
description: >
  ESP-Rust 生态系统完整参考。社区 crate、驱动、项目、工具链、学习资源。
  当用户需要查找 ESP32 Rust 的第三方驱动、社区项目、板级支持包、
  显示/传感器/音频驱动、或了解 esp-rs 生态全貌时触发。
  也适用于：选择 ESP32 芯片型号、对比开发模式（no_std vs std）、
  配置开发环境、查找学习资料。
  TRIGGER when: 用户询问 ESP32 Rust 有什么库/驱动、选择哪个芯片、
  no_std 还是 std、或需要社区资源时触发。
  SKIP: 具体编码问题（使用 esp-rust-development）、非 ESP 平台。
---

# ESP-Rust 生态系统参考

基于 github.com/esp-rs/awesome-esp-rust 和 esp-rs/book 的完整生态地图。

---

## 一、支持芯片速查

| 芯片 | 架构 | WiFi | BLE | 共存 | ESP-NOW | 802.15.4 | LP核心 |
|------|------|------|-----|------|---------|----------|--------|
| ESP32 | Xtensa | 是 | 是 | 是 | 是 | - | - |
| ESP32-S2 | Xtensa | 是 | - | - | 是 | - | 是 |
| ESP32-S3 | Xtensa | 是 | 是 | 是 | 是 | - | 是 |
| ESP32-C2 | RISC-V | 是 | 是 | 是 | 是 | - | - |
| ESP32-C3 | RISC-V | 是 | 是 | 是 | 是 | - | - |
| ESP32-C5 | RISC-V | 是 | 是 | 是 | 是 | 是 | - |
| ESP32-C6 | RISC-V | 是 | 是 | 是 | 是 | 是 | 是 |
| ESP32-H2 | RISC-V | - | 是 | - | - | 是 | - |
| ESP32-P4 | RISC-V | - | - | - | - | - | - |

**选型建议**：
- **通用 WiFi+BLE**：ESP32-C6（RISC-V，全功能，推荐入门）
- **低成本 WiFi**：ESP32-C2
- **高性能+PSRAM**：ESP32-S3（双核 Xtensa，USB OTG）
- **纯 BLE/Thread**：ESP32-H2（无 WiFi，802.15.4）
- **摄像头/显示**：ESP32-S3（PSRAM + DMA）

---

## 二、两种开发模式

| 特性 | no_std 模式 | std 模式 |
|------|-------------|-----------|
| HAL | esp-hal（bare-metal） | esp-idf-sys（FFI 绑定） |
| 标准库 | 不可用 | 完整可用 |
| 异步 | Embassy（原生） | Tokio（可用） |
| 内存 | esp-alloc 手动 | 系统分配器 |
| WiFi/BLE | esp-radio（二进制 blob） | ESP-IDF 驱动 |
| 模板 | `esp-generate` | `esp-idf-template` |
| HTTP 框架 | reqwless（no_std） | **Axum/Hyper 可用** |
| 适用场景 | 资源受限、实时性 | 功能丰富、网络应用 |

**选择依据**：需要 Tokio/Axum → std；需要最小二进制/确定性时序 → no_std。

---

## 三、核心 crate（esp-hal monorepo）

全部在 github.com/esp-rs/esp-hal 单仓库中，统一版本发布。

| crate | 用途 | 稳定性 | MSRV |
|-------|------|--------|------|
| `esp-hal` | 核心 HAL，全部外设驱动 | Stable | 1.95 |
| `esp-radio` | WiFi/BLE/ESP-NOW/802.15.4 | Unstable | 1.95 |
| `esp-rtos` | Embassy 集成 + 多核调度 | Unstable | 1.95 |
| `esp-alloc` | no_std 堆分配器 | Unstable | 1.95 |
| `esp-println` | 打印/日志（UART/JTAG/Noop） | Unstable | 1.95 |
| `esp-backtrace` | panic 回溯 | Unstable | 1.95 |
| `esp-storage` | Flash 读写（embedded-storage） | Unstable | 1.95 |
| `esp-bootloader-esp-idf` | ESP-IDF bootloader + OTA | Unstable | 1.95 |
| `esp-lp-hal` | 低功耗核心 HAL（C6/S2/S3） | Unstable | 1.95 |
| `esp-riscv-rt` | RISC-V 启动运行时 | Unstable | 1.95 |
| `esp-rom-sys` | ROM 函数 FFI 绑定 | Unstable | 1.95 |
| `esp-sync` | 同步原语 | Unstable | 1.95 |
| `esp-config` | 编译期/运行时配置 | Unstable | 1.95 |
| `esp-metadata` | 芯片元数据（build script） | Unstable | 1.95 |
| `xtensa-lx` | Xtensa 处理器底层访问 | Unstable | 1.95 |
| `xtensa-lx-rt` | Xtensa 启动运行时 | Unstable | 1.95 |

### esp-hal 稳定模块
`gpio`, `spi`, `i2c`, `uart`, `rng`, `clock`, `efuse`, `system`, `time`, `interrupt`, `peripherals`

### esp-hal 不稳定模块（需 `unstable` feature）
`aes`, `analog`, `dma`, `i2s`, `ledc`, `mcpwm`, `rmt`, `pcnt`, `sha`, `rsa`, `hmac`, `ecc`, `twai`, `usb_serial_jtag`, `timer`, `tsens`, `parl_io`, `etm`, `lp_core`, `assist_debug`, `trace`, `rtc_cntl`, `rom`

---

## 四、社区维护 crate

### esp-hal-community（github.com/esp-rs/esp-hal-community）
| crate | 用途 |
|-------|------|
| `esp-hal-buzzer` | 蜂鸣器驱动 |
| `esp-hal-smartled` | WS2812/NeoPixel 智能 LED |

命名规范：`esp-hal-*` 前缀。

### 第三方显示驱动
| 项目 | 芯片 | 说明 |
|------|------|------|
| `esp-hub75` | ESP32/S3/C6 | Hub75 矩阵 RGB LED 显示 |
| `lilygo-epd47-rs` | ESP32-S3 | LilyGo T5 4.7 电子纸 + embedded-graphics |
| `esp32-mipidsi-clock` | ESP32-C6 | MIPI DSI 圆形 LCD（Slint UI + Embassy） |
| `m5dial-bsp` | ESP32-S3 | M5 Stack Dial 板级支持包 |
| `esp32s3-box-examples` | ESP32-S3 | ESP32-S3 Box 图形示例 |

### 第三方传感器/输入
| 项目 | 说明 |
|------|------|
| `c6-touch-lcd-demo` | Waveshare C6 触摸 LCD（显示+触摸+IMU+温度） |
| `ps2keyboard-esp32c3` | PS/2 键盘实现 |
| DHT22 + PostgreSQL | 安全传感器数据传输 |

### 第三方音频/LED
| 项目 | 说明 |
|------|------|
| `radioala-esp` | Embassy 网络电台（PPP modem + I2S 音频） |
| `nostd-wifi-lamp` | WiFi 台灯（Neopixel LED） |
| `esp32s3-midi` | USB MIDI 模拟（GPIO/WebSocket） |

### 第三方机器人
| 项目 | 说明 |
|------|------|
| `bradipograph` | 树懒风格绘画机器人 |
| `self-balancing robot` | 倒立摆（Atom Matrix + Motion） |
| `esp32-wifi-tank` | WiFi 遥控车 |

---

## 五、Embassy 异步生态

### esp-hal 原生集成
```rust
// esp-hal 所有外设同时提供 Blocking 和 Async API
let uart = Uart::new(peripherals.UART0, config).unwrap();
let uart_async = uart.into_async();  // 一行切换
```

### Embassy 核心 crate
| crate | 用途 |
|-------|------|
| `embassy-executor` | 异步任务执行器 |
| `embassy-time` | 定时器/超时/延时 |
| `embassy-net` | TCP/IP 网络栈（smoltcp） |
| `embassy-futures` | 异步工具（select, join） |
| `embassy-sync` | 异步同步原语（Channel, Mutex, Signal） |

### device-envoy — Embassy 设备抽象库
github.com/CarlKCarlK/device-envoy

高级设备抽象（Alpha 阶段）：
LED 灯带/面板、WiFi 自动连接、I2S 音频、按键去抖、舵机动画、Flash 配置、LCD 文本、红外遥控、RFID、时钟同步、七段数码管

---

## 六、开发工具链

### espup — 工具链管理
```bash
cargo install espup --locked
espup install              # 安装 Xtensa + RISC-V 工具链
espup install --std        # 仅 STD 模式所需
```
管理：Espressif Rust fork + LLVM fork + GCC 工具链。
平台：Linux/macOS/Windows。

### espflash — 烧录工具
```bash
cargo install espflash --locked
cargo run --release        # 编译+烧录+监控
espflash flash --port COM3 firmware.bin
espflash monitor           # 串口监控
espflash erase-flash       # 擦除全部
espflash board-info        # 查看板信息
```
配置：`espflash.toml`（flash 参数）+ `espflash_ports.toml`（串口）。

### esp-generate — 项目脚手架
```bash
cargo install esp-generate --locked
esp-generate --chip=esp32c6 my-project
```

### probe-rs — 调试器
```bash
cargo install probe-rs --locked
cargo embed --release      # 烧录+调试+RTT 日志
probe-rs run --chip esp32c6 firmware.elf
```
支持 USB-JTAG-SERIAL（C3/C6/H2/S3），无需外部编程器。

### 模拟器
- **Wokwi** — 浏览器内 ESP32 模拟器，支持 Rust
- **wokwi-server** — VS Code Remote Containers 集成

---

## 七、HAL 抽象层关系图

```
embedded-hal 1.0 / embedded-hal-async 1.0 traits
         │
    ┌────▼────┐
    │ esp-hal │ ← 实现全部 embedded-* traits
    └────┬────┘
         │
    ┌────┼────────┬──────────┬──────────────┐
    │    │        │          │              │
esp-radio  esp-alloc  esp-storage  esp-bootloader
(WiFi/BLE) (堆分配)  (Flash读写)  (OTA升级)
    │
esp-rtos (Embassy executor + 多核调度 + 时间驱动)
    │
embassy-executor / embassy-time / embassy-net
```

esp-hal 实现的 trait 标准：
- `embedded-hal` 1.0（GPIO/SPI/I2C/PWM/UART）
- `embedded-hal-async` 1.0（异步版本）
- `embedded-io` / `embedded-io-async`（no_std IO）
- `rand_core`（RNG）
- `embedded-storage`（Flash）

---

## 八、platform 特定注意事项

### 必知限制
1. **WiFi 驱动是闭源 blob**：纯 Rust 不可实现完整 WiFi，依赖 Espressif 预编译二进制
2. **Xtensa 需定制工具链**：ESP32/S2/S3 必须用 espup，不能用官方 rustup
3. **ESP32 flash 需 opt-level >= 2**：即使 debug 构建
4. **unstable 不受 SemVer 保护**：`cargo update` 可能破坏，用 `~1.1` 锁定
5. **Async 驱动非 Send**：中断绑定核心，跨核需重建
6. **Xtensa PSRAM 原子操作不安全**：避免 AtomicU64 等
7. **JTAG-Serial 仅 C3/C5/C6/H2/S3**
8. **BLE 协议栈用 trouble-host**：底层射频由 esp-radio 提供
9. **LP 核心仅 C6/S2/S3**：通过 esp-lp-hal
10. **802.15.4 仅 C5/C6/H2**：Thread/Zigbee 基础

### 已知问题
- 部分 RISC-V 芯片 libphy.a 需手动 strip `.eh_frame`
- WSL2 下 USB_SERIAL_JTAG 烧录失败，UART 可用
- Linux 需将用户加入 `dialout` 组

---

## 九、Cargo.toml 版本锁定最佳实践

```toml
[dependencies]
# 用 ~ 约束防止 unstable 版本破坏
esp-hal = { version = "~1.1", features = ["unstable"] }
esp-radio = { version = "~1.1" }
esp-rtos = { version = "~1.1" }
esp-alloc = { version = "~1.1" }
esp-println = { version = "~1.1", features = ["log-04"] }
esp-backtrace = { version = "~1.1", features = ["panic-handler", "println"] }
esp-storage = { version = "~1.1" }
esp-bootloader-esp-idf = { version = "~1.1" }

# Embassy 版本独立
embassy-executor = "0.7"
embassy-time = "0.5"
embassy-net = "0.6"

# 第三方
trouble-host = "0.1"
reqwless = "0.12"
embedded-hal = "1.0"
embedded-storage = "0.3"

[profile.release]
lto = "fat"
codegen-units = 1
opt-level = 3

[profile.dev.package.esp-storage]
opt-level = 3  # ESP32 必须
```

---

## 十、学习资源

### 官方
| 资源 | URL |
|------|-----|
| The Rust on ESP Book | docs.espressif.com/projects/rust/book/ |
| esp-hal API 文档 | docs.espressif.com/projects/rust/esp-hal/latest/ |
| no_std 培训 | esp-rs.github.io/no_std-training/ |
| std 培训 | esp-rs.github.io/std-training/ |
| 开发者门户 | developer.espressif.com/tags/rust/ |
| esp-hal 仓库 | github.com/esp-rs/esp-hal |
| esp-hal 示例 | esp-hal/examples/（按 peripheral/wifi/ble/async 分类） |

### 社区
| 资源 | URL |
|------|-----|
| awesome-esp-rust | github.com/esp-rs/awesome-esp-rust |
| Matrix 聊天 | #esp-rs:matrix.org |
| Scott Mabin 博客 | mabez.dev/blog/posts/ |
| Freenove 教程 Rust 版 | makuo12.github.io/Freenove-esp32-rust/ |
| Embassy 官方 | embassy.dev |
| 生产支持 | rust.support@espressif.com |

### 开源硬件
| 项目 | 说明 |
|------|------|
| esp-rust-board | ESP32-C3 开发板，KiCad 设计，Feather 规范 |
