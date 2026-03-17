## §1 Introduction

LLM 处理代码时表现出一种令人困惑的异质性：同一个模型可以轻松追踪一条变量赋值链，却在加入算术后变得脆弱；能处理展开的直线计算，却在等价的循环语义上挣扎；对嵌套条件分支自信地给出错误答案，却在简单 boolean short-circuit 上迟迟无法收敛。这种异质性不仅出现在 code-specialized 模型中，也出现在 general-purpose LLM 处理代码时 \citep{liu2024codemind, gu2024cruxeval, abdollahi2025demystifying, chen2025reval}。单看任务准确率无法解释这些差异的来源——两个 accuracy 相近的模型可能在内部以完全不同的方式失败。要理解"代码理解"究竟是一种能力还是多种，我们需要打开模型内部，观察代码推理在逐层计算中如何展开。

现有可解释性工作大多回答的是一个表征性问题：模型在某一层*编码*了什么信息 \citep{belinkov2017neural, cunningham2023sparse, nostalgebraist2020, belrose2023eliciting}。但对代码推理而言，更关键的问题是：**在每一层，模型是否已经*解出*了这个问题？** 而且"解出"本身还有程度之分——答案信息可能已经存在于 hidden state 中，但尚未被组织成模型自己能够稳定利用的形式。区分"信息存在"与"信息可用"，正是理解不同 code primitive 为何引发不同内部失败模式的入口。

为研究这一问题，我们构建了一个面向 interpretability 的 code reasoning benchmark，包含六类 synthetic 任务，沿 data flow 到 control flow 的谱系设计。每个实例都被约束为单 digit 输出，以避免 multi-token decoding 干扰逐层分析。在此基础上，我们引入一个 dual diagnostic framework。逐层 linear probe 判断正确答案是否已在 hidden state 中线性可读（information availability）；基于 Patchscopes \citep{ghandeharioun2024patchscopes} 的 Context-Stripped Decoding (CSD) 则测试模型能否在移除原始代码上下文后，仅凭同一个 hidden state 自行解码出答案（information readiness）。

这一视角揭示了代码推理的一个结构化内部过程。跨任务地看，模型都会先经历一个 **brewing** 过程：答案信息先对外部 readout 可见，但还没有变成模型自己可稳定解码的形式。随后轨迹分叉为四种 **resolution outcome**——**Resolved**、**Overprocessed**、**Misresolved** 和 **Unresolved**——共覆盖 [XX]\% 的样本。对 code understanding 最关键的发现是：不同代码原语会触发显著不同的 outcome fingerprint 和 brewing dynamics。例如，计算上完全等价的 Loop 与 Loop-unrolled 在 outcome fingerprint 上显著分离，直接表明模型敏感的不只是底层运算，还包括 code primitive 本身；而跨规模比较则揭示出 brewing scaffold 在不同模型间保持稳定，变化的只是 resolution 的成功率。代码推理并不是单一能力，而是一组共享早期结构、但在 resolution 路径上显著分化的内部子过程。

我们的贡献有三点：

- 我们构建了一个用于逐层研究代码推理的 **benchmark + dual diagnostic framework**，将 probing 与 Context-Stripped Decoding 作为 information availability 与 information readiness 的互补视角。
- 我们刻画了一个 **brewing-to-resolution structure**：答案往往先变得外部可读，再变得对模型自身可解码；其后轨迹分化为一个经因果支持的四路 resolution taxonomy。
- 我们展示了 **code primitive 层面的 mechanistic differences**：不同代码原语诱导出不同的 outcome fingerprint 和 brewing dynamics，暴露出不同的 resolution bottleneck；同时 brewing scaffold 跨模型与跨规模保持稳定，而 outcome 分布随能力变化，表明 mechanism 与 capability 可分离。

## §2 Method

