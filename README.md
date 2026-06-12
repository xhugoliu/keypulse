# KeyPulse

KeyPulse 是一个面向多种键盘固件的物理按键数据项目。支持接入的键盘固件会把物理按键活动上抛给桌面接收端，用户可以在不读取系统文本输入的前提下，观察打字负载、按键热力图、层使用情况、按住时长和人体工学趋势。

项目目前处于“文档先行”的设计阶段。这个仓库会先把协议、固件边界和桌面端架构写清楚，再开始写代码，避免第一版实现很小的时候就把抽象做歪。

## 范围

第一阶段范围：

- 固件体系：QMK 和 ZMK。
- 桌面平台：macOS 和 Windows。
- 应用外壳：Tauri 桌面应用，Rust 作为后端。
- 首批键盘：X.Tips miniX 和 pskeeb5。
- 核心数据：物理按键事件、层状态、设备身份、transport 健康状态、本地聚合指标。

第一阶段暂不做：

- Linux 打包。
- 移动端应用。
- 云同步。
- 系统级键盘监听。
- 推断或保存用户输入的文本。
- 兼容 `minix-insight` 实验阶段使用的 `KS` 报文。

## 设计目标

- Raw HID 只是 transport 的一种，不是协议本身。
- 核心事件模型不绑定 QMK、ZMK、矩阵形状或 USB 接口类型。
- 为受限固件提供低开销的二进制帧。
- 固件适配器要足够小，可以轻量接入现有 keymap 或模块。
- 所有按键数据默认只保存在本地。
- 设备 profile 负责描述布局、手区、物理位置和展示名称，不把这些写死进应用代码。
- 优先使用显式 capability，而不是让接收端猜。

## 文档索引

- [产品简报](docs/product-brief.md) - 问题、目标、非目标和 MVP 定义。
- [架构](docs/architecture.md) - 系统层次、数据流和组件边界。
- [Protocol v0 草案](docs/protocol-v0.md) - 逻辑事件模型和 transport profile。
- [固件适配器](docs/firmware-adapters.md) - QMK 和 ZMK 的接入策略。
- [桌面应用](docs/desktop-app.md) - 面向 macOS 和 Windows 的 Tauri 接收端设计。
- [设备 Profiles](docs/device-profiles.md) - profile 职责和首批键盘 profile 计划。
- [路线图](docs/roadmap.md) - 分阶段实现计划。
- [参考资料](docs/references.md) - 上游文档和本地实验记录。

## 相关项目

KeyPulse 的设计直接建立在 miniX、pskeeb5 以及它们现有构建链路之上。相关 GitHub 仓库、本地工作区、工具链目录和脚本入口见 [参考资料](docs/references.md)。

## 核心想法

KeyPulse 把三件事拆开：

1. 事件模型：物理键盘活动到底表示什么。
2. Transport：固件如何把事件发送到主机。
3. Profile：如何解释设备的矩阵、位置、手区和布局。

这是最重要的架构决策。这样 QMK 可以使用 Raw HID，ZMK 可以使用更自然的 USB serial 或自定义 HID，桌面应用仍然只消费一条归一化事件流。
