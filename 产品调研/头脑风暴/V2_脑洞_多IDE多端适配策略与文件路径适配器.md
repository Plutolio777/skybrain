# SkyBrain 解决方案 V2 — 脑洞探索：多 IDE/多端适配策略与文件路径适配器

> 状态：讨论中
> 阶段：V2 解决方案与产品雏形
> 核心问题：开发团队使用的 vibecoding 工具繁多（Cursor / Cline / Windsurf / Copilot / Trae / Claude Code / Antigravity），每种工具的规则文件路径和格式各有差异。SkyBrain 的 Local Daemon 如何做到"一套知识，多端无缝投递"？

---

## 一、问题缘起：vibecoding 工具碎片化

SkyBrain 的 L1 轨道（静态上下文注入）依赖在项目目录中写入特定格式的规则文件。但现实是：

> **每个 vibecoding 工具读取的规则文件路径都不同。** 如果不做适配，SkyBrain 只能服务单一工具（如 Cursor），产品覆盖范围严重受限。

同时，每个工具的规则能力也存在差异 —— 有的支持按文件 glob 条件生效，有的只支持全量生效，有的支持 YAML frontmatter，有的只能纯 Markdown。

---

## 二、核心设计理念：内容标准化 + 文件路径适配器

> **SkyBrain Local Daemon 内部维护一套统一的 Markdown 规则表示，然后根据检测到的当前 IDE/工具类型，将规则写入对应的文件路径。本质上就是一个"多路径文件同步器"。**

三层架构：

