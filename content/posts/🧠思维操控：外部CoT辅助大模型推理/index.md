---
title: "🧠思维操控：外部CoT辅助大模型推理"
date: 2025-09-02T07:22:16+08:00
categories:
  - 算法
  - 论文
tags:
  - 论文
  - 算法
  - CoT压缩
---

## 🧠 思维操控：外部 CoT 辅助大模型推理

Thought Manipulation: External Thought Can Be Efficient for Large  Reasoning Models

Training Free

## 摘要

近期在大型推理模型（LRMs）方面的进展证明了扩展测试时计算规模能够提升多个任务中的推理能力。然而，LRMs 通常会遭受“过度思考”问题，即模型生成大量冗余的推理步骤，而这些步骤仅带来有限的性能提升。现有的工作依赖微调来缓解过度思考的问题，但这需要额外的数据、非传统的训练设置、可能引发的安全性错位风险，以及较差的泛化能力。

通过实证分析，我们揭示了 LRM 行为的一个重要特性：将由较小模型生成的外部 CoTs（链式思维）放置在思考标记（<think> 和 </think>）之间，可以有效地引导模型生成更少的推理步骤。基于这一洞察，我们提出了一种简单而高效的流程，称为 ThoughtMani，以使 LRMs 能够跳过不必要的中间步骤，并显著降低计算成本。我们进行了广泛的实验以验证 ThoughtMani 的实用性和效率。例如，在 LiveBench/Code 数据集上应用于 QwQ32B 时，ThoughtMani 在几乎不增加 CoT 生成器开销的情况下，保持了原始性能并减少了约 30% 的输出 token 数量。此外，我们发现 ThoughtMani 平均提升了 10% 的安全性对齐。由于模型供应商通常同时提供不同尺寸的模型，ThoughtMani 为构建更高效且易于使用的 LRM 提供了一种有效途径，从而适用于现实世界的应用场景。

## Motivation

- 计算资源
- 基于微调的方法通常需要额外的数据收集，从而导致成本增加。此外，微调可能会引发安全性的对齐问题

## 实证

使用 LLM（ Qwen-Max、Qwen-Plus、Qwen2.5-7B-Instruct 和 Qwen-2.5-3B-Instruct）生成推理，然后使用<think>和</think>包裹其推理，并将用户的 Question+ 构造的 think 一同输入到 LRM（QwQ 和 Deepseek-Distillation-Qwen2.5-32b(14b)-instruct）中，用于生成最终答案。

> **生成推理的 Prompt：**
> “If you are a teacher, you are listing the important key points for solving the problem and no calculation details should be included. You are not allowed to produce any final answer. Add <STOP> when the key points are finished. You may provide **only very high-level ideas** for solving the problem, no calculation details should be included.”

![](/post_imgs/%F0%9F%A7%A0%E6%80%9D%E7%BB%B4%E6%93%8D%E6%8E%A7%EF%BC%9A%E5%A4%96%E9%83%A8CoT%E8%BE%85%E5%8A%A9%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%8E%A8%E7%90%86/IPa3bpCbMojwwHxQjvMc7fWXnEc.png)

**发现使用 RL 的 LRM 在很多情况下仍然会再生成自己的推理链，**但是如果外部 CoT 是质量较高的（比如 Qwen-Max）生成的，那么 LRM 再自己推理的概率会小很多。

但是基于蒸馏的 LRM 很少会再生成自己的推理链，作者推测基于蒸馏的 LRMs 可能并未真正“理解”推理或思考的概念。相反，它们的行为主要由监督微调期间学到的模式跟随技能驱动。

## 方法

1. 使用 LLM 生成 high level 的 CoT

> “If you are a teacher, you are listing the important key points for solving the problem and no calculation details should be included. You are not allowed to produce any final answer. Add <STOP> when the key points are finished. You may provide **only very high-level ideas** for solving the problem, no calculation details should be included. If you feel that you cannot solve it, output <STOP> and return.”

1. 如果 LLM 生成的 CoT 为空（Prompt 中写了如果模型觉得自己无法解决，那就直接输出 STOP），就抛弃掉 LLM 的 CoT 转而让 LRM 自行推理

![](/post_imgs/%F0%9F%A7%A0%E6%80%9D%E7%BB%B4%E6%93%8D%E6%8E%A7%EF%BC%9A%E5%A4%96%E9%83%A8CoT%E8%BE%85%E5%8A%A9%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%8E%A8%E7%90%86/YqJLbxoRqoN6rgxnbnscuTqpn1d.png)

## 实验

数据集

- 数学：AIME-2024、GSM-8k、MATH-500
- 编程：LiveBench
- 安全对齐：WildJailbreak

模型：

- CoT 模型：Qwen-Max, Qwen-Plus, Qwen-2.5-7B-Instruct, and Qwen-2.5-3B-Instruct.
- LRM：QwQ-32B（RL）、Deepseek-Distillation-Qwen-2.5-14B（32B）-Instruct（Distill）
- 模型均为 4-bit 量化版本

实验设置：

- 框架：vLLM
- 超参：temperature 0.7, top-k 0.95, maxlen 30000（AIME）、 20000 （Others）

Baselines：

- Empty Thought，将推理模型的 think 部分替换为空
- Truncation，在到达原 Reasoning 一半的长度时添加</think>截断推理
- Prompt Reduction，（Let’s quickly conclude the answer without showing step-by-step reasoning.）
- 其他 CoT 压缩：Tokenskip、CoT-Valve

![](/post_imgs/%F0%9F%A7%A0%E6%80%9D%E7%BB%B4%E6%93%8D%E6%8E%A7%EF%BC%9A%E5%A4%96%E9%83%A8CoT%E8%BE%85%E5%8A%A9%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%8E%A8%E7%90%86/ZYaJbpjljoVOydxL5b5coMHJnIf.png)

![](/post_imgs/%F0%9F%A7%A0%E6%80%9D%E7%BB%B4%E6%93%8D%E6%8E%A7%EF%BC%9A%E5%A4%96%E9%83%A8CoT%E8%BE%85%E5%8A%A9%E5%A4%A7%E6%A8%A1%E5%9E%8B%E6%8E%A8%E7%90%86/BtQJbW6N4ohf0HxRQpJcyQF0nag.png)

{{< lead >}}
感觉如果不做 Training free，填入 CoT 之后再基于填入的 CoT 进行 SFT 应该还蛮有潜力的
{{< /lead >}}
