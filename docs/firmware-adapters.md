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