```
┌─────────────────────────────────────┐
│         SkyBrain 云端大脑             │
│  统一规则内容 (Markdown + 元数据)      │
└─────────────┬───────────────────────┘
              │ WebSocket 实时推送
┌─────────────▼───────────────────────┐
│       Local Daemon (本地)            │
│  ┌─────────────────────────────┐    │
│  │     工具探测引擎              │    │
│  │  (检测当前使用的 IDE/工具)     │    │
│  ├─────────────────────────────┤    │
│  │     规则翻译引擎              │    │
│  │  (统一格式 → 各工具原生格式)   │    │
│  ├─────────────────────────────┤    │
│  │     文件路径适配器            │    │
│  │  (写入各工具对应的文件路径)    │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

---

## 三、调研成果：主流 vibecoding 工具配置全景

> 调研时间：2026年5月 · 仅聚焦于支持"AI Agent 模式 + MCP"的 vibecoding 工具  
> JetBrains 系列、传统编辑器不在此列（它们不属于 vibecoding 工具范畴）

### 3.1 各工具规则文件路径速查

#### 项目级（Workspace-level）规则路径

```
项目根/
│
├── Cursor ─────────────────────────────────────────
│   .cursor/rules/*.mdc          ← 【v2 推荐】多文件规则目录（含 YAML frontmatter）
│   .cursorrules                 ← 【v1 已废弃】单文件
│
├── Windsurf ───────────────────────────────────────
│   .windsurfrules              ← 项目级单文件规则
│   .windsurf/rules/*.md        ← 【Wave 8+】多文件规则目录
│
├── Cline ──────────────────────────────────────────
│   .clinerules/                ← 多文件规则目录（自动兼容 .cursorrules/.windsurfrules/AGENTS.md）
│   .clinerules                  ← 单文件（旧格式，仍支持）
│
├── GitHub Copilot ──────────────────────────────────
│   .github/copilot-instructions.md               ← 仓库级，对所有 Copilot 请求生效
│   .github/instructions/*.instructions.md        ← 路径级，仅匹配文件时激活
│
├── Trae ───────────────────────────────────────────
│   .trae/rules/*.md            ← 项目规则目录
│
├── Claude Code ────────────────────────────────────
│   CLAUDE.md                   ← 项目根指令（会话启动时自动加载）
│   .claude/rules/*.md          ← 多文件规则目录
│
├── Antigravity (Google) ──────────────────────────
│   .agent/rules/*.md           ← 工作区级规则目录
│
└── Roo Code (VS Code 扩展) ───────────────────────
    .roo/rules/*.md             ← 工作区规则目录（推荐）
    .roorules                   ← 单文件（回退方案）
```

#### 全局级（Global/User-level）规则路径

| 工具 | macOS / Linux | Windows |
|---|---|---|
| **Cursor** | `~/.cursor/` (CLI config) | `%USERPROFILE%\.cursor\` |
| **Windsurf** | `~/.codeium/windsurf/memories/global_rules.md` | `%USERPROFILE%\.codeium\windsurf\memories\global_rules.md` |
| **Cline** | `~/Documents/Cline/Rules/` 主 · `~/.cline/rules/` · `~/Cline/Rules/` 回退 | `Documents\Cline\Rules\` |
| **GitHub Copilot** | `$HOME/.copilot/copilot-instructions.md` | `%USERPROFILE%\.copilot\copilot-instructions.md` |
| **Trae** | 通过 IDE Settings 中心配置 | 同 |
| **Claude Code** | `~/.claude/CLAUDE.md` · `~/.claude/rules/` | `%USERPROFILE%\.claude\` |
| **Antigravity** | `~/.gemini/GEMINI.md` | `%USERPROFILE%\.gemini\GEMINI.md` |
| **Roo Code** | `~/.roo/rules/` | `%USERPROFILE%\.roo\rules\` |

### 3.2 各工具规则能力差异

| 工具 | 规则格式 | 条件规则 | 条件语法 | 规则类型/模式 | 跨工具兼容 |
|---|---|---|---|---|---|
| **Cursor** | Markdown + YAML frontmatter | ✅ | `globs: "src/**/*.tsx"` | Always / Auto Attached / Agent Requested | — |
| **Windsurf** | 纯 Markdown | ⚠️ 手动 | 无标准语法 | 全量 / 手动引用 | — |
| **Cline** | Markdown / .txt | ✅ | `paths: "src/components/**"` (YAML frontmatter) | 无条件 / 条件匹配 | ✅ 自动读取 .cursorrules / .windsurfrules / AGENTS.md |
| **GitHub Copilot** | 纯 Markdown | ✅ | `.github/instructions/*.instructions.md` 路径分离 | 仓库级 / 路径级 | — |
| **Trae** | Markdown | ✅ | `globs: "*.js, src/**/*.ts"` | Always / Apply to Specific Files / Intelligent / Manual | — |
| **Claude Code** | 纯 Markdown | ❌ | — | 全量加载 | ✅ CLAUDE.md 通用格式 |
| **Antigravity** | 纯 Markdown | ⚠️ 待确认 | 待确认 | 全局 / 工作区 | ⚠️ 与 Gemini CLI 共用 `~/.gemini/GEMINI.md` |
| **Roo Code** | Markdown / .txt | ⚠️ 模式级隔离 | `.roo/rules-{mode}/` 目录分离 | 全模式 / 特定模式 | ✅ 可使用 Cursor .mdc 文件 |

### 3.3 关键发现

1. **路径规律极其明显** — 所有工具的项目级规则都在 `.工具名/rules/` 或 `.工具名规则文件` 模式下，枚举成本很低。Local Daemon 的工具探测只需扫描项目根是否存在这些已知目录/文件。

2. **格式高度统一** — 全部是 Markdown（最多加 YAML frontmatter）。SkyBrain 只需生成一套内容，适配器只管"写到哪里"。

3. **Cline 的兼容策略值得借鉴** — 它自动读取 `.cursorrules`、`.windsurfrules`、`AGENTS.md`，说明行业正在自发趋向互认。SkyBrain 如果写入 `.cursorrules`，实际上同时服务了 Cursor 和 Cline 用户。

4. **Antigravity 与 Gemini CLI 存在路径冲突** — 两者共用 `~/.gemini/GEMINI.md`，已有社区 GitHub issue，未来 Google 可能会拆分。SkyBrain 目前可以不处理该冲突，等 Google 官方解决。

5. **Copilot 的 `.github/` 路径是版本控制的** — 写入 `.github/copilot-instructions.md` 会产生 Git 变更，需考虑是否加入 `.gitignore` 或提供独立于 Git 的同步机制。

---

## 四、基于调研的适配器设计