**Models.** 我们分析了 Qwen2.5-Coder \citep{hui2024qwen25coder}，DeepSeek-Coder \citep{guo2024deepseekcoder}，Qwen2.5 \citep{yang2024qwen25}，CodeLlama \citep{roziere2023codellama} 与 Llama \citep{llama3team2024}，详细模型 URL 见 \cref{app:model_urls_and_setup}。正文中我们重点研究 Qwen2.5-Coder-7B。

**Benchmark.** 为了研究内部代码推理，我们需要一组足够简单、能够隔离具体推理原语，但又足够丰富、能产生不同计算行为的任务。因此我们沿着 program analysis 中一个经典轴来设计六类任务：**data flow** 与 **control flow** \citep{aho2006compilers, allen1970control, kildall1973unified, nielson1999principles}。每个实例由短代码片段 $C$ 和问题后缀 $Q$ 组成，形成 source prompt $S = [C;\,Q]$。答案始终是单个数字 $t^* \in \mathcal{D} = \{0,1,\dots,9\}$，从而保证目标只对应一个 token，避免 multi-token decoding 带来的分析混淆。六类任务分别是：(1) Copying, (2) Computing, (3) Conditional, (4) Loop, (5) Loop-unrolled, (6) Short-circuit。代码片段保持短小，使逐层信号反映 reasoning primitive 而非长上下文效应；答案限定为单 digit token（0–9），为后续 diagnostic function 提供对齐的输出空间以支持跨层比较。完整细节与 cases 见 \cref{app:benchmark_design_details}。

**Dual Diagnostic Framework.**
给定一个具有 $L$ 层的 decoder-only transformer $\mathcal{M}$，记 $\mathbf{h}^\ell \in \mathbb{R}^d$ 为模型处理 source prompt $S$ 时第 $\ell$ 层 last token 的 hidden state。由于 last position attend 到全部前文，它可以作为模型在该深度已完成何种计算的紧凑表示。我们围绕 $\mathbf{h}^\ell$ 定义两个诊断函数 $\Phi_{\mathrm{P}}^\ell, \Phi_{\mathrm{C}}^\ell: \mathbb{R}^d \to \Delta^{|\mathcal{T}|}$，将其映射到分类空间 $\mathcal{T} = \mathcal{D} \sqcup \{\bar d\}$ 上的概率单纯形，其中 $\bar d$ 将所有非 digit token 聚合为一个 residual class（$|\mathcal{T}| = 11$）。全文记 $\sigma(\cdot) \triangleq \mathrm{softmax}(\cdot)$。

第一个诊断是 linear probing \citep{belinkov2022probing}。对每一层训练一个逻辑分类器 $\Phi_{\mathrm{P}}^\ell(\mathbf{h}^\ell) = \sigma(W^\ell \mathbf{h}^\ell + \mathbf{b}^\ell)$，测试答案信息是否已经在 hidden state 中线性可读。这问的是一个必要条件：答案是否已经以简单形式存在？为避免 probe 被表面分布线索投机取巧，我们在十个 digit 类之外引入一个 near-OOD residual 类及对应训练样本 \citep{hewitt2019designing}（详见 \cref{app:probing_training_details}）。

