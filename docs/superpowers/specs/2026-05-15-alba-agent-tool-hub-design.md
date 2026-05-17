---
title: agenthub — Cross-Tool Skills/Commands/Agents/Plugins/MCP Management Hub
date: 2026-05-15
status: APPROVED — design complete, ready for implementation plan
owner: Alba
schema_version: 1
---

# agenthub 设计文档

## §0 目标

构建一个个人工具栈，**统一管理跨 AI 编程工具的扩展内容**（skills、slash commands、subagents、plugins、MCP servers），并满足：

1. **三家工具体验对等**：Claude Code / Codex / OpenCode 一视同仁，不以任一为特权源；
2. **profile 切换**：单条命令切换工具的整套状态（dotfiles + 启用内容）；
3. **多机同步**：内容 hub 通过 git remote 同步，本机状态保持机器独立；
4. **远程源跟踪**：可从 anthropic/skills 等上游 git 仓库拉取 skill 更新，并保留本地化改动；
5. **plugin 整体管理**：把整个 superpowers 之类的 plugin 作为粗粒度单元纳入 profile；
6. **MCP 服务器跨工具配置**：单一定义，渲染到三家各自的配置文件；
7. **fork 友好**：工具代码和个人内容分仓，让别人轻松 fork 工具而不被个人数据污染。

## §1 整体架构

### 1.1 两个仓库

```
Repo A: agenthub                      Repo B: alba-agent-tool-hub
github.com/alba/agenthub              github.com/alba/alba-agent-tool-hub
(工具 — pipx installable)              (个人内容)

agenthub/                             alba-agent-tool-hub/
├── cli/                              ├── .agenthub/
│   └── agenthub/                     │   └── version.toml          (schema_version=1)
│       ├── __main__.py               ├── skills/
│       ├── config.py                 │   └── <name>/
│       ├── detect.py                 │       ├── SKILL.md          (含 metadata.tags)
│       ├── apply.py                  │       ├── .agenthub-source.toml      (可选)
│       ├── sync.py                   │       └── .agenthub-source.lock.toml (可选)
│       ├── project.py                ├── commands/
│       ├── source.py                 │   └── <name>.md (+ 可选成对 sidecar/lock)
│       ├── plugin.py                 ├── agents/
│       ├── mcp.py                    │   └── <name>.md (+ 可选成对 sidecar/lock)
│       ├── adopt.py                  ├── plugins/
│       ├── render.py                 │   ├── claude/<name>.toml
│       └── targets/                  │   ├── codex/<name>.toml
│           ├── base.py               │   └── opencode/<name>.toml
│           ├── claude.py             ├── mcp-servers/
│           ├── codex.py              │   └── <name>.toml
│           └── opencode.py           ├── profiles/
├── pyproject.toml                    │   ├── claude/
├── tests/                            │   │   ├── default/
│   ├── unit/                         │   │   │   ├── profile.toml
│   └── integration/                  │   │   │   ├── includes.d/         (可选)
├── docs/                             │   │   │   ├── settings.json
│   └── hub-schema.md                 │   │   │   ├── CLAUDE.md
└── README.md                         │   │   │   └── statusline.sh
                                      │   │   ├── work/
                                      │   │   └── minimal/
                                      │   ├── codex/
                                      │   └── opencode/
                                      ├── projects/
                                      │   └── <id>/config.toml
                                      └── README.md
```

### 1.2 本机非同步状态

```
~/.agenthub/
├── config.toml                       # 工具配置：hub 路径、CLI 路径覆盖
├── state.json                        # active profile per target、cwd→project-id 映射
├── secrets.env                       # gitignored；${VAR} 占位符的真值
├── staging/                          # apply 临时区（写好 → 原子换位 → 清空）
├── last-apply.json                   # 最近一次成功 apply 的清单（doctor 用）
├── stash/                            # source update 冲突时的本地改动备份
└── content-cache/                    # source import 时的 shallow clone 缓存
```

实际软链产物落地：
```
~/.<tool>/                            # 全局通道
└── skills/<name>     → ~/agent-hub/skills/<name>
<project>/.<tool>/                    # 项目通道
└── skills/<name>     → ~/agent-hub/skills/<name>
```