### 4.1 工具探测机制（检测层）

Local Daemon 通过**文件系统扫描**即可确定当前开发者使用的工具：

```
探测逻辑（伪代码）：
1. 扫描项目根目录：
   - 存在 .cursor/rules/  → Cursor
   - 存在 .windsurfrules   → Windsurf
   - 存在 .clinerules/     → Cline
   - 存在 .github/copilot-instructions.md → Copilot
   - 存在 .trae/rules/     → Trae
   - 存在 CLAUDE.md        → Claude Code
   - 存在 .agent/rules/    → Antigravity
   - 存在 .roo/rules/      → Roo Code

2. 进程扫描（辅助）：
   - 检查运行中进程名含 "Cursor" / "Windsurf" / "Trae" 等

3. 开发者手动选择（兜底）：
   - Local Daemon 托盘菜单提供工具列表
```

### 4.2 工具能力画像（兼容层）

提前为每个工具维护一个能力画像，确保规则生成时做对降级：

```json
{
  "cursor": {
    "project_rules_dir": ".cursor/rules/",
    "file_extension": ".mdc",
    "conditional_rules": true,
    "condition_syntax": "yaml_frontmatter.globs",
    "max_file_size": "unlimited"
  },
  "windsurf": {
    "project_rules_file": ".windsurfrules",
    "project_rules_dir": ".windsurf/rules/",
    "file_extension": ".md",
    "conditional_rules": false,
    "condition_syntax": null
  },
  "cline": {
    "project_rules_dir": ".clinerules/",
    "file_extension": ".md",
    "conditional_rules": true,
    "condition_syntax": "yaml_frontmatter.paths"
  },
  "copilot": {
    "project_rules_file": ".github/copilot-instructions.md",
    "project_rules_dir": ".github/instructions/",
    "file_extension": ".instructions.md",
    "conditional_rules": true,
    "condition_syntax": "separate_files_by_path"
  },
  "trae": {
    "project_rules_dir": ".trae/rules/",
    "file_extension": ".md",
    "conditional_rules": true,
    "condition_syntax": "frontmatter.globs"
  },
  "claude_code": {
    "project_rules_file": "CLAUDE.md",
    "project_rules_dir": ".claude/rules/",
    "file_extension": ".md",
    "conditional_rules": false
  },
  "antigravity": {
    "project_rules_dir": ".agent/rules/",
    "file_extension": ".md",
    "conditional_rules": false
  },
  "roo_code": {
    "project_rules_dir": ".roo/rules/",
    "file_extension": ".md",
    "conditional_rules": true,
    "condition_syntax": "mode_directory_separation"
  }
}
```

### 4.3 优雅降级策略

SkyBrain 始终以"最大能力"格式存储规则，适配器负责降维翻译：

| 降级场景 | 策略 |
|---|---|
| Cursor → Windsurf | 去 YAML frontmatter，条件规则的 globs 转为文字描述（如"以下规则仅适用于 `src/components/` 目录"） |
| Cursor → Copilot | 条件规则拆分为独立 `.instructions.md` 文件放入 `.github/instructions/` |
| Cursor → Claude Code | 所有规则合并为纯 Markdown，去 YAML frontmatter，写入 `CLAUDE.md` |
| Cursor → Antigravity | 同 Claude Code 处理，写入 `.agent/rules/` |

---

## 五、L1/L2 静态规则同步策略：手动同步优先，L3 无同步需求

### 5.1 三层知识中哪些需要同步

根据三层知识分级模型，同步需求按投递通道划分：

| 层级 | 知识类型 | 投递通道 | 同步需求 | 原因 |
|---|---|---|---|---|
| **L1** | SkyBrain 元知识 | 磁盘写入（静态文件） | ✅ 需要同步 | 涉及**磁盘写入**操作，必须由开发者主动触发 |
| **L2** | 项目级固化知识 | 磁盘写入（静态文件） | ✅ 需要同步 | 同 L1，来自云端团队大脑，项目知识变更后需更新本地文件 |
| **L3** | 动态扩展知识 | MCP 实时查询 | ❌ 不需要同步 | 每次调用都是**实时查询云端**，天然就是最新数据 |

