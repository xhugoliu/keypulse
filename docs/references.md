# 参考资料

KeyPulse 从本地实验和上游固件/应用平台能力出发。

## 本地先验

### `minix-insight`

GitHub: https://github.com/xhugoliu/minix-insight

`minix-insight` 验证了第一条端到端链路：

- QMK miniX 固件发出物理按键事件。
- macOS 从 vendor-defined HID interface 接收事件。
- 事件写入 SQLite。
- 菜单栏应用展示摘要和热力图。

KeyPulse 应复用这些经验，但不复用旧报文格式。

本地代码约束：

- `Sources/MinixInsightCore/TelemetryPacket.swift` 解析 32 字节 `KS` 报文，同时容忍 macOS HID 回调里多出的前置 report ID 字节。
- `Sources/MinixInsightCore/TelemetryCollector.swift` 使用 IOHIDManager 匹配 miniX 的 VID/PID `0x5262:0x4e4b`、usage page `0xff60`、usage `0x61`。
- 同一 Raw HID interface 会被 Vial 或 WebHID 页面占用，实验应用已经把 open 失败显示为 `Raw HID busy`。
- `Sources/MinixInsightCore/SQLiteDatabase.swift` 当前存储字段为 `host_time_iso`、`host_time_ns`、`qmk_time_ms`、`seq`、`row`、`col`、`pressed`、`layer`、`keycode`。
- `Sources/MinixInsightCore/TelemetrySummary.swift` 通过同一 row/col 的 down/up 事件配对计算按住时长，使用 QMK 毫秒计时，并丢弃超过 12 小时的单次长按来规避设备重启或计时异常。

### `QMK-Keyboard`

GitHub: https://github.com/xhugoliu/QMK-Keyboard

`xhugoliu/QMK-Keyboard` 的 `minix-key-stats` 分支包含当前 miniX 实验：

- 从 `pre_process_record_user` 发出 Raw HID packet。
- Python HID logger。
- CSV summary script。
- 本地 QMK build helper。

本地代码约束：

- `minix/keymaps/vial/keymap.c` 直接 `#include "raw_hid.h"`，在 `pre_process_record_user` 中调用 `raw_hid_send()`。
- 现有报文是旧 `KS` 格式：magic、version、type、row、col、pressed、active layer、`timer_read32()`、keycode、sequence。它没有 `device_hello`、capabilities、profile hash、position、source side 或 layer bitmap。
- `minix/keymaps/vial/rules.mk` 只显式设置 `VIA_ENABLE = yes`、`VIAL_ENABLE = yes`、`VIAL_INSECURE = yes`。在本地 `vial-qmk/builddefs/common_features.mk` 中，`VIA_ENABLE` 会把 `RAW_ENABLE` 置为 yes；KeyPulse 适配器不应把这个间接关系当作唯一前提。
- miniX `072c` 和 `103c` 的 `keyboard.json` 都定义同一套 6 rows x 5 columns 分体矩阵、VID/PID `0x5262:0x4e4b` 和 `dynamic_keymap.layer_count = 9`。

### `zmk-config-pskeeb5`

GitHub: https://github.com/xhugoliu/zmk-config-pskeeb5

`zmk-config-pskeeb5` 是第一个 ZMK 目标。它有价值是因为它包含：

- Position-first ZMK keymaps。
- Hold-tap 和 layer-tap 行为。
- Combos。
- Encoders。
- Trackpoint 和 mouse 相关控制。
- 真实分体键盘配置。

本地代码约束：

- `config/pskeeb5.keymap` 是当前用户 keymap 覆盖，包含 6 层、hold-tap、layer-tap、combos、conditional layer、双 encoder 绑定和 mouse/pointing 行为。
- 实际 shield 文件在 `.zmk/workspace/zmk/app/boards/shields/pskeeb5/`。`pskeeb5_layouts.dtsi` 定义 10 columns x 4 rows 的 matrix transform，以及 38 个 physical-layout key attributes。
- `pskeeb5_right.overlay` 对 `default_transform` 设置 `col-offset = <5>`；`Kconfig.defconfig` 把右半边设为 `ZMK_SPLIT_ROLE_CENTRAL`。因此第一版如果只选一个 ZMK 发射端，应优先验证右半边/central side。
- 本地 build matrix 和 `scripts/local-zmk-build.sh` 都对右半边使用 `-S studio-rpc-usb-uart`；生成配置显示右半边启用 USB CDC ACM、serial、ZMK Studio RPC、USB HID、mouse/pointing，左半边是 split peripheral 且未启用 serial。
- 右半边已有 PS/2 trackpoint 通过 UART driver 接入，ZMK Studio 也占用 USB UART/RPC 语义。KeyPulse 不能默认把现有 `studio-rpc-usb-uart` 当作空闲 byte stream。

## 相关项目与本地环境

为了避免新会话只记得 GitHub 仓库、却忘记本地构建链路，这里把相关对象按角色拆开。

### 源码仓库

- `keypulse`
  GitHub: https://github.com/xhugoliu/keypulse
  角色：当前主仓库，承载协议、桌面应用、固件适配器和 profile 设计。

