# AGENTS.md

论文翻译与解读仓库。所有产物规程以 `.skills/paper-reader/SKILL.md`（七步工作流）为唯一标准：
翻译前先读 `refs/translation-rules.md`，解读前先读 `refs/interpretation-guide.md`。产物全部用中文撰写。

## 仓库结构

- `papers/<topic>/<slug>/` — 每篇论文一个文件夹，含 `translation.md`（全译）、`interpretation.md`（通俗解读）、`architecture.md`（Mermaid 架构图）、原 PDF（经 `mv` 移入，文件名永不改）
- `papers/README.md` — 主题分类表 + 论文索引表；每新增一篇必须同步更新数量与索引行
- `classic-ai/` — 特例：文件夹与产物文件用 `NN-<slug>` 编号前缀（如 `04-transformer-architecture.md`），其余主题用裸 `architecture.md`
- `article-structure.md` / `article-structure.drawio` — 由 `tmp/gen_structure_drawio.py` 生成的仓库结构图，日常不更新
- `tmp/` — 已 gitignore 的临时区，杂散 PDF 会落在那里

## 关键规程（源自 SKILL.md，勿偏离）

- 主题无固定列表：能匹配 `papers/README.md` 现有主题就复用，否则按内容新建 kebab-case 主题
- 团队子主题（如 `math-foundations/dengyu`、`wanghong`）：同团队 ≥2 篇且同主题才建；新团队名先征询用户
- 论文移动用 `mv`，绝不 `cp`
- 架构图：Mermaid flowchart，节点/边标签一律双引号（内含 `"` 转义为 `&quot;`），禁止嵌套 subgraph；存量 `.drawio` 文件不删除
- 长论文（>25 页或 section >6 个）：分段翻译，先 Write 文件头再按 section Edit 追加，新 section 前 Grep 已译文件沿用术语
- 不翻译：公式、引用标记、参考文献、代码块；术语首次出现用 `中文（English）`
- 网络调用仅限 arxiv.org；PDF 文本用 `pdftotext -layout <pdf> -`（或 pdfplumber），扫描件不做 OCR 仅告知用户

## 陷阱

- arXiv 下载失败只告知用户手动链接，不反复重试