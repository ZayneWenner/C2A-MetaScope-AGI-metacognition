# 拿来说明 / Attribution

**作者 / Author:** 贾静文（Jia Jingwen）
**挑战 / Challenge:** C2A — Track 2 Metacognition（元认知）
**日期 / Date:** 2026-10-07

---

## 借鉴来源 / References

| 来源 / Source | 链接 / Link | 核心思想 / Key Idea |
|---------------|-------------|---------------------|
| DeepMind 认知框架 (2026) | [PDF](https://storage.googleapis.com/deepmind-media/DeepMind.com/Blog/measuring-progress-toward-agi/measuring-progress-toward-agi-a-cognitive-framework.pdf) | 10 个认知能力分类法 + 三阶段评估协议；元认知被定义为"幻觉的根源" |
| Kadavath et al. (2022) | [arXiv:2207.05221](https://arxiv.org/abs/2207.05221) | 口语化置信度（P(True)/P(IK)）与校准；奠基性证明 LLM 具备一定自我知识 |
| Guo et al. (2017, ICML) | [arXiv:1706.04599](https://arxiv.org/abs/1706.04599) | 期望校准误差（ECE）与温度缩放，经典校准度量 |
| Steyvers & Peters (2025) | [Lacuna 综述](https://lacuna.tiptreesystems.com/work/metacognition-and-uncertainty-communication-in-humans-and-large-language-models/) | 区分"元认知敏感度（AUC）"与"校准（ECE）"两个维度 |
| Mirror (Wang, 2026) | [arXiv:2604.19809](https://arxiv.org/abs/2604.19809) | 多层元认知校准基准；发现组合校准误差与"知-行鸿沟" |
| TRIAGE (2026) | [arXiv:2605.13414](https://arxiv.org/abs/2605.13414) | 预算约束下的前瞻元认知控制；顺序 accept/reject 设计 |
| Mei et al. (2025) | [arXiv:2506.18183](https://arxiv.org/abs/2506.18183) | 推理模型的"自信错误悖论"：推理越深越过度自信 |
| Nelson & Narens (1990) | Metamemory: A Theoretical Framework | 元认知"监控 / 控制"二分法的理论基础 |
| KSTAR 课程框架 | 课程资料 | ΔR（认知偏差）作为学习/元认知信号；抑制（inhibition）机制 |

---

## 拿来的部分 / Adopted

### 从 Kadavath et al. (2022) 拿了什么
- **拿了：** "口语化置信度 + P(True) 自评"的范式——让模型先作答再报告置信度。
- **为什么好用：** 这是目前最直接、可复现的元认知测量接口，无需额外训练。

### 从 Guo et al. (2017) 拿了什么
- **拿了：** ECE（期望校准误差）作为校准度量，按置信度分箱比较预测频率与真实频率。
- **为什么好用：** 标准、可解释、确定性计算，适合作为基准主指标之一。

### 从 Steyvers & Peters (2025) 拿了什么
- **拿了：** 把"敏感度（AUC）"与"校准（ECE）"拆成两个独立维度分别报告。
- **为什么好用：** 避免把"能排序对错"和"置信度准"混为一谈，让提案指标更严谨。

### 从 DeepMind 框架 + KSTAR 拿了什么
- **拿了：** 三阶段评估协议（基线 → 任务 → 人类对比）的理念，改造为我们的 Phase 1/2 + 人类基线；将 KSTAR 的 inhibition 与 ΔE 映射为"低置信时 abstain"与 ΔE_KU。
- **为什么好用：** 有人类基线参照，才能有意义地解读 AI 的 ΔE_KU 得分。

---

## 去掉的部分 / Removed

| 来源 | 他们做了什么 | 我不要 | 原因 |
|------|------------|--------|------|
| Kadavath (2022) | 报告单一聚合 ECE | 去掉单一 ECE | 聚合 ECE 混淆已知/未知，掩盖了"边界过度自信"这一真正的元认知失败模式 |
| Mirror (2026) | 跨域组合校准误差（CCE） | 去掉跨域组合视角 | 我想聚焦更基础的"已知-未知边界"，并用对抗陷阱主动诱发过度自信，而非被动观测跨域失败 |
| TRIAGE (2026) | 顺序 accept/reject 的预算控制 | 去掉顺序决策 | TRIAGE 自身承认无法暴露"组合层面联合分配"，我用组合级联合选择补上这一缺口 |
| Mei et al. (2025) | 推理模型校准（IUQ 后处理） | 去掉推理专属设定 | 我们测通用元认知，不绑定特定推理范式；但其"自信错误"发现指导了陷阱项设计 |

---

## 我的改进 / My Improvements

### 改进 1：已知-未知边界 + 自信错误陷阱（核心创新）

- **现有方法的问题：** Kadavath / MMLU 的校准用单一聚合 ECE，无法区分"在已知项上准"还是"在未知项上也准"；Mirror 关注跨域组合，却不主动构造已知-未知边界。
- **我的改进：** 程序化生成一条受控的 knownness 边界（K/U 标签），并专门埋设"近因陷阱"项（人物互换、单位错误、off-by-one）主动诱发自信错误。
- **效果：**
  - 提出主信号 **ΔE_KU = |ECE_U − ECE_K|**——直接量化"跨边界的过度自信"，比聚合 ECE 更能定位幻觉根源。
  - 陷阱过度自信率独立成指标，捕捉最危险的"答错却 c>0.8"模式。

### 改进 2：监控 + 控制双通道

- **现有方法的问题：** 多数基准只测"口语化置信度"（监控），不测"是否据此行动"（控制）。
- **我的改进：** 增加 Phase 2——在共享作答预算下做组合层面的"尝试 / abstain"联合选择（而非 TRIAGE 的顺序决策）。
- **效果：** 把 Nelson & Narens 的监控/控制二分落到可测行为，并对应 KSTAR 的抑制机制；暴露了 TRIAGE 无法测的组合分配能力。

### 改进 3：与 KSTAR 的可执行对齐

- **现有方法的问题：** 评估指标常缺乏认知科学理论支撑，或与课程框架脱节。
- **我的改进：** 将 KSTAR 的 ΔE（认知偏差）显式映射为 ΔE_KU；将 inhibition 映射为"低置信 → abstain"的控制动作。
- **效果：** 每个指标都有明确的认知科学解释（DeepMind / KSTAR / Nelson & Narens），而非纯统计数字，呼应课程"AI+X"的对口要求。
