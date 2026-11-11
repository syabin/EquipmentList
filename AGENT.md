# Agent Guide — 设备清单制作工具

## 项目简介

工业设备清单管理的单页 Web 工具，帮助工程师快速创建、编辑和管理设备清单。将 Excel 设备清单转化为结构化 Web 表格，支持智能搜索、批量操作、工时统计。

## 技术栈

- 纯 HTML/CSS/JavaScript（无框架依赖）
- xlsx.js (SheetJS) 处理 Excel 导入导出
- localStorage 本地存储 + GitHub Contents API 云端同步（可选）
- 单文件部署：`index.html` 即全部代码

## 文件结构

```
EquipmentList/
├── index.html    # 主程序（HTML + CSS + JS 全部内联）
└── README.md     # 项目文档
```

## 核心架构

所有代码在 `index.html` 一个文件中，结构为：

1. **CSS**（`<style>`）— 全局样式、表格、弹窗、存档面板 v2、云同步面板、通用弹窗、撤回提示
2. **HTML**（`<body>`）— 头部按钮栏、筛选栏、表格主体、确认框、存档面板、云同步面板、通用弹窗
3. **JavaScript**（`<script>`）— 全部业务逻辑（存档系统 + 云同步模块都在 `Archive System v2` / `GitHub 云端同步` 两段）

## 关键模块

| 模块 | 主要函数 | 说明 |
|------|---------|------|
| 数据层 | `rows[]`, `modelRef[]`, `snapshot()` | 行数据数组、型号对照表 |
| 撤销/重做 | `pushHistory()`, `undo()`, `redo()` | 50步历史栈，Ctrl+Z/Y；`pushHistory()` 同时调度云同步 |
| 自动暂存 | `autoSave()`, `checkAutoSave()` | 每15秒存 localStorage；顺带定时自动快照 + 云同步兜底 |
| 存档系统 v2 | `getArchives()`, `setArchives()`, `normalizeArchive()`, `saveArchive()`, `loadArchive()`, `renderArchiveList()`, `importArchives()` | 项目级存档；写入保护/配额/自动快照/撤回/搜索排序分页/批量/导入三策略 |
| 云同步 | `ghConfig()`, `ghUpload()`, `ghPull()`, `ghShowDiff()`, `runGhAutoSync()` | GitHub Contents API；仓库 `syabin/task-data`，路径 `equipment-list/equipment-data.json` |
| 工序批量 | `openProcessBatch()`, `applyProcessBatch()` | 表头「工序 ▾」；覆盖 / 仅填空 / 清空；`process` 也在 `bindEvents()` 的批量填充白名单内 |
| 分组视图 | `archiveProjectNos()`, `archiveGroups()`, `renderGroupedArchives()` | 按 `rows[].projectNo` 折文件夹；多项目号重复出现；`deviceList_archiveView` 记住平铺/分组 |
| 项目号点选 | `pickArchiveProjectNo()`, `fillArchiveNameFromNo()` | 点存档名称 (`data-act="pick-no"`) 或文件夹「🏷 填入」(`pick-group-no`)，把项目号填进 `archiveName` 保存框；多项目号弹窗选择、无项目号回退存档名；**只改输入框，不写 `rows[].projectNo`** |
| 表格导航 | `handleCellArrow()`, `focusCell()`, `flushCellBeforeMove()` | 方向键移格；重绘前先派发 change 落库，避免丢输入 |
| 自动填充 | `disableAutofill(root)` | `render()` 末尾与 `initArchiveSystem()` 各调一次，压住浏览器/密码管理器浮层 |
| 自动编号 | `autoNumberMatch()`, `isAutoNumberDept()` | 按 `mainEquip='1'` 分组写 `matchMark`；**仅 `dept` 为 `AUTO_NUMBER_DEPT`（'ME'）的行写入**，其它部门跳过且保留原值、不打断分组 |
| 型号搜索 | `loadRefFromFile()`, `showAutocomplete()`, `acMatchScore()`, `acHighlight()`, `selectAcItem()` | 从 Excel 加载对照表（字段 `model/name/attr/dept/biz`）；**型号与设备名称都可搜**，型号命中优先，命中片段高亮；选中后回填型号 + 名称 + 属性 + 部门 + 业务归属 |
| 工时计时 | `timerStart()`, `timerPause()`, `timerReset()` | 内置工时计时器 |
| 渲染 | `render()` | 虚拟滚动渲染表格（性能优化） |
| 导入导出 | `saveExcel()`, `exportExcel()` | xlsx.js 生成 Excel |

## 数据模型

每行设备数据 (`rows[]` 中的对象) 包含字段：
- `id`, `year`, `projectNo`, `virtualCode`, `contractNo`
- `equipName`, `model`, `subModel`, `attr`, `blockOwner`
- `process`, `designNote`, `qty`, `purchase`, `position`
- `contractAttr`, `biz`, `dept`, `paint`, `nameplate`, `line`, `specialMark`
- `selected` (UI 状态), `line` (条线分组色)

