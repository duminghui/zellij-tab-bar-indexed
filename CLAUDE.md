# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

本仓库是 zellij 内置 `default-plugins` 中 `tab-bar` 与 `compact-bar` 两个插件的抽取版（fork），在上游 [zellij-org/zellij](https://github.com/zellij-org/zellij) 源码之上叠加两项相互独立的定制：

1. `show_tab_indices`：bool 配置项，在 tab 标签前渲染 `<1>`、`<2>` 等序号前缀（上游 PR zellij-org/zellij#4606 的先行版）。
2. Permissions handshake：以 `request_permission(&[ReadApplicationState, ChangeApplicationState])` 取代启动时的 `set_selectable(false)`，并订阅 `EventType::PermissionRequestResult` 处理 Granted/Denied 分支——文件型插件读取应用状态需用户在 zellij 内手动授权（README 的 First Run 流程）。

核心约束：除这两项定制外，`*/src` 应与上游 `default-plugins` 对应版本逐字一致。直接改动上游代码会给下次 rebase 制造冲突；任何源码改动都应能归入上述两项定制之一。

## 常用命令

```bash
rustup target add wasm32-wasip1   # 一次性准备 target
cargo check                       # 快速编译检查
cargo build --release             # 发布构建
```

Rust 工具链位于 `/root/.local/share/{cargo,rustup}`（`CARGO_HOME`/`RUSTUP_HOME` 非默认路径）。正常情况下 cargo 直接可用（环境继承自用户 shell）；若会话环境的 PATH 中没有 `/root/.local/share/cargo/bin`（Claude Code 未从配置好的 shell 启动），调用前需显式注入：

```bash
export CARGO_HOME=/root/.local/share/cargo RUSTUP_HOME=/root/.local/share/rustup PATH="/root/.local/share/cargo/bin:$PATH"
```

- `.cargo/config.toml` 已设默认 target 为 `wasm32-wasip1`，无需显式 `--target`。
- 产物位于 `target/wasm32-wasip1/release/`，文件名为 `tab-bar.wasm` / `compact-bar.wasm`（连字符）。
- 仓库内没有测试。验证手段 = 编译零 warning + 与上游 tag 做 `diff -ruN` 核对（见下）。
- `Cargo.lock` 已 gitignore，不提交。
- 两个插件 `Cargo.toml` 故意不写 `[lib]` 段——它们是面向 wasm32-wasip1 的二进制 crate，不要「补全」。

## 架构

Cargo workspace，成员 `tab-bar`（顶部丝带样式 tab 栏）与 `compact-bar`（底部单行：模式指示 + tab 栏，额外支持 tooltip）。两者依赖 `zellij-tile` / `zellij-tile-utils`，版本号跟随上游 zellij release（当前 0.44）。

两插件内部同构：

- `main.rs`：`State` 实现 `ZellijPlugin` 生命周期——`load`（解析配置、request_permission、subscribe）→ `update`（按事件更新状态、返回是否重绘）→ `render`（拼 ANSI 输出到单行）。鼠标点击/滚动切 tab 依赖 `LinePart.len` 累计宽度做命中测试。
- `line.rs`：整行布局——按可用宽度装填 tab、活动 tab 居中、放不下时两侧折叠为 `← +N` / `+N →`。
- `tab.rs`：单个 tab 的样式（`tab_style` → `render_tab`）与点击命中（`get_tab_to_focus`）。

compact-bar 独有：`tooltip.rs`（浮动 tooltip 渲染）、`keybind_utils.rs` + `action_types.rs`（从 keymap 反查按键提示）、`clipboard_utils.rs`（复制状态提示文案）。tooltip 机制：主实例通过 `MessageToPlugin` + `zellij:OWN_URL` 以 `is_tooltip true` 配置把自身再实例化为浮动 pane，两实例间靠 `pipe` 私有消息通信（toggle / persist / 随 mode 变化挪坐标）。注意 zellij 限制：`tooltip` 配置项对文件型插件无效（README 有载明）。

两项定制在源码中的落点（两插件对称）：

- `show_tab_indices`：`main.rs` 的 `State.show_tab_indices` 解析与渲染循环传参；`tab.rs` 的 `render_tab`/`tab_style` 增加 `tab_index: Option<usize>` 参数与 `<N>` 前缀构建（`index_color` 用 emphasis 色）。
- permissions handshake：`load`/`setup_subscriptions` 的 `request_permission` 与订阅列表；`update` 的 `PermissionRequestResult` 分支——Granted 后 `set_selectable(false)` 并重新订阅，Denied 打日志。compact-bar 按 `is_tooltip` 拆分订阅逻辑，多一个 `resubscribe_events`。
- 关键交汇点：Granted 后的重新订阅必须包含 `EventType::InitialKeybinds`。这是 fork patch 与上游改动唯一重叠之处，遗漏会在冷启动时静默破坏 Locked 模式的 keybind 显示。

## 维护工作流：rebase on upstream

本仓库的常规维护操作是把两项定制重新套到新版上游 zellij 之上（用户说「rebase on upstream」「升级到 zellij vX.Y.Z」等即触发），完整流程见 `.claude/skills/rebase-on-upstream-zellij/SKILL.md`。要点：

- 策略为 snapshot + reapply：整体覆盖上游新版 `default-plugins` 源码与 Cargo.toml，再把两项定制作为独立 commit 逐一套回。只 bump `zellij-tile` 版本不可行——上游 patch release 常改插件源码。
- 辅助脚本 `scripts/three_way_diff.sh <old_tag> <new_tag>`（位于该 skill 目录）克隆两个上游 tag 到 `/tmp` 并预生成 fork patch 与 upstream patch。
- 上一个 squash commit（如 `b69aaff` rebase customizations on zellij v0.44.3）的 diff 是定制的权威参照。
- 收尾校验：`diff -ruN <upstream-new>/default-plugins/<plugin>/src <plugin>/src` 的输出必须恰好只剩两项定制；compact-bar `tab.rs` 的格式漂移（操作数顺序、单行/多行调用）是常见偏差来源。
- `zellij-tile` 新版若未上 crates.io，回退为 git 依赖指向 tag。

发布：push tag `v*.*.*` 或 `v*.*.*-*` 触发 CI 构建并把两个 `.wasm` + sha256sum 附到 GitHub Release。tag 号与上游 zellij 版本对齐（如 `v0.44.3`），同版迭代加 `-N` 后缀（如 `v0.44.0-1`）。origin 为 `duminghui/zellij-tab-bar-indexed`。
