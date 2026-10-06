# C2A Proposal: MetaScope — 已知-未知边界的元认知校准基准

# MetaScope: A Known–Unknown Calibration Gap Benchmark for LLM Metacognition

**赛道 / Track:** Track 2 — Metacognition（元认知）
**作者 / Author:** 贾静文（Jia Jingwen）
**日期 / Date:** 2026-10-07

---

## 1. 赛道选择与动机 / Track Selection & Motivation

### 为什么选择 Metacognition 赛道？ / Why Track 2?

当我们说一个 AI"知道答案"，我们到底在说什么？当前 LLM 评估长期混淆两件事：**知识（knowledge）** 与 **元认知（metacognition）**。一个模型在 MMLU 上拿高分，只说明它"记得住或算得出"，并不说明它"知道自己什么时候会错"。而 DeepMind 认知框架（2026）明确把元认知定义为"对自身认知过程的监控与理解"，并指认它是**幻觉的根源**——模型往往不是不知道，而是不会表达"我不知道"。这正是五项评估缺口中缺口最大、也最危险的能力之一。

When we say an AI "knows the answer," what do we actually mean? Current LLM evaluation conflates two distinct things: knowledge and metacognition. A high MMLU score shows a model can retrieve or compute, but not that it knows when it is wrong. DeepMind's cognitive framework (2026) defines metacognition as "monitoring and understanding of one's own cognitive processes" and identifies it as the root cause of hallucination — the model often does not know less than it fails to express its uncertainty. Among the five tracks, metacognition has both the largest evaluation gap and the highest stakes for safe deployment.

### KSTAR 连接 / KSTAR Connection

KSTAR 用 **ΔE（认知偏差 / epistemic delta）** 刻画"系统的预期置信与实际结果之间的偏差"。真正的元认知，等价于要求模型在看到结果之前，其**自我预测的置信度 R̂_E 逼近真实正确率 R_E**——即 R̂_E ≈ R_E。我们的基准将直接测量这个偏差，并把 KSTAR 的"抑制（inhibition）"机制落地为"当置信度低时选择 abstain（不答）"的控制行为：模型不仅要能监控自己，还要能据此行动。

In KSTAR, ΔE (epistemic delta) captures the gap between a system's predicted confidence and its actual outcomes. Genuine metacognition requires the model's self-predicted confidence R̂_E to approximate its true accuracy R_E *before* seeing the result. Our benchmark measures exactly this gap, and operationalizes KSTAR's inhibition as a control behavior: when confidence is low, the model abstains rather than answers. Metacognition here means not just monitoring, but acting on that monitoring.

---

## 2. Benchmark 设计思路 / Benchmark Design

### 2.1 核心设计：已知-未知边界与"自信错误陷阱" / Core Design: Known–Unknown Boundary & Confident-Error Traps

MetaScope 的核心思想是**程序化构造一条"已知 / 未知"边界**，并专门埋设"自信错误陷阱"项。每个测试项都来自一个受控的 knownness 标签：

- **K 项（已知 / Known）：** 取自模型训练分布内的标准事实或规则（如 MMLU、TriviaQA 子集），模型应当会且自信。
- **U 项（未知 / Unknown）：** 构造为"记忆答案必然失效"的新颖或对抗项，包括 ① 虚构实体项（用虚构国家 / 元素替换真实事实）；② 近因陷阱项（对已知事实做细微扰动——人物互换、单位 / 量纲错误、年份 off-by-one）；③ 新颖规则实例。

模型**不知道**哪些项是 K 或 U，无法靠标签作弊——这与 SynthRule 用"程序生成全新规则"防记忆泄漏是同一思路，但针对的是元认知而非学习能力。

MetaScope's core idea is to procedurally construct a controlled known–unknown boundary, with deliberate "confident-error traps." Each item carries a hidden knownness label: K-items are drawn from the model's training distribution (standard facts/rules from MMLU, TriviaQA); U-items are novel or adversarially perturbed so memorized answers fail — fictitious-entity items, near-miss trap items (subtle perturbations: swapped figures, unit errors, off-by-one years), and novel rule instances. The model is never told which items are K or U, so it cannot cheat via the label — the same anti-memorization logic as SynthRule, but aimed at metacognition rather than learning.

### 2.2 任务描述 / Task Description

**Phase 1 — 监控（Monitoring）：** 对每个项，模型先作答，再输出置信度 c ∈ [0,1]（或 P(True) 式自评）。

**Phase 2 — 控制（Control）：** 给模型一组混合了 K/U 的项，并设一个"作答预算"B（如只能尝试其中 60%）。模型须在**看到答案之前**，自行选择尝试哪些、放弃（abstain）哪些。

