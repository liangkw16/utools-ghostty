# utools-ghostty 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 uTools 中完整复刻 Raycast Ghostty 扩展的 7 项能力（新窗口/新标签/搜索标签/YAML 启动配置/Git 仓库打开/拖拽打开/配置编辑器）。

**Architecture:** preload（Node）承载全部系统能力（osascript、ps、fs、utools.dbStorage），经 `window.services` 暴露；纯逻辑（YAML 校验、布局展开、AppleScript 生成、config 解析、模糊匹配）放 `src/lib/`，UMD-lite 模式（浏览器 `<script>` 下为全局函数，Node `require` 下为 module.exports），用 Node 内置 test runner 单测；渲染层单 `index.html` 按 `onPluginEnter` code 切视图，共享键盘导航列表组件。

**Tech Stack:** vanilla HTML/CSS/JS（无构建、无框架）、Node `node:test`、vendor 的 js-yaml 4.x、AppleScript/JXA（osascript）。

**Spec:** `docs/superpowers/specs/2026-11-17-utools-ghostty-design.md`（计划以 spec 为准，执行者须同时读 spec）

## Global Constraints

- `platform: ["darwin"]`；`pluginId` 保持 `com.liangkw.ghostty` 不变
- 无构建步骤：vanilla HTML/CSS/JS，禁止引入前端框架；运行时零网络请求
- 唯一运行时依赖：js-yaml（vendor 于 `src/vendor/js-yaml.min.js`，UMD，preload 中 `require`）
- 测试仅用 Node 内置 `node --test test/`，不引入测试框架；Node ≥ 18
- `src/lib/*.js` 一律 UMD-lite：顶层函数声明 + 文件尾
  `if (typeof module !== "undefined") module.exports = { ... }`
- UI 文案中文；列表渲染性能目标 < 300ms（输入过滤需防抖）
- `~/.config/ghostty/config` 写入必须保留注释/空行/顺序（行级编辑），写前备份一次 `config.bak`
- 别名（改名）不落插件存储：由 Ghostty `name override` 持有
- Ghostty ≥ 1.2；改名依赖 `name override`（> f54340ad7 构建），运行时缺失则提示升级、不崩溃
- preload 是渲染层唯一后端出口：渲染层只调 `window.services.*` 与 `utools.*` UI API

## Review Focus

1. **重复执行同一启动配置必须聚焦已有**（dedup 分支返回 `focused-existing`，不重开窗口）→ Task 3 测试 `launch dedups to focused-existing`
2. **`commands[]` 的 `- exec:` 对象与纯字符串混用都要能跑**（Raycast/Warp 兼容）→ Task 1 测试 exec 对象解析、Task 2 测试 join
3. **surface 列表过期（tab 已关/位置变了）**：聚焦按 id 寻址而非 index，AppleScript 报错归一化提示 → Task 3 测试 id 寻址、Task 6 测试错误归一化
4. **config 含注释、空行、值内含 `=`**：解析与回写只动目标行 → Task 5 测试
5. **Ghostty 未运行/零窗口 + 路径含空格/引号/`~`**：先 `launch` 再操作；`quoteAppleString` 转义 + `~` 展开 → Task 3 测试 launch 守卫与转义

---

### Task 1: 测试底座 + js-yaml vendor + 启动配置校验模块

**Files:**
- Create: `package.json`、`src/vendor/js-yaml.min.js`、`src/lib/launch-config.js`、`test/launch-config.test.js`