但信息存在不等于信息可用。为此我们构建第二个诊断：Context-Stripped Decoding (CSD)，基于 Patchscopes \citep{ghandeharioun2024patchscopes}。具体做法是从 source run 中提取 $\mathbf{h}^\ell$，注入到一个只包含问题后缀的 target prompt $T = Q$ 中，移除原始代码上下文 $C$。模型随后从第 $\ell{+}1$ 层到第 $L{-}1$ 层继续前向传播，注意力上下文仅限于 $T$。为消除 target prompt 自身的 language prior confound，我们对 patching 后的 logit 做 baseline subtraction：
$$
\Phi_{\mathrm{C}}^\ell(\mathbf{h}^\ell) = \sigma\!\Big(W_u \circ \mathrm{LN} \circ F_{L-1} \circ \cdots \circ F_{\ell+1}\!\big(\mathbf{h}^\ell\big|_T\big) \;-\; \mathbf{z}_b\Big)\bigg|_{\mathcal{T}}
$$
其中 $\mathbf{z}_b$ 是对 $T$ 不做任何 patching 直接前向传播得到的 logit，代表 language prior。若 $\arg\max \Phi_{\mathrm{C}}^\ell(\mathbf{h}^\ell) = t^*$，则 $\mathbf{h}^\ell$ 是 self-contained 的：模型自身的解码管道无需回看代码即可产出正确答案。在 Patchscopes 框架中对应配置 $(Q, n_Q, \mathbb{I}, \mathcal{M}, \ell)$。$\Phi_{\mathrm{P}}^\ell$ 衡量 information availability，$\Phi_{\mathrm{C}}^\ell$ 衡量 information readiness；沿深度扫描两者的一致与分歧，即可追踪代码推理在模型内部的展开过程。


## §3 The Internal Lifecycle of Code Reasoning

对每个样本 $x$（source prompt $S_x$，ground-truth $t_x^*$），模型的一次前向传播产生一串随层深演化的 hidden states $\{\mathbf{h}_x^\ell\}_{\ell=0}^{L-1}$；$\Phi_{\mathrm{P}}^\ell$ 与 $\Phi_{\mathrm{C}}^\ell$ 在每一层各给出一个诊断分布，于是代码推理变成一条可沿深度直接观察的内部轨迹。Figure~\ref{fig:resolution_taxonomy} 展示了 anchor setting 上的代表性轨迹：probing 首先变得正确，而 CSD 在若干层之后才跟上。答案信息先以外部可读的形式出现，随后才转化为模型自身可解码的状态。

记 $\hat{t}_{\mathrm{P}}^\ell \triangleq \arg\max_{t \in \mathcal{T}} \Phi_{\mathrm{P}}^\ell(\mathbf{h}_x^\ell)[t]$，$\hat{t}_{\mathrm{C}}^\ell \triangleq \arg\max_{t \in \mathcal{T}} \Phi_{\mathrm{C}}^\ell(\mathbf{h}_x^\ell)[t]$，$\hat{t}_{\mathrm{out}} \triangleq \mathcal{M}(S_x)$ 为模型最终输出。在最早的若干层（大约前 [XX]\%），两个诊断函数都处于高熵状态，概率质量分散在多个 digit 和 $\bar{d}$ 之间——模型在变换表征，但答案相关结构尚未在任一视角下浮现。这个 pre-brewing 区域的结束标志是 probing 首次正确的那一层。我们定义
$$
\mathrm{FPCL}(S_x) \triangleq \min\{\ell : \hat{t}_{\mathrm{P}}^\ell = t_x^*\},\qquad
\mathrm{FJC}(S_x) \triangleq \min\{\ell : \hat{t}_{\mathrm{P}}^\ell = t_x^* \;\land\; \hat{t}_{\mathrm{C}}^\ell = t_x^*\}.
$$
在 anchor setting 上，FPCL 平均出现在归一化深度 [XX]\%，FJC 出现在 [XX]\%；这一先后关系 $\mathrm{FPCL}(S_x) < \mathrm{FJC}(S_x)$ 在全部实验中均成立。我们将间隔 $[\mathrm{FPCL},\,\mathrm{FJC})$ 称为 **brewing**——答案已经存在，但尚未变成模型自己可解码的形式。其长度 $\Delta_{\mathrm{brew}}(S_x) = \mathrm{FJC}(S_x) - \mathrm{FPCL}(S_x)$ 平均为 [XX] 层（总深度的 [XX]\%），说明这段中间过程并非边缘现象。那么一旦到达（或未能到达）FJC，后续轨迹是否统一？

