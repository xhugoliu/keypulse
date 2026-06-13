# 设备 Profiles

设备 profile 描述如何解释某把键盘发来的物理事件。它们应该放在应用代码之外，这样添加新键盘时不需要改仪表盘引擎。

## 职责

一个 profile 应定义：

- 稳定的 `profile_id`。
- 人类可读名称。
- 固件体系。
- 预期 transport profile。
- USB 或 serial 匹配线索。
- 已知矩阵大小。
- Position 列表。
- Row/column 映射。
- 视觉布局坐标。
- 手区。
- 拇指区。
- 可选层名称。
- 可选按键标签。

Profile 不应定义：

- 用户私密输入文本。
- 应用级分析策略。
- Transport 解析行为。

## 建议 Profile 形状

这是设计草图，不是最终 schema。

```json
{
  "profile_id": "xtips-minix",
  "version": 1,
  "name": "X.Tips miniX",
  "firmware_family": "qmk",
  "transports": [
    {
      "kind": "raw_hid",
      "vendor_id": "0x5262",
      "product_id": "0x4e4b",
      "usage_page": "0xff60",
      "usage": "0x61"
    }
  ],
  "matrix": {
    "rows": 6,
    "cols": 5
  },
  "positions": [
    {
      "position": 0,
      "row": 0,
      "col": 0,
      "x": 0,
      "y": 0,
      "hand": "left"
    }
  ],
  "layers": [
    { "index": 0, "name": "Base" }
  ]
}
```

## 匹配策略

Profile 匹配应综合多种线索：

- Device hello 中的 `profile_hash32`。
- USB VID/PID。
- HID usage page 和 usage。
- Serial vendor/product 元数据。
- Firmware family。
- 用户手动覆盖。

当自动匹配存在歧义时，应用应该支持手动选择 profile。

## 初始 Profiles

### `xtips-minix`

用途：

- 验证 QMK Raw HID 和第一版桌面仪表盘。

已知形态：

- 分体 3x5 + 3x5 物理布局。
- QMK 矩阵表示为 6 rows x 5 columns。
- 左手 rows：0 到 2。
- 右手 rows：3 到 5。
- `072c` 和 `103c` 本地 `keyboard.json` 使用相同 layout map：左半 rows 0..2、右半 rows 3..5 交错成三行，每行左 5 键 + 右 5 键，总计 30 个物理键。
- USB 匹配线索来自本地实验：VID/PID `0x5262:0x4e4b`，QMK/Vial Raw HID usage page `0xff60`、usage `0x61`。
- 固件声明 `dynamic_keymap.layer_count = 9`，profile 应至少允许 0..8 层命名或保留未命名层。

profile 迁移要求：

- position 应按 layout 顺序稳定编号，而不是简单使用 `row * cols + col` 作为展示顺序。
- row/col 仍应保留，用于调试、向后导入旧 `KS` 数据和验证 QMK hook。

### `pskeeb5`

用途：

- 验证 ZMK 作为一等固件体系。

已知形态：

- 分体键盘。
- ZMK position-first keymap。
- 多层、combos、hold-tap 行为、encoders 和 pointing device 控制。
- 当前 shield 的 `pskeeb5_layouts.dtsi` 定义 10 columns x 4 rows matrix transform。
- physical layout 中有 38 个 key attributes：前三行 30 个主键，第四行 8 个拇指/功能键。
- 右半边 overlay 设置 `col-offset = <5>`，所以主机 profile 要能表达 split half 到全局 position/row/col 的映射。
- 当前用户 keymap 有 6 层，且 combos 使用 ZMK key positions，例如 `<1 11>`、`<13 16>` 等。
- 右半边是 central side，并且带 ZMK Studio、USB CDC ACM、USB HID、mouse/pointing、PS/2 trackpoint 相关配置。

待确认：

- 最终 transport。
- KeyPulse 事件由 central side 统一发送，还是每半边各自发送。
- KeyPulse profile 是否直接复用 ZMK physical layout 坐标，或转换成 KeyPulse 自己的 profile schema。
- serial/RPC/custom HID 中哪个 transport 不会干扰 ZMK Studio 和现有 pointing device。

## X.Tips QMK/Vial 扩展

miniX 跑通后，其他 X.Tips QMK/Vial 型号应按以下步骤加入：

1. 从键盘 layout 元数据创建 profile。
2. 启用共享 QMK adapter。
3. 验证 row/col 到 position 的映射。
4. 确认 Raw HID interface 身份。
5. 检查 flash size 影响。

应用不应该为每个 X.Tips 型号编写定制 UI。