**Interfaces:**
- Consumes: 无（首个任务）
- Produces:
  - `parseLaunchConfig(yaml: string): LaunchConfig`（失败 throw Error，中文消息）
  - `validateLaunchConfigYaml(yaml?: string): string | undefined`
  - `isDirectoryOverrideCompatible(config: LaunchConfig): boolean`（全配置唯一 cwd ≤ 1）
  - `slugify(name: string): string`（小写、trim、空白→`-`）
  - `mergeConfigStore(store: Record<string, string>, yaml: string, prevSlug?: string): { store: Record<string,string>, slug: string }`（重命名时删旧 key）
  - `deleteConfigFromStore(store: Record<string, string>, slug: string): Record<string, string>`
  - 类型：`LaunchConfig = { name: string, windows: Array<{ tabs: Array<{ title?: string, layout: PaneNode }> }> }`；`PaneNode = { cwd?: string, commands?: Array<string | { exec: string }>, split_direction?: "vertical" | "horizontal", panes?: PaneNode[] }`
  - `require("../../src/vendor/js-yaml.min.js")` 提供 `.load/.fail`（后续任务在 Node 侧使用）

- [ ] **Step 1: 写失败测试**

```js
// test/launch-config.test.js（节选关键断言，其余同风格补全）
const test = require("node:test"); const assert = require("node:assert");
const { parseLaunchConfig, validateLaunchConfigYaml, isDirectoryOverrideCompatible,
        slugify, mergeConfigStore, deleteConfigFromStore } = require("../src/lib/launch-config");

test("parse 接受 exec 对象与纯字符串两种 commands", () => {
  const cfg = parseLaunchConfig(`name: demo\nwindows:\n  - tabs:\n      - layout:\n          commands:\n            - exec: npm run dev\n            - echo hi\n`);
  assert.equal(cfg.name, "demo");
  const cmds = cfg.windows[0].tabs[0].layout.commands;
  assert.deepEqual(cmds, [{ exec: "npm run dev" }, "echo hi"]);
});
test("parse 缺 name / 缺 windows 抛中文错误", () => { /* ... */ });
test("exec 对象缺 exec 字段报错", () => { /* ... */ });
test("isDirectoryOverrideCompatible：多 cwd → false，单/零 cwd → true", () => { /* ... */ });
test("slugify 与 store 增删改（重命名删旧 key）", () => { /* ... */ });
```

- [ ] **Step 2: 运行确认失败**

Run: `node --test test/launch-config.test.js`
Expected: FAIL（模块不存在）

- [ ] **Step 3: 建底座与实现 `src/lib/launch-config.js`**

- `package.json`：`{ "name": "utools-ghostty", "private": true, "scripts": { "test": "node --test test/" } }`（零依赖）
- vendor：`curl -fsSL https://cdn.jsdelivr.net/npm/js-yaml@4.1.0/dist/js-yaml.min.js -o src/vendor/js-yaml.min.js` 后提交（离线使用）
- 实现：`require("../vendor/js-yaml.min.js")` 解析；递归收集全部 `cwd` 判兼容；遍历 PaneNode 校验 commands 元素为 string 或含非空 `exec` 的对象

- [ ] **Step 4: 运行确认通过**

Run: `npm test`
Expected: 全部 PASS

- [ ] **Step 5: Commit**

```bash
git add package.json src/vendor src/lib/launch-config.js test/launch-config.test.js
git commit -m "feat: launch config validation with vendored js-yaml"
```

---

### Task 2: 布局展开模块（YAML → 扁平 pane 计划）

**Files:**
- Create: `src/lib/layout.js`、`test/layout.test.js`

**Interfaces:**
- Consumes: Task 1 的 `LaunchConfig` / `PaneNode` 类型
- Produces: `buildPlan(config: LaunchConfig, opts?: { directory?: string, forceNewWindow?: boolean }): Plan`
  - `Plan = { workspaceName: string, directory: string, windowTitle: string, tabs: Array<{ title: string, workingDirectory: string, panes: Array<{ cwd: string, command: string | null, splitFrom: number, direction: "vertical" | "horizontal" }> }> }`
  - 规则（测试逐条钉死）：根 pane 的 `splitFrom = 0`；嵌套 `panes` 深度优先展平，节点第 i(i>0) 个子 pane 的 `splitFrom` = 父 pane 在结果数组中的下标、`direction` = 父节点 `split_direction`（缺省 vertical）；`command` = commands 的 exec 提取后以 `"; "` 连接，无则 null；`opts.directory` 覆盖全部 cwd；title 兜底 = 有效目录最后一段（`~/x/y` → `y`）；`windowTitle` = 首个有效目录最后一段；`~` 前缀展开为 `os.homedir()`

