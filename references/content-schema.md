# Content Schema — Reading Assistant

AI 生成内容的 JSON schema。Claude 在阶段 2 直接输出此格式。

---

## 完整 JSON 结构

```json
{
  "title": "文章标题（中文译名）",
  "author": "作者名",
  "summary": "100字以内的中文摘要，覆盖核心论点",
  "sections": [
    {
      "id": "s1",
      "heading": "章节标题（中文）",
      "paragraphs": [
        {
          "en": "原文英文段落",
          "zh": "中文翻译（流畅自然，非机器直译）"
        }
      ],
      "knowledge_cards": [
        {
          "term": "术语或概念",
          "explanation": "简短解释（30字以内）"
        }
      ]
    }
  ],
  "bloom": {
    "beginner": [
      { "tag": "记忆", "q": "问题（记忆层：能复述文章中的关键信息吗？）", "a": "参考答案" },
      { "tag": "理解", "q": "问题（理解层：能用自己的话解释核心概念吗？）", "a": "参考答案" }
    ],
    "intermediate": [
      { "tag": "应用", "q": "问题（应用层：能将概念用于具体场景吗？）", "a": "参考答案" },
      { "tag": "分析", "q": "问题（分析层：能分解论点、找出依据吗？）", "a": "参考答案" }
    ],
    "expert": [
      { "tag": "评价", "q": "问题（评价层：能判断论点的合理性和局限性吗？）", "a": "参考答案" },
      { "tag": "创造", "q": "问题（创造层：能基于文章观点提出新想法吗？）", "a": "参考答案" }
    ]
  },
  "knowledge_base": {
    "term_key": {
      "title": "术语名称",
      "beginner": "入门解释（类比日常生活）",
      "intermediate": "进阶解释（技术原理）",
      "expert": "专家解释（学术/业界视角）"
    }
  },
  "course": {
    "modules": [
      {
        "id": "m1",
        "title": "模块标题（对应章节主题）",
        "subtitle": "副标题（一句话点出核心）",
        "screens": [
          {
            "type": "concept",
            "content": {
              "pairs": [
                {
                  "code": "原文关键句或术语（英文）",
                  "plain": "通俗中文解释（用日常类比）"
                }
              ]
            }
          },
          {
            "type": "quiz",
            "content": {
              "question": "单选题题目",
              "options": ["A. 选项1", "B. 选项2", "C. 选项3", "D. 选项4"],
              "correct": 1,
              "explanation": "为什么这个答案是对的（2-3句）"
            }
          },
          {
            "type": "dragdrop",
            "content": {
              "instruction": "把每个概念拖到正确的分类",
              "chips": ["概念A", "概念B", "概念C", "概念D"],
              "zones": [
                { "label": "分类1", "correct": ["概念A", "概念B"] },
                { "label": "分类2", "correct": ["概念C", "概念D"] }
              ]
            }
          },
          {
            "type": "application",
            "content": {
              "scenario": "情景描述（具体、贴近生活）",
              "question": "在这个情景下，你会如何应用本章学到的内容？",
              "reference": "参考答案（给出思路框架，不唯一）"
            }
          }
        ]
      }
    ]
  }
}
```

---

## 生成规则

### sections
- 每个原文章节对应一个 section
- paragraphs 保留原文英文 + 流畅中文译文
- knowledge_cards 挑出该章节的 2-4 个关键术语

### bloom
- 严格6道题，每级2道
- 问题难度梯度清晰：记忆→理解→应用→分析→评价→创造
- 答案具体，包含文章中的关键信息

### knowledge_base
- key 为术语的英文小写（用下划线连接），如 `attention_mechanism`
- 三层解释体现不同深度

### course.modules
- 每个 section 对应一个 module（id: m1, m2...）
- 每个 module 固定 4 个 screens，顺序：concept → quiz → dragdrop → application
- concept：选 2-3 对最有代表性的「原文表达 ↔ 通俗解释」
- quiz：考查该章节最核心的一个概念，4个选项，只有1个正确
- dragdrop：chips 4-6个，zones 2-3个，每个 zone 至少接受1个 chip
- application：情景要具体（不能太抽象），参考答案给框架不给唯一答案

---

## 输出规则（NON-NEGOTIABLE）

- **MUST**：输出纯 JSON，不包含任何 markdown 标记（禁止 ```json 代码块）
- **MUST**：JSON 必须合法——检查所有引号闭合、逗号、嵌套层级，输出前自我校验
- 所有中文内容用简体中文
- 如果文章没有明显章节，按主题自行划分为 3-5 个 section
- knowledge_base 的 key 与 article 正文 `data-term` 属性保持一致
