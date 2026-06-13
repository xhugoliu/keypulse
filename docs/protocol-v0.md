# KeyPulse Protocol v0 草案

本文描述第一版协议方向。在 miniX/QMK 和 pskeeb5/ZMK 都有可工作的适配器之前，它仍然是草案。

## 协议目标

- 表达物理按键活动。
- 同时适配 QMK 和 ZMK。
- 能放进 32 字节 Raw HID report 这类受限 transport。
- 不采集文本输入。
- 允许桌面应用附加主机侧元数据。
- 从第一天开始对消息做版本化。

## 概念

### Device

发出 KeyPulse frame 的物理键盘或键盘半边。

### Profile

主机侧元数据，用来描述 position 如何映射到视觉布局、手区、标签和可选的固件名称。

### Position

设备 profile 内稳定的物理按键编号。ZMK 天然以 key position 为核心；QMK 可以通过 row/col 和 profile 的矩阵顺序推导 position。

### Matrix Coordinate

固件里的物理行列坐标。它对调试很有用，对 QMK 这类以矩阵坐标作为原生来源的键盘尤其重要。

### Host Time

桌面接收端收到有效 frame 时分配的时间戳。固件不发送 host time。

## 本地实验基线

KeyPulse v0 不是 `minix-insight` 旧协议的重命名。当前 miniX/QMK 实验使用 32 字节 `KS` Raw HID 报文，字段为 row、col、pressed、active layer、QMK time、keycode 和 sequence。它已经证明物理按键事件链路可行，但缺少 v0 需要的 hello、capabilities、profile 匹配、position、source side 和 layer bitmap。

实现 v0 parser 时可以保留旧 `KS` fixture 用于迁移测试或导入测试，但 live protocol 应以 `KP` frame32 为准。

## 消息类型

| Type | 名称 | 用途 |
| --- | --- | --- |
| `0x01` | `device_hello` | 声明设备、固件体系、transport 和 capabilities。 |
| `0x02` | `key_event` | 上报物理按键按下或松开。 |
| `0x03` | `layer_state` | 上报当前层或层 bitmap 变化。 |
| `0x04` | `heartbeat` | 空闲时保持连接新鲜度。 |
| `0x7f` | `error` | 上报固件侧协议或缓冲区错误。 |

如果第一版中层状态可以嵌入 key event，实际实现可以只先支持 `device_hello`、`key_event` 和 `heartbeat`。

## 标准 Key Event 字段

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `sequence` | `u32` | 是 | 尽可能在每次设备启动后单调递增。 |
| `device_time_ms` | `u32` | 是 | 固件单调毫秒计时，允许回绕。 |
| `position` | `u16` | 是 | 当前 profile 中稳定的物理按键位置。 |
| `row` | `u8` | 可选 | 未知时为 `0xff`。 |
| `col` | `u8` | 可选 | 未知时为 `0xff`。 |
| `pressed` | `bool` | 是 | `true` 表示按下，`false` 表示松开。 |
| `active_layer` | `u8` | 可选 | 最高活动层；未知时为 `0xff`。 |
| `layer_state` | `u32` | 可选 | 固件能低成本提供时使用 bitmap。 |
| `keycode` | `u16` | 可选 | 固件原始 keycode；未知时为 `0xffff`。 |
| `source_side` | `u8` | 可选 | unknown、single、left、right、central。 |
| `flags` | `u16` | 是 | 可选状态 bitfield。 |

接收端会附加：

- `host_time_ns`
- `transport_id`
- `device_instance_id`
- `profile_id`
- 解析状态

## Compact Frame32 Envelope

`frame32` 是第一版 wire format。它设计为可以放进 QMK Raw HID 固定 32 字节 report，同时也可以在 serial 上承载。

所有多字节整数都使用 little-endian。