- [ ] **Step 1: 写失败测试**

覆盖：单 tab 单 pane；嵌套 panes 的 splitFrom/direction 矩阵；commands join（exec 对象+字符串混用）；directory 覆盖；title/windowTitle 兜底；`~` 展开；多窗口配置（tabs 分属两个 window）。

Run: `node --test test/layout.test.js` → FAIL

- [ ] **Step 2: 实现 `buildPlan`**

纯函数；展平用显式栈/递归均可（测试已定行为）。

- [ ] **Step 3: 运行确认通过**

Run: `npm test` → PASS

- [ ] **Step 4: Commit**（`feat: layout flattening for launch configs`）

---

### Task 3: AppleScript/JXA 生成模块

**Files:**
- Create: `src/lib/applescript-gen.js`、`test/applescript-gen.test.js`

**Interfaces:**
- Consumes: Task 2 的 `Plan`
- Produces（全部纯函数返回脚本字符串）：
  - `quoteAppleString(s: string): string`（转义 `\` 与 `"`，外层加引号）
  - `listSurfacesJXA(): string`（JXA：遍历 `Application("Ghostty")` 的 windows→tabs→terminals，输出 `JSON.stringify` 的数组，元素 `{ windowId, tabId, surfaceId, name, workingDirectory, pid, tty }`，取各对象 `id()`）
  - `newWindowScript(directory: string | null): string`（有目录时经 `new surface configuration` 设 `initial working directory`）
  - `newTabScript(directory: string | null): string`
  - `focusSurfaceScript(t: { windowId: string, tabId: string, surfaceId: string }): string`（`activate window (first window whose id is …)` + `select tab (first tab whose id is …)` + `focus (first terminal whose id is …)`，全部按 id 寻址）
  - `renameSurfaceScript(surfaceId: string, title: string): string`（`set name override of (first terminal whose id is …) to <title>`；`title === ""` 时 `to missing value` 恢复）
  - `launchScript(plan: Plan, opts: { autoFocus: boolean, forceNewWindow: boolean }): string`——Raycast 对齐语义（spec ④）：未运行先 `launch`；**查重**：已有 window/tab 名字匹配 workspaceName/windowTitle 或 tab 内 surface 的 working directory == plan.directory → `select tab`+`activate window`+`focus` 并 `return "focused-existing"`；否则首 tab：无窗口或 forceNewWindow → `new window with configuration`，否则 front window `new tab`；后续 tab 均 `new tab in targetWindow`；pane 展开按 splitFrom `split <parent> direction <dir> with configuration`；命令 `input text <cmd> to <pane>` + `send key "enter"`；tab 标题 `perform action "set_tab_title:<title>"`；结束 `return "opened-new"`

- [ ] **Step 1: 写失败测试**

关键断言（对生成文本做片段匹配）：
- `quoteAppleString('a"b\\c')` 输出 `"a\"b\\c"`
- launchScript 含 `if not running`/`launch` 守卫（Ghostty 未运行）
- **dedup**：脚本含名字/目录匹配循环与 `return "focused-existing"`（Review Focus #1）
- 有窗口且非 forceNewWindow：首 tab 走 `new tab in` front window 分支；forceNewWindow 走 `new window`
- panes：splitFrom=1 的 pane 生成 `split ... direction vertical`；command 生成 `input text` + `send key "enter"`
- focusSurfaceScript 三段均含 `whose id is`（Review Focus #3）
- renameSurfaceScript 两种形态：设置/`missing value`
- 路径含空格/引号时经 quoteAppleString 转义（Review Focus #5）

Run: `node --test test/applescript-gen.test.js` → FAIL

- [ ] **Step 2: 实现生成器**

