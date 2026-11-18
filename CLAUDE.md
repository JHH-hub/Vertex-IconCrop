# CLAUDE.md · Vertex-IconCrop AI 协作导航文档

> 本文档与 `AGENTS.md` 保持同步，为 Claude Code 及其他 AI 编码助手提供 `Vertex-IconCrop` 仓库的架构全景、核心代码索引、运行/打包命令与开发约束。修改代码前请先阅读本文档。

---

## 1. 项目定位与架构概览

`Vertex-IconCrop`（Icon 裁切工具 / Sprite Cutter v1.0）是一个专为游戏美术与找茬关卡制作打造的**大图批量框选、裁切、AI 批量命名与 ZIP 导出工具**：
- **核心设计**：
  - **纯单文件前端主程序**：`icon裁剪工具.html` 内置了全部 UI 样式、Canvas 交互引擎、JSZip v3.10.1 源码、Base64 图标以及公司内网 Venus 大模型代理调用逻辑，无需后端服务或网络即可完成离线框选与 ZIP 打包导出。
  - **轻量级 EXE 包装器**：通过 `launcher.py` + `Icon裁剪工具.spec`（PyInstaller 单文件打包），双击 `dist\Icon裁剪工具.exe` 时自动将内嵌的 `icon裁剪工具.html` 解压释放到系统临时目录 `%TEMP%\IconCropper\icon裁剪工具.html` 并调用系统默认浏览器打开，随后启动器进程自行退出。
  - **跨仓库关系**：该工具同时内嵌于 `Vertex-AssetDesk/tools/icon-crop/index.html`（通过 `/tools/icon-crop` 提供服务）。

---

## 2. 核心文件索引

| 文件路径 | 作用与核心职责 |
| :--- | :--- |
| `icon裁剪工具.html` | **核心主程序（单文件 HTML，约 2912 行）**。内嵌 JSZip v3.10.1、三栏式暗色紫调工作台 UI、选区与撤销栈管理、Venus AI 解析/OCR/智能匹配/AI 增强放大、多级抗锯齿缩放与 ZIP/标注图导出。 |
| `launcher.py` | **PyInstaller 启动器脚本**。通过 `sys._MEIPASS` 定位打包资源，释放 `icon裁剪工具.html` 到 `%TEMP%\IconCropper\` 并调用 `webbrowser.open()` 打开，失败时通过 `ctypes.windll.user32.MessageBoxW` 弹窗提示。 |
| `Icon裁剪工具.spec` | **版本受控的 PyInstaller 打包配置**。指定 `launcher.py` 为入口，`datas=[('icon裁剪工具.html', '.')]`，`console=False`，`upx=True`，图标 `裁剪.ico`，输出单文件 `Icon裁剪工具.exe`。 |
| `build.bat` | **一键打包脚本**。自动检查并按需通过腾讯 PyPI 镜像安装 `pyinstaller`，清理 `build/` 与 `dist/`，执行 `python -m PyInstaller --noconfirm --clean "Icon裁剪工具.spec"`。 |
| `裁剪.ico` | Windows 可执行文件应用图标。 |
| `animation-tracker.html` | 历史保留的资产台快照页面（注意：**不属于** Icon 裁剪工具打包产物，修改 Icon 裁剪功能时切勿误改此文件）。 |
| `README.md` | 面向用户的中文功能特性、快捷键说明与使用文档。 |
| `AGENTS.md` / `CLAUDE.md` | 面向 AI Agent 的上下文导航与开发约束（两文件保持内容同步）。 |

---

## 3. `icon裁剪工具.html` 核心模块速查

1. **状态与快照撤销栈（State & Undo Stack）**：
   - `srcImage` (`HTMLImageElement`)、画布缩放比 `zoom`、选区数组 `selections`（元素结构 `{x, y, w, h, name}`）、名字池 `namePool`（`[{text, assigned}]`）、当前激活选区索引 `activeIdx`。
   - `undoStack`（上限 `UNDO_LIMIT = 50`）：采用深拷贝快照 `snapshotSelections()`，`pushHistory()` 在增删、移动、缩放、正方形化、清空前记录状态；方向键连续微调通过 `beginNudgeHistory()`（800ms 防抖窗口）合并为单次撤销记录。
2. **画布渲染与选区交互（Canvas & Selection Overlay）**：
   - `render()` / `drawSelections()`：主画布关闭插值平滑（`setPixelSharp(ctx)`）以保持像素锐利；每个选区渲染 `.sel-overlay` DOM 覆盖层，当前激活选区额外渲染 8 个方向缩放手柄（`nw/n/ne/e/se/s/sw/w`）。
   - `startMoveSelection(idx, e)` / `startResizeSelection(idx, dir, e)`：处理选区主体拖拽平移与 8 手柄拖拽缩放，实时同步左侧坐标输入框（`#manX/#manY/#manW/#manH`）。
   - `squarify(mode)`：以每个选区中心点 `(cx, cy)` 为基准，按最长边（`max`）或最短边（`min`）一键正方形化并限制在原图边界内。
   - 右侧列表支持通过 `.rdrag`（`⠿`）拖拽重排选区顺序，或将 `.name-chip` 拖放到对应选区行完成命名。
