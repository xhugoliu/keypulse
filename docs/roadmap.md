# 路线图

这份路线图优先保证协议清晰和一个可靠实现，再逐步扩展到更多固件与 transport。

## Phase 0 - 文档和决策

- 定义项目范围。
- 定义 protocol v0 草案。
- 定义 QMK 和 ZMK 适配器边界。
- 定义 Tauri 桌面端架构。
- 定义 profile 职责。
- 对齐本地真实约束：miniX/QMK `KS` 实验报文、`minix-insight` HID/SQLite/聚合实现、pskeeb5/ZMK split central 与 USB CDC ACM/ZMK Studio 状态。

退出标准：

- 仓库在开始写代码之前，已经说明清楚要构建什么。
- 下一阶段的模拟数据能够覆盖旧 miniX 事件、KeyPulse v0 frame、profile 映射和按住时长聚合。

## Phase 1 - 桌面骨架和协议核心

- 创建 Tauri 桌面项目。
- 增加 Rust 侧 `frame32` 协议解析。
- 增加本地 SQLite schema。
- 增加模拟事件导入或 replay，至少覆盖：
  - KeyPulse `KP` frame32。
  - 旧 miniX `KS` CSV/fixture 导入。
  - macOS HID 前置 report ID 偏移。
  - miniX row/col 到 position 映射。
- 从模拟数据渲染仪表盘。
- 先复刻 `minix-insight` 已验证的指标：今日 press、7 日趋势、按住时长、active keys、左右手负载和热力图。

退出标准：

- 不接硬件时，应用也能展示 KeyPulse 数据。
- parser、storage 和 aggregation 有回归测试，能处理 `u32` device time 回绕和设备重启导致的异常长按。

## Phase 2 - miniX QMK Raw HID

- 构建可复用 QMK adapter helper。
- 把 miniX 从实验 `KS` 报文迁移到 KeyPulse Protocol v0。
- 增加 `xtips-minix` profile。
- 实现 macOS 和 Windows HID discovery。
- 记录 live miniX events。
- 明确 adapter build 开关，不依赖 `VIA_ENABLE` 间接启用 `RAW_ENABLE`。
- 验证 Vial/WebHID 占用 Raw HID 时的 UI 状态。

退出标准：

- miniX 在 macOS 和 Windows 上端到端工作。
- 同一 miniX profile 能解释 072c 和 103c 的共享 layout。

## Phase 3 - QMK 泛化

- 清理 QMK adapter API。
- 文档化如何把 adapter 加入另一把 QMK/Vial 键盘。
- 增加一个额外 X.Tips QMK/Vial profile。
- 验证 flash size 和 Raw HID 行为。

退出标准：

- 第二把 QMK 键盘不需要应用定制代码即可工作。

## Phase 4 - pskeeb5 ZMK 原型

- 创建 ZMK adapter/module 原型。
- 选择 pskeeb5 第一版 transport：独立 USB CDC ACM、复用 ZMK Studio RPC 或 custom HID 三选一，不能默认占用现有 Studio RPC stream。
- 发出 KeyPulse `frame32` 消息。
- 增加 `pskeeb5` profile。
- 记录 live pskeeb5 events。
- 优先验证右半边 central side 发事件，因为当前本地配置中右半边是 split central 且已有 USB CDC ACM/ZMK Studio。
- 验证 position、source side、layer state 和 profile 映射，不在固件端解释 hold-tap、combos 或 pointing 行为。

退出标准：

- pskeeb5 至少在一个桌面系统上端到端工作。
- 不破坏现有 ZMK Studio、split BLE、encoder 和 trackpoint 行为。

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
