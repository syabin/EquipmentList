# Agent Guide — 设备清单制作工具

## 项目简介

工业设备清单管理的单页 Web 工具，帮助工程师快速创建、编辑和管理设备清单。将 Excel 设备清单转化为结构化 Web 表格，支持智能搜索、批量操作、工时统计。

## 技术栈

- 纯 HTML/CSS/JavaScript（无框架依赖）
- xlsx.js (SheetJS) 处理 Excel 导入导出
- localStorage 本地存储
- 单文件部署：`index.html` 即全部代码

## 文件结构

```
EquipmentList/
├── index.html    # 主程序（HTML + CSS + JS 全部内联）
└── README.md     # 项目文档
```

## 核心架构

所有代码在 `index.html` 一个文件中，结构为：

1. **CSS**（`<style>` 标签，~600 行）— 全局样式、表格、弹窗、存档面板
2. **HTML**（`<body>` 部分）— 头部按钮栏、筛选栏、表格主体、弹窗
3. **JavaScript**（`<script>` 标签，~1500+ 行）— 全部业务逻辑

## 关键模块

| 模块 | 主要函数 | 说明 |
|------|---------|------|
| 数据层 | `rows[]`, `modelRef[]`, `snapshot()` | 行数据数组、型号对照表 |
| 撤销/重做 | `pushHistory()`, `undo()`, `redo()` | 50步历史栈，Ctrl+Z/Y |
| 自动暂存 | `autoSave()`, `checkAutoSave()` | 每15秒存 localStorage |
| 存档系统 | `saveArchive()`, `loadArchive()`, `exportAllArchives()` | 项目级存档管理 |
| 型号搜索 | `loadRefFromFile()`, 自动匹配逻辑 | 从 Excel 加载对照表，输入型号时自动填充 |
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
ATTRS = ["3D-UT管道", "EE-电控系统", "LSU-UT元件", ...]  // 属性选项
DEPTS = ["EC-单机", "EE", "LSU", "ME", "PSU", "SE", "3D"]  // 部门
BIZS = ["RICH", "ROSS"]  // 业务线
```

## 修改指南

- **新增字段**：在 `COLS` 定义中添加列宽 → `<thead>` 添加表头 → `render()` 中添加单元格 → `rows` 对象添加默认值
- **修改样式**：CSS 变量在 `:root` 中定义，表格列宽在 `.col-*` 类中
- **导入逻辑**：参考 `loadRefFromFile()` 处理 xlsx 解析
- **存档格式**：JSON 结构为 `{ version, exportTime, archives: [...] }`
