---
name: reading-assistant
description: |
  阅读辅助工具。将任意文章（URL / 本地 PDF / 粘贴文本）转化为一个自包含的中文阅读 HTML 页面，
  包含：双语切换、Bionic 阅读、聚焦遮罩、划线批注、布鲁姆分级问题、**课程模块（交互测试）**。
  触发词：「帮我做阅读辅助」「生成阅读页面」「/reading-assistant」「把这篇文章做成阅读工具」
trigger:
  - /reading-assistant
  - 帮我做阅读辅助
  - 生成阅读页面
  - 把这篇文章做成阅读工具
allowed-tools:
  - Read
  - Write
  - Bash
  - WebFetch
---

# /reading-assistant

将任意文章转为带课程模块的中文阅读辅助 HTML 页面。

---

## ⚠️ 已知陷阱（生成 HTML 前必读）

以下是真实踩过的 bug，每条都曾导致页面功能异常：

1. **TDZ 崩溃**：`let courseData / courseState / courseVisible` 的声明必须出现在
   `loadCourseData([...])` 调用**之前**。声明在后会导致整个 script 崩溃，所有按钮失效。

2. **双语索引错位**：`initParas()` 必须用 `.filter()` 排除
   `blockquote / .inline-knowledge / .warn-box / .rule-box / .bloom-section`
   内的 `<p>`，否则中英段落会错位对应。

3. **quiz 永远判错**：`screen-quiz` div 上必须有 `data-correct="N"` 属性，
   缺失时 `parseInt(undefined)` = NaN，所有选项都判错。

4. **课程页无法滚动**：`.cv-body / .course-module / .course-screen` 的 flex
   链上每一层都必须加 `min-height: 0`，否则 `overflow-y: auto` 无效。

5. **内容首屏空白**：`.section` 初始 `opacity:0`，IntersectionObserver 异步触发
   可能在首屏不触发，必须在 `requestAnimationFrame` 回调里给可见区域的
   section 补加 `.visible`。

---

## 输入

用户提供以下任意一种：
- **URL**：`/reading-assistant https://example.com/article`
- **本地 PDF 路径**：`/reading-assistant /path/to/file.pdf`
- **粘贴文本**：直接附在命令后，或单独发送文章内容

---

## 执行阶段

### 阶段 0 — 预检（失败立即终止）

```bash
ls ~/.claude/skills/reading-assistant/references/template.html \
  && echo "✅ 模板就绪" \
  || echo "❌ 模板不存在，请检查 skill 目录"
```

模板不存在时：停止执行，告知用户重新安装 skill 或确认路径后重试。

---

### 阶段 1 — 内容获取

**如果是 URL**：使用 WebFetch 获取页面内容，提取正文（忽略导航/广告/页脚）。

**如果是 PDF**：
```bash
python3 -c "
import pdfplumber, sys
text = []
with pdfplumber.open(sys.argv[1]) as pdf:
    for page in pdf.pages:
        t = page.extract_text() or ''
        text.append(t)
print('\n'.join(text))
" /path/to/file.pdf
```
如果 pdfplumber 不可用，提示用户安装：`pip install pdfplumber`

**如果是粘贴文本**：直接使用全文。

提取后输出：「✅ 内容获取完成，约 XXX 字，开始结构化处理…」

---

### 阶段 2 — AI 结构化

**立即读取 schema（不得跳过）**：
使用 Read 工具读取 `~/.claude/skills/reading-assistant/references/content-schema.md`

读完后，严格按照其中的字段结构生成 JSON，不得凭记忆猜测字段名。

**本 skill 在 Claude Code 中运行，由 Claude 自身直接生成结构化内容，不需要任何外部 API Key。**

根据文章内容直接生成符合 `content-schema.md` 格式的 JSON，包含：
- `title`、`sections[]`（翻译段落 + 知识卡片）
- `bloom`（6道布鲁姆问题）
- `course.modules[]`（每章节对应1个模块，每模块4个屏幕）

输出：「✅ 结构化完成，生成 X 个章节，X 个课程模块」

---

### 阶段 3 — 生成 HTML

**Step 0 — 读取课程组件模式**（生成 course screens 前必须先读）：
使用 Read 工具读取 `~/.claude/skills/reading-assistant/references/course-patterns.md`

**Step 1 — 定位替换点**：
```bash
grep -n "document.title\|<article\|const knowledge\|loadCourseData\|nav-course-btn" \
  ~/.claude/skills/reading-assistant/references/template.html
```

