# 架构

## 系统形态

```text
键盘固件
  -> 固件适配器
  -> transport profile
  -> KeyPulse frame
  -> 桌面 transport driver
  -> 协议解析器
  -> 归一化事件流
  -> 本地存储
  -> 仪表盘、导出和诊断
```

KeyPulse 把固件侧事件捕获和主机侧解释拆开。桌面应用不应该知道 QMK hook 怎么写，固件也不应该知道 UI 如何渲染热力图。

## 分层

### 固件适配器

适配器运行在键盘固件里，职责是：

- 观察物理按键状态变化。
- 在可用时读取当前层或层状态。
- 把固件特有概念映射为 KeyPulse 字段。
- 通过选定 transport 发出紧凑帧。
- 保持可选，方便从键盘构建中移除。

适配器不应该：

- 在键盘上保存用户分析数据。
- 发送输入文本。
- 依赖某个桌面 UI。
- 假设固定矩阵大小。

### Transport Profile

Transport profile 定义主机通信方式。

第一批 profile：

- `qmk-raw-hid-32`：QMK Raw HID，使用 32 字节 report。
- `zmk-usb-serial-frame32`：ZMK USB serial 候选 profile，承载 32 字节 KeyPulse frame。

未来可以加入自定义 HID endpoint、WebHID 友好封装、BLE GATT 或文件导入。

本地约束说明：miniX/QMK 已证明 Raw HID 可行；pskeeb5/ZMK 当前右半边已有 USB CDC ACM/ZMK Studio RPC 和 PS/2 trackpoint UART 约束，所以 ZMK serial 仍是候选而非已定实现。

### 协议核心

协议核心定义归一化消息：

- 设备 hello。
- 物理按键位置事件。
- 层状态事件。
- 心跳。
- 错误或 capability 报告。

同一个逻辑消息应该不关心它来自 HID 还是 serial。

### 桌面接收端

接收端是 Tauri 应用：

- Rust 后端负责设备发现、transport driver、协议解析、存储和聚合。
- 前端负责可视化、控制、设备选择和导出流程。
- 前后端通信中，请求-响应操作走 command，实时事件流走 event 或 channel。

### 存储

存储只在本地。第一版建议使用 SQLite，因为它跨平台、可检查，也足够承载按键事件这类时间序列数据。

建议表：

- `devices`：已发现设备身份和关联 profile。
- `events`：归一化按键事件。
- `layer_events`：可选的层状态变化。
- `sessions`：应用记录区间和 transport 状态。
- `profiles`：已安装 profile 元数据和版本。

存储层应保留足够原始事实，以便重算派生指标。`minix-insight` 已验证的最小事实集是 host time、device time、sequence、row、col、pressed、layer、keycode；KeyPulse 还需要补充 device/profile/transport 身份、position、source side、layer_state、协议版本和解析状态。

## 数据流

1. 键盘固件观察到按键按下或松开。
2. 适配器构造 KeyPulse frame。
3. Transport 把 frame 发给桌面端。
4. Rust transport driver 接收字节。
5. 解析器校验 magic、版本、消息类型和 payload。
6. 事件归一化层附加主机时间和设备身份。
7. 存储层追加事件。
8. 聚合器更新实时摘要。
9. 前端渲染状态和指标。

## 边界规则

- 固件适配器发送事实，不发送 UI 标签。
- 设备 profile 负责布局解释。
- Transport driver 只处理字节，不计算人体工学指标。
- 协议解析器只验证消息，不承载产品策略。
- 聚合器基于已存储的归一化事件计算派生指标。
- UI 不直接访问 HID 或 serial 设备。

## 目标仓库结构

这是规划目标，不要求立即落地成代码。

```text
keypulse/
  apps/
    desktop-tauri/
  crates/
    keypulse-protocol/
    keypulse-core/
    keypulse-hid/
    keypulse-serial/
    keypulse-storage/
  firmware/
    qmk/
    zmk/
  profiles/
    xtips-minix.json
    pskeeb5.json
  docs/
```

## 早期架构决策

- QMK 和 ZMK 共享一套归一化事件模型。
- 固件到主机使用紧凑二进制 frame。
- QMK Raw HID 是受 32 字节约束的实现 profile。
- ZMK 是一等固件体系，不是 QMK Raw HID 的兼容层；pskeeb5 的第一版 transport 需要在独立 USB CDC ACM、ZMK Studio RPC 和 custom HID 之间验证。
- 键盘 profile 不写死在前端编译产物里。