**示例项（U 类近因陷阱）：**
```
问：光在真空中的速度约为多少？（单位：英里 / 秒）
模型已知：约 3×10^8 m/s（= 186,282 mi/s）
陷阱：若误用 km/s 或忘记换算，会自信给出 300,000
正确：≈186,282 英里 / 秒；错误但"看似熟悉"的答案：300,000
```
模型必须对"我这个单位换算到底靠不靠谱"给出校准的置信度——这正是元认知在边界上的真实考验。

Phase 1 (Monitoring): for each item the model answers and outputs a confidence c∈[0,1]. Phase 2 (Control): given a mixed K/U set and a fixed attempt budget B (e.g., may attempt only 60%), the model must choose, before seeing answers, which items to attempt and which to abstain. Example U-trap: "Speed of light in vacuum in miles per second?" — a model that confuses units may confidently output 300,000 (the km/s figure) instead of ≈186,282. The model must calibrate its confidence about whether its unit handling is reliable — exactly where metacognition is truly tested at the boundary.

### 2.3 为什么它测量"元认知"而非其他能力 / Why This Measures Metacognition

我们刻意把**置信度与正确率分开计分**，从而隔离"自我评估"与"知识本身"；并用 K/U 切分**控制难度变量**：一个只会"在看起来难的项上降低置信度"、却分不清"已知难题"和"未知"的模型，会在 U 项上严重过度自信，表现为巨大的 ΔE_KU 落差。真正的元认知最小化这个落差。Phase 2 进一步要求"依自我认知行动"，把 Nelson & Narens（1990）的监控 / 控制二分落到可测行为上。

We score confidence separately from accuracy, isolating self-assessment from knowledge itself; and we use the K/U split to control for difficulty. A model that merely lowers confidence on "hard-looking" items but cannot distinguish known-hard from unknown will be overconfident on U-items, yielding a large ΔE_KU gap. Genuine metacognition minimizes this gap. Phase 2 further requires acting on self-knowledge, operationalizing Nelson & Narens' (1990) monitoring/control distinction into measurable behavior.

### 2.4 关键指标 / Key Metrics

| 指标 | 公式 / 含义 | 测量什么 |
|------|----------|----------|
| ECE_K / ECE_U | 已知 / 未知项的期望校准误差 | 分别在两类上的校准质量 |
| **ΔE_KU** | \|ECE_U − ECE_K\|（**主信号**） | 跨已知-未知边界的过度自信 |
| 元认知敏感度 AUC | 置信度对"对错"的排序能力 | 能否区分自己的对 / 错（Steyvers & Peters, 2025） |
| 陷阱过度自信率 | 陷阱项答错且 c > 0.8 的比例 | 最危险的"自信错误"模式 |

Key metrics: ECE_K and ECE_U (expected calibration error on known/unknown items); **ΔE_KU = |ECE_U − ECE_K|** as the primary signal of overconfidence across the boundary; Metacognitive Sensitivity (AUC) of confidence vs. correctness (Steyvers & Peters, 2025); and Trap Overconfidence Rate (fraction of trap items answered wrongly with c>0.8).

### 2.5 设计灵感 / Design Inspiration

- **Kadavath et al. (2022, arXiv:2207.05221)：** 采纳"口语化置信度 + P(True) 校准"框架；舍弃其单一聚合 ECE 与 OOD 泛化顾虑。
- **Guo et al. (2017, ICML)：** 采纳 ECE 指标；扩展为分层 ECE_K / ECE_U。
- **Steyvers & Peters (2025)：** 采纳"敏感度 vs 校准"二分，用 AUC 测敏感度。
- **Mirror (Wang, 2026, arXiv:2604.19809)：** 采纳多层元认知评估；我们的差异点在于**聚焦已知-未知边界并埋设对抗陷阱**，报告落差 ΔE_KU 而非组合校准误差（CCE）。
- **TRIAGE (2026, arXiv:2605.13414)：** 采纳"预算约束下的前瞻控制"；我们的差异点在于**组合层面的联合选择**（而非顺序 accept/reject），正好补上 TRIAGE 自承"无法暴露"的组合分配难题。
- **DeepMind 框架 (2026) + KSTAR：** 采纳元认知定义与三阶段协议；将 inhibition 与 ΔE 映射为控制阶段与 ΔE_KU。

