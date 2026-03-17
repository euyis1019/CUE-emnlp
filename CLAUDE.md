# CUE-emnlp LaTeX Project

## 项目结构

- **Markdown 工作目录**: `/Users/eric/Documents/Obsidian Vault/科研/LLM可解释性/CUE/` — 用户编写正文、梳理逻辑、记录进展
- **LaTeX 目录（本目录）**: `overleaf/CUE-emnlp/` — 英文论文 LaTeX 源码，最终投稿版本
- Appendix 的相关实验目录 `/Users/eric/Documents/Obsidian Vault/科研/LLM可解释性/CUE/Appendix`
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


### Git 配置

已通过 `gh` 配置全局 git 身份（`git config --global`）：
- `user.name`: Eric
- `user.email`: euyis1019@users.noreply.github.com

push 前先 `git fetch` 检查远程是否有新提交，如有则 `git pull --rebase` 后再 push。
