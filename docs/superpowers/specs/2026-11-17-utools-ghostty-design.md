# utools-ghostty 设计文档

日期：2026-11-17
状态：待用户批准

## 意图与动机

用户以 uTools 为日常启动器（不使用 Raycast），希望把 Raycast 平台 jarry_chung 的 Ghostty
扩展（https://www.raycast.com/jarry_chung/ghostty，约 1.85 万安装）的 7 项能力**完整**复刻到
uTools，装入本仓库 `utools-ghostty`（当前仅有最小骨架：打开 Ghostty.app）。

用户已明确范围：**全部 7 项都要**。

## 目标

- 7 项能力全部落地，交互按 uTools 习惯适配（关键字唤起、键盘优先）
- 离线可用：无 CDN、无外部网络依赖，node 依赖以 vendor 文件形式打包
- 不依赖 Ghostty shell integration（通过 AppleScript + pid/ps 获取信息）
- 启动配置 YAML 与 Raycast/Warp 格式兼容，已有配置可直接粘贴迁移

## 非目标

- 不做 Windows/Linux（Ghostty AppleScript 仅 macOS，`platform: ["darwin"]`）
- 不做主题切换器（Raycast 版也没有）
- 不复刻 Raycast 的"鼠标移到窗口中心"（CGWarpMouseCursorPosition）：需要辅助功能授权，
  收益低。聚焦仅用 `activate window`

## 反目标（技术成了也算失败）

- 卡顿：列表渲染超过 300ms，或按键导航明显延迟
- 写坏用户的 `~/.config/ghostty/config`（必须保留注释、空行、顺序，仅改动目标行）
- 频繁权限弹窗（AppleScript 自动化授权应只需一次，授予 uTools 控制 Ghostty）

## 约束

- uTools 插件 = Electron 渲染进程 + preload（Node 环境）。所有系统能力（AppleScript、
  ps、文件读写）经 preload 中 `child_process` / `fs` 完成
- Ghostty 1.2+ 的 AppleScript API（本机 1.3.2-main+tip 已验证 sdef：`new window` /
  `new tab` / `split` / `focus` / `activate window` / `select tab` / `input text` 命令，
  `id` / `pid` / `tty` / `working directory` / `initial working directory` /
  `initial input` / `name` 属性）。其中改名（③）依赖 `name override`（上游 2026-10-26
  合入，需 > f54340ad7 构建）；运行时检测到缺失则提示升级，其余能力仍 1.2+ 可用
- 无构建步骤：vanilla HTML/CSS/JS，不引入框架
- 最终经 uTools 开发者工具打包为 `.upx` 上架插件市场

## 方案（Approaches）

范围（7 项全做）已由用户确认。技术路线取舍如下：

### 自动化底座

- **A. AppleScript 为主（选定）**：`osascript` 调 Ghostty AppleScript API，与 Raycast 版
  同源。覆盖枚举/聚焦/改名/建窗/建标签/分屏/注入命令全部需求。代价：首次使用触发一次
  macOS「uTools 想控制 Ghostty」授权；Ghostty < 1.2 不可用（提示升级）。
- B. CLI `open -na Ghostty.app --args` 为主：无自动化授权，但无法枚举/聚焦/改名已有
  surface，能力③做不到，④ 的命令注入不可靠。**放弃**。

### 前端栈

- **A. vanilla 无构建（选定，我的裁定）**：零依赖、包小、市场审核简单；代价是配置
  编辑器 UI 代码量偏大。
- B. Vue3 内嵌（global build 打进包里）：编辑器好写，但需 vendor 约 120KB 框架。
  **放弃**。

### 入口结构（我的裁定）

uTools 惯例是每个能力一个关键字；同时保留一个 hub 入口兼顾可发现性：