Inspiration: Kadavath et al. (2022) for verbalized confidence and calibration (we drop the single aggregate ECE); Guo et al. (2017) for ECE (we stratify it); Steyvers & Peters (2025) for the sensitivity/calibration split; Mirror (2026) for multi-level evaluation (we target the known–unknown boundary with adversarial traps and report ΔE_KU instead of compositional CCE); TRIAGE (2026) for prospective control under budget (we use portfolio-level joint selection, which TRIAGE's sequential design explicitly cannot expose). DeepMind (2026) and KSTAR supply the definition, three-stage protocol, and the inhibition/ΔE mapping.

---

## 3. 人类基线考量 / Human Baseline Considerations

### 预期人类表现分布 / Expected Human Performance

| 人群 | K 项（已知） | U 项（未知） | ΔE_KU 预期 |
|------|-------------|-------------|-----------|
| 普通成人 | 高准确 + 校准良好 | 易过度自信（"未知未知"） | 中 |
| 逻辑 / 科学训练者 | 高准确 + 校准 | 更敢说"不知道" | 较小 |
| 儿童（8–12 岁） | 中等 | 高度过度自信 | 大 |

### 区分度设计 / Discrimination Design

人类与 AI 在 K 项上都应高准确且较校准——这是基线校准；区分度来自 U 项与 ΔE_KU：

- **过易风险：** 部分 K 项人人都会，不构成区分；我们用 U 项（尤其陷阱）制造难度。
- **过难风险：** 极端新颖项可能让所有人都随机；我们控制 U 项占 50%，其中"清晰未知"（虚构实体，人类能理性给出低置信）与"自信陷阱"（诱导自信错误）各半，避免地板效应。
- **难度梯度：** 50% K / 50% U；U 内 50% 虚构实体 + 50% 近因陷阱。

Humans and AI should both be accurate and reasonably calibrated on K-items — that is the baseline. Discrimination comes from U-items and ΔE_KU. To avoid ceiling effects, U-items (especially traps) provide difficulty; to avoid floor effects, we cap U at 50% with half clearly-unknown (fictitious entities, where humans rationally give low confidence) and half near-miss traps (which induce confident errors). Gradient: 50% K / 50% U.

---

## 4. 预期创新点与可行性 / Innovation & Feasibility

### 4.1 创新点 / What's New

| 现有 Benchmark 的局限 | MetaScope 如何解决 |
|----------------------|-------------------|
| Kadavath / MMLU 校准：单一聚合 ECE，混淆已知 / 未知 | 分层 ECE_K / ECE_U + 落差 ΔE_KU |
| Mirror：跨域组合校准误差，无对抗已知-未知边界 | 显式 AKUS + 自信错误陷阱 |
| TRIAGE：顺序 accept/reject，无法暴露组合分配 | 组合层面联合选择（共享预算） |
| 仅口语化置信度：只有监控，无控制 | 监控 + 控制双通道 |

**最核心的创新：** 把"元认知"从"整体校准好不好"细化为"在已知-未知边界上是否过度自信"，并补上"能否依此行动"的控制通道——直接对应 KSTAR 的 ΔR→抑制闭环。

Core innovation: we refine metacognition from "is overall calibration good?" to "is the model overconfident specifically across the known–unknown boundary?" and add a control channel testing whether it can act on that self-knowledge — directly mapping to KSTAR's ΔR→inhibition loop.

### 4.2 可行性 / Feasibility

| 资源 | 方案 | 时间 |
|------|------|------|
| K 项数据 | MMLU / TriviaQA / TruthfulQA 子集（公开） | 1 天 |
| U 项生成器 | Python 模板程序生成虚构实体 + 近因陷阱约 1000 项 | 1.5 天 |
| 评分函数 | ECE 分箱 + AUC（sklearn）+ 控制分公式，确定性 | 1 天 |
| 模型测试 | API 调用 2–3 个前沿模型（取 logprobs / 口语化置信） | 2 天 |
| 文档 + 提交 | Kaggle Community Benchmarks 格式 | 1.5 天 |

全部基于文本 / API 推理，无需 GPU 训练；U 项程序生成保证"答案不可能在训练数据中"，彻底消除数据污染。可在 **7 天**内完成。

All components are text/API-based, requiring no GPU training. The programmatic U-item generator guarantees answers are absent from training data, eliminating contamination. Feasible within 7 days using public K-item datasets, a Python trap generator (~1000 items), deterministic scoring (ECE binning, AUC via sklearn, control-score formula), and API calls to 2–3 frontier models.

---

## 参考文献 / References

1. DeepMind (2026). *Measuring Progress Toward AGI: A Cognitive Framework*.
2. Kadavath, S. et al. (2022). Language Models (Mostly) Know What They Know. *arXiv:2207.05221*.
3. Guo, C. et al. (2017). On Calibration of Modern Neural Networks. *ICML*.
4. Steyvers, M. & Peters, M. A. K. (2025). Metacognition and Uncertainty Communication in Humans and Large Language Models.
5. Wang, J. Z. (2026). Mirror: A Hierarchical Benchmark for Metacognitive Calibration in LLMs. *arXiv:2604.19809*.
6. (2026). TRIAGE: Evaluating Prospective Metacognitive Control in LLMs under Resource Constraints. *arXiv:2605.13414*.
7. Mei, Z. et al. (2025). Reasoning about Uncertainty: Do Reasoning Models Know When They Don't Know? *arXiv:2506.18183*.
8. Nelson, T. O. & Narens, L. (1990). Metamemory: A Theoretical Framework and Some New Findings.
