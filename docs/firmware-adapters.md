# 固件适配器

KeyPulse 固件适配器是嵌入现有键盘固件的小型集成层。它们的职责是观察物理按键活动，并发出 KeyPulse frame。

## 适配器职责

- 尽量在最接近物理按键位置的地方捕获按下和松开事件。
- 包含固件时间和序列号。
- 在可用时包含 position、矩阵坐标、当前层和原始 keycode。
- 通过选定 transport 发出协议 frame。
- 避免发送文本或主机侧上下文。

## QMK 适配器

### 可行性

QMK 是最容易落地的第一目标。QMK Raw HID 提供了通过 HID interface 与主机双向通信的能力，并且 report 是固定 32 字节 buffer，很适合 `frame32` 草案。

### 初始集成形态

适配器应提供类似能力：

```text
keypulse_init()
keypulse_send_hello()
keypulse_send_key_event(keycode, record)
keypulse_task()
```

预期 keymap 集成方式：

```text
pre_process_record_user(...)
  -> keypulse_send_key_event(...)
```

具体 C API 可以在实现时调整。重要边界是：一把键盘的 keymap 最好只需要一个 include、一个 rules/config 开关、一个 hook 调用。

### QMK 数据映射

| KeyPulse 字段 | QMK 来源 |
| --- | --- |
| `row` | `record->event.key.row` |
| `col` | `record->event.key.col` |
| `pressed` | `record->event.pressed` |
| `device_time_ms` | `timer_read32()` |
| `active_layer` | `get_highest_layer(layer_state)` |
| `layer_state` | 可用时取 `layer_state` |
| `keycode` | hook 参数 |
| `position` | profile 根据 row/col 查表 |

### miniX 现有实验约束

当前 `QMK-Keyboard/minix/keymaps/vial/keymap.c` 已经在 `pre_process_record_user` 中发送旧 `KS` Raw HID 报文。这证明 hook 位置和 Raw HID 链路可用，但它不是 KeyPulse adapter 的最终形态。

迁移到 KeyPulse 时需要显式处理：

- 旧 `KS` 报文没有 hello、capabilities、profile hash、position、source side 或 `layer_state` bitmap。
- 当前 miniX 只发送 row/col，position 应由 `xtips-minix` profile 从 QMK layout 顺序推导。
- miniX keymap 依赖 `VIA_ENABLE` 间接启用 `RAW_ENABLE`；共享 adapter 应提供明确的 build 开关，避免只在 Vial keymap 中可用。
- `raw_hid_send()` 和 Vial/VIA 共用 Raw HID interface，host 侧必须把 Vial/WebHID 占用视为正常状态，而不是协议错误。
- 旧 `KS` 报文可以作为本地 replay/fixture 来源，但 KeyPulse v0 不应承诺在线兼容。

### 首批 QMK 目标

- X.Tips miniX。
- miniX profile 和 helper 稳定后，扩展到其他 X.Tips QMK/Vial 型号。

### QMK 风险

- Raw HID 可能被其他工具使用，或与 Vial/WebHID 使用方式产生接口占用冲突。
- 某些键盘 flash 空间可能紧张。
- 分体键盘需要清楚定义 source-side 策略。
- 矩阵坐标不一定等于用户看到的视觉顺序，必须由 profile 负责映射。

## ZMK 适配器

### 可行性

ZMK 是一等目标，但不应被强行塞进 QMK Raw HID 的形状里。适配器应该使用 ZMK 的事件模型观察 key position 和 layer activity，再通过 macOS/Windows 上最可靠的 transport 发出同一套 KeyPulse 协议。

### 初始 Transport 方向

第一版优先 transport：

- USB serial 承载 `frame32` 消息。

原因：

- ZMK 键盘经常有不同于 QMK 的主机通信假设。
- serial 在 bring-up 阶段更容易检查。
- 它避免把整个协议绑定到 Raw HID。

如果 serial 对发布版太别扭，再评估 custom HID。

### pskeeb5 现有本地约束

`zmk-config-pskeeb5` 当前没有 KeyPulse adapter，但已经暴露出第一版实现必须尊重的边界：

- build target 是 `nice_nano_v2` + `pskeeb5_left` / `pskeeb5_right`。
- 右半边是 ZMK split central side；左半边是 BLE split peripheral。
- 右半边 build 使用 `studio-rpc-usb-uart` snippet，生成配置启用 USB CDC ACM、ZMK Studio RPC、USB HID、serial、mouse/pointing。
- `pskeeb5_right.overlay` 已经用 UART driver 接 PS/2 trackpoint；不要假设硬件 UART 空闲。
- `pskeeb5_layouts.dtsi` 定义 10x4 transform 和 38 个 physical positions，右半边 `col-offset = <5>`。
- 当前用户 keymap 有 6 层、combos、conditional layer、hold-tap/layer-tap、encoders 和 pointing device 行为；KeyPulse v0 应先记录物理 position 事件和 layer 状态，不要试图在固件端解释这些高级行为。

因此，ZMK 原型的第一个问题不是“serial 能不能发 32 字节”，而是“KeyPulse frame 应走独立 USB CDC ACM、复用 Studio RPC transport、还是改成 custom HID”。在没有验证前，`zmk-usb-serial-frame32` 只能视为候选 transport。

### ZMK 数据映射

| KeyPulse 字段 | ZMK 来源 |
| --- | --- |
| `position` | ZMK key position event |
| `pressed` | position state |
| `device_time_ms` | uptime 或 kernel timer |
| `active_layer` | layer state event 或 helper |
| `layer_state` | 可访问时取 layer state bitmap |
| `keycode` | 可用时取 keycode state event |
| `row` / `col` | profile 根据 position 查表 |
| `source_side` | 构建期 side 配置 |

### 首批 ZMK 目标

- pskeeb5。

### ZMK 风险

- 最佳 transport 可能需要自定义 ZMK module。
- USB serial 行为必须在 macOS 和 Windows 上验证。
- BLE 不属于 v0 范围。
- 分体键盘归属需要谨慎：可以由每半边各自发事件，也可以由 central side 为两边发归一化事件。

## Capability Flags

适配器应该报告 capabilities，而不是让主机猜。

候选 flags：

- 有 matrix row/col。
- 有稳定 position。
- 有 active layer。
- 有 layer bitmap。
- 有 raw keycode。
- 有 source side。
- 有 heartbeat。
- 支持 host commands。

## Host Commands

MVP 不要求 host-to-device commands。未来可能加入：

- 请求 hello。
- 设置按键数据开关。
- 设置采样模式。
- 请求固件版本。
- 设置 heartbeat interval。

在真正需要之前，适配器可以先作为单向事件发射器。