### 1.3 五大支柱

```
① Content sync          —— skills/commands/agents 细粒度同步
② Profile orchestration —— 一键切换工具全套状态
③ Source ingestion      —— 上游 git 跟踪 + 拉取更新
④ Plugin management     —— 整 plugin 安装/版本（含 OpenCode npm 特例）
⑤ MCP server management —— 跨工具 MCP server 统一表达 + 三家渲染
```

### 1.4 两条绑定通道

```
GLOBAL  通道 —— active profile 决定 ─→  ~/.<tool>/...
PROJECT 通道 —— hub/projects/<id>/config.toml 决定 ─→ <project>/.<tool>/...
```

强保证：两条通道在物理层面完全正交。项目级操作绝不污染全局目录；反之亦然。

### 1.5 通用 schema envelope

所有持久化 toml/json 文件以下面这两行开头：
```toml
apiVersion = "agenthub.dev/v1"
kind = "Profile" | "ProfileFragment" | "ProjectConfig" |
       "SourceIntent" | "SourceLock" |
       "PluginRef" | "McpServer" |
       "Config" | "State" | "HubVersion"
```

agenthub 启动时握手：未知 major → 拒绝运行 + 提示升级；未知 minor → 保留未知字段往回写。

### 1.6 实机验证清单（实现阶段处理）

- Codex plugin 安装 CLI 入口和约定
- Codex 用户级独立 command 是否支持（疑似仅 plugin-bundled）
- Codex MCP server 配置具体落点（settings vs plugin .mcp.json vs config.toml）
- Codex 项目级 MCP 是否支持
- OpenCode 的 MCP `type` 取值范围
- OpenCode plugin npm 安装的具体调用约定（bun vs npm vs pnpm）

## §2 Profile 系统

### 2.1 Profile 是什么

`profiles/<tool>/<name>/` 是一个目录，包含：
- `profile.toml`：声明该 profile 启用哪些 skill/command/agent/plugin/mcp（用 `<name>` 字面 或 `@tag` 引用）
- 该工具的所有 dotfile 文件（settings.json / CLAUDE.md / statusline.sh / config.toml / opencode.json 等），可包含 `${VAR}` 占位符
- 可选 `includes.d/*.toml` 片段目录（fragment composition）

**无 profile 间继承**——每个 profile 完整声明自己。`default` 只是名字约定，不是特殊角色。

### 2.2 profile.toml schema

```toml
apiVersion = "agenthub.dev/v1"
kind       = "Profile"

[skills]
include = ["frontend-design", "@react"]

[commands]
include = ["review", "@workflow"]

[agents]
include = []

[plugins]
include = ["superpowers"]

[mcp]
include = ["filesystem", "github"]
```

每节都支持 `<name>` 字面和 `@tag` 引用。`@tag` 在 apply 时解析为全集（去重）。`@tag` 解析为空集 → warning，不 error。

### 2.3 Tag 声明

每个 skill 在自己的 SKILL.md frontmatter 里：
```yaml
---
name: frontend-design
description: ...
metadata:
  tags: [core, web, react]
---
```

commands / agents 同理（在 .md frontmatter 内 `metadata.tags`）。
plugins / mcp-servers 在自己的 toml 的 `[meta] tags = [...]` 字段。

Tag 是 free-form 小写字符串，无中心化注册表。

### 2.4 fragment composition（conf.d 模式）

```
profiles/claude/work/
├── profile.toml
├── includes.d/                # 可选目录
│   ├── 10-base.toml
│   ├── 20-python.toml
│   └── 30-machine-laptop.toml
├── settings.json
└── CLAUDE.md
```

apply 时把 `profile.toml` 和 `includes.d/*.toml` 的所有 include 列表做并集。dotfile 文件**不参与片段化**（dotfile 整文件管，没有干净的合并语义）。

片段文件 schema 等同 profile.toml 但通常只填一两节：
```toml
# includes.d/20-python.toml
apiVersion = "agenthub.dev/v1"
kind       = "ProfileFragment"

[skills]
include = ["@python", "pytest-helper"]

[mcp]
include = ["pip-resolver"]
```

### 2.5 Active profile 状态