## 常量定义

```javascript
ATTRS = [...]  // 属性选项
DEPTS = [...]  // 部门
BIZS = ["RICH", "ROSS"]  // 业务线
AUTO_NUMBER_DEPT = 'ME'  // 自动编号唯一允许的责任部门（isAutoNumberDept() 做 trim + toUpperCase 比较）

// 存档
ARCHIVE_KEY = 'deviceList_archives'
ARCHIVE_VERSION = 2
ARCHIVE_MAX_BYTES = 4 * 1024 * 1024   // 写前校验，超限拒绝且不覆盖原库
ARCHIVE_AUTO_KEEP = 5                  // 自动快照保留份数
ARCHIVE_PAGE_SIZE = 20
ARCHIVE_GROUP_PAGE_SIZE = 10           // 分组视图每页文件夹数
ARCHIVE_VIEW_KEY = 'deviceList_archiveView'

// 云同步（localStorage key 前缀 el-gh-*）
EL_GH_CONFIG_KEY / EL_GH_TOKEN_KEY / EL_GH_SYNC_KEY
默认仓库 syabin/task-data（与任务管理器共用）
默认路径 equipment-list/equipment-data.json（任务管理器用 data/task-data.json）
```

## 修改指南

- **新增字段**：在 `COLS` 定义中添加列宽 → `<thead>` 添加表头 → `render()` 中添加单元格 → `rows` 对象添加默认值
- **修改样式**：CSS 变量在 `:root` 中定义，表格列宽在 `.col-*` 类中
- **导入逻辑**：参考 `loadRefFromFile()` 处理 xlsx 解析
- **存档格式**：导出 JSON 为 `{ app:'EquipmentList', version:2, exportTime, archives:[...] }`；单条存档含 `id/name/note/time/timeStr/rows/nextId/timerElapsed/modelRef/kind/source/version/size`。`normalizeArchive()` 负责 v1 迁移与结构校验，坏数据返回 `null`
- **存档写入**：任何改动都必须经 `setArchives()`（返回 boolean），不要直接 `localStorage.setItem(ARCHIVE_KEY, ...)`；改动后调 `renderArchiveList()` 与 `scheduleGhAutoSync()`
- **云数据格式**：`buildCloudData()` 产出 `{ app, version, rows, nextId, timerElapsed, modelRef, archives }`（`withMeta=true` 时加 `exportTime`）；指纹 `ghLocalFingerprint()` 用不带时间戳的版本
- **云同步**：`ghUpload({silent|force})` / `ghPull()` / `ghShowDiff()`；冲突依据 `ghGetSync().sha` 与远端 sha 比对；`applyCloudData()` 会先留本地快照再覆盖，覆盖存档库时保留该快照
- **自动同步触发点**：`pushHistory()`（编辑）、`autoSave()`（15秒兜底）、所有存档写操作后调 `scheduleGhAutoSync()`；拉取期间用 `_ghSuppressAuto` 抑制回传
- **UI 弹窗**：确认框 `showModal()`，多选用 `showChoiceModal()`，输入用 `showPromptModal()`，差异对比用 `openMiniModal(..., '760px')`；列表一律事件委托 `data-act`，不要在 HTML 里拼 `onclick` 参数
- **存档事件委托**：`onArchiveBodyClick()` 必须先判 `[data-act]` 再判 `.archive-group-head` 折叠——分组头里的「🏷 填入」按钮带 `data-act`，否则点它会连带折叠文件夹
- **方向键 / 自动填充**：方向键走全局 `keydown` 委托（不要放进 `bindEvents()`，虚拟滚动会重复绑定）；跳转前必须 `flushCellBeforeMove()` 落库。新增输入框由 `render()` 末尾的 `disableAutofill(tbody)` 与 init 时的 `disableAutofill(document)` 统一处理
- **型号联想**：搜索字段在 `acMatchScore()` 里集中定义（目前 `m.model` / `m.name`，属性列只展示）。返回值越小优先级越高，`-1` 表示不命中；`showAutocomplete()` 按 `score → 原索引` 稳定排序后截 50 条。新增搜索字段必须同时更新 `acHighlight()` 的渲染列，且用 `acText()` 做空值安全，不要用 `m.xxx.toLowerCase()` 裸调（旧缓存可能缺字段）
- **自动编号**：改部门限制只需改 `AUTO_NUMBER_DEPT`；`autoNumberMatch()` 遍历 `rows` 时先过 `isAutoNumberDept(r)`，非目标部门必须 `return`（不写 `matchMark`、不改 `inGroup`/`subNo`），否则会污染其它部门的配套标识。无目标部门行时提前 return，不要 `pushHistory()` 以免产生空撤销步。`pushHistory()` 已内含云同步调度，无需另调
- **验证**：抽取内联 script 用 `node --check` 检查语法，再用轻量 DOM mock 跑存档/云同步/自动编号冒烟用例
