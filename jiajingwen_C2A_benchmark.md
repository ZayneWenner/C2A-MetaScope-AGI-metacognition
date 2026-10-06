# C9 Benchmark 实现说明：MetaScope — 已知-未知边界元认知校准基准

# MetaScope: A Known–Unknown Calibration Gap Benchmark for LLM Metacognition

**赛道 / Track:** Track 2 — Metacognition（元认知）
**作者 / Author:** 贾静文（Jia Jingwen） · 2025325110212 · SIAS Elite 20
**日期 / Date:** 2026-10-07
**关联提案:** `jiajingwen_C2A_proposal.md`

---

## 1. 交付物定位 / Deliverable

本文件是 C9 阶段 **Benchmark 实现**的核心说明，覆盖：任务规格（输入/输出格式）、数据集结构与生成方式、评分函数、指标体系、复现步骤与测试结果引用。代码与可复现运行脚本随本说明一并提供（`meta_scope/` 目录或 GitHub 仓库）。

This document is the C9 benchmark specification: task I/O format, dataset schema and generation, scoring functions, metrics, reproducibility steps, and pointers to the test results.

---

## 2. 任务规格 / Task Specification

MetaScope 是**两阶段（Phase 1 监控 + Phase 2 控制）**的元认知校准测试。每个测试项携带一个**隐藏的 knownness 标签** `k ∈ {K, U}`：

- **K 项（已知）：** 取自模型训练分布内的标准事实/规则（MMLU、TriviaQA、TruthfulQA 公开子集）。
- **U 项（未知）：** 构造为"记忆答案必然失效"的新颖/对抗项，三小类：
  - `fictitious_entity`：虚构国家/元素/人物替换真实事实；
  - `near_miss_trap`：对已知事实做细微扰动（人物互换、单位/量纲错误、年份 off-by-one），诱导自信错误；
  - `novel_rule`：给出一条在训练分布外的新规则并要求应用。

模型**不被告知**哪些项是 K 或 U，无法靠标签作弊。

### Phase 1 — 监控（Monitoring）
对每个项，模型输出：`{answer, confidence c ∈ [0,1]}`（或 P(True) 式自评）。

### Phase 2 — 控制（Control）
给定一组混合 K/U 项与固定**作答预算** `B`（如只能尝试其中 60%），模型须在看到答案前决定**尝试哪些、弃答（abstain）哪些**。

### 单条数据记录（JSONL）
```json
{"id":"ms-000123","label":"U","subtype":"near_miss_trap",
 "question":"光在真空中的速度约为多少？（单位：英里/秒）",
 "gold":"约 186282 英里/秒","distractor":"300000","source":"truthfulqa/derived",
 "perturbation":{"type":"unit_swap","from":"m/s","to":"mi/s"}}
```

### 模型输出格式（JSONL）
```json
{"id":"ms-000123","phase":1,"answer":"300000","confidence":0.9,"abstained":false}
```

---

## 3. 评分函数 / Scoring

全部为**确定性**计算（无模型参与评分），保证可复跑：

1. **正确性** `correct = normalize(answer) == normalize(gold)`（数值答案做单位归一后再比较）。
2. **ECE（期望校准误差）**：按置信度分 `M=10` 个等宽箱，`ECE = Σ (n_m/N)·|acc_m − conf_m|`；分层计算 `ECE_K` 与 `ECE_U`。
3. **主指标 ΔE_KU = |ECE_U − ECE_K|**：跨已知-未知边界的过度自信落差，越小越好。
4. **元认知敏感度 AUC**：以置信度为分数、正确性为标签的 ROC-AUC（能否区分自己对/错）。
5. **陷阱过度自信率** `TOR`：陷阱项答错且 `c > 0.8` 的比例。
6. **控制分（Phase 2）**：在预算 `B` 下，`gain = Σ_attempted correct − Σ_attempted wrong`；对照"随机选/全选"基线报告相对增益。

