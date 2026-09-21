# CodeBuddy 支持（本 fork 额外添加）

本 fork 在官方 herdr 基础上，增加了对腾讯 **CodeBuddy**（CLI 命令 `codebuddy`）的 agent 检测支持，使 herdr 能识别正在运行的 CodeBuddy 会话，并在侧边栏显示其 working / blocked / idle 状态。

> 说明：官方仓库（herdrdev/herdr）截至本提交尚未合并 CodeBuddy 支持（相关 PR #1064 未合并，feature request 见 #567）。本 fork 自行维护该支持。

## 改动清单

- `src/detect/mod.rs`
  - `Agent` 枚举新增 `Codebuddy` 变体
  - `Agent::ALL` 与 `Agent::SCREEN_MANIFEST_AGENTS` 数组加入该变体（常量长度同步）。当前值：`ALL` = 25、`SCREEN_MANIFEST_AGENTS` = 23（上游另有 `Letta`、`Muse`）
  - `agent_label` → `"codebuddy"`、`interactive_agent_executable` → `"codebuddy"`
  - `lookup_agent` 增加 `"codebuddy"` 映射（识别运行中的 codebuddy 进程）
- `src/detect/manifests/codebuddy.toml`（新增）
  - working / blocked / idle 的 OSC 标题与屏幕状态检测规则
- `src/detect/manifest.rs`
  - `BUNDLED_MANIFESTS` 注册 `codebuddy` manifest
- `src/config/sound.rs`
  - 补齐 `for_agent` 对 `Agent::Codebuddy` 的穷尽 match 分支
- `distribution/agent-detection/codebuddy.toml` + `index.toml`
  - 上游新增了发布目录校验：凡 `BUNDLED_MANIFESTS` 里的 agent 都必须出现在 `distribution/agent-detection/`，否则 `just release-docs-check`（`scripts/agent_detection_manifest_check.py --require-all-published`）会失败。codebuddy 采用**正式发布**而非 `UNPUBLISHED_BUNDLED_MANIFESTS` 的 version+sha256 例外，因为例外一旦改动 manifest 就会失效。发布副本必须与 bundled 版本一致且**逐字节相同**，改动 `src/detect/manifests/codebuddy.toml` 后需同步 `cp` 过去。

## 合并上游的注意点

上游每个版本都会新增 agent 变体（本轮是 `Letta`、`Muse`），`Agent::ALL` / `SCREEN_MANIFEST_AGENTS` 的长度常量会因此再次错位。`git merge upstream/master` 本身无冲突，但合并后这两个常量需要手工修正，否则编译失败。

## 构建要求

herdr 内置的 `libghostty-vt` 需要 **Zig 0.16.0**（v0.9.1 起从 0.15.2 升级）。系统里的 `zig` 若已是 0.16，直接构建即可：

```bash
cargo build --release
# 产物： ./target/release/herdr
```

若 PATH 上的 `zig` 版本不对，用 `ZIG` 环境变量指向 0.16 的二进制：

```bash
ZIG=/path/to/zig-0.16 cargo build --release
```

## 规则依据（已验证）

以下结论来自 CodeBuddy CLI **2.156.0** 自带 bundle 中的常量，以及在真实 Herdr pane 上用 `herdr agent explain <pane> --json` 抓取的各种状态。

- **终端标题**：`TerminalTitleUpdater` 写入 `\x1B]0;${prefix} ${topic}\x07`。working 时 `prefix` 取 `SPINNER_FRAMES = ["\u280B","\u2819","\u2839","\u2838","\u283C","\u2834","\u2826","\u2827","\u2807","\u280F"]`，其他时候取 `IDLE_PREFIX = "\u2733"`。全部帧都落在 braille 区 U+2800-U+28FF，所以 `osc_title_working` 只需 braille 区间，**不要**照抄 Claude 2.1.228 新增的 U+25D0-U+25D3 半圆区间（CodeBuddy 没有这个 spinner）。
- **标题可能整体缺失**：`CODEBUDDY_CODE_DISABLE_TERMINAL_TITLE` 被设置、或 stdout 不是 TTY 时标题更新器直接关闭。因此 working 不能只依赖 OSC 标题。
- **live turn 行**：输入框上方显示 `<frame> <动作>\u2026 (<秒>s \xB7 running <工具> \xB7 \u2026)`，字形用 cli-spinners 的 "star" 动画 `["\u2736","\u2738","\u2739","\u273A","\u2739","\u2737"]`（70ms），即区间 U+2736-U+273A。`live_turn_working` 依据这条。回合结束后该行变成 `\u2714 Worked for 9m 54s`，所以字形不会滞留（idle 画面里扫不到任何 U+2736-U+273A）。
- **输入框与状态行**：输入框本体是裸 `>`，它下面才是状态行 `\u276F <cwd> \xB7 <model> \xB7 <已用时长>`，再下面是模式徽标行（`\u23F8` plan、`\u23F5\u23F5` bypass/accept/auto、`\u21E2` delegate）。`live_prompt_box` 因此要匹配 `>`；状态行里的 `\u276F` 和模式徽标是**常驻**的，绝不能当成活动信号。
- **对话框选项**：确认对话框（模块 83563，interrupt-approval）把选中项渲染成 `> `、其余渲染成两个空格；其它列表（agents / model / skills picker）才用 `\u276F`。权限对话框形态为 `> 1. Yes / 2. Yes, allow reading from <dir> during this session / 3. No`，因此 `bash_permission_prompt` / `generic_permission_prompt` 的选项正则带可选 `[>❯]?` 前缀。对话框自身的快捷键提示是 `(shift + tab)` / `(escape)`，**不含** `esc to cancel`。
- `contains` 匹配器是**大小写不敏感**的（`lower_text.contains(needle)`），所以 `"showing detailed transcript"` 能匹配 CodeBuddy 实际渲染的 `Showing detailed transcript \xB7 ctrl+o to toggle`。
- `tab to amend` / `ctrl+e to explain` 在 2.156.0 中查无此文案，属沿用自 Claude 的常量；保留作为前向兼容，当前不会命中。