AppleScript 语句骨架参照 spec ④ 与 Raycast `ghostty-api.ts`（设计文档已转录关键行）；JXA 枚举注意 `id()`/属性取值要函数调用。

- [ ] **Step 3: 运行确认通过**

Run: `npm test` → PASS

- [ ] **Step 4: Commit**（`feat: applescript/jxa generators`）

---

### Task 4: 模糊匹配模块

**Files:**
- Create: `src/lib/match.js`、`test/match.test.js`

**Interfaces:**
- Produces: `fuzzyScore(query: string, fields: string[]): number`——0 = 不命中；大小写不敏感；任一 field 含 query 子串 → 高分；各 field 分词首字母缩写含 query → 次高分（如 `gt` 命中 `ghostty tools`）；空 query 恒 > 0；多 field 命中叠加
- **与 spec 的偏差（实现者照此执行）**：spec ③ 的"简易拼音首字母"在此简化为英文分词首字母缩写 + 中文字符子串命中（零依赖约束下不引拼音表；拼音匹配留作后续增强）

- [ ] **Step 1: 写失败测试**（子串命中/缩写命中/不命中/空 query/大小写/中文子串）
- [ ] **Step 2: `node --test test/match.test.js` → FAIL**
- [ ] **Step 3: 实现（分词按非字母数字边界切）**
- [ ] **Step 4: `npm test` → PASS**
- [ ] **Step 5: Commit**（`feat: fuzzy matcher`）

---

### Task 5: Ghostty 配置文件解析/编辑模块

**Files:**
- Create: `src/lib/ghostty-config.js`、`test/ghostty-config.test.js`

**Interfaces:**
- Produces:
  - `parseConfig(raw: string): Line[]`，`Line = { kind: "entry" | "comment" | "blank", raw: string, key?: string, value?: string }`；entry 以**首个** `=` 切分并 trim（Review Focus #4）
  - `serialize(lines: Line[]): string`（`\n` 连接，保序）
  - `editEntryValue(lines: Line[], index: number, value: string): Line[]`（仅该行 raw 变为 `${key} = ${value}`）
  - `appendEntry(lines: Line[], key: string, value: string): Line[]`
  - `detectHexColor(value: string): string | null`（`#rgb/#rrggbb/#rrggbbaa`）

- [ ] **Step 1: 写失败测试**（注释/空行保留、值内 `=`、编辑只动一行、追加、色值检测）
- [ ] **Step 2: FAIL → 实现 → `npm test` PASS**（一步一验，同前任务节奏）
- [ ] **Step 3: Commit**（`feat: ghostty config line editor`）

---

### Task 6: preload 侧 Node services

**Files:**
- Create: `src/services/osa.js`、`src/services/tabs.js`、`src/services/workspaces.js`、`src/services/config-file.js`、`test/workspaces.test.js`、`test/osa.test.js`

**Interfaces:**
- Consumes: Task 3 生成器、Task 1 store 函数
- Produces（CommonJS，可注入依赖便于测试）：
  - `osa.js`：`createOsa(execFile)` → `{ run(script, opts?: { lang?: "AppleScript" | "JavaScript", timeoutMs?: number }): Promise<string>, runJSON<T>(jxaSource): Promise<T> }`；默认导出 `createOsa(child_process.execFile)`；超时 kill；stderr 含 `-1743` / `not authorized` → `Error("需要授权 uTools 控制 Ghostty：系统设置 → 隐私与安全性 → 自动化")`，其余 stderr 原样透传（Review Focus #3）
  - `tabs.js`：`toSurfaces(json: unknown): Surface[]`（纯函数，单测）+ `createTabs(osa, execFile)` → `{ listSurfaces(): Promise<Surface[]> }`；`Surface = { surfaceId, windowId, tabId, name, workingDirectory, pid, tty, process: string | null }`；对每个 pid 跑 `ps -p <pid> -o comm=`，失败的（shell 已退出）丢弃；process 取 comm 尾段（路径→命令名）
  - `workspaces.js`：`listGitRepos(root: string, maxDepth: number): Promise<Repo[]>`，`Repo = { name, directory, lastModifiedMs }`；含 `.git`（目录或文件）即命中并停止下探；realpath 去重；并发递归；root 支持 `~` 前缀（`os.homedir()` 展开——设置值原样存 `~/Developer`）
  - `config-file.js`：`CONFIG_PATH`（`~/.config/ghostty/config`）、`readConfig(): Promise<string>`、`writeConfig(next: string): Promise<void>`（`config.bak` 不存在时先备份再写）