**Step 2 — 只读需要替换的行段**（根据 Step 1 的行号，用 Read 工具）：
- `<title>` 附近 ±5 行
- `<article>` 到 `</article>` 全段
- `const knowledge = {` 到 `}` 结束
- `loadCourseData([` 到 `])` 结束

其余 CSS / JS 部分**不读取**，输出时原样保留。

**Step 3 — 替换清单（只改这5处，其他一律不动）**：
1. `<title>` 内容 → 新文章标题
2. nav 中的标题文字 → 新文章标题
3. `<article>` 内容 → 新章节 + 布鲁姆问题
4. `const knowledge = {...}` → 新知识库数据
5. `loadCourseData([...])` → 新课程数据

---

> **NON-NEGOTIABLE：**
> 1. `initParas()` **必须使用**下方过滤版本，禁止简化为 `querySelectorAll('article p')`
> 2. 所有 JS 函数只替换数据，**禁止重写或删除**任何现有函数体

**`initParas()` 固定写法（禁止修改）**：
```javascript
function initParas() {
  artParas = Array.from(document.querySelectorAll('article p')).filter(p =>
    !p.closest('blockquote') &&
    !p.closest('.inline-knowledge') &&
    !p.closest('.warn-box') &&
    !p.closest('.rule-box') &&
    !p.closest('.bloom-section')
  );
  artParas.forEach((p, i) => p.setAttribute('data-pi', i));
}
```

**❌ 绝不删除以下函数**（即使你认为当前文章用不到）：

| 函数 | 用途 |
|------|------|
| `toggleCourseView()` | 加强理解面板开关 |
| `loadCourseData()` | 注入课程数据 |
| `renderCourseView()` / `updateCourseView()` | 课程渲染与导航 |
| `courseNav()` / `jumpToModule()` | 课程翻页 |
| `checkQuiz()` | 测验判题 |
| `initDnD()` / `dropChipToZone()` | 拖拽匹配 |
| `showAppAnswer()` | 应用题展开答案 |
| `setLang()` / `initParas()` | 双语切换 |
| `toggleAdhd()` / `applyAdhd()` | Bionic 阅读 |
| `toggleMaskInline()` | 聚焦遮罩 |
| `saveNote()` / `openNote()` | 笔记功能 |
| `handleUrlInput()` / `enhanceImportedContent()` | 导入面板 |
| `togglePanel()` | 设置/导入面板开关 |

**DO NOT（以下操作一律禁止）**：
- ❌ 修改任何 CSS 变量（`:root` 里的 `--` 变量）
- ❌ 重构或"优化"现有 JS 函数
- ❌ 删除 `<script src="pdf.min.js">` 或 `<script src="mammoth.browser.min.js">`
- ❌ 把 `onclick=` 属性改为 addEventListener
- ❌ 修改 `#course-view` 的 CSS（`position:fixed` 等布局已精确调试）
- ❌ 在 `<article>` 外添加任何新的全局样式

HTML 生成完成后，Write 到文件：
```
{文章标题（中文，去除特殊符号）}-阅读辅助.html
```
保存位置：与输入文件同目录，或当前工作目录。

---

### 阶段 4 — 输出 & 预览

```bash
open "{output_file}.html"
```

告知用户：
```
✅ 阅读辅助页面已生成：{filename}.html

功能说明：
📖 阅读模式  — 双语切换 · Bionic 阅读 · 聚焦遮罩 · 划线批注
🧠 布鲁姆问题 — 入门/进阶/专家三级问题，点击展开答案
🎓 加强理解  — 点击导航栏「加强理解」按钮进入，包含：
   · 概念解析（原文 ↔ 通俗解释）
   · 小测验（单选题 + 即时反馈）
   · 匹配练习（拖拽分类）
   · 应用思考（情景问题 + 参考答案）
```

---

## 参考文件

- `references/content-schema.md` — AI 生成内容的 JSON schema
- `references/course-patterns.md` — 课程模块交互组件模式
- `references/template.html` — 基础模板（完整 CSS + JS，随 skill 分发）

---

## 错误处理

| 场景 | 处理方式 |
|------|---------|
| 模板文件不存在 | 停止执行，提示重新安装 skill |
| URL 无法访问 | 提示用户粘贴文本或使用 PDF |
| PDF 解析失败 | 提示安装 pdfplumber，或请用户复制文本 |
| 文章过短（< 500字） | 警告内容可能不足，继续生成 |
| 文章过长（> 50000字） | 截取前 40000 字并告知用户 |
| 生成 JSON 格式错误 | 重试一次，仍失败则降级：只生成翻译，跳过课程模块 |
