# 论文翻译与解读项目

使用 `.skills/paper-reader/` 中的 Skill 对学术论文进行完整翻译和通俗解读。

## 使用方式

把论文 PDF 放入 `papers/` 目录，然后对话中说：

```
帮我翻译解读 papers/xxx.pdf
```

或直接给 arXiv ID：

```
帮我读这篇论文 2401.12345
```

## 目录结构

```
article/
├── .skills/
│   └── paper-reader/                # 论文翻译与解读 Skill
│       ├── SKILL.md                 # Skill 定义（七步工作流）
│       └── refs/
│           ├── translation-rules.md       # 翻译规范细则
│           └── interpretation-guide.md    # 解读写作指南
├── papers/                          # 论文存放与产物目录
│   ├── README.md                    # 主题分类表 + 论文索引
│   └── <topic>/<paper-slug>/        # 按主题分类，每篇论文一个文件夹
│       ├── <原始文件名>.pdf         # 原文（mv 移入，文件名不改）
│       ├── translation.md           # 完整翻译
│       ├── interpretation.md        # 通俗解读
│       └── architecture.md          # Mermaid 架构图
│   （classic-ai/ 特例：文件夹与产物文件用 NN-<slug> 编号前缀，见 AGENTS.md）
├── article-structure.md/.drawio     # 仓库结构图（由 tmp/gen_structure_drawio.py 生成）
├── AGENTS.md                        # 仓库规程
└── .gitignore                       # tmp/ 等已忽略
```