| 入口 | 类型 | 行为 |
|---|---|---|
| `ghostty` | 关键字 | hub 列表：新窗口 / 新标签 / 搜索标签 / 启动配置 / Git 仓库 / 编辑配置 / 设置 |
| `gtw` / `gtn` | 关键字 | 直达：新窗口 / 新标签 |
| `gtt` | 关键字 | 直达：搜索标签 |
| `gtl` | 关键字 | 直达：启动配置列表 |
| `gtr` | 关键字 | 直达：Git 仓库列表 |
| `gtc` | 关键字 | 直达：配置编辑器 |
| 拖拽文件/文件夹进 uTools | over 指令 | 用 Ghostty 打开（对应 Raycast ⑥） |

所有关键字进入同一 `index.html`，按 `onPluginEnter` 的 `code` 分发视图。

## 各能力设计

### ① 新建窗口 / ② 新建标签

- `osascript` 执行 `tell application "Ghostty" to make new window`（新建标签在窗口
  上下文 `make new tab`）
- 关键点（承接 Raycast 的修复）：在**当前桌面空间**打开新窗口，不抢焦点切换空间——
  实现期对照 Raycast `src/utils/applescript.ts` 的写法
- 设置项「打开后聚焦 Ghostty」默认开启；关闭时不 activate，仅建窗

### ③ 搜索标签

- 枚举：windows → tabs → terminals，逐 surface 取 `id` / `pid` / `tty` /
  `working directory` / `name`
- 前台进程：preload 中 `ps -p <pid> -o comm=` 解析
- 过滤维度：surface 标题、tab 标题、工作目录、前台进程、tty（子串匹配 + 简易拼音
  首字母）；shell 已退出的 surface（pid 不存活）自动隐藏
- 动作：回车按稳定 `id` 聚焦（`activate window` + `select tab`，地址用 id 避免列表
  过期后误跳）；`⌘.` 复制工作目录；`⌘E` 改名——**用户已定夺（2026-11-17）：升级
  Ghostty 至最新 tip**，用 `set name override of <surface>`（与 Raycast 完全一致）：
  可给单个分屏设置别名，置空恢复原标题；别名由 Ghostty 持有、随会话生灭，插件不落盘
- UI：输入框过滤 + 列表（标题 / 目录 / 进程三行信息）

### ④ 启动配置（YAML）

- schema 与 Raycast/Warp 完全一致：
  `name`（必填）、`windows[]`（每项一窗）、`tabs[]`（每项一标签）、`title`（可选，
  缺省取目录名）、`layout.cwd`（支持 `~`）、`layout.commands[]`（顺序执行）、
  `layout.split_direction`（vertical=右侧 / horizontal=下方）、嵌套 `panes[]`
  （首个并入当前 pane，其余开分屏）
- 存储：`utools.dbStorage`，key 为 slugify(name)，value 为 YAML 原文；与 Raycast 存储
  格式同构，可直接粘贴迁移
- 管理：插件内列表（选择执行 / ⌘E 编辑 / ⌘⌫ 删除 / 新建=粘贴 YAML + js-yaml 校验，
  错误提示行号）
- 执行（与 Raycast 行为完全对齐，已核实其 `ghostty-api.ts`）：
  - **查重优先**：已有 window/tab 的名字匹配 workspace 名/窗口标题，或 tab 内某
    surface 的 working directory 等于目标目录时，直接聚焦该 tab（`select tab` +
    `activate window` + `focus`），不重复打开
  - **首个 window**：Ghostty 无已有窗口、或设置「启动配置强制新窗口」开启时 →
    `new window with configuration`；否则首个 tab 作为 front window 的 `new tab`
    （后续 window 才各开新窗）
  - 分屏按 panes 树 `split <parent> direction <dir> with configuration`
  - 命令注入：`input text <cmd> to <pane>` + `send key "enter"`（与 Raycast 相同）
  - tab 标题：`perform action "set_tab_title:<title>"`；tab/window 无标题时按 Raycast
    规则用目录名兜底

### ⑤ 从 Git 仓库打开