- `QMK-Keyboard`
  GitHub: https://github.com/xhugoliu/QMK-Keyboard
  本地路径：`/Users/xhugoliu/Projects/QMK-Keyboard`
  角色：X.Tips QMK/Vial 固件源码仓库。miniX 以及后续其他 X.Tips QMK 键盘的源文件以这里为准。

- `minix-insight`
  GitHub: https://github.com/xhugoliu/minix-insight
  本地路径：`/Users/xhugoliu/Projects/minix-insight`
  角色：miniX 的早期实验接收端，用来验证物理按键数据从固件到本地 SQLite 和 UI 的整条链路。

- `zmk-config-pskeeb5`
  GitHub: https://github.com/xhugoliu/zmk-config-pskeeb5
  本地路径：`/Users/xhugoliu/Projects/zmk-config-pskeeb5`
  角色：pskeeb5 的 ZMK 配置仓库，也是第一批 ZMK 接入目标。

### 本地构建工作区

- `vial-qmk`
  GitHub: https://github.com/vial-kb/vial-qmk
  本地路径：`/Users/xhugoliu/Projects/vial-qmk`
  角色：QMK/Vial 构建工作树，不是 miniX 源码真身。`QMK-Keyboard/minix` 会被临时复制到 `vial-qmk/keyboards/xtips/minix` 再执行编译。
  备注：当前本地工作树存在未提交的 `keyboards/xtips/`，并且分支相对 `origin/vial` 落后 1 个提交。

### 本地工具与工具链

- `QMK CLI venv`
  本地路径：`/Users/xhugoliu/Projects/.qmk-venv`
  角色：本地 `qmk` 命令所在的 Python 虚拟环境。`QMK-Keyboard/scripts/build_minix_103c.sh` 默认从这里调用 `qmk`。

- `ARM GNU Toolchain`
  本地路径：`/Users/xhugoliu/Projects/toolchains`
  角色：QMK 构建 miniX 时使用的 ARM 工具链目录。默认路径由 `QMK-Keyboard/scripts/build_minix_103c.sh` 指向其中的 `arm-none-eabi/bin`。

### ZMK 本地状态与工作目录

- `zmk-config-pskeeb5/.zmk`
  本地路径：`/Users/xhugoliu/Projects/zmk-config-pskeeb5/.zmk`
  角色：pskeeb5 本地 ZMK 状态目录，不是源码仓库本体。里面承载：
  - `workspace/`：west 初始化后的 ZMK/Zephyr 工作区。
  - `venv/`：本地 Python 虚拟环境。
  - `toolchains/`：Zephyr SDK 和 arm-zephyr-eabi 工具链。
  - `downloads/`：bootstrap 下载的 SDK 压缩包。
  - `bin/`：本地辅助脚本，例如 `wget` shim。

### 关键脚本入口

- `QMK-Keyboard/scripts/build_minix_103c.sh`
  角色：miniX 的本地 QMK 构建入口。负责同步源码到 `vial-qmk`、调用 `qmk compile`、复制产物到 `QMK-Keyboard/dist/`。

- `zmk-config-pskeeb5/scripts/local-zmk-env.sh`
  角色：pskeeb5 本地 ZMK 环境变量入口，定义 `ZMK_STATE_DIR`、`ZMK_WORKSPACE`、`ZMK_VENV`、`ZEPHYR_SDK_INSTALL_DIR`。

- `zmk-config-pskeeb5/scripts/local-zmk-bootstrap.sh`
  角色：pskeeb5 本地 ZMK/Zephyr 初始化入口。负责创建 `.zmk` 状态目录、安装 `west/cmake/ninja`、初始化 west 工作区、下载并安装 Zephyr SDK。

- `zmk-config-pskeeb5/scripts/local-zmk-build.sh`
  角色：pskeeb5 本地构建入口。负责在 `.zmk/workspace` 中执行 `west build`，并把左右半边和 reset 固件复制到 `zmk-config-pskeeb5/firmware/`。

## miniX / pskeeb5 链路摘要

- miniX：
  源码在 `QMK-Keyboard`，构建在 `vial-qmk`，命令来自 `.qmk-venv`，编译依赖 `toolchains`，实验接收端经验来自 `minix-insight`。

- pskeeb5：
  源码在 `zmk-config-pskeeb5`，本地工作区和 SDK 状态放在 `.zmk/`，构建通过 `west build` 完成，产物输出到 `firmware/`。

## 上游参考

- QMK Raw HID: https://docs.qmk.fm/features/rawhid
- ZMK new behavior and event development: https://zmk.dev/docs/development/new-behavior
- Tauri calling Rust from the frontend: https://v2.tauri.app/develop/calling-rust/
- Tauri calling the frontend from Rust: https://v2.tauri.app/develop/calling-frontend/
- Rust `hidapi`: https://docs.rs/hidapi/latest/hidapi/

## 从参考资料得到的设计提示

- QMK Raw HID 使用固定大小 report，目前是 32 字节，这强烈影响第一版二进制 frame。
- QMK Raw HID 暴露默认 usage page 和 usage，桌面应用可以用这些信息过滤设备。
- Tauri 的 Rust 后端适合承担 HID、serial、storage 和 parser 职责。
- ZMK 应被视为独立固件环境，而不是 QMK Raw HID 的克隆。