`~/.agenthub/state.json`:
```json
{
  "apiVersion": "agenthub.dev/v1",
  "kind": "State",
  "spec": {
    "active_profiles": {
      "claude": "work",
      "codex": "default",
      "opencode": "default"
    },
    "project_mapping": {
      "/abs/path/to/projA": "projA-id"
    }
  }
}
```

`agenthub apply --target claude --profile work` 修改 `active_profiles.claude`。
不传 `--profile` 时沿用 active；state.json 无值时默认 `default`。

### 2.6 切换 profile 的物理动作

切换 profile **不删 hub 内文件**，只动本机软链：
- 拔旧 profile 的所有软链；
- 渲染 dotfiles 到 `~/.agenthub/staging/<tool>/.agenthub-rendered/` → 原子换位到 `~/.<tool>/.agenthub-rendered/` → 软链到 `~/.<tool>/`；
- 建新 profile 的内容软链；
- 写 `last-apply.json`。

`agenthub gc` 清理 `.agenthub-rendered/` 里无人指向的旧产物。

## §3 项目绑定系统

### 3.1 数据模型

```
hub/projects/<id>/config.toml           ← 跨机器同步（git tracked）
~/.agenthub/state.json.project_mapping  ← 本机映射，不同步
```

### 3.2 projects/<id>/config.toml schema

```toml
apiVersion = "agenthub.dev/v1"
kind       = "ProjectConfig"

[meta]
description = "..."
created_at  = "2026-05-15"

[claude.skills]
include = ["frontend-design", "@react"]

[claude.commands]
include = ["review"]

[claude.agents]
include = []

[claude.plugins]
include = ["superpowers"]

[claude.mcp]
include = ["filesystem"]

[codex.skills]
include = ["@core"]

# 其他 target 类推；不需要的 target 节段省略
```

### 3.3 项目身份识别

**显式命名 + 本机映射缓存**。首次 `agenthub project init` 时由 Alba 给项目起名（默认 cwd basename），mapping 存 `~/.agenthub/state.json.project_mapping`。

允许多个 cwd 映射到同一 id（多 worktree）；不允许单个 cwd 映射到多个 id。

### 3.4 项目流程

**3.4.1 首次接入**
```
cd /abs/path/to/newproj
agenthub project init
  ↓
1. 检测 cwd 是否已在 mapping 中 → 已有：报错，提示 unlink 或 --force
2. hub 内 projects/<id>/ 已存在 → 询问是否复用
3. 否则在 hub 创建 projects/<id>/config.toml（meta 段 + 五个空 target 节）
4. state.json.project_mapping 写入 cwd → id
5. 提示下一步：agenthub project add ...
```

**3.4.2 项目级 apply**
```
agenthub project apply
  ↓
1. 从 state.json 取 cwd 对应 id
2. 读 hub/projects/<id>/config.toml
3. 对每个 target 节段：
   - 解析 include（含 @tag 展开）
   - 渲染 ${VAR}（如需）
   - 软链/写入到 <cwd>/.<tool>/...
4. 严格不动 ~/.<tool>/...
```

**3.4.3 新机器初次进入已存在 project**
```
machine B 上，cd 到该项目目录，agenthub project init
  ↓
agenthub 扫 hub/projects/ 已存在 id，按 cwd basename 模糊匹配：
  "Found existing project 'projA-id' in hub. Link this cwd to it? (Y/n)"
```

**3.4.4 孤儿检测**
```
agenthub project doctor 报告：
  · 本机 mapping、hub 无 id → 提示 unlink 或 init 新 id
  · hub 有 project、本机无 mapping → 信息提示（可能别机器用）
  · cwd 在 mapping 中但目录不存在 → 提示 unlink
```

## §4 Source Ingestion

### 4.1 Sidecar 机制

外来内容（skill/command/agent，**不含** plugin/mcp）通过 **sidecar + lock 双文件**追踪：

| 内容类型 | sidecar 位置 | lock 位置 |
|---|---|---|
| skill（目录） | `<skill>/.agenthub-source.toml` | `<skill>/.agenthub-source.lock.toml` |
| command（.md） | `<name>.agenthub-source.toml`（成对） | `<name>.agenthub-source.lock.toml` |
| agent（.md） | 同上 | 同上 |

