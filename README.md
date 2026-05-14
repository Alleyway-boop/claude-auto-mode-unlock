# Claude Code Unlocker

解除 Claude Code CLI 的相关功能限制，解锁 auto 模式~~和 API 接入情况下 buddy 互动功能~~。

本项目包含两个独立的补丁工具：

- **`claude-auto-mode-patcher.mjs`** — 解锁 auto 模式，无需逐条确认权限，自动执行。
- ~~**`claude-buddy-patcher.mjs`** — 解锁 buddy 互动功能，小伙伴用你配置的 haiku 模型发表评论。（**buddy 仅为 claude 愚人节限定活动而已，新版本已剔除**）~~

---

> ## 🔥 无需补丁开启 Auto 模式（推荐）
>
> **[Claude Code 官方文档](https://code.claude.com/docs/en/permission-modes#eliminate-prompts-with-auto-mode)表明：通过 API 使用 Sonnet 4.6, Opus 4.6, or Opus 4.7 等就可以开启 auto 模式。**
>
> 所以基本上直接指定 `ANTHROPIC_BASE_URL` 和 `ANTHROPIC_AUTH_TOKEN` 两个环境变量即可使用 auto 模式，当然前提是，你所使用的 API 支持 model 设置为 claude-sonnet-4-6，经过测试 deepseek/glm 都支持将 sonnet 映射到自己的模型中，其他厂商有待测试。
>
> ### 方式一：直接指定环境变量
>
> ```bash
> # DeepSeek — 不指定时默认模型为 deepseek-v4-flash
> export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
> export ANTHROPIC_AUTH_TOKEN=your-deepseek-api-key
> 
> # 智谱 GLM — 不指定时默认模型为 glm-4.7
> export ANTHROPIC_BASE_URL=https://open.bigmodel.cn/api/paas/v4
> export ANTHROPIC_AUTH_TOKEN=your-glm-api-key
> ```
>
> ⚠️ **注意：** 此方法仅适用于 Anthropic 兼容的 API 端点（DeepSeek、GLM 等），且要求后端能接受 Claude 模型名，而且不支持指定模型比如 `deepseek-v4-pro` 和 `glm-5.1`。我们推荐使用下方的方式。
>
> ### 方式二：使用 [claude-custom-router](https://github.com/zzturn/claude-custom-router)（推荐）
>
> 这个项目最初是为了做**多 API 负载均衡**，但意外发现它天然解决了 auto 模式解锁问题——因为它作为本地代理转发请求，Claude Code 感知到的模型名始终是 Claude 系列，自动通过所有门控。
> 
> 除了 auto 模式解锁，它还提供：
> - **多 API 源负载均衡** — 在 DeepSeek、GLM、Anthropic 之间自动切换
>- **故障自动转移** — 单个 API 挂了自动切到下一个
> - **统一配置管理** — 不用每次切换环境变量
> 
> 详见 👉 [zzturn/claude-custom-router](https://github.com/zzturn/claude-custom-router)
> 
> ---
> 
> 通过上述任一方式设置后直接使用，无需运行任何补丁脚本，且长期有效：
> 
>```bash
> claude --permission-mode auto
>```
> 

---

## 环境要求

- **Claude Code CLI**（环境变量方式跨版本通用；补丁脚本需精确匹配已支持版本）
- Node.js 18+
- **Windows / macOS / Linux**

## 支持版本

| 版本 | 平台 | 状态 |
|------|------|------|
| v2.1.92 | Windows | 已验证 |
| v2.1.96 | macOS/Linux | 已验证 |
| v2.1.104 | Windows | 已验证 |
| v2.1.105 | Windows | 已验证 |
| v2.1.109 | Windows | 已验证 |
| v2.1.133 | Linux | 已验证 |
| v2.1.137 | Linux | 已验证 |
| v2.1.138 | Linux | 已验证 |
| v2.1.139 | Linux | 已验证 |
| v2.1.141 | Windows (Bun) | 已验证 |

**版本匹配策略**：精确匹配，不支持回退。每个版本的混淆变量名不同，补丁无法跨版本复用。不支持的版本会直接报错。

> **跨平台说明**：各平台（Windows/macOS/Linux）的 Bun 编译二进制混淆变量名不同，即使版本号相同也无法互用补丁。

## 快速开始

**推荐：** 使用上方环境变量方式，无需运行补丁。

**备选：** 使用补丁脚本（仅 v2.1.96 macOS Bun 二进制）：

```bash
git clone https://github.com/zzturn/claude-auto-mode-unlock.git
cd claude-auto-mode-unlock

node claude-auto-mode-patcher.mjs    # auto mode 补丁
node claude-buddy-patcher.mjs        # buddy 补丁
```

---

## Auto Mode 补丁

解锁 auto 模式，自动执行无需逐条确认。

### 用法

```bash
# 应用补丁
node claude-auto-mode-patcher.mjs

# 检查补丁状态
node claude-auto-mode-patcher.mjs --check

# 恢复原始文件
node claude-auto-mode-patcher.mjs --restore

# 手动指定路径
CLAUDE_BIN=/path/to/claude node claude-auto-mode-patcher.mjs   # macOS/Linux
set CLAUDE_BIN=C:\path\to\claude && node claude-auto-mode-patcher.mjs  # Windows
```

### 使用

```bash
claude --permission-mode auto    # 启动时启用
# 或在会话中按 Shift+Tab 切换
```

### 原理

Claude Code 使用 [Bun](https://bun.sh/) 编译为独立二进制文件，JavaScript 源码以明文嵌入。本脚本通过**等长字节替换**修改权限检查函数（7 个基础 + 可选 classifier 补丁，v2.1.139+ 共 9 个）：

| # | 目标 | 效果 |
|---|------|------|
| 1 | `modelSupportsAutoMode` — provider 检查 | 绕过 firstParty/anthropicAws 限制 |
| 2 | `modelSupportsAutoMode` — model 正则（外层返回值） | 修改函数尾部 `return!1` → `return!0` |
| 3 | `modelSupportsAutoMode` — model 正则（内层返回值） | `return regex.test(K)` → `return!0`，确保非 Claude 模型通过 |
| 4 | `isAutoModeGateEnabled` | 始终返回 `true` |
| 5 | `isAutoModeCircuitBroken` | 始终返回 `false` |
| 6 | `verifyAutoModeGateAccess` | 强制走 happy path |
| 7 | `carouselAvailable` | 始终为 `true`（Shift+Tab 可切换） |
| 8 | `classifier unavailable` | fail-open：分类器不可用时允许而非阻止（v2.1.137+） |

> **v2.1.105 新增 `model-return` 补丁**：早期版本仅修改了 `modelSupportsAutoMode` 函数外层的 `return!1`，但内层的 `return/^claude-(opus|sonnet)-4-6/.test(K)` 会先执行并直接返回结果，导致外层修改永远不会生效。非 Claude 模型（如 glm-5.1）因此被误拒。新增的 `model-return` 补丁直接将内层正则返回替换为 `return!0`，从根源解决了此问题。

### 版本差异示例

每个版本的 7 个补丁点结构相同，仅混淆变量名不同。以 gate-enabled 补丁为例：

```javascript
// v2.1.92
function ty(){if(cv?.isAutoModeCircuitBroken()??!1)return!1;...}
// v2.1.96
function oN(){if(C0?.isAutoModeCircuitBroken()??!1)return!1;...}
// v2.1.104
function qL(){if(IV?.isAutoModeCircuitBroken()??!1)return!1;...}
// v2.1.105
function DL(){if(Lf?.isAutoModeCircuitBroken()??!1)return!1;...}
// v2.1.141
function fR(){if(ak?.isAutoModeCircuitBroken()??!1)return!1;...}
```

> **注意**：v2.1.105 的 `circuit-broken` 补丁结构有变化，从 `return <variable>` 改为 `return <object>.circuitBroken`，补丁已适配。

> **注意**：v2.1.139+ 的 `modelSupportsAutoMode` 函数结构大幅重构，不再使用正则匹配，改用 `$.includes("claude-3-")` + 多个 `||$===` 字符串比较来枚举模型。补丁策略相应调整为 3 个子补丁（`model-allow-list`、`model-rate-limit`、`model-outer-return`），加上 `classifier-unavailable`（fail-open）共 9 个补丁点。v2.1.141 沿用相同结构。

---

## Buddy 补丁

解锁 buddy companion 互动功能。小伙伴会用你配置的 haiku 模型（`ANTHROPIC_DEFAULT_HAIKU_MODEL`）发表评论。

### 用法

```bash
node claude-buddy-patcher.mjs           # 应用
node claude-buddy-patcher.mjs --check   # 检查状态
node claude-buddy-patcher.mjs --analyze # 诊断分析（不改文件）
node claude-buddy-patcher.mjs --restore # 恢复
CLAUDE_BIN=/path/to/claude node claude-buddy-patcher.mjs  # 指定路径
```

### 使用

在 Claude Code 中输入 `/buddy` 孵化小伙伴：

| 命令 | 作用 |
|------|------|
| `/buddy` | 孵化一个小伙伴 |
| `/buddy pet` | 摸摸它，触发反应 |
| `/buddy off` | 关闭小伙伴评论 |
| `/buddy on` | 重新开启 |

### 原理

基于源码分析的 5 阶段 patching：

1. **LOCATE** — 通过函数签名锚点定位 `Fa_`（buddyReact）函数
2. **VALIDATE** — 用 3 个源码派生的结构验证器确认目标正确
3. **BOUNDARY** — 花括号平衡扫描确定函数边界（支持正则字面量、模板字面量）
4. **REPLACE** — 动态生成等长本地 LLM 替换（含 JS 语法验证）
5. **VERIFY** — 补丁后完整性验证

原始 `Fa_` 有 4 层门控（auth provider、rate limit、org UUID、OAuth token），最终调用远程 API。补丁替换整个函数体，使用与 `wE7`（companion 生成）相同的 `Y0`/`ZP()` 本地 LLM 调用模式，直接用配置的 haiku 模型生成 reaction。

---

## 项目文件

| 文件 | 说明 |
|------|------|
| `claude-auto-mode-patcher.mjs` | Auto mode 补丁脚本 |
| `claude-buddy-patcher.mjs` | Buddy 补丁脚本 |
| `buddy-source-extracted.js` | 从二进制提取并标注的 buddy 系统源码 |
| `GUIDE.md` | 详细使用说明 |
| `METHODOLOGY.md` | 源码驱动的二进制 Patch 方法论文档 |

## 恢复原版

```bash
node claude-buddy-patcher.mjs --restore
node claude-auto-mode-patcher.mjs --restore
```

## 安全性

- 补丁前自动创建带版本号的备份文件（`.auto-mode-backup` / `.buddy-backup`）
- 所有替换严格等长，不破坏文件结构
- macOS 上自动执行 `codesign --force --sign -` 重新签名
- Windows 无需签名
- 可通过 `--restore` 完全恢复原始文件

## 添加新版本支持

当 Claude Code 更新到未支持的新版本时，脚本会报错并提示不支持的版本。需要手动添加：

### 步骤

1. 安装新版本 Claude Code
2. 在目标文件中搜索 7 个补丁点的混淆变量名
3. 在 `VERSION_PATCHES` 中添加新版本条目

### 搜索命令

```bash
# 设置目标文件路径
CLI="node_modules/@anthropic-ai/claude-code/cli.js"  # Windows npm (旧版)
# CLI="$(readlink -f ~/.local/bin/claude)"            # macOS/Linux

# 1. provider 检查
grep -oP 'if\([A-Za-z0-9_]+!=="firstParty"&&[A-Za-z0-9_]+!=="anthropicAws"\)return!1' "$CLI"

# 2. model 正则（外层返回值）— v2.1.137 及更早
grep -oP 'claude-\(opus\|sonnet\)-4-6/\.test\([A-Za-z0-9_]+\)\}return!1\}' "$CLI"

# 2. model allow-list — v2.1.139+（新结构）
grep -oP 'if\([A-Za-z0-9_]+\.includes\("claude-3-"\).*?\)return!1' "$CLI"

# 2.5 model 正则（内层返回值）— v2.1.105~v2.1.137
grep -oP 'return/\^claude-\(opus\|sonnet\)-4-6/\.test\([A-Za-z0-9_]+\)' "$CLI"

# 3. gate 函数
grep -oP 'function [a-zA-Z0-9_]+\(\)\{if\([a-zA-Z0-9_]+\?\.isAutoModeCircuitBroken\(\)\?\?!1\)return!1;if\([a-zA-Z0-9_]+\(\)\)return!1;if\(![a-zA-Z0-9_]+\([a-zA-Z0-9_]+\(\)\)\)return!1;return!0\}' "$CLI"

# 4. circuit-broken 函数
grep -oP 'function [A-Za-z0-9_]+\(\)\{return [A-Za-z0-9_]+\.circuitBroken\}' "$CLI"

# 5. can-enter
grep -oP 'if\([a-zA-Z0-9_]+\)return\{updateContext:\([^)]+\)=>[^}]+\};let [a-zA-Z0-9_]+;' "$CLI"

# 6. carousel
grep -oP '[A-Za-z0-9_]+=!1;if\([A-Za-z0-9_]+!=="disabled"&&[^;]+' "$CLI" | grep enabled

# 7. classifier unavailable — v2.1.137+
grep -oP '[A-Za-z0-9_]+\("tengu_iron_gate_closed",!0,[A-Za-z0-9_]+\)' "$CLI"

# 8. model rate-limit — v2.1.139+
grep -oP 'if\([A-Za-z0-9_]+\(\)&&\([A-Za-z0-9_]+==="claude-opus-4-6"\|\|[A-Za-z0-9_]+==="claude-sonnet-4-6"\)\)return!1' "$CLI"
```

> **对于 Bun 编译二进制**（`~/.local/bin/claude` 或 `claude.exe`），`grep` 无法直接搜索。需用 Node.js 从二进制中提取字符串后再分析：
> ```bash
> node -e "
> const fs = require('fs');
> const buf = fs.readFileSync(process.argv[1]);
> const str = buf.toString('latin1');
> const idx = str.indexOf('firstParty');
> if (idx > 0) console.log(str.substring(idx - 100, idx + 300));
> " ~/.local/bin/claude
> ```

## 注意事项

- **推荐使用环境变量方式**：设置 `ANTHROPIC_MODEL=claude-sonnet-4-6-20250514` 或 `claude-opus-4-7` 等即可开启 auto 模式，无需补丁，跨版本通用（详见上方「无需补丁开启 Auto 模式」章节）
- **补丁版本必须精确匹配**：每个版本的混淆变量名不同，不支持跨版本回退
- **补丁跨平台不通用**：各平台（Windows/macOS/Linux）的 Bun 编译二进制混淆名不同，即使同版本号也无法互用
- **升级后需重新打补丁**：Claude Code 更新会替换文件，需重新运行脚本（环境变量方式不受此影响）
- **auto 模式安全分类器仍生效**：仅解除入口限制，`classifyYoloAction` 仍会评估安全性

## 常见问题

<details>
<summary>提示 "vX.Y.Z is not supported"</summary>

当前安装的 Claude Code 版本尚未添加补丁支持。参考上方「添加新版本支持」章节手动添加。
</details>

<details>
<summary>脚本显示 "SKIP" 或 "No patches applied"</summary>

二进制可能已被补丁（运行 `--check` 查看），或版本不在支持列表中。推荐改用环境变量方式（设置 `ANTHROPIC_MODEL=claude-sonnet-4-6-20250514` 或 `claude-opus-4-7`），无需补丁。
</details>

<details>
<summary>补丁后 claude 命令无法启动</summary>

```bash
node claude-auto-mode-patcher.mjs --restore
# 或
node claude-buddy-patcher.mjs --restore
```
</details>

<details>
<summary>Windows 上找不到目标文件</summary>

设置环境变量手动指定：
```cmd
set CLAUDE_BIN=%USERPROFILE%\.local\bin\claude.exe
node claude-auto-mode-patcher.mjs
```
</details>

<details>
<summary>macOS 上 codesign 失败</summary>

```bash
codesign --force --sign - "$(node -e 'console.log(require("fs").realpathSync(process.argv[1]))' ~/.local/bin/claude)"
```
</details>

<details>
<summary>Buddy 不说话？</summary>
反应有 30 秒冷却，需要足够对话上下文。叫它名字或 `/buddy pet` 可触发。
</details>

## License

MIT