- [ ] **Step 1: 写失败测试**
  - `test/osa.test.js`：注入假 execFile（记录 argv、回调可控）断言：`-l JavaScript` 参数、超时 kill、-1743 归一化文案、runJSON 解析 stdout
  - `test/workspaces.test.js`：tmp 目录夹具（嵌套仓库、`.git` 文件形式、符号链接环路、maxDepth 截断）断言输出
- [ ] **Step 2: FAIL → 实现 → `npm test` PASS**
- [ ] **Step 3: Commit**（`feat: node services for osa, tabs, workspaces, config file`）

---

### Task 7: 渲染底座 + plugin.json 全量 + hub + ①② 新窗口/新标签

**Files:**
- Modify: `plugin.json`（全量 7 feature）、`src/index.html`（重写为外壳）、`src/preload.js`（重写为 services 出口）
- Create: `src/ui.css`、`src/ui/list.js`、`src/views/hub.js`、`src/views/open.js`

**Interfaces:**
- Consumes: Task 3/4/6 全部
- Produces（渲染层约定，后续视图任务沿用）：
  - `UI.list(container, opts: { placeholder: string, items: any[], renderRow(item, el), filterFields(item): string[], onSelect(item): void, actions?: Array<{ key: string, label: string, run(item): void }> })`——过滤输入防抖（120ms）经 `fuzzyScore`；↑↓ 移动、回车 onSelect、`⌘数字` 触发对应 action；`UI.toast(msg)`、`UI.setViewHeight(px)`（内部 `utools.setExpendHeight`）
  - 视图注册：`src/views/*.js` 挂全局 `Views.<name> = (ctx) => void`；`index.html` 以 `VIEW_BY_CODE` 映射（`ghostty→hub`、`gtw/gtn→open`、`gtt→tabs`、`gtl→launch`、`gtr→repos`、`gtc→config-editor`、`open-with-ghostty→over`、任意 code+`⌘,`→settings）
  - `window.services`（preload.js 出口）：`{ launch: { newWindow(dir?|null), newTab(dir?|null), runPlan(yaml, opts?) }, tabs: { listSurfaces(), focusSurface(t), renameSurface(id, title) }, repos: { list(root, maxDepth) }, ghosttyConfig: { read(), save(text) }, settings: { get(), save(patch) }, launchStore: { list(), save(yaml, prevSlug?), remove(slug) } }`
  - `settings` 默认值（`DEFAULT_SETTINGS`，放 `src/lib/settings.js`，UMD-lite）：`{ gitRoot: "~/Developer", maxDepth: 3, ghosttyPath: "/Applications/Ghostty.app", autoFocus: true, forceNewWindow: false }`；存 `utools.dbStorage.getItem("settings")`，读取时与默认合并
  - `launchStore` 存 `utools.dbStorage.getItem("launchConfigs")` → `Record<slug, yaml>`（经 Task 1 纯函数操作）
  - plugin.json：features = `ghostty`/`gtw`/`gtn`/`gtt`/`gtl`/`gtr`/`gtc`（关键字同 code）+ `{ "code": "open-with-ghostty", "cmds": [{ "type": "over", "label": "用 Ghostty 打开", "fileType": ["file", "directory"] }] }`；`pluginSetting: { "single": true }`

- [ ] **Step 1: 重写 preload.js / plugin.json / index.html + ui.css + ui/list.js + hub / open 视图**

- [ ] **Step 2: 手动验证（uTools 开发者工具，已导入本插件）**

