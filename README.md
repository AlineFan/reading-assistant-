# reading-assistant

> 将任意文章转为带课程模块的中文阅读辅助 HTML 页面的 Claude Code Skill

## 效果预览

生成的页面包含：

| 功能 | 说明 |
|------|------|
| 📖 双语切换 | 中文 / 英文 / 中英对照三种模式 |
| ⚡ Bionic 阅读 | 加粗词头，提升阅读速度（ADHD 友好） |
| 🎯 聚焦遮罩 | 遮挡非阅读区域，减少分心 |
| ✏️ 划线批注 | 选中文字高亮 + 添加个人笔记 |
| 🧠 布鲁姆问题 | 记忆→理解→应用→分析→评价→创造，六级问题 |
| 🎓 加强理解模块 | 概念解析 / 单选测验 / 拖拽匹配 / 应用思考，四屏交互课程 |

---

## 安装

```bash
git clone https://github.com/AlineFan/reading-assistant- \
  ~/.claude/skills/reading-assistant
```

---

## 使用方法

在 Claude Code 中输入以下任意一种：

```bash
# 网址
/reading-assistant https://example.com/article

# 本地 PDF
/reading-assistant /path/to/paper.pdf

# 粘贴文本（在命令后直接附上文章内容）
/reading-assistant 在这里粘贴文章全文…
```

生成完成后自动打开 HTML 文件，文件名格式为 `{文章标题}-阅读辅助.html`。

---

## 工作原理

```
阶段 0  预检模板文件是否存在
阶段 1  获取文章内容（URL / PDF / 文本）
阶段 2  Claude 直接结构化：翻译 + 知识卡片 + 布鲁姆问题 + 课程数据
阶段 3  填入模板，生成完整 HTML（无需外部 API Key）
阶段 4  输出文件并打开预览
```

> 本 skill 在 Claude Code 内运行，由 Claude 自身完成所有 AI 处理，不需要配置任何额外的 API Key。

---

## 文件结构

```
reading-assistant/
├── SKILL.md                    # skill 执行逻辑（含 Gotchas、NON-NEGOTIABLE）
├── README.md
└── references/
    ├── template.html           # 基础模板（完整 CSS + JS）
    ├── content-schema.md       # AI 结构化输出 JSON schema
    └── course-patterns.md      # 课程模块 HTML/JS 组件模式
```

---

## 要求

- [Claude Code](https://claude.ai/code) CLI
- 处理 PDF 时需要：`pip install pdfplumber`