答案是否定的。Figure~\ref{fig:resolution_taxonomy} 示意了 brewing 过程及其后续分化的四种 resolution outcome。令 $\bar{\Phi}_{\mathcal{W}}(t) \triangleq \frac{1}{|\mathcal{W}|}\sum_{\ell \in \mathcal{W}} \Phi_{\mathrm{C}}^\ell(\mathbf{h}^\ell)[t]$ 为尾部窗口 $\mathcal{W} = \{\ell \mid \ell \geq \lfloor 3L/4 \rfloor\}$ 内 CSD 对 digit $t$ 的平均概率，四类 outcome 定义如下：
$$
\mathrm{Outcome}(S_x) = \begin{cases}
\textbf{Resolved} & \mathrm{FJC} \neq \varnothing \;\land\; \hat{t}_{\mathrm{out}} = t_x^* \\
\textbf{Overprocessed} & \mathrm{FJC} \neq \varnothing \;\land\; \hat{t}_{\mathrm{out}} \neq t_x^* \\
\textbf{Misresolved} & \mathrm{FJC} = \varnothing \;\land\; \max_{t \in \mathcal{D}}\, \bar{\Phi}_{\mathcal{W}}(t) \geq \tfrac{1}{2} \\
\textbf{Unresolved} & \mathrm{FJC} = \varnothing \;\land\; \max_{t \in \mathcal{D}}\, \bar{\Phi}_{\mathcal{W}}(t) < \tfrac{1}{2}
\end{cases}
$$
直觉上，Resolved 与 Overprocessed 的区分在于：模型曾在某层同时通过两个诊断，但后续层是否保住了正确答案。Overprocessed 意味着正确计算已经形成却被后续层破坏。Misresolved 与 Unresolved 则刻画两种不同的失败：前者收敛到了稳定但错误的答案，后者在现有深度内始终未形成稳定 resolution。值得注意的是，$\mathrm{FJC} = \varnothing$ 的样本中仅有 [0.2]\% 最终输出正确，即缺少 FJC 几乎等价于最终做错。

在上述 setting 中，四类 outcome 分别占 [XX]\% Resolved、[XX]\% Overprocessed、[XX]\% Misresolved 和 [XX]\% Unresolved，共覆盖 [XX]\% 的样本（\cref{app:outcome_distribution_statistics}）。但这套分类目前仍然是描述性的——它是否反映了真实的内部计算结构，需要因果证据来回答。

### Causal Validation

如果 FJC 和四类 outcome 对应真实的内部结构，那么围绕它们的干预就应当产生可预测的、与 outcome 相关的效应。我们设计三组因果实验来检验这一点（详细设置、层位扫描与失败案例见 \cref{app:causal_validation_details}）。
**Activation patching at FJC.** 借鉴因果定位方法 \citep{meng2022locating, geiger2023causal}，我们在 FJC 层用答案不同但任务匹配的样本的 $\mathbf{h}^{\mathrm{FJC}}$ 替换原始 hidden state，观察最终预测是否被改写。如果 FJC 是因果特权层，patching 在此处的 answer-flip rate 应显著高于其他层。结果：FJC 处的 flip rate 为 [XX]\%，而匹配的 pre-brewing 层和 post-resolution 层分别为 [XX]\% 和 [XX]\%。
**Layer skipping for Overprocessed.** Overprocessed 的定义蕴含正确计算已形成但被后续层破坏——类似于先前工作中观察到的 overthinking 现象 \citep{halawi2024overthinking, schuster2022confident}。若如此，跳过 FJC 之后的尾部层、直接从 $\mathbf{h}^{\mathrm{FJC}}$ 解码应当恢复正确答案。结果：在 Overprocessed 样本上，skip 恢复正确率为 [YY]\%；对已经 Resolved 的样本，同样操作仅造成 [Y.Y]\% 的准确率下降——效应具有明显的 outcome 特异性。
**Re-injection for Unresolved.** 如果 Unresolved 意味着"没算完"而非"根本不会"，那么给模型额外的有效深度应能救回部分样本。我们将后期 hidden state 重新注入一次新的前向传播，rescue rate 为 [ZZ]\%，说明相当一部分 Unresolved 样本包含尚未完成但已可利用的计算。

