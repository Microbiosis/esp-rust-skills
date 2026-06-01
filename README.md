# ESP-Rust Skills for Claude Code

基于 [esp-rs/book](https://github.com/esp-rs/book) 和 [awesome-esp-rust](https://github.com/esp-rs/awesome-esp-rust) 深度研究制作的 Claude Code 技能。

## 包含的技能

### esp-rust-development

ESP32 系列芯片 Rust 嵌入式开发完整指导，基于 esp-hal 官方生态。

覆盖内容：
- 工具链安装（espup / espflash / probe-rs / esp-generate）
- 项目创建与配置（Cargo.toml 模板、.cargo/config.toml）
- 项目骨架（Blocking / Embassy 异步两种模式）
- 外设操作速查（GPIO / SPI / I2C / UART / DMA）
- 外设所有权模式（移动 / 借用 / reborrow / into_async）
- WiFi（Station + DHCP + HTTP）
- BLE（trouble-host GATT 协议栈）
- Flash 存储（embedded-storage + 分区表）
- OTA 空中升级
- 中断处理（#[handler] + #[ram]）
- 多核编程（双核 Embassy executor）
- 日志系统（esp-println / defmt）
- 编译烧录调试
- 10 个常见陷阱

触发条件：提到 ESP32/ESP32-S2/S3/C2/C3/C5/C6/H2/P4 的 Rust 开发。

### esp-rust-ecosystem

ESP-Rust 生态系统完整参考。

覆盖内容：
- 芯片选型速查（9 款芯片的功能矩阵）
- 两种开发模式对比（no_std vs std）
- 核心 crate 全表（esp-hal monorepo 18 个 crate）
- 社区维护 crate 和第三方驱动
- Embassy 异步生态（device-envoy 等）
- 开发工具链（espup / espflash / esp-generate / probe-rs）
- HAL 抽象层关系图
- 平台特定注意事项和限制（11 条）
- Cargo.toml 版本锁定最佳实践
- 学习资源（官方 + 社区）

触发条件：需要查找 ESP32 Rust 的库/驱动、选择芯片、对比开发模式。

## 安装方式

### 方式 1：cc-switch 安装

在 cc-switch 仓库管理中添加本仓库：
```
Microbiosis/esp-rust-skills
```

### 方式 2：手动安装

```bash
# 克隆到 Claude Code 技能目录
git clone https://github.com/Microbiosis/esp-rust-skills.git /tmp/esp-rust-skills
cp -r /tmp/esp-rust-skills/esp-rust-development ~/.claude/skills/
cp -r /tmp/esp-rust-skills/esp-rust-ecosystem ~/.claude/skills/
```

### 方式 3：Claude Code 安装

```
/install-skill https://github.com/Microbiosis/esp-rust-skills esp-rust-development
```

## 数据来源

- [The Rust on ESP Book](https://docs.espressif.com/projects/rust/book/preface.html)
- [awesome-esp-rust](https://github.com/esp-rs/awesome-esp-rust)
- [esp-hal 仓库](https://github.com/esp-rs/esp-hal)

## 许可证

MIT