**没有 sidecar = Alba 原创，不参与 update**。
plugin/mcp 不走 sidecar：plugin 有自己的 `hub/plugins/<tool>/<name>.toml` 引用文件；mcp 本来就是本地手写。

### 4.2 Sidecar + Lock schema

```toml
# .agenthub-source.toml（人写，intent）
apiVersion = "agenthub.dev/v1"
kind       = "SourceIntent"

[source]
type     = "git-subdir"     # git | github | git-subdir | github-release | http | npm
url      = "https://github.com/anthropic/skills"
path     = "skills/frontend-design"      # 子路径；为空时整 repo 是一个 skill
ref      = "main"                        # 分支名 / tag / 引用；具体 sha 在 lock
ref_type = "branch"                      # branch | tag | commit
```

```toml
# .agenthub-source.lock.toml（机器写，resolved）
apiVersion = "agenthub.dev/v1"
kind       = "SourceLock"

sha          = "abc1234567890..."
commit_title = "feat: add experimental react helpers"
imported_at  = "2026-05-15T11:30:00+08:00"
last_updated = "2026-05-15T11:30:00+08:00"
```

### 4.3 CLI

```
agenthub source import --url <git-url> [--path <subpath>] [--kind skill|command|agent] \
                       [--name <local-name>] [--ref <ref>]
agenthub source update <name> [--ref <ref>]    # 默认 autostash+rebase
agenthub source update --all
agenthub source ls                              # 现场扫所有 sidecar
agenthub source attach <name> --url ... --path ...    # 原创 → 追踪上游
agenthub source detach <name>                   # 追踪上游 → 原创
```

### 4.4 Import 流程

```
agenthub source import --url <url> --path <path>
  1. shallow clone 到 ~/.agenthub/content-cache/<url-hash>/
  2. 读 HEAD 的 sha（用作默认 pin）
  3. 拷贝 <repo>/<path>/ 整体到 hub/skills/<name>/
  4. 写 sidecar + lock
  5. 不自动 git commit，提示 Alba review 后提交
  6. 提示：未加入任何 profile/project；如何启用
```

### 4.5 Update 流程

默认 **autostash + rebase**（chezmoi 模式）：

```
agenthub source update frontend-design
  1. 读 sidecar.ref + lock.sha → 目标 ref
  2. fetch/checkout 目标 ref 到 cache
  3. 比较 hub 当前内容 vs cache(目标 ref) vs cache(lock.sha 历史)
  4. 三态：
     a) hub == lock 历史（Alba 没改）
        → 直接拷贝目标内容、更新 lock.sha + last_updated
        → 打印 diff 摘要，提示 review + git commit
     b) hub != lock 历史（Alba 改过）
        → autostash 到 ~/.agenthub/stash/<timestamp>/<skill>/
        → 应用上游内容
        → 尝试 replay 本地改动（unified patch）
        → 成功：完事，提示 review + commit
        → 失败：弹 5 选项 (stash/abort/diff/merge/detach)
     c) 目标 ref 已是 lock.sha → "already up to date"
  5. 始终不自动 git commit
```

5 选项中 **detach** = 删 sidecar + lock，从此把它当作原创、不再追踪上游。

## §5 Plugin 管理

### 5.1 Plugin 引用 schema

每个 plugin 在 hub 内由一个**引用文件**描述（**不 vendor 内容**，仅存指针；内容由原生工具按需拉取/安装）：

**Claude Code / Codex（基于 marketplace + plugin 名）**

```toml
# hub/plugins/claude/<name>.toml
apiVersion = "agenthub.dev/v1"
kind       = "PluginRef"

[meta]
description = "Bundle of skills and subagents for advanced workflows"
tags        = ["productivity"]

[plugin]
type        = "marketplace"
marketplace = "https://github.com/obra/superpowers"
plugin      = "superpowers"          # plugin 在 marketplace 中的名字
version     = "5.0.7"                # 可选；不写则 latest
```

**OpenCode（基于 npm 包）**

```toml
# hub/plugins/opencode/<name>.toml
apiVersion = "agenthub.dev/v1"
kind       = "PluginRef"

[meta]
description = "Multi-agent orchestration harness"
tags        = ["productivity"]

[plugin]
type        = "npm"
npm_package = "@code-yeongyu/oh-my-openagent"
version     = "latest"
```