3. **批量命名与 Venus AI 能力（需公司内网）**：
   - 代理端点：`VENUS_URL = 'http://v2.open.venus.oa.com/llmproxy/chat/completions'`。
   - Token 存储：`localStorage` 键名 `VENUS_TOKEN_KEY = 'iconCrop.venusToken'`（通过 `getVenusToken()` / `promptVenusToken()` 管理，严禁硬编码）。
   - `callVenus(modelList, messages, timeoutMs)`：通用请求封装，遇到 HTTP `403`/`404` 或模型下线（`4030`）自动遍历 fallback 模型列表。
   - **文本解析（`aiParse`）**：模型链 `['deepseek-v3', 'hunyuan-pro', 'hunyuan-standard', 'gpt-4o-mini']`；支持点击「🤖 AI 解析」或在 `#nameRawText` 粘贴多行文本后自动触发。
   - **截图 OCR（`aiParseImage`）**：模型链 `['hunyuan-vision', 'gpt-4o', 'gpt-4o-mini', 'qwen-vl-plus']`；支持点击「📷 截图识别」选图或在文本框直接 `Ctrl+V` 粘贴剪贴板截图。
   - **智能匹配（`autoMatchSelectionNames`）**：调用 `buildSelectionContactSheet()` 将所有选区缩略图拼接为带 `#编号` 的网格总览图，发送给视觉模型自动匹配 `namePool` 中的候选名字（或无候选池时直接视觉命名）。
4. **高清导出与 AI 增强放大（Export & AI Enhance）**：
   - `drawSelectionToCanvas()`：
     - 1:1 导出或放大时使用 `setPixelSharp` 避免浏览器默认双线性插值抹糊；
     - 缩小时采用**多级二分降采样（step-down halving）+ `imageSmoothingQuality = 'high'`**，消除单步大幅缩小产生的锯齿与模糊。
   - `aiEnhanceSelectionImage(sel)`（勾选 `#aiEnhanceCheck` 时生效）：当源选区尺寸 `< 100×100`（`AI_ENHANCE_THRESHOLD = 100`）且需放大导出时，调用 `VENUS_IMAGE_MODELS = ['gemini-3.1-flash-image', 'gemini-3-pro-image']` 进行重绘高清化；通过 `urlToBase64()` 将返回的远程 COS URL 转为本地 Base64 避免 Canvas 跨域污染（`SecurityError`）；单张 AI 增强失败时自动 catch 并回退到普通缩放，不阻塞批量 ZIP 导出。
   - `getIconQualityWarnings()`：检测 JPEG 透明背景变黑风险、非正方形拉伸风险及源选区过小风险。
   - 支持导出带彩色边框与编号/名称标签的整图（`#exportAnnotatedBtn` → `annotated_preview.png`）。

---

## 4. 运行与打包命令

### 开发调试（直接运行）
```powershell
# 方式 1：直接在浏览器打开主文件（推荐）
start icon裁剪工具.html

# 方式 2：测试启动器逻辑
python launcher.py
```

### 打包独立 EXE
```powershell
# 一键清理并按 Icon裁剪工具.spec 打包为 dist\Icon裁剪工具.exe
.\build.bat

# 或手动执行 PyInstaller（必须指定 .spec 文件以保持配置一致）
python -m PyInstaller --noconfirm --clean "Icon裁剪工具.spec"
```

---

## 5. 开发约束与安全红线（AI 必读）

1. **严禁硬编码 Venus Token 或内网凭据**：
   - 所有 AI 请求的 Token 必须从 `localStorage.getItem('iconCrop.venusToken')` 读取，绝不允许将真实 Token 写入 `icon裁剪工具.html` 或任何配置文件中。
2. **保持 `icon裁剪工具.html` 单文件自包含**：
   - 因 `launcher.py` 仅释放 `icon裁剪工具.html` 单个文件到 `%TEMP%\IconCropper\`，任何新增样式、脚本或图标必须内联在 `icon裁剪工具.html` 内，禁止拆分为外部 `.js`/`.css` 文件（否则打包成 EXE 后释放到临时目录将丢失静态资源）。
3. **打包配置唯一性**：
   - 修改打包参数（如图标、内嵌数据文件、隐藏控制台等）时，必须修改版本受控的 `Icon裁剪工具.spec`，不要在 `build.bat` 中改用裸命令行参数覆盖 `.spec`。
4. **不要提交构建产物**：
   - `build/`、`dist/`、`*.exe` 已在 `.gitignore` 中忽略，切勿强制添加到 Git 版本库。
5. **文档同步要求**：
   - 修改 `AGENTS.md` 时必须同步更新 `CLAUDE.md`。
