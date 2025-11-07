---
title: "Saber：一种针对扩散语言模型的自适应加速与回溯增强的高效采样方法"
date: 2025-11-07T15:56:05+08:00
categories:
  - 算法
  - 论文
tags:
  - 论文
  - 算法
---

## Saber：一种针对扩散语言模型的自适应加速与回溯增强的高效采样方法

Saber: An Efficient Sampling with Adaptive Acceleration and Backtracking Enhanced Remasking for Diffusion Language Model

{{< lead >}}
这篇文章设计了 DLM 的 remask 机制，是对于已经解码出来的 token 的 remask
{{< /lead >}}

## 摘要

扩散语言模型（DLMs）正作为一种强大且有前景的替代方案，逐渐取代主流的自回归范式，在并行生成和双向上下文建模方面具有固有优势。然而，在代码生成任务中，由于更强的结构约束，DLMs 的性能受到推理速度与输出质量之间关键权衡的显著阻碍。我们观察到，通过减少采样步骤来加速代码生成过程通常会导致性能的灾难性下降。在本文中，我们引入了一种新的**无需训练**的采样算法——自适应加速与回溯增强重掩码（即 Saber），用于 DLMs 以在代码生成中实现更好的推理速度和输出质量。具体而言，Saber 受到 DLM 生成过程中的两个关键洞察启发：**1）随着更多代码上下文的建立，可以自适应地加速；2）需要一个回溯机制来逆转生成的标记**。在多个主流代码生成基准上的广泛实验表明，Saber 在主流 DLM 采样方法上平均提升了 1.9% 的 Pass@1 准确率，同时实现了平均 251.4% 的推理加速。通过利用 DLMs 的固有优势，我们的工作显著缩小了代码生成中与自回归模型之间的性能差距。

{{< lead >}}
这篇工作局限在代码生成，可能是 Motivation 里观察到的现象在代码生成中更加直观
也可能是相关工作中提到 ReMDM 与该方法类似，但是代码生成任务上并不理想
{{< /lead >}}

## Motivation

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/KwDebpIBYoen9Ux6IJpcmeNVnrh.png)

{{< lead >}}
根据图 a 大概整体 confidence 是随着降噪进行越来越大的
图 b 没说是具体是哪个 token 的置信度，但是看起来可能是红色的 sorted
{{< /lead >}}

图 a 表明，DLM 生成过程难度逐渐降低。

在最开始时，由于大量 token 是[MASK]，DLM 的不确定性很高，但随着逐渐解码，DLM 的不确定性逐渐降低，**可以在这里使用一个自适应的加速策略**，在采样初期保持谨慎，在后面逐渐激进

图 b 表明，DLM 生成具有动态上下文

DLM 具有动态的上下文，随着越来越多的[MASK]被解码，DLM 对于已经生成的 token 的置信度会发生变化，DLM 可能会越来越觉得之前生成的某个 token 是错误，**但是传统的 DLM 在采样中是不可逆的，无法将一个 unmask 出来的 token 重新 mask**

## 方法

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/OwmZb1vI9obWccxLQUVc3RRin3e.png)

### 动态 unmask 自适应加速

动态设置置信度阈值 tau_t，将其设置为之前步骤中 unmask 的 token 的平均置信度，c_max 是一个超参

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/CnzoblWTsooKvsx3sNpcWwqpn2e.png)

如果某些 token 的置信度超过 tau_t，就并行解码出来，作为 Draft，记作 D_t

### Remask

先基于上一步 decode 的激进程度，判断这一步需要 remask 的 token 数量，mu 是一个超参

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/AMiRbM21WolqeAxksSKcWhtYnXd.png)

然后计算**先前已有的 token 的置信度变化（不包括刚刚 decode 出来的 token）**：

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/TmQ3bO64Comwo7xvzXSc5udhnrc.png)

选出 top-mu_t 的 token，将其重新回退为[MASK]

## 实验

A6000（48G）、LLaDA-8B-Instruct，max_new_tokens 设置为 256、block_size 设置为 128

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/JilbblbzZoOtypxpPXWcPnI5nZc.png)

在其他模型上面的效果（HumanEval）

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/YO14bzrwmoDXvfx5EwqciBLSnJh.png)

### 消融

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/AcTabfvakofykYx881PcLL0mnrc.png)

## 一些其他想法

在另一篇文章中看到以下内容：

![](/post_imgs/Saber%EF%BC%9A%E4%B8%80%E7%A7%8D%E9%92%88%E5%AF%B9%E6%89%A9%E6%95%A3%E8%AF%AD%E8%A8%80%E6%A8%A1%E5%9E%8B%E7%9A%84%E8%87%AA%E9%80%82%E5%BA%94%E5%8A%A0%E9%80%9F%E4%B8%8E%E5%9B%9E%E6%BA%AF%E5%A2%9E%E5%BC%BA%E7%9A%84%E9%AB%98%E6%95%88%E9%87%87%E6%A0%B7%E6%96%B9%E6%B3%95/OrWfb7JDro2YBGx2yQJcPYIBnXb.png)

DLM 相比同参数的 AR，可以说很慢了