`[plugin].type ∈ {marketplace, npm}`。决定 apply 时走哪条安装路径。

### 5.2 Per-tool 安装机制

| Target | 安装方式 |
|---|---|
| Claude Code | delegate `claude plugin marketplace add <url> && claude plugin install <name> [--version ...]` |
| Codex | delegate Codex 原生 plugin install（**待实机验证** 具体 CLI 语法） |
| OpenCode | 编辑 `~/.config/opencode/package.json` 的 `plugin` 数组 + 调用 `bun install`（CLI 路径在 `~/.agenthub/config.toml` 可覆盖） |

项目级 plugin（如有）：写到 `<project>/.<tool>/` 对应位置。

### 5.3 CLI

```
agenthub plugin add    -t claude --marketplace <url> --plugin <name> [--version ...]
                       -t opencode --npm-package <name> [--version ...]
agenthub plugin ls     -t <tool>
agenthub plugin show   -t <tool> <name>
agenthub plugin rm     -t <tool> --plugin <name>
agenthub plugin update -t <tool> --plugin <name> [--version ...]
```

**启用粒度**：all-or-nothing。一次启用整个 plugin，**不做** "只启用 plugin 中的某些 skill"（plugin 内部依赖不透明，拆开容易坏；要细控就 fork plugin 再做）。

**安装手段**：delegate 到原生 CLI / npm install，agenthub **不直接管** `~/.claude/plugins/cache/` 等目录（避免在 Claude Code 升级 plugin 模型时碎掉）。

### 5.4 Plugin 不走 sidecar 的原因

- skill/command/agent 是 **vendored**（hub 内有完整内容，sidecar 记录上游）—— 适合本地少量改动
- plugin 是 **referenced**（hub 内仅引用文件，内容由原生工具拉）—— plugin 通常大、常被上游 release 推动、本地很少改

这是有意的非对称设计，对应 Pillar 3 vs Pillar 4 的本质差异。

## §6 MCP Server 管理

### 6.1 Hub MCP schema

```toml
# hub/mcp-servers/<name>.toml
apiVersion = "agenthub.dev/v1"
kind       = "McpServer"

[meta]
description = "Read-only filesystem access under HOME/work"
tags        = ["core", "fs"]

[server]
type    = "local"        # local | remote
command = "npx"
args    = ["-y", "@modelcontextprotocol/server-filesystem", "${HOME}/work"]
# remote 模式用：
# type    = "remote"
# url     = "https://mcp.example.com"
# headers = { Authorization = "Bearer ${MCP_TOKEN}" }

[env]
NODE_ENV = "production"

# 仅在差异化时填
[targets.claude]
disabled = false
```

### 6.2 Per-tool 渲染落点

| Target | 写入位置 | 写入形式 |
|---|---|---|
| Claude Code | `~/.claude/settings.json` 的 `mcpServers.<name>` | 合并写入 |
| Codex | 待实机验证 | TBD |
| OpenCode | `~/.config/opencode/opencode.json` 的 `mcp.<name>` | 合并写入 |

项目级落到 `<project>/.<tool>/` 对应文件。

### 6.3 CLI

```
agenthub mcp add --name <n> --type local|remote ... [--tags ...]
agenthub mcp ls
agenthub mcp show <name>
agenthub mcp rm  <name>
```

合并写入语义：agenthub 解析、修改、回写完整 settings.json/opencode.json，**仅替换 mcp 相关节段**，不动其他键。删除"上次 apply 装了但本次 profile 不再含"的条目。

## §7 CLI 全貌

### 7.1 全局选项

```
--hub <path>           覆盖 hub 仓库位置
--target <tool>        指定 target（claude|codex|opencode|all）
--profile <name>       指定 profile
--project <path|id>    指定项目
--dry-run              不写、只输出 plan
--verbose              文件层细节
--diff                 模板内容 unified diff
--yes / -y             跳过交互
--json                 NDJSON 输出
```

### 7.2 命令分组

