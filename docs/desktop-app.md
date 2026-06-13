# 桌面应用

桌面应用是面向 macOS 和 Windows 的 Tauri 应用。它是 KeyPulse 键盘的本地接收端、仪表盘和导出工具。

## 职责

- 发现受支持的设备。
- 打开 transport interface。
- 解析 KeyPulse frame。
- 附加主机时间戳。
- 将事件匹配到设备 profile。
- 在本地保存事件。
- 计算实时和历史摘要。
- 清楚展示 transport 和协议状态。
- 导出原始事件和统计摘要。

## 非职责

- 监听操作系统键盘 hook。
- 读取文本输入。
- 充当固件配置器。
- 默认上传按键数据。
- 负责固件构建流程。

## 后端服务

### Device Manager

跟踪已发现设备及其状态：

- Missing。
- Connected。
- Recording。
- Paused。
- Interface busy。
- Unsupported protocol。
- Missing profile。

### Transport Drivers

第一批 driver：

- HID driver，对应 `qmk-raw-hid-32`。
- Serial driver，对应 `zmk-usb-serial-frame32`。

Driver 应向协议解析器暴露归一化的字节流或 frame 流。

本地实验提示：

- macOS HID 回调可能给 32 字节 Raw HID report 加一个前置 report ID 字节，HID driver 应在 transport 层容忍 32 或 33 字节输入。
- miniX 现有匹配线索是 VID/PID `0x5262:0x4e4b`、usage page `0xff60`、usage `0x61`。
- Vial 或 WebHID 占用同一 Raw HID interface 时，打开失败应显示为 interface busy。
- pskeeb5 右半边当前生成 USB CDC ACM 是为了 ZMK Studio RPC；Serial driver 必须区分 KeyPulse stream、Studio RPC stream 和普通 debug/logging serial。

### Protocol Parser

校验内容：

- Magic bytes。
- major/minor version。
- message type。
- reserved byte 约束。
- 已知 payload 形状。

无效 frame 应计数并在诊断里展示，不应静默当作按键事件。

### Event Normalizer

附加主机侧字段：

- 主机时间戳。
- 设备实例。
- Transport 身份。
- Profile 身份。
- 解析状态。

它也负责在 profile 能帮忙时补齐可选字段，例如从 position 映射 row/col，或从 row/col 映射 position。

### Storage

SQLite 是第一版预期存储后端。

优先保存原始归一化事实。派生摘要可以重算，除非后续性能证明需要缓存。

`minix-insight` 已验证的最小事件事实包括 host time、device time、sequence、row、col、pressed、layer 和 keycode。KeyPulse schema 应在此基础上增加 device/profile/transport 身份、position、source side、layer_state、parse status 和 protocol version。

### Aggregator

计算：

- 按下次数。
- 按住时长。
- 当前 active keys。
- 每日 totals。
- 按 position 的热力图。
- 层使用情况。
- 左右手负载。
- 拇指键负载。
- 高频位置。

按住时长的第一版算法可以沿用实验项目的策略：同一 device/profile/position 的 down/up 配对，使用固件单调时间计算 elapsed，并显式处理 `u32` 回绕和设备重启后的离谱长按。

## 前端视图

MVP 视图：

- 设备状态。
- 记录控制。
- 今日摘要。
- 7 日趋势。
- 物理热力图。
- Top keys 或 top positions。
- 导出动作。
- 诊断面板。

后续视图：

- 多设备对比。
- 分层热力图。
- Hold-tap 行为概览。
- Session 对比。
- Profile 编辑器。

## macOS 注意事项

- HID 访问可能需要用户关闭 Vial、WebHID 页面或其他已经打开同一接口的工具。
- 应用不应为了按键数据记录请求 Accessibility 权限。
- Serial 设备选择应展示足够身份信息，以便区分键盘端口和其他 serial 设备。

## Windows 注意事项

- HID 和 serial 身份匹配要保守。
- Interface busy 和 driver permission 错误需要一等 UI 提示。
- 正常用户安装不应要求开发工具。

## 前后端契约

请求-响应 commands 用于：

- 列出设备。
- 开始或暂停记录。
- 选择 profile。
- 导出数据。
- 打开数据库位置。
- 二次确认后清空本地数据。

流式 event 或 channel 用于：

- 实时按键事件。
- 设备状态变化。
- 聚合后的仪表盘更新。
- 诊断计数器。

## 数据安全

应用应该明显展示这些状态：

- 正在记录。
- 记录已暂停。
- 已连接但没有可用的按键数据输入。
- 正在本地保存。
- 导出完成。

任何 UI 都不应该暗示 KeyPulse 会采集用户输入文本。
