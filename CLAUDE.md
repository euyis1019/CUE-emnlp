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

### 画图规范

生成论文用的 matplotlib 图片时，必须遵守以下规范：

1. **所有文字必须是英文**，包括标题、坐标轴标签、图例、标注等
2. **字体必须足够大** — 图片会被缩放嵌入到双栏论文中，字号太小在 PDF 中完全看不清。参考磅数：
   - 子图标题 / 大标题: `fontsize=72`
   - 坐标轴标签 (xlabel/ylabel): `fontsize=60`
   - 刻度标签 (tick labels): `fontsize=50`
   - 图例 (legend): `fontsize=44~50`
   - 标注 (annotations): `fontsize=42~46`
   - colorbar 标签/刻度: `fontsize=50~54`
3. **不加 suptitle** — 标题由 LaTeX 的 `\caption{}` 提供，图片本身不需要总标题
4. **figsize 要配合大字号** — 通常 figsize 在 `(27, 18)` 到 `(48, 30)` 之间，配合上述字号
5. **线宽/marker** — linewidth 至少 3.5，markersize 至少 10，error bar capsize 至少 8
6. **输出** — `dpi=150`，`bbox_inches='tight'`，`facecolor='white'`

画图脚本位置: `/Users/eric/Desktop/tmp_Project/cue 画图/`
