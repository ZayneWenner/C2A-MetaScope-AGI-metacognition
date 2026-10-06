# MetaScope · 已知–未知边界元认知校准基准

**C2A / C9 挑战交付** · Track 2 Metacognition（元认知）· 作者：贾静文（Jia Jingwen）· 2026-10-07

## 一句话

MetaScope 用**受控的「已知 / 未知」分界**把元认知拆成两条可测通道——**监控**（置信度是否校准）与**控制**（是否据此 abstain），主信号是跨边界的校准落差 **ΔE_KU**。

## 主指标

| 指标 | 定义 | 回答的问题 |
|---|---|---|
| **ΔE_KU**（主信号） | \|ECE_U − ECE_K\| | 模型在「不知道」时是否过度自信 |
| AUC | 置信度对正确/错误的排序能力 | 能否区分答对与答错（敏感度） |
| TOR | 错误且 c>0.8 的样本比率 | 最危险的「自信错误」有多常见 |
| 控制增益 | Phase 2 组合选择在预算 B 下的得分增益 | 监控信号是否真的驱动了行动 |

## 设计与规模

- 约 **1000 项**：50% K（MMLU 学科分层抽样）/ 50% U（程序模板生成，答案必然不在训练数据中）
- U 内再分半：`fictitious_entity`（清晰未知）+ `near_miss_trap`（近因陷阱：人物互换 / 单位错误 / off-by-one）
- ECE 采用 **M = 10** 个置信度分箱；全部评分确定性、固定 `seed=42`

## 两阶段协议

- **Phase 1（监控）**：先作答、再报告置信度 → ECE_K / ECE_U / ΔE_KU / AUC / TOR
- **Phase 2（控制）**：在共享作答预算 **B = 0.6** 下做**组合层面**的「尝试 / abstain」联合选择 → 控制增益（相对 TRIAGE 的顺序 accept/reject 设计补齐组合分配缺口）

## 交付物

| 文件 | 内容 |
|---|---|
| `jiajingwen_C2A_proposal.md` | 提案：问题、设计、指标、人类基线与评估计划 |
| `jiajingwen_C2A_benchmark.md` | 基准实现规格：任务 I/O、K/U 数据构造、确定性评分、复现步骤、与已有工作的关系 |
| `jiajingwen_C2A_AI日志.md` | AI 协作日志（本轮 AI 使用过程，含反向举证） |
| `jiajingwen_C2A_AAR.md` | KSTAR 式复盘：ΔR 归因与改进项 |
| `jiajingwen_C2A_拿来说明.md` | 借鉴来源、去掉了什么、我的改进（学术诚信声明） |

## 复现

```bash
python -m meta_scope.gen   --seed 42 --n 1000 --out data/metascope.jsonl
python -m meta_scope.run   --data data/metascope.jsonl --model <MODEL_ID> --phase 1 --out runs/<model>.p1.jsonl
python -m meta_scope.run   --data data/metascope.jsonl --model <MODEL_ID> --phase 2 --budget 0.6 --out runs/<model>.p2.jsonl
python -m meta_scope.score --p1 runs/<model>.p1.jsonl --p2 runs/<model>.p2.jsonl --json
```

## 交付状态（如实说明）

本轮为**基准设计阶段**交付：任务构造、评分接口与预注册式分析计划均已确定且可复现；**未执行前沿模型实测**，待验证的设计预期（H1–H3）见 `jiajingwen_C2A_benchmark.md` 第 6 节。

## 理论来源

DeepMind (2026) 认知框架 · Kadavath et al. (2022) · Guo et al. (2017) · Steyvers & Peters (2025) · Nelson & Narens (1990) · KSTAR 课程框架（详见 `jiajingwen_C2A_拿来说明.md`）