```
─── 装机与配置 ────────────────────────────
agenthub init --hub <path>
agenthub bootstrap
agenthub config get|set <key> [<value>]
agenthub --version

─── 主流程 ─────────────────────────────────
agenthub apply [--target ...] [--profile ...]
agenthub sync
agenthub doctor
agenthub gc

─── Profile 管理 ───────────────────────────
agenthub profile ls [--target ...]
agenthub profile show <profile> --target <tool>
agenthub profile new <name> --target <tool> [--copy-from <existing>]
agenthub profile rm <name> --target <tool>
agenthub profile edit <name> --target <tool> \
        --add-skill/--add-tag/--add-plugin/--add-mcp/--add-command/--add-agent \
        --rm-* (对称)

─── Project 管理 ───────────────────────────
agenthub project init [--id <name>] [--use-existing <id>]
agenthub project add -t <tool> <skill...>
agenthub project rm  -t <tool> <skill...>
agenthub project ls
agenthub project show [<id>]
agenthub project apply
agenthub project unlink
agenthub project delete <id>

─── Source 管理 ────────────────────────────
agenthub source import --url <url> [...]
agenthub source update <name> [--ref ...]
agenthub source update --all
agenthub source ls
agenthub source attach <name> --url ... --path ...
agenthub source detach <name>

─── Plugin 管理 ────────────────────────────
agenthub plugin add    -t <tool> --marketplace <url> --plugin <name> [...]
                       -t opencode --npm-package <name> [...]
agenthub plugin ls     -t <tool>
agenthub plugin rm     -t <tool> --plugin <name>
agenthub plugin update -t <tool> --plugin <name> [...]

─── MCP 管理 ───────────────────────────────
agenthub mcp add --name <n> --type local|remote ...
agenthub mcp ls
agenthub mcp show <name>
agenthub mcp rm  <name>

─── 本地 → hub 收编 ───────────────────────
agenthub adopt skill   --from <path> [--as <name>] [--tags ...]
agenthub adopt command --from <path> [--as <name>] [--tags ...]
agenthub adopt agent   --from <path> [--as <name>] [--tags ...]

─── 状态与诊断 ─────────────────────────────
agenthub state show
agenthub state rollback
agenthub state migrate
```

### 7.3 输出约定

- 默认所有写操作前打印 plan 块，确认后执行；`--yes` 跳过；
- `--json` 输出 NDJSON；
- 错误码：`0` 成功 / `1` 用户错误 / `2` 环境/状态错误 / `3` 上游/网络错误。

### 7.4 三层 dry-run

```
默认（语义层）：
  + enable skill foo@1.2.3 (via tag @react)
  - disable skill bar
  ~ update mcp baz: 1.0.0 → 1.0.1

--verbose（文件层）：
  ln -s /Users/alba/agent-hub/skills/foo  ~/.claude/skills/foo
  rm    ~/.claude/skills/bar
  write ~/.claude/settings.json  (mcp.baz)

--diff（内容层）：
  --- a/.claude/CLAUDE.md
  +++ b/.claude/CLAUDE.md
  @@ -1,3 +1,4 @@
  ...
```

## §8 Bootstrap：新机器到可用

```bash
# 1. 装工具
pipx install git+https://github.com/alba/agenthub.git

# 2. clone 内容仓
git clone git@github.com:alba/alba-agent-tool-hub.git ~/agent-hub

# 3. 注册 hub 位置
agenthub init --hub ~/agent-hub
   → 创建 ~/.agenthub/config.toml 与空 secrets.env 模板

# 4. 探测三家工具 + 初始化 state
agenthub bootstrap
   → 探测顺序：CLI in PATH → 配置目录存在 → 任一满足即视为安装
   → 缺失 CLI 时让用户在 config.toml 填 [targets.<tool>].cli_path
   → 扫所有 dotfile 模板的 ${VAR}，对比 secrets.env 提示缺失变量

# 5. 首次 apply
agenthub apply --target all --profile default

# 6. 项目接入（如有）
cd ~/code/my-project
agenthub project init
agenthub project apply
```

### 8.1 ~/.agenthub/config.toml schema

```toml
apiVersion = "agenthub.dev/v1"
kind       = "Config"

hub_path = "/Users/alba/agent-hub"

[targets.claude]
enabled  = true
cli_path = "claude"        # 或绝对路径

[targets.codex]
enabled  = true
cli_path = "codex"

[targets.opencode]
enabled  = true
cli_path = "opencode"

[render]
secrets_env_path = "~/.agenthub/secrets.env"
```