- 输入 `ghostty` → hub 列表 7 项（含设置），子串过滤正常，回车进入对应能力
- `gtw` → 打开新 Ghostty 窗口（不切换桌面空间）；`gtn` → 当前窗口新标签；Ghostty 未运行时先拉起
- 首次执行触发系统授权弹窗，同意后不再弹（Review Focus #5 的运行时确认）
- `npm test` 仍全绿

- [ ] **Step 3: Commit**（`feat: renderer shell, hub, new window/tab`）

---

### Task 8: ③ 搜索标签视图（gtt）

**Files:**
- Create: `src/views/tabs.js`

**Interfaces:**
- Consumes: `services.tabs.*`、`UI.list`、`fuzzyScore`（filterFields = [name, workingDirectory, process, tty]）
- Produces: `Views.tabs`；actions：`⌘E` 改名（输入框，空值恢复）、`⌘.` 复制工作目录（`utools.copyText`）

- [ ] **Step 1: 实现视图**（进入即枚举，防抖过滤；行显示 标题/目录/进程）
- [ ] **Step 2: 手动验证**

- 多窗口多标签下搜索命中标题/目录/进程/tty；回车聚焦到**精确 surface**（含分屏场景）
- 改名后 Ghostty 内该分屏标题变化、列表同步；置空恢复原标题；本机构建无 `name override` 时显示升级提示不崩溃（Review Focus #3 运行时面）
- 已退出的 shell surface 不出现在列表

- [ ] **Step 3: Commit**（`feat: surface search view`）

---

### Task 9: ④ 启动配置视图（gtl）

**Files:**
- Create: `src/views/launch.js`

**Interfaces:**
- Consumes: `services.launchStore.*`、`services.launch.runPlan(yaml, { directory?, forceNewWindow? })`（preload 内部：Task 1 parse → Task 2 buildPlan → Task 3 launchScript → osa.run）
- Produces: `Views.launch`——列表（执行/`⌘E` 编辑/`⌘⌫` 删除/`⌘N` 新建）；编辑页为 textarea + 实时校验（`validateLaunchConfigYaml`，显示行号错误）

- [ ] **Step 1: 实现视图**
- [ ] **Step 2: 手动验证**

- 粘贴 Raycast README 三个示例 YAML 均通过校验并可执行（spec 验证节）
- 重复执行同一配置 → 聚焦已有不重开（Review Focus #1 运行时面）
- 执行后回车自动 `utools.hideMainWindow()`
- 校验错误显示中文消息

- [ ] **Step 3: Commit**（`feat: launch config view`）

---

### Task 10: ⑤ Git 仓库视图（gtr）

**Files:**
- Create: `src/views/repos.js`

**Interfaces:**
- Consumes: `services.repos.list(root, maxDepth)`（设置值）、`services.launchStore.list()`、Task 1 `isDirectoryOverrideCompatible`
- Produces: `Views.repos`——仓库列表（mtime 排序，行显示 目录名/相对路径）；选中后二级列表：兼容的启动配置 + 「仅打开 shell」（= runPlan 目录覆盖、无配置场景 `services.launch.newWindow(repoPath)`）

- [ ] **Step 1: 实现视图**
- [ ] **Step 2: 手动验证**（在设置的根目录下放 2-3 个 git 仓库夹具：普通、worktree/file `.git`、深于 maxDepth 的应缺席）
- [ ] **Step 3: Commit**（`feat: git repo launcher view`）

---

### Task 11: ⑥ 拖拽打开视图（over）

**Files:**
- Create: `src/views/over.js`

**Interfaces:**
- Consumes: `onPluginEnter` payload（`type === "over"`，`payload` 为路径数组）、`services.launch.newWindow/newTab`
- Produces: `Views.over`——确认列表（目录直接用；文件取父目录；首项开窗口、其余开标签）

- [ ] **Step 1: 实现视图**
- [ ] **Step 2: 手动验证**（拖 1 个文件夹、1 个文件、多选混合三种情况）
- [ ] **Step 3: Commit**（`feat: drag-to-open view`）

