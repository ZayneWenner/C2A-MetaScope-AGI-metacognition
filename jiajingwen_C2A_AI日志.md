# AI 生成日志 / AI Generation Log

**作者 / Author:** 贾静文（Jia Jingwen）
**挑战 / Challenge:** C2A — Track 2 Metacognition（元认知）
**日期 / Date:** 2026-10-07

---

## 工具 / Tools Used

- **WorkBuddy（AI 助手）** — 论文精读、benchmark 调研、提案撰写、迭代优化、PDF 解析、本次全部交付物生成
- **WebSearch（联网检索）** — 核实 benchmark 真实出处与最新进展（2024–2026）
- **Python 3.13 + pdfplumber** — 解析资料包中的两份 PDF（DeepMind 摘要、C2A 指南）与解压 starter 模板

---

## 论文精读 / Paper Analysis

- **方法：** 将资料包中的 `deepmind-agi-cognitive-framework-summary.pdf` 交给 AI 解析，提取：
  1. 10 个认知能力的定义与层级（8 项基础 + 2 项复合）
  2. 五个评估缺口最大赛道（Learning / Metacognition / Attention / Executive Functions / Social Cognition）的具体描述
  3. 三阶段评估协议（认知评估 → 人类基线 → 认知画像）
  4. Metacognition 赛道被定义为"幻觉的根源"——这一表述直接成为我选题的核心动机
- **关键提取：** 论文明确指出元认知是"对自身认知过程的监控与理解"，且当前评估缺口大。AI 帮我把它与课程 KSTAR 框架的 **ΔE（认知偏差）** 对齐：元认知 ≈ 要求模型在结果揭晓前，自我预测置信 R̂_E ≈ 真实正确率 R_E。
- **迭代：** 1 轮精读 + 1 轮追问（确认 DeepMind 原文与课程指南在截止日期上的差异：课程冻结定义为 12/31，指南 PDF 写 4/9，我以课程 `challenge.json` 的 12/31 为准）。

---

## Benchmark 调研 / Benchmark Research

- **搜索策略（共 3 组并行检索，覆盖 8+ 个相关基准）：**
  - Query 1: `Kadavath 2022 Language Models Mostly Know What They Know verbalized confidence`
  - Query 2: `LLM metacognition benchmark confidence calibration 2024 2025`
  - Query 3: `DeepMind measuring progress toward AGI cognitive framework metacognition track`
- **发现的关键参考（按与本项目相关度）：**
  - **Kadavath et al. (2022, arXiv:2207.05221)：** 奠基性——P(True) / P(IK) 口语化置信度与校准，但只用单一聚合 ECE，且 OOD 校准泛化差。
  - **Steyvers & Peters (2025)：** 明确区分"元认知敏感度（AUC）"与"校准（ECE）"两个维度，成为我的指标设计依据。
  - **Mirror (Wang, 2026, arXiv:2604.19809)：** 多层元认知校准基准，发现"组合校准误差"与"知-行鸿沟"——但聚焦跨域组合，未构造对抗性的已知-未知边界。
  - **TRIAGE (2026, arXiv:2605.13414)：** 预算约束下的前瞻元认知控制，但采用"顺序 accept/reject"，其论文自承无法暴露"组合层面联合分配"难题。
  - **Mei et al. (2025, arXiv:2506.18183)：** 推理模型的"自信错误悖论"——推理越深越过度自信，提示陷阱项设计的现实必要性。
  - **Guo et al. (2017, ICML)：** ECE 与温度缩放，经典校准指标。
- **评估结果：** 调研 8 个 benchmark / 论文，最终选定 Kadavath、Guo、Steyvers&Peters、Mirror、TRIAGE 作为主借鉴与差异化对象，确保"站在肩膀上"且清楚自己补了什么缺口。

---

## 提案撰写 / Proposal Writing

- **初稿生成：**
  - Prompt：提供赛道（Track 2）+ 调研结论 + 核心想法（"构造已知-未知边界 + 自信错误陷阱 + 监控/控制双通道"），要求生成中英双语、含四部分、附示例项的提案初稿。
  - AI 生成了完整四部分结构的初稿。
- **迭代次数：** 3 轮
  - 第 1 轮：初稿对"为什么它测量元认知而非其他能力"论证偏弱。要求强化"置信度与正确率分开计分"与"K/U 切分控制难度变量"的逻辑链。
  - 第 2 轮：Phase 2 控制通道与 KSTAR 的"抑制"对应不够清晰。重新把 inhibition 明确映射为"低置信时 abstain"，并补上 ΔE_KU 与 ΔE 的对应。
  - 第 3 轮：补充了示例测试项（光速单位陷阱）、关键指标表、可行性时间表的具体天数。
- **人工修改（贾静文）：**
  - 调整了中英文双语的呈现方式（采用"段内紧邻双语"，中文叙述 + 英文对应，便于 Kaggle 国际评审）。
  - 修正了已知-未知样本中"虚构实体 vs 近因陷阱"的比例（定为各半，避免地板效应）。
  - 把"核心创新"一句话收敛为"从整体校准细化为边界过度自信 + 补上控制通道"，使创新点更聚焦。
  - 核实并统一了所有引用的 arXiv 编号与年份，剔除无法核实的表述。

---

## 手动步骤说明 / Manual Steps Justification

| 手动步骤 | 为什么没用 AI |
|----------|-------------|
| 赛道最终选定（Metacognition） | 需结合个人兴趣与课程 KSTAR 契合度做判断，AI 只提供五赛道对比，最终取舍由我定 |
| 难度梯度比例（50/50，U 内虚构实体与陷阱各半） | 基于对认知科学"地板/天花板效应"的经验判断，AI 缺乏对不同人群实测分布的把握 |
| 截止日期以 12/31 为准的判定 | 需对照 `challenge.json`（冻结定义）与指南 PDF 的冲突，做权威性判断而非直接采信 |
| K/U 示例项中的具体数值与单位陷阱设计 | 需保证科学事实准确（如光速 186,282 mi/s），AI 易在数字上出错，我做了人工核验 |

---

## AI 段位自评 / AI Usage Level

**🔵 进阶 —** 多轮对话迭代优化 prompt，审阅并修正 AI 输出，且设计了"调研 → 差异化定位 → 撰写 → 核验引用"的完整 workflow。核心设计思路（已知-未知边界 + 自信错误陷阱 + 双通道）由我提出并与 AI 协同打磨，不是纯 AI 生成，也不是纯人工。