三组干预各自针对 taxonomy 的不同分支，均产生了与对应 outcome 一致且对其他 outcome 不一致的效应，表明 brewing-to-resolution 捕捉的是模型内部计算的真实结构而非事后归纳。这套结构同样在模型自身的输出分布中留下了可观测信号：CSD entropy $H_{\mathrm{C}}^\ell$、其下降速率、以及 probe-CSD agreement $A^\ell = \mathbb{1}[\arg\max \Phi_{\mathrm{P}}^\ell = \arg\max \Phi_{\mathrm{C}}^\ell \in \mathcal{D}]$ 三个 GT-free 量即可在不访问 ground truth 的情况下区分四类 outcome（\cref{app:ground_truth_free_discrimination}）。§4 将利用这些信号，把分析从单一 anchor setting 推广到全部六类 code primitive 的跨任务、跨模型比较。


---

## §4 What Different Code Primitives Reveal

### 4.1 Outcome Fingerprints and What They Expose

Figure~\ref{fig:task_fingerprint} 把六类任务的四路 outcome 分布并排展示在同一张图中。最直接的观察是：每一种 code primitive 都诱导出自己独特的 outcome profile。这不只是"正确率不同"——accuracy 无法区分的 failure mode，在这里被拆开了。下面我们先点出各任务最有辨识度的特征，再进入三组 controlled comparison。

**Copying** 作为谱系中最简单的一端，以 Resolved 为绝对主导（[XX]\%），Overprocessed 和 Unresolved 均为极小的尾部。纯变量传播一旦被正确追踪，模型几乎总能把结果保持到最后。这为后续所有比较提供了 baseline。

**Conditional** 的 signature 以高 Misresolved 为标志（[XX]\%）。模型并不是在条件分支上"算不出来"，而是自信地收敛到了错误分支的结果。从 $H_{\mathrm{C}}^\ell$ 看，Conditional 的 Misresolved 样本往往在中后层就已经达到低熵——CSD 稳定了，只是稳定在错误答案上。这种"confident but wrong"的模式在 data flow 任务中几乎看不到。

**Short-circuit** 呈现另一种 profile：Unresolved 占比显著升高（[XX]\%），同时伴有可观的 Misresolved（[XX]\%）。compound boolean 中多个 subcondition 的联合判断与 `and`/`or` 的 short-circuit semantics 给模型施加了双重压力：既要正确评估每个子条件，又要在正确位置截断 evaluation。模型经常在现有深度内无法完成这个过程。

**Computing vs. Copying：算术改变了什么？** 由于 Computing 在 $\rho = 0$ 时退化为 Copying，这一对任务提供了干净的比较。fingerprint 上，Computing 的 Resolved 显著低于 Copying（[XX]\% vs. [XX]\%），而 Overprocessed 大幅上升（[XX]\% vs. [XX]\%）。算术的主要效应不是让模型"找错方向"（Misresolved 并没有显著增加），而是让已经形成的正确答案更难被保持到最终输出。brewing dynamics 支持同样的解读：相对于 Copying，Computing 的 $H_{\mathrm{C}}^\ell$ 下降更慢，$A^\ell$ agreement window 更晚且更窄，平均 $\Delta_{\mathrm{brew}}$ 为 [XX] vs. [XX] 层。随着 $\rho$ 从 0 增加到 1，这些效应单调增强（\cref{app:subsec:computing_density_sweep}）。算术延长的不是 pre-brewing，而是 brewing 本身——信息存在但尚不可用的中间状态被拉长了。