---

### Task 12: ⑦ options.json 生成 + 配置编辑器视图（gtc）

**Files:**
- Create: `scripts/gen-options.mjs`、`src/assets/options.json`（生成后提交）、`src/views/config-editor.js`

**Interfaces:**
- Consumes: `services.ghosttyConfig.*`、Task 5 解析/编辑、options.json
- Produces:
  - `scripts/gen-options.mjs`：抓取 https://ghostty.org/docs/config/reference 提取选项 → `options.json`：`Array<{ key: string, group: string, type: "string" | "boolean" | "number" | "color" | "enum", options?: string[], defaultValue?: string, description: string }>`；group 按 spec ⑦ 前缀规则（Font/Colors/Cursor/Window/Quick Terminal/macOS/Mouse & Scroll/Clipboard/Shell/Keybindings/General）；类型推断：*-color/background 等已知色键→color，文档枚举→enum，true/false→boolean，数值默认→number，兜底 string
  - `Views.config-editor`：分组浏览（侧栏组 + 列表）+ 搜索（键名/说明）；色值行显色板、enum 下拉、boolean 开关、其余文本行内编辑；未知 key 可编辑；「添加选项」从 options.json 搜索后追加到文件末尾；保存 = read → parse → 行级编辑 → write（Task 5/6）

- [ ] **Step 1: 写 gen-options.mjs 并生成 options.json**

Run: `node scripts/gen-options.mjs && node -e "const o=require('./src/assets/options.json'); console.log(o.length)"` 
Expected: 条数 > 200，且含 `theme`/`background`/`font-size`，group 均在 11 组集合内

- [ ] **Step 2: 实现编辑器视图**
- [ ] **Step 3: 手动验证**

- 对真实 `~/.config/ghostty/config`（含注释/空行）改一个值、保存，`diff` 仅目标行变化；`config.bak` 生成一次
- 添加选项写入末尾；色板颜色与值一致

- [ ] **Step 4: Commit**（`feat: ghostty config editor with options metadata`）

---

### Task 13: 设置视图 + README 重写

**Files:**
- Create: `src/views/settings.js`
- Modify: `README.md`（重写）

**Interfaces:**
- Consumes: `services.settings.*`、`DEFAULT_SETTINGS`
- Produces: `Views.settings`——表单（gitRoot/maxDepth/ghosttyPath/autoFocus/forceNewWindow），保存写 dbStorage；README：功能表（7 能力 + 关键字表）、首次授权说明（macOS 自动化授权一次性）、Ghostty ≥1.2 与改名需最新 tip 的说明、YAML 示例

- [ ] **Step 1: 实现设置视图**
- [ ] **Step 2: 手动验证**（改 gitRoot 后 gtr 生效；关 autoFocus 后 gtw 不抢焦点；关 forceNewWindow 后启动配置首 tab 进当前窗口）
- [ ] **Step 3: 重写 README**
- [ ] **Step 4: Commit**（`feat: settings view; docs: rewrite readme`）

---

### Task 14: E2E 手动清单 + 打包收尾

**Files:**
- Create: `docs/manual-e2e-checklist.md`
- Modify: 如清单发现缺陷，修至全过

**Interfaces:**
- Consumes: 全部
- Produces: 通过的手动清单 + `.upx` 包

- [ ] **Step 1: 写清单文件**（按 spec「验证」节展开：每能力一条步骤+预期；含 Raycast README 三示例 YAML 回放、config 注释保留 diff 验证、多窗口分屏聚焦、改名/恢复、拖拽三态）
- [ ] **Step 2: 逐项执行并勾选**（在 uTools 开发者工具中；发现的问题当场修复并重跑相关项）
- [ ] **Step 3: 打包**：开发者工具「打包」生成 `.upx`（确认包内无 node_modules/test）
- [ ] **Step 4: Commit + push**（`chore: e2e checklist and packaging`）
