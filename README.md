# 论文翻译与解读项目

使用 `.skill/` 中的 Skill 对学术论文进行完整翻译和通俗解读。

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
├── .skill/
│   ├── SKILL.md              # Skill 定义
│   └── refs/
│       ├── translation-rules.md       # 翻译规范细则
│       └── interpretation-guide.md    # 解读写作指南
├── papers/                    # 论文存放与产物目录
│   ├── README.md              # 论文索引
│   └── <paper-slug>/          # 每篇论文一个文件夹
│       ├── original.pdf       # 原文副本
│       ├── translation.md     # 完整翻译
│       └── interpretation.md  # 通俗解读
└── .gitignore
```