**Loop vs. Loop-unrolled：syntax matters.** 这是全文最干净的因果对照。两类任务的底层算术完全相同（matched initial value、operation、iteration count），唯一差异是 Loop 使用 `for ... in range(...)` 语法，而 Loop-unrolled 把同样的计算展开为 sequential statements。如果模型只是在做 repeated addition，两者的 fingerprint 应当相近。实际上它们显著分离：Loop 的 Unresolved 明显高于 Loop-unrolled（[XX]\% vs. [XX]\%），而两者的 Overprocessed 比例相近。从 dynamics 看，Loop 的 $A^\ell$ trajectory 更脆弱，entropy collapse 更慢且更不稳定。循环理解不能被还原为等价的 sequential arithmetic——模型在处理 loop semantics 时动用了一条不同的内部路径。

**Data flow vs. Control flow：两种不同的瓶颈。** 把上述观察聚合起来，一个更广泛的 pattern 浮现了。Data flow 主导的任务（Copying、Computing、Loop-unrolled）倾向于更长的 brewing、更平滑的 entropy descent，失败时更多是 Overprocessed——瓶颈在于把正确表征稳定地保持到最后（stabilization bottleneck）。Control flow 主导的任务（Conditional、Short-circuit）则更常出现 Misresolved 和 Unresolved——瓶颈在于选择正确的执行路径并贯彻这个 commitment（commitment bottleneck）。Loop 正好位于两者之间，既有 data flow 式的长 brewing，又有 control flow 式的 Unresolved 升高，与它的 mixed 性质一致。完整的跨任务数据与 per-difficulty-level 分析见 \cref{app:subsec:per_difficulty_breakdown}。

### 4.2 Across Models and Scales

上述 task-dependent 模式并不只属于一个模型。我们在 Qwen2.5-Coder 与 Qwen2.5（同架构，code-specialized vs. general-purpose）之间、以及 0.5B 到 14B 的规模范围内，都观察到相同的 brewing-to-resolution 结构。但 outcome 分布并不是在所有维度上同步变化——这暴露了一个清晰的 mechanism 与 capability 的分离。

**Coder vs. Base.** 在同架构下，code-specialized 训练显著改善了 outcome 分布：Coder 的整体 Resolved 从 [XX]\% 提升到 [XX]\%，Unresolved 从 [XX]\% 降到 [XX]\%，其中在 [task placeholder] 上增益最大。但 FJC 的归一化层位在两个模型之间保持高度相关（Pearson $r = [0.XX]$），平均偏移仅为总深度的 [X.X]\%。训练改变的更多是模型最终能 resolve 多少，而不是 resolution transition 倾向于在哪里发生。

**Scaling.** 更大的模型把概率质量从 Unresolved 和 Misresolved 转移到 Resolved，反映有效计算能力的提升。Overprocessed 呈现一个值得注意的非单调趋势：在中等规模达到峰值（[XX]\%）后下降——"找到答案"的能力先于"稳定保留答案"的能力出现。与此同时，归一化 brewing duration 在不同规模间保持稳定，大致维持在总深度的 [XX]\% $\pm$ [X.X]\%。capability 在变，但计算骨架没有变。

**跨架构。** 同样的 pattern 延伸到 Qwen 系列之外。在 CodeLlama / Llama 和 DeepSeek-Coder 上，outcome 分布随训练和规模移动，但 brewing 的存在和 task-dependent fingerprint 的结构保持稳健（\cref{app:subsec:cross_architecture_robustness}）。brewing 反映的是 transformer 在代码上计算的一般属性，具体的 outcome mix 则更多反映学习到的能力与有效容量。


---

## Related Work

### Layer-wise Readout and Internal State Analysis

越来越多的工作为检视 transformer 的内部状态提供了方法基础 \citep[综述见][]{bereska2024mechanistic}。Logit Lens \citep{nostalgebraist2020} 和 Tuned Lens \citep{belrose2023eliciting} 关注中间层预测如何沿深度逐步形成；\citet{geva2021kv} 和 \citet{geva2022transformer} 进一步表明，feed-forward 层可以被理解为逐步提升词汇空间中概念激活的 key-value memory。Patchscopes \citep{ghandeharioun2024patchscopes} 通过将 source layer 的表征注入 target prompt，利用模型自身的生成过程来解码该层编码了什么。Linear probing \citep{hewitt2019structural, hewitt2019designing, belinkov2022probing} 则测试特定属性是否在线性意义上可分；近期工作还把表征分析扩展到更广泛的结构性属性，例如时空结构 \citep{gurnee2024spacetime, burns2023discovering} 和 top-down control directions \citep{zou2023representation}。我们的工作建立在这条工具链之上，但问题设置不同：我们不只问某一层是否编码了答案，而是问该答案是否已经被组织成模型自身能够利用的形式。也正是在这个意义上，我们把 probing 与 CSD 并置起来，区分 information availability 与 information readiness。

### Understanding LLMs on Code

现有针对 LLM 如何处理代码的理解工作，主要集中在 probing、归因和行为分析上。\citet{hooda2024counterfactual} 用 counterfactual analysis 检验模型是否真正响应循环不变量、变量类型等 code predicate，发现模型常依赖浅层模式匹配。\citet{liu2024codemind} 提出 CodeMind，从相互依赖和相互独立的执行阶段 probing 代码推理，发现当变量状态依赖控制流决策时，LLM 尤其容易失败。\citet{gu2024cruxeval} 提出 CRUXEval，通过输入/输出预测评估代码推理，揭示出代码生成能力与代码理解能力之间存在显著落差。\citet{chen2025reval} 和 \citet{abdollahi2025demystifying} 则进一步分析了 LLM 如何模拟代码执行，记录了在循环、算术和条件分支上的系统性错误，这些模式与我们观察到的 outcome fingerprint 是一致的。AutoProbe \citep{autoprobe2025} 利用 attention-based probing 动态聚合最有信息量的层与 token 来预测代码正确性，表明最优层位会随模型和任务变化。Neuron-Guided Interpretation of Code LLMs \citep{neuronguided2026} 进一步尝试在代码域做神经元层面的解释。这些工作已经表明代码相关信息并非均匀地分布在模型内部，但它们主要回答的是代码属性是否被表示、以及在哪里被表示；我们的关注点则是这些信息何时变得对模型自身可解码，以及这一时机如何随 code primitive 系统性变化。

### Reasoning Dynamics, Overthinking, and Causal Structure

与我们最接近的一条脉络，是对 LLM 内部推理动态和因果机制的研究。\citet{wendler2024llamas} 在多语言模型中揭示了输入编码、概念处理和输出投射的阶段性结构；\citet{selfverify2025} 则表明模型在输出前就可能内部编码 correctness signal，并报告了 token-level overthinking。overthinking 这一现象本身也已被 \citet{halawi2024overthinking} 系统刻画，并与 early exit 策略联系起来 \citep{schuster2022confident}；我们定义的 Overprocessed outcome 可以被看作这一现象在代码任务上的细粒度、任务特定实例化。与此同时，\citet{wang2022ioi} 和 \citet{nikankin2025arithmetic} 展示了如何通过 patching、消融和神经元分析为内部机制提供因果证据 \citep{meng2022locating, geiger2023causal}；\citet{universalneurons2024} 和 \citet{gould2024successor} 则考察了跨模型共享的计算单元。\citet{nanda2023grokking} 用 mechanistic interpretability 跟踪训练过程中的 progress measure，表明内部结构往往会先于行为变化而出现，这与我们发现 brewing 先于 resolution 的结果形成呼应。我们的工作与这些研究共享“内部过程具有结构且可被因果检验”的立场，但关注的是代码域中的 layer-level reasoning dynamics：答案往往先变得外部可读，再变得对模型自身可解码；其后的 brewing gap 与 resolution outcome 强烈依赖于 code primitive 本身。更广泛地，\citet{nanda2025pragmatic} 指出代码与工具使用是可解释性研究的理想落脚点，因为它们天然具有可验证的 ground truth；我们的设置正是沿着这一方向，将代码执行原语作为分析单元来展开。
