# 路线图

这份路线图优先保证协议清晰和一个可靠实现，再逐步扩展到更多固件与 transport。

## Phase 0 - 文档和决策

- 定义项目范围。
- 定义 protocol v0 草案。
- 定义 QMK 和 ZMK 适配器边界。
- 定义 Tauri 桌面端架构。
- 定义 profile 职责。

退出标准：

- 仓库在开始写代码之前，已经说明清楚要构建什么。

## Phase 1 - 桌面骨架和协议核心

- 创建 Tauri 桌面项目。
- 增加 Rust 侧 `frame32` 协议解析。
- 增加本地 SQLite schema。
- 增加模拟事件导入或 replay。
- 从模拟数据渲染仪表盘。

退出标准：

- 不接硬件时，应用也能展示 KeyPulse 数据。

## Phase 2 - miniX QMK Raw HID

- 构建可复用 QMK adapter helper。
- 把 miniX 从实验报文迁移到 KeyPulse Protocol v0。
- 增加 `xtips-minix` profile。
- 实现 macOS 和 Windows HID discovery。
- 记录 live miniX events。

退出标准：

- miniX 在 macOS 和 Windows 上端到端工作。

## Phase 3 - QMK 泛化

- 清理 QMK adapter API。
- 文档化如何把 adapter 加入另一把 QMK/Vial 键盘。
- 增加一个额外 X.Tips QMK/Vial profile。
- 验证 flash size 和 Raw HID 行为。

退出标准：

- 第二把 QMK 键盘不需要应用定制代码即可工作。

## Phase 4 - pskeeb5 ZMK 原型

- 创建 ZMK adapter/module 原型。
- 选择 pskeeb5 第一版 transport，大概率是 USB serial。
- 发出 KeyPulse `frame32` 消息。
- 增加 `pskeeb5` profile。
- 记录 live pskeeb5 events。

退出标准：

- pskeeb5 至少在一个桌面系统上端到端工作。

## Phase 5 - ZMK 固化

- 验证 macOS 和 Windows serial 行为。
- 判断 custom HID 是否优于 serial。
- 稳定 ZMK 构建集成方式。
- 文档化分体键盘 source-side 行为。

退出标准：

- ZMK 支持足够可重复，能进入真实日常使用。

## Phase 6 - 产品打磨

- 增加诊断 UI。
- 增加 profile 管理。
- 增加导出流程。
- 增加 installer packaging。
- 在应用内增加隐私说明。
- 为 parser、storage 和 aggregation 增加回归测试。

退出标准：

- KeyPulse 可以作为日常本地桌面工具使用。

## 后续想法

- BLE transport。
- Linux 支持。
- Profile editor。
- Firmware build helper。
- 分层热力图。
- Hold-tap timing analysis。
- 多键盘对比。
- 可匿名分享的 layout research 导出格式。

