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

## blocked 在弹窗期间不可靠（已实机复现并修复）

症状：CodeBuddy 停在权限确认弹窗上时，herdr 报 `working`，看不出它在等人。这个症状在 firstmate 侧已被实证两次（2026-09-15 ryfund、2026-09-16 Pi 权限策略 worker，见其 `cb-dialog-item.md`）。

### 实测复现（2026-09-21，CodeBuddy 2.156.0 + herdr 0.9.1）

在一个 `--permission-mode default` 的临时 tab 里让 codebuddy 跑 `rm -rf <scratch>/probe-rm`，弹窗期间抓到的屏幕：

```
● Bash(rm -rf /Users/.../cb-blocked-probe/scratch/probe-rm)
────────────…（对话框上边框）
 Bash command
   rm -rf /Users/.../cb-blocked-probe/scratch/probe-rm
   Remove probe-rm directory
────────────…（对话框 footer 上边框）
 Do you want to proceed(High risk operation detected - requires confirmation every time)?
 > 1. Yes
   2. Yes, and don't ask again for session (shift + tab)
   3. No, and tell CodeBuddy what to do differently (escape)
```

此时 herdr 报 `working`，标题仍在转 `⠸`，但 `probe-rm` 目录还在（命令没执行）。**Bug 稳定复现。**

### 根因（修正我早先的误判）

关键在于：**弹窗是替换掉输入框的，不是叠在它上面**。所以弹窗打开时裸 `>` 提示行消失，而 `after_last_horizontal_rule` 恰好解析成弹窗正文。这修正了我先前"对话框渲染在输入框上方、该 region 看不到它"的推断 - region 从来不是问题。

真正的三层原因：

1. **优先级掩蔽（决定性）**：引擎取优先级最高的命中规则（`manifest.rs:482-485`，blocked 没有特殊加权）。弹窗期间 `TerminalTitleUpdater` 继续写 braille spinner，`osc_title_working`（1100）命中；而本 manifest 里 blocked 最高只有 980 → 无论 blocked 规则命中与否都会输。上游其它 manifest 正是为此把 blocker 抬到 1200-1400（`letta` 1400、`grok` 1300、`qwen` 1200、`amp`/`codex`/`hermes` 1100）。
2. **现有 blocked 规则没有一条能匹配这个弹窗**（实测全部 `matched=false`）：
   - `live_blocked_form` / `generic_permission_prompt`：region 是**对的**（实测预览就是上面那段弹窗正文），但 `contains` 用的是 Claude 的 `esc to cancel` / `enter to select`，而 CodeBuddy 只渲染每个选项后面的 `(shift + tab)` / `(escape)`。
   - `bash_permission_prompt`：`contains = ["do you want to proceed?"]` **匹配不上高风险形态** - 风险原因被插在 `?` 前面，实际文本是 `proceed(High risk...)?`。低风险 Bash（无 reason）才会渲染成 `proceed?`。
   - `legacy_no_prompt_blocker`：实测 `matched=true`（`do you want to` + `1. Yes`），但优先级只有 300。
3. **`whole_recent` 兜底有误报风险**：firstmate 那份笔记记录过反向误报 - 一个健康 worker 因为输出正文里引用了弹窗文案（"Do you want to proceed (High risk...)"）就被判成"停在弹窗"，根因就是扫了整块区域。

### 修复

新增 `confirm_dialog_blocked`（priority **1200**，region `after_last_horizontal_rule`），用"编号选项行 `[>❯]?\s*[1-3]\.\s*(yes|no)`"作**形状条件**，再要求同屏出现标题族文案（`do you want to` / `would you like to` / `about to delete`）。

- 高优先级解决第 1 层。
- 底部 region + 形状条件解决第 3 层：idle 时该 region 只有状态行和模式徽标，绝不会出现编号选项；所以正文里引用弹窗文案不会被误判。
- 实测还顺带回答了先前挂起的疑问：弹窗期间输入框被替换、裸 `>` 消失，所以 `legacy_no_prompt_blocker` 的 `not` 守卫**不会**在弹窗期间误伤（实测该规则 `matched=true`）。把守卫从 `❯` 改成 `[>❯]` 是安全的。

### 验证结果

| 场景 | 修复前 | 修复后 |
|---|---|---|
| 弹窗停在 High risk Bash 确认 | `working`（matched `osc_title_working`） | **`blocked`**（matched `confirm_dialog_blocked`，1200 压过 1100） |
| 按 Escape 拒绝后 | - | `idle`（`live_prompt_box` + `osc_title_idle`），弹窗规则不再命中 |
| 2 个空闲 pane | `idle` | `idle`，弹窗规则 `matched=false` |
| 2 个工作中 pane | `working` | `working`，弹窗规则 `matched=false` |

拒绝后确认 `probe-rm` 目录仍存在（命令确实没执行）。验证走的是本地 override + `herdr server reload-agent-manifests` 热重载，验证完 override 已删除。

### 仍未验证

- 其它形态的弹窗：`Write`/`Edit` 用 `Do you want to create …?` / `Do you want to make this edit to …?`，`Fetch` 用 `Do you want to allow … to fetch this content?`，批量删除用 `This command is about to delete N items.`。这些标题不含 `do you want to` 的常规形态，但 `about to delete` 已覆盖批量删除；其余几种需要用同样的方法各触发一次确认。
- 渲染 `Esc to cancel` 那一类菜单（`live_blocked_form` 覆盖的 agents / skills picker）在弹窗期间是否也被 1100 掩蔽，未验证。
- 工作区信任弹窗（`Do you trust the files in this folder`）未触发过 - 本次实验用的是已信任路径。
- `dynamic_workflow_prompt`、`model_picker_menu`、`transcript_viewer` 仍未做过实机触发验证。