> L1（元知识）的更新走 Daemon 版本升级通道，L2（固化知识）的更新走云端同步通道。两者最终都写入磁盘，因此共享同一套同步策略。

### 5.2 初期方案：手动点击同步

**核心原则**：L1 + L2 规则写入磁盘是一个**有副作用的操作**（可能覆盖开发者已有规则），不应在后台静默执行。

```
开发者工作流：
  1. 打开 Local Daemon 托盘面板
  2. 查看当前静态规则状态（红色标记 = 有更新未同步）
     - L1 元知识版本 / 状态
     - L2 项目知识版本 / 状态 / 最后同步时间
  3. 点击 [同步] 按钮
  4. Daemon 拉取最新 L1（如有）+ 云端最新 L2 → 合并渲染为工具原生格式 → 写入目标文件
  5. 同步完成提示 + 知识变更摘要展示
```

**Local Daemon 面板信息**：
- 当前已激活的 IDE 工具图标
- L1 元知识状态（Daemon 内置，通常不变）
- L2 项目知识版本号 / 最后同步时间
- 是否有待同步更新（红色小圆点标记）
- [立即同步] 按钮

### 5.3 后期可选优化

| 方案 | 触发方式 | 适用场景 |
|---|---|---|
| **启动时提醒** | Daemon 启动时检测到 L2 版本落后，托盘图标变红，但不自动写入 | 让开发者感知到有更新，但保留控制权 |
| **智能静默同步** | 仅当检测到开发者**没有**该工具的已有自定义规则时，自动执行 | 新项目冷启动，无冲突风险 |
| **TL 广播通知** | TL 审批关键规则时，可选 "全员推送通知" | 紧急安全规范更新 |

### 5.4 同步范围

每次同步时，Local Daemon 为所有已激活的工具生成规则文件（L1 元知识 + L2 固化知识合并写入）：

```
┌─────────────────────────────────────────────────┐
│              Local Daemon 同步引擎               │
├─────────────────────────────────────────────────┤
│  已激活工具: [Cursor, Cline]                     │
│                                                 │
│  开发者点击 [同步] 后:                            │
│    1. 合并 L1 元知识 + 云端最新 L2 固化知识        │
│    2. 按工具能力画像降级翻译                       │
│    3. 为 Cursor 渲染 → .cursor/rules/*.mdc       │
│    4. 为 Cline  渲染 → .clinerules/*.md          │
│                                                 │
│  注：因为 Cline 已自动读取 .cursorrules，         │
│  写 Cursor 格式可能同时服务 Cline 用户。          │
│  但仍单独写 .clinerules/ 以保证条件规则正确加载。   │
└─────────────────────────────────────────────────┘
```

---

## 六、待进一步探讨的问题

1. **与开发者已有规则的共存**：追加？覆盖？标记冲突？
2. **L1 + L2 的 Token 预算裁剪**：哪些知识进静态文件，哪些仅保留在 L3 MCP 按需检索？
3. **规则变更通知**：TL 审批新规则后，开发者如何感知到需要同步更新？

> 这些将在后续单独的子脑洞中逐一展开探讨。

---

## 七、结论

通过系统调研，我们确认了 SkyBrain 多端适配与规则同步的核心策略是可行的：

> **"内容标准化 + 文件路径适配器 + 手动同步"** —— SkyBrain 生成一套统一的 Markdown 规则内容（L1 元知识 + L2 固化知识），Local Daemon 通过工具探测识别当前 IDE，再根据工具能力画像将规则翻译并写入对应路径。开发者通过托盘面板手动触发同步，L3 动态知识完全无需同步（MCP 实时查询）。

这一策略的优势在于：
- **低维护成本**：工具端的规则路径和格式差异在适配层隔离，核心内容不受影响
- **天然兼容 Cline**：Cline 已支持读取 Cursor/Windsurf 格式，写一份可能服务多端
- **扩展友好**：未来新出 vibecoding 工具时，只需新增一条能力画像 + 路径映射即可接入
- **安全可控**：手动同步避免了静默覆盖开发者已有规则的风险，L3 无需同步减少了不必要的磁盘写入