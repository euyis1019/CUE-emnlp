# CUE-emnlp LaTeX Project

## 项目结构

- **Markdown 工作目录**: `/Users/eric/Documents/Obsidian Vault/科研/LLM可解释性/CUE/` — 用户编写正文、梳理逻辑、记录进展
- **LaTeX 目录（本目录）**: `overleaf/CUE-emnlp/` — 英文论文 LaTeX 源码，最终投稿版本
- **内容流向**: Markdown 是思考与内容源头，LaTeX 是格式化输出。写/改 LaTeX 前，先查对应 Markdown 文件的最新版本。（除非是小修改/语法、渲染上的小修改）

## 规则

### Label 规范

所有 section、subsection、table、figure，以及 appendix 中的 section、subsection，都**必须**有清晰的 `\label{}`：

- Section: `\label{sec:xxx}`
- Subsection: `\label{subsec:xxx}`
- Figure: `\label{fig:xxx}`
- Table: `\label{tab:xxx}`
- Appendix section: `\label{app:xxx}`
- Appendix subsection: `\label{app:subsec:xxx}`
- Equation (如需引用): `\label{eq:xxx}`

正文引用统一使用 `\cref{}`（已加载 `cleveref` 宏包）。仅在需要特殊前缀时（如 §3）局部用 `\ref{}`。

### 符号系统

`Method/notation_v1.md` 是符号定义的唯一权威来源。LaTeX 中所有数学符号必须与该文件一致。

### 引用格式

使用 natbib：
- 括号引用: `\citep{key}` → (Author et al., 2024)
- 文内引用: `\citet{key}` → Author et al. (2024)
- 禁止使用 `\cite{}`

### Markdown ↔ LaTeX 对应关系

| Markdown 文件 | LaTeX 文件 |
|---|---|
| `introduction/introduction_v1.md` | `sections/introduction.tex` |
| `related_paper/relatedwork_v0.1.md` | `sections/related_work.tex` |
| `Method/method_v0.md` | `sections/method.tex` |
| `experiment/experiment_reading_notes_v0.md` | `sections/experiments.tex` |
| `Appendix/appendix_A_benchmark_v0.md` | `appendix/benchmark_details.tex` |
| `Appendix/appendix_B_probing_training_v0.md` | `appendix/probing_training.tex` |