- 设置页配置：扫描根目录（默认 `~/Developer`，首次使用引导设置）+ maxDepth（默认 3）
- 递归扫描含 `.git` 的目录（`.git` 为目录或文件均算，覆盖 worktree/submodule），
  realpath 去重防符号链接环路（算法同 Raycast `workspaces.ts`）
- 选择仓库后：列出「目录覆盖兼容」的启动配置（全配置 ≤1 个唯一 cwd，算法同 Raycast）
  + 「仅打开 shell」兜底项
- 选中后走 ④ 的执行逻辑，所有 cwd 覆盖为所选仓库路径
- 列表按 mtime 排序，显示仓库目录名与相对路径

### ⑥ 用 Ghostty 打开（over 拖拽指令）

- feature 声明：`{ "type": "over", "label": "用 Ghostty 打开", "fileType": ["file", "directory"] }`
- payload 为路径数组：目录直接打开；文件取父目录；多选则首项开窗口、其余开标签
- 进入后显示待打开路径确认列表，回车执行

### ⑦ 配置编辑器

- 读写 `~/.config/ghostty/config`；解析按行处理，保留注释/空行/顺序，仅改写目标行
  （行号级编辑，不重排整文件）
- 已知选项元数据打包为 `options.json`（key / group / type / enum options / 默认值 /
  中文说明），构建期用脚本 `scripts/gen-options.mjs` 从 ghostty.org/docs/config/reference
  抓取生成 + 人工校对核心项，可重复运行更新
- 视图：分组浏览（Font / Colors / Cursor / Window / macOS / Mouse & Scroll /
  Clipboard / Shell / Keybindings / General，同 Raycast 分组）+ 搜索（键名/说明）
- 编辑：色值显示色板（hex 检测）、enum 下拉、boolean 开关、其余文本；未知键可自由
  编辑值；「添加选项」从已知元数据搜索添加，写入文件末尾（附中文注释）
- 写入前备份一次 `config.bak`（覆盖式，仅一份）

### 设置页

`utools.dbStorage` 存储：Git 扫描根目录 / maxDepth / Ghostty.app 路径（默认
`/Applications/Ghostty.app`）/ 打开后是否聚焦 / 启动配置是否强制新窗口（对应 Raycast
的 openInNewWindow 偏好）。别名由 Ghostty 自身持有（`name override`），插件不存别名数据。

## 技术底座

- `src/preload.js`：Node 侧唯一出口。封装 `osascript` 执行（execFile + 超时）、
  js-yaml（vendor 于 `src/vendor/js-yaml.min.js`）、ps 解析、config 文件读写、
  git 仓库扫描；经 `exports` 暴露为 `window.services.*`
- 渲染层：单 `src/index.html`，`onPluginEnter` 按 code 切换视图；共享列表渲染器
  （↑↓ 选择、回车执行、`⌘数字` 快捷动作、输入过滤防抖）；样式贴近 uTools 原生列表观感
- 权限说明（写入 README）：首次使用触发 macOS 自动化授权（uTools 控制 Ghostty），
  一次性；Ghostty 版本 < 1.2 时给出升级提示
- `plugin.json`：7 个 feature code（hub + 6 直达 + over），`platform: ["darwin"]`

## 验证

- 手动 E2E 清单（uTools 开发者工具中逐项验证）：每个能力一条带预期结果的步骤
- YAML 兼容：Raycast README 的三个示例配置原样粘贴可解析、可执行
- 配置编辑器：对带注释的现有 config 改一项后 diff，仅目标行变化
- 搜索标签：改名/复制目录/聚焦三个动作在多窗口多标签下正确寻址

## 留给实现者的

- AppleScript 具体语句以 sdef 实测为准（如 `new tab` 需在 window 上下文中执行的细节）
- `input text` 与 `initial input` 的注入可靠性实测，按结果选择主/备路径
- over 指令 payload 的具体结构以 uTools 开发者文档实测为准
- `options.json` 抓取脚本的网络解析细节与核心项校对范围
