# AGENTS.md

本文件是给进入 `keypulse` 仓库工作的 agent 看的快速入口，不是给人读的完整项目说明。

## 项目定位

- `keypulse` 是一个面向多种键盘固件的物理按键数据项目。
- 当前阶段以文档和架构设计为主，代码尚未开始。
- 第一阶段目标：QMK + ZMK，桌面端为 macOS + Windows，应用外壳为 Tauri。
- 第一批键盘：X.Tips miniX 和 pskeeb5。
- 不兼容 `minix-insight` 里的旧 `KS` 报文。

## 首先读哪里

- [README.md](/Users/xhugoliu/Projects/keypulse/README.md)
  作用：给人快速理解项目目标和文档入口。
- [docs/product-brief.md](/Users/xhugoliu/Projects/keypulse/docs/product-brief.md)
  作用：产品目标、非目标、MVP。
- [docs/architecture.md](/Users/xhugoliu/Projects/keypulse/docs/architecture.md)
  作用：系统分层和边界。
- [docs/protocol-v0.md](/Users/xhugoliu/Projects/keypulse/docs/protocol-v0.md)
  作用：KeyPulse 协议草案。
- [docs/references.md](/Users/xhugoliu/Projects/keypulse/docs/references.md)
  作用：相关源码仓库、本地构建工作区、工具链目录和脚本入口。

## 术语约定

- 面向用户与文档正文时，优先使用：
  - `按键数据`
  - `物理按键数据`
  - `按键事件`
  - `事件流`
- 避免优先使用：
  - `遥测`
  - `telemetry`

## 当前上下文

- miniX 的源码真身在 `QMK-Keyboard`，不是 `vial-qmk`。
- `vial-qmk` 是本地 QMK/Vial 构建工作树。
- pskeeb5 的源码真身在 `zmk-config-pskeeb5`。
- pskeeb5 的本地 ZMK/Zephyr 工作区与工具链状态在 `zmk-config-pskeeb5/.zmk/`。
- 如果需要回看这些链路，优先读 `docs/references.md`，不要把这些细节重新塞回 `README.md`。

## 入口分层

- `README.md`
  面向人类读者，应保持精简。
- `AGENTS.md`
  面向 agent，应保持精简，但要提供足够快的上下文切入点。
- `docs/*.md`
  放详细设计、链路地图和参考资料。