### 8.2 secrets.env 格式

标准 `.env` 风格的 `KEY=VALUE` 文件（每行一对）：

```ini
# ~/.agenthub/secrets.env
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GITHUB_TOKEN=ghp_...
MCP_SLACK_TOKEN=xoxb-...

# 注释以 # 开头
# 值不带引号；含空格的值要带引号
SOME_PATH="/Users/alba/Important Folder"
```

agenthub apply 时把所有 dotfile / mcp-server / 等模板里出现的 `${VAR}` 替换为这里对应的值；任一引用变量在文件里找不到 → apply 报错并指明缺失变量名。

## §9 Apply Pipeline（实现关键）

### 9.1 分阶段执行（pre-flight collision + 临时目录原子换位）

```
agenthub apply --target claude --profile work
  1. 解析阶段
     · 读 profile.toml + includes.d/*.toml → 合并 include 列表
     · @tag 展开 → 最终内容集合
     · 解析 ${VAR}（缺值立即报错）
  2. 计划阶段
     · 对比期望状态 vs live 状态 → 计划项列表
     · 默认输出语义层 plan
     · 等待确认（除非 --yes）
  3. 预检阶段（pre-flight）
     · 扫 live 目录里"非 agenthub 管理"的文件
     · 撞上目标路径 → abort，未写任何东西
  4. 构建阶段
     · 在 ~/.agenthub/staging/<tool>/ 构建新软链树 + 渲染 dotfile
  5. 提交阶段
     · 写 ~/.agenthub/last-apply.json 清单（先于真实换位）
     · os.rename() 单目录原子换位（POSIX 同 fs 保证）
     · 更新 state.json.active_profiles
  6. 清理阶段
     · staging/ 清空
     · 触发可选 gc（清理孤儿 rendered）
```

### 9.2 崩溃恢复

`agenthub doctor` 通过对比 `last-apply.json` 和 live 状态识别半成品：
- staging/ 残留 → 清理
- last-apply 中的目标位置缺文件/软链 → 提示 re-apply
- live 中有不在 last-apply 的 agenthub-managed 文件 → 提示 stale

## §10 测试与维护

### 10.1 测试金字塔

| 层 | 数量 | 工具 |
|---|---|---|
| E2E smoke | ≤5 场景，手动+偶发 CI | 真 CLI、真文件系统 |
| Integration | ~30 场景 | pytest + tmp_path + mock CLI |
| Unit | 覆盖核心模块 | pytest + pytest-mock |

### 10.2 工具链

- 测试：`pytest` + `pytest-xdist` + `pytest-mock`
- Lint：`ruff`（含 import 排序）
- Type check：`pyright`（Protocol 友好）
- Coverage：`coverage.py`，目标 ≥80%、核心 ≥90%
- CI：GitHub Actions，矩阵 Python 3.11/3.12/3.13 × macOS/Linux

### 10.3 关键 fixture

- `tmp_hub`：最小合法 hub 仓库
- `fake_target_dirs`：模拟 ~/.claude/ ~/.codex/ ~/.config/opencode/
- `mock_target_cli`：mock 三家 CLI 调用

### 10.4 必测场景（≥10 必选）

1. apply 干净状态：profile 全装，软链都建立
2. apply 切换 profile：旧软链拔、新软链建
3. apply 预检冲突：unmanaged 文件挡道 → abort 在写之前
4. apply 中途崩溃 → `doctor` 检测半成品
5. sync 远端冲突：报错不静默 merge
6. source update 顺路径：autostash + rebase 走顺
7. source update 冲突路径：rebase 失败弹 5 选项
8. project add → apply：只写 cwd，不污染 ~/
9. schema_version 不兼容：拒绝运行 + 提示
10. ${VAR} 缺失：apply 报错指明哪个变量

### 10.5 维护节奏

- agenthub 工具走 semver；hub schema_version 单调整数
- CHANGELOG.md 每版必填 user-facing 变更
- deprecation：标记后保留 1 个 major 再删
- MVP 范围冻结后新功能进 v2 backlog

## §11 设计原则与不做的事

### 11.1 强保证

- **Pre-flight collision check**：apply 在任何写操作前确认目标位置无冲突
- **原子换位**：staging → live 走 `os.rename`，单 dir 内 POSIX 原子
- **Plan-before-mutate**：所有写操作默认先 plan、确认后执行
- **不自动 git commit**：始终 Alba review 后手动 commit
- **不静默 stash**：本地改动备份到 `~/.agenthub/stash/`，不污染 git stash 栈
- **不污染项目目录**：项目级 init 不写文件到项目根（无 `.agenthub-id` 之类）

### 11.2 明确不做（MVP 范围外）

- 真正的 profile 继承（用 conf.d 片段替代）
- 中心化 sources/manifest 表（YAGNI；现场扫 sidecar）
- 自动定期 update（永远 Alba 显式触发）
- 跨 skill 依赖跟踪（让上游解决）
- 移动目录做启停（用软链有无）
- Lua/bash 第三方 target 插件（用 Python entry-points）
- Whole-world rebuild（per-symlink ops + manifest）
- 静默销毁式 sync（永远 plan 先行）

## §12 Phase 2 候选清单

- `agenthub mcp adopt --from-settings ...` 从 settings.json 反抽 MCP
- `agenthub mcp test <name>` 启动验证
- 真 profile 继承（`extends = "..."`）
- 真正的 Target plugin API（外部 `pip install agenthub-target-zed`）
- 1Password / Keychain 集成替代 secrets.env
- `agenthub web` 本地 UI 浏览 hub
- `agenthub export --format claude-plugin` 派生 marketplace.json
- 项目模板：`agenthub project init --template python` 预填一组 skill

## §13 决策摘要表

| # | 维度 | 选择 |
|---|---|---|
| 1 | CLI 语言 | Python 3.11+ |
| 2 | CLI 命令名 | `agenthub` |
| 3 | 工具范围 | 五大支柱（content / profile / source / plugin / mcp） |
| 4 | Profile 语义 | 宽义（dotfiles + 启用内容） |
| 5 | Profile 作用域 | per-tool |
| 6 | Profile 继承 | 无；用 conf.d 片段组合 |
| 7 | 项目级安装 | CLI 驱动 + hub 内集中 config |
| 8 | 项目身份 | 显式命名 + 本机映射缓存 |
| 9 | 多机同步 | hub repo 走 git；本机 state 不同步；sync ≠ apply |
| 10 | 密钥处理 | `${VAR}` 模板 + secrets.env |
| 11 | 仓库切分 | 双仓（工具仓 + 内容仓） |
| 12 | 内容子集声明 | name + @tag |
| 13 | Tag 位置 | SKILL.md frontmatter `metadata.tags`（Codex 风） |
| 14 | Source 元数据 | sidecar（intent）+ lock 双文件，对齐 marketplace.json `source.source` 分类 |
| 15 | Apply 流程 | 分阶段 + pre-flight + staging + 原子换位 + manifest |
| 16 | 冲突处理 | autostash+rebase 默认；失败弹 stash/abort/diff/merge/detach |
| 17 | Dry-run | 三层（语义/文件/内容 diff） |
| 18 | Target 抽象 | Protocol + entry-point 注册（内部 seam，MVP 不暴露） |
| 19 | Envelope | apiVersion/kind 写在每个持久化文件顶部 |
| 20 | Schema 版本 | 单调整数；启动握手；migrate 工具留待 v2 |

## §14 附：Alba 本机状况快照（实现阶段参照）

- Claude Code skills 已装：agent-browser, create-skill, find-skills, frontend-design, obsidian-cli, obsidian-markdown, generating-tutorials（7 个，多为符号链接到 ~/.agents/skills/）
- Claude Code plugin：superpowers@claude-plugins-official v5.0.7
- Codex：`~/.codex/{config.toml, skills/.system/, plugins/cache/, memories/}` 已有结构
- OpenCode：`~/.config/opencode/{opencode.json, oh-my-openagent.json, package.json, bun.lock}` 已有配置（`oh-my-openagent` 是 npm plugin）
- 项目根：`/Users/fsm/project/MyProject/alba-agent-tool-hub`（已 init git）