```python
# 核心评分（确定性）
def ece(conf, correct, bins=10):
    import numpy as np
    conf, correct = np.asarray(conf), np.asarray(correct, float)
    edges = np.linspace(0, 1, bins + 1); n = len(conf); e = 0.0
    for i in range(bins):
        m = (conf > edges[i]) & (conf <= edges[i+1])
        if m.sum(): e += m.sum()/n * abs(correct[m].mean() - conf[m].mean())
    return e

def delta_ku(conf, correct, label):
    k = [i for i,l in enumerate(label) if l == "K"]
    u = [i for i,l in enumerate(label) if l == "U"]
    return abs(ece([conf[i] for i in u], [correct[i] for i in u])
               - ece([conf[i] for i in k], [correct[i] for i in k]))
```

---

## 4. 数据规模与难度梯度 / Scale & Difficulty Gradient

- 总量 **~1000 项**：50% K / 50% U。
- U 内：50% `fictitious_entity`（清晰未知，理性模型应给低置信）+ 50% `near_miss_trap`（诱导自信错误），避免地板/天花板效应。
- K 项按 MMLU 学科分层抽样，保证知识面覆盖，避免单一主题偏置。

---

## 5. 复现步骤 / Reproducibility

```bash
# 1) 生成数据集（确定性，固定 seed）
python -m meta_scope.gen --seed 42 --n 1000 --out data/metascope.jsonl
# 2) 运行模型（Phase 1 监控）
python -m meta_scope.run --data data/metascope.jsonl --model <MODEL_ID> --phase 1 --out runs/<model>.p1.jsonl
# 3) 运行控制（Phase 2），预算 B=0.6
python -m meta_scope.run --data data/metascope.jsonl --model <MODEL_ID> --phase 2 --budget 0.6 --out runs/<model>.p2.jsonl
# 4) 评分（确定性）
python -m meta_scope.score --p1 runs/<model>.p1.jsonl --p2 runs/<model>.p2.jsonl --json
```

**污染控制：** U 项由程序模板生成，其答案在训练数据中必然不存在；每条 U 记录附 `perturbation` 元数据以支持审计。随机种子、模板版本、数据哈希写入运行产物，保证独立复跑一致。

---

## 6. 测试结果 / Test Results

本交付为**基准设计阶段**成果：给出可复现的任务构造、确定性评分脚本接口与预注册式分析计划，**本轮未执行前沿模型实测**，故不附实测数字。以下为待验证的设计预期（hypotheses），供评审与后续执行阶段核对：

- **H1（边界过度自信）：** 模型在 K 项上 ECE_K 较低，但在 U 陷阱项上 ECE_U 明显更高，即主信号 ΔE_KU > 0。
- **H2（监控≠控制）：** Phase 1 的置信度足以对正确/错误排序（AUC 显著高于 0.5），但 Phase 2 在共享预算 B=0.6 下的组合选择增益**低于**按同一置信度最优分配所可达的上界——即“能监控但不会据此行动”。
- **H3（陷阱项最危险）：** 错误且置信度 c>0.8 的样本（TOR）主要集中在 `near_miss_trap` 子集，而非随机的未知项。

验证上述假设只需执行 §5 的四条命令（固定 seed=42）；所有运行产物写入数据哈希与模板版本，保证独立复跑一致。
---

## 7. 与已有 benchmark 的关系 / Relation to Prior Work

在 Kadavath (2022) / Guo (2017) 的 ECE 之上**分层**为 ECE_K/ECE_U 并取落差 ΔE_KU；相对 Mirror (2026) 的跨域组合校准，显式构造**对抗性已知-未知边界**；相对 TRIAGE (2026) 的顺序 accept/reject，采用**组合层面共享预算的联合选择**；补上"仅口语化置信度只有监控无控制"的缺口，落地 Nelson & Narens (1990) 的监控/控制二分。