## 已知缺口：blocked 在弹窗期间不可靠

症状：CodeBuddy 停在权限确认弹窗上时，herdr 报 `working`，看不出它在等人。这个症状在 firstmate 侧已被实证两次（2026-09-15 ryfund、2026-09-16 Pi 权限策略 worker，见其 `cb-dialog-item.md`）。

根因是三层叠加，**主因是规则优先级掩蔽**，不是我先前以为的 region：

1. **优先级掩蔽（决定性）**：引擎取**优先级最高**的命中规则（`manifest.rs:482-485`，blocked 没有任何特殊加权）。本 manifest 里 blocked 最高只有 980，而 `osc_title_working` 是 1100 → 只要 braille spinner 还在写，无论 blocked 规则命中与否都会输。实测印证：停在弹窗时 herdr 报 `working`，说明弹窗期间标题确实还在转（`TerminalTitleUpdater` 由 run state 驱动，interrupt-approval 未结束该回合）。
   上游其它 manifest 正是为此把 OSC-title blocker 抬到 1200-1400：`letta` 1400、`grok` 1300、`qwen` 1200、`amp`/`codex`/`hermes` 1100。CodeBuddy 与 Claude 的优先级形状相同（blocked 最高都是 980），差别只在弹窗期间标题是否继续转 spinner；Claude 那边未独立验证。
2. **`live_blocked_form` / `generic_permission_prompt` 两条都够不着**：region 是 `after_last_horizontal_rule`，实测该区域等于状态行 + 模式徽标（输入框的上下边框就是那两条水平线）；且两条都 `contains = ["esc to cancel"]`，而对话框不渲染该文案。
3. **兜底规则挂在 `whole_recent` 上**：`bash_permission_prompt`（850）的 `any` 是 Bash 专属文案，只覆盖 Bash 类确认（标题 `Bash command`），Read/Write/Edit/Fetch 类不命中；`legacy_no_prompt_blocker`（300）面很宽（`do you want to` + `yes`），唯一的假阳性防线是空闲输入框守卫，而该守卫在弹窗期间是否有效见下面第 2 条待测项。

即使某一层修好，第 1 层的 1100 仍会盖住它，所以必须一起改。

### 需要一次实机验证才能定稿

必须在一个**默认权限模式**的 codebuddy 会话里触发一次确认弹窗，用 `herdr agent explain <pane> --json` 抓两件事：

- **弹窗渲染在哪**：`above_prompt_box`（会话区，即 `whole_recent` 能覆盖）还是输入框**下方**。这决定新规则的 region。
- **弹窗期间输入框 region 是什么**：`prompt_box_body` 是否仍是裸 `>`。若不是，`legacy_no_prompt_blocker` 的 `not` 守卫不会误伤；若仍是，则该守卫会把兜底规则也压掉（这是本轮把守卫从 `❯` 改成 `[>❯]` 后引入的未验证交互，方向上是拿"空闲误报 blocked"换"blocked 漏报"，需要实测确认哪边更靠得住）。

定稿方向（有上游先例，但在拿到上述两条读数前不要落地）：

- region 限定在**底部提示区**，不要用 `whole_recent`。firstmate 那份笔记记录过反向误报：一个健康 worker 因为**输出正文里引用了弹窗文案**（"Do you want to proceed (High risk...)"）就被判成"停在弹窗"，根因就是扫了整块区域。判定应带**形状条件**（编号选项行 `> 1. Yes` 之类 + 同屏文案），否则任何"提到弹窗文案"的 worker 都会被误判。
- 给 blocked 规则优先级 **> 1100**，照 `codex` / `grok` / `qwen` / `hermes` / `amp` / `letta` 的先例。

另外 `dynamic_workflow_prompt`、`model_picker_menu`、`transcript_viewer` 也还没做过实机触发验证。

