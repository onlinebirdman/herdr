# CodeBuddy 支持（本 fork 额外添加）

本 fork 在官方 herdr 基础上，增加了对腾讯 **CodeBuddy**（CLI 命令 `codebuddy`）的 agent 检测支持，使 herdr 能识别正在运行的 CodeBuddy 会话，并在侧边栏显示其 working / blocked / idle 状态。

> 说明：官方仓库（herdrdev/herdr）截至本提交尚未合并 CodeBuddy 支持（相关 PR #1064 未合并，feature request 见 #567）。本 fork 自行维护该支持。

## 改动清单

- `src/detect/mod.rs`
  - `Agent` 枚举新增 `Codebuddy` 变体
  - `Agent::ALL`（21→22）与 `Agent::SCREEN_MANIFEST_AGENTS`（19→20）加入该变体
  - `agent_label` → `"codebuddy"`、`interactive_agent_executable` → `"codebuddy"`
  - `lookup_agent` 增加 `"codebuddy"` 映射（识别运行中的 codebuddy 进程）
- `src/detect/manifests/codebuddy.toml`（新增）
  - 基于 `claude.toml` 的屏幕状态检测规则（working / blocked / idle），依据是 codebuddy 与 Claude Code 2.1.220 在终端协议上对齐
- `src/detect/manifest.rs`
  - `BUNDLED_MANIFESTS` 注册 `codebuddy` manifest
- `src/config/sound.rs`
  - 补齐 `for_agent` 对 `Agent::Codebuddy` 的穷尽 match 分支

## 构建要求

herdr 内置的 `libghostty-vt` 需要 **Zig 0.15.2**。官方 CI 在 macOS 上使用 Homebrew 安装：

```bash
brew install zig@0.15
```

然后通过 `ZIG` 环境变量指向它来构建（避免与系统里可能已装的 `zig` 0.16 冲突）：

```bash
ZIG="$(brew --prefix zig@0.15)/bin/zig" cargo build --release
# 产物： ./target/release/herdr
```

## 待校准

`codebuddy.toml` 中的 OSC 标题规则（`osc_title_working` 的 braille 前缀、`osc_title_idle` 的 `✳ ` 前缀）直接沿用自 Claude 的规则，依据是“codebuddy 与 Claude Code 2.1.220 在终端协议上对齐”的推断。需用真实 `codebuddy` 运行时的终端标题校准；若 working / idle 识别不准，调整这两条规则即可。