```text
byte 0      magic_0 = 'K'
byte 1      magic_1 = 'P'
byte 2      protocol_major = 0
byte 3      protocol_minor = 1
byte 4      message_type
byte 5      header_flags
byte 6..9   sequence
byte 10..31 message payload
```

Envelope 中的 `sequence` 是 transport 序列号。对于 key event，它应该与 key event 自身的 sequence 一致。其他消息也可以复用同一个计数器。

## Frame32 Key Event Payload

当 `message_type = 0x02`：

```text
byte 10..13 device_time_ms
byte 14..15 position
byte 16     row
byte 17     col
byte 18     pressed
byte 19     active_layer
byte 20..23 layer_state
byte 24..25 keycode
byte 26     source_side
byte 27     event_flags
byte 28..31 reserved
```

v0 中 reserved 字节必须为 0，接收端应忽略它们。

## Frame32 Device Hello Payload

当 `message_type = 0x01`：

```text
byte 10     firmware_family
byte 11     transport_family
byte 12     source_side
byte 13     capability_flags_low
byte 14..17 profile_hash32
byte 18..21 firmware_build_hash32
byte 22..25 device_uid32
byte 26..31 reserved
```

Device hello 不需要携带完整的人类可读名称。接收端通过 `profile_hash32`、USB 身份、serial 身份和用户配置来选择 profile。

## 枚举

### Firmware Family

| Value | 含义 |
| --- | --- |
| `0x00` | unknown |
| `0x01` | QMK |
| `0x02` | ZMK |

### Transport Family

| Value | 含义 |
| --- | --- |
| `0x00` | unknown |
| `0x01` | Raw HID |
| `0x02` | USB serial |
| `0x03` | Custom HID |
| `0x04` | BLE |
| `0x05` | File import |

### Source Side

| Value | 含义 |
| --- | --- |
| `0x00` | unknown |
| `0x01` | single-piece keyboard |
| `0x02` | left half |
| `0x03` | right half |
| `0x04` | central receiver |

## Transport Profiles

### `qmk-raw-hid-32`

- 每个 QMK Raw HID report 承载一个 `frame32`。
- 需要 QMK `RAW_ENABLE = yes`。
- 使用 QMK Raw HID 接口，通常通过 VID、PID、usage page 和 usage ID 过滤。
- 某些主机 API 可能带一个前置 report ID 字节，接收端必须容忍。
- miniX 现有 Vial build 中，`VIA_ENABLE` 会间接打开 `RAW_ENABLE`。KeyPulse adapter 应显式声明自己的依赖，避免依赖 Vial/VIA 的副作用。

### `zmk-usb-serial-frame32`

- 候选方案：通过 USB serial stream 发送 `frame32` 消息。
- stream framing 仍待确认。候选方案包括 COBS、SLIP，或 magic scan + checksum。
- 只有在 macOS 和 Windows 上验证 serial read 足够稳定时，第一版才可以直接使用 32 字节 chunk。
- pskeeb5 当前右半边已经通过 `studio-rpc-usb-uart` 启用 USB CDC ACM 和 ZMK Studio RPC。KeyPulse 不能默认占用同一个 stream，必须先验证独立 CDC ACM、复用 RPC、自定义 HID 三种路径中的哪一种最稳。

## 兼容策略

不承诺兼容 `minix-insight` 里的 `KS` 报文。

v1 之前：

- 允许 breaking change。
- 所有 breaking change 都必须更新本文档。
- 固件适配器和桌面解析器应一起版本化。

v1 之后：

- major version 可以破坏 wire format。
- minor version 必须保留已有消息解析能力。
- 接收端遇到更新的 major version 时，应拒绝并显示明确状态。

## 隐私规则

协议不应该包含：

- 文本字符。
- 应用名称。
- 窗口标题。
- 系统焦点元素数据。
- 输入法文本。

协议可以包含：

- 物理 position。
- 矩阵坐标。
- 固件 keycode。
- 层状态。
- 时间信息。
- 用于 profile 匹配的设备身份。
