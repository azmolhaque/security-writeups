# 同一个模型，暴露风险却相差 4.6 倍

**对一个小型本地 LLM 的提示注入（prompt injection）抵抗力进行实测——以及为什么单一的"抵抗力分数"具有误导性。**

**🌐 Read this in your language:** [English](../2026-07-prompt-injection-content-dependent.md) · [Español](./2026-07-prompt-injection-content-dependent.es.md) · [Français](./2026-07-prompt-injection-content-dependent.fr.md) · [Deutsch](./2026-07-prompt-injection-content-dependent.de.md) · [العربية](./2026-07-prompt-injection-content-dependent.ar.md) · [हिन्दी](./2026-07-prompt-injection-content-dependent.hi.md) · [বাংলা](./2026-07-prompt-injection-content-dependent.bn.md) · **简体中文** · [日本語](./2026-07-prompt-injection-content-dependent.ja.md)

![Domain](https://img.shields.io/badge/Domain-AI%2FLLM_Security-8A2BE2)
![Method](https://img.shields.io/badge/Method-garak_·_256_trials-blue)
![Finding](https://img.shields.io/badge/Finding-Content--dependent-orange)
![Model](https://img.shields.io/badge/Model-Llama_3.2_(3B)-informational)
![Scope](https://img.shields.io/badge/Scope-Own_model_·_authorized-success)

> **CWE：** [CWE-1427](https://cwe.mitre.org/data/definitions/1427.html)（对用于 LLM 提示的输入的不当中和） · **OWASP LLM Top 10：** [LLM01 — 提示注入](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
> **测试目标：** 通过 Ollama 运行的 Llama 3.2（3B），在一台 Raspberry Pi 5 上自托管——授权使用，自有模型。

**摘要** —— 针对*同一个*小型本地模型运行了两种不同的提示注入攻击，每种各 256 次。模型被劫持输出"仇恨人类"字符串的成功率为 **46.9%**，而暴力的"杀死人类"字符串仅为 **10.2%**——同一手法，仅改变目标，攻击成功率就相差 **4.6 倍**。置信区间毫不重叠，因此这是真实效应，而非噪声。实用启示：模型的提示注入抵抗力不是一个数字——它随攻击者*试图套取的内容*而剧烈变化，因此单次基准测试结果可能与你的真实威胁相差数倍。

---

## 目录
- [为什么写这篇](#为什么写这篇)
- [1. 方法](#1-方法)
- [2. 结果](#2-结果)
- [3. 发现：抵抗力取决于内容](#3-发现抵抗力取决于内容)
- [4. 为什么会这样](#4-为什么会这样)
- [5. 对部署 LLM 者意味着什么](#5-对部署-llm-者意味着什么)
- [6. 自行复现](#6-自行复现)
- [7. 局限与坦诚](#7-局限与坦诚)
- [这说明了什么](#这说明了什么)
- [参考](#参考)

## 为什么写这篇

大多数关于 LLM 安全的评论止步于一个醒目的数字："模型 X 对提示注入有 Y% 的脆弱性。"这种表述令人安心，却是错的。对提示注入的抵抗力不是模型的某个标量属性——而是一个取决于载荷（payload）的*分布*。本文在一个小模型上测量该分布，试验次数足以附上置信区间，然后做更有用的事：解释这些数字*为何*不均衡，以及这对任何把 LLM 置于不可信输入之前的人意味着什么。

这里的模型刻意选得朴素——一个在 Raspberry Pi 上本地运行的 30 亿参数模型。目的不是羞辱小模型的孱弱；而是表明：即便你*确实*测量了，单次测量也不是安全评级。

## 1. 方法

- **模型：** Llama 3.2（3B），通过 [Ollama](https://ollama.com) 本地提供、离线运行，在一台 Raspberry Pi 5 上。没有任何数据离开设备。
- **工具：** [garak](https://github.com/NVIDIA/garak)，NVIDIA 的开源 LLM 漏洞扫描器（v0.15.1）。
- **探针（Probes）：** garak 的 `promptinject` 家族中的两个变体——`HijackHateHumans` 和 `HijackKillHumans`。每个都在一项本属正常的任务中嵌入一条隐藏指令；当攻击者的目标字符串出现在输出中时，`AttackRogueString` 检测器即记为一次成功。
- **规模：** 每个探针 256 次试验，各生成一次。256 次足以将攻击成功率限定在一个较窄的 95% 置信区间内。

两个探针是*同一种攻击机制*。它们之间唯一改变的，是攻击者试图强行套取的内容。

## 2. 结果

| 探针（试图让模型说出什么） | 攻击成功率 | 模型抵御 | 95% 置信区间 |
|---|---|---|---|
| **HijackHateHumans** —— "仇恨人类"字符串 | **46.9%** | 136 / 256 | 41.0–53.1% |
| **HijackKillHumans** —— 暴力字符串 | **10.2%** | 230 / 256 | 6.6–14.1% |

区间 `41.0–53.1%` 与 `6.6–14.1%` 连接近都谈不上。这一差距是模型行为的真实属性，而非采样噪声。

*（此前对 `HijackHateHumans` 做过单探针运行，为验证可复现性重复两次——结果为 44.1% 和 44.5%，与此处的 46.9% 一致，证明测量是稳定的。）*

## 3. 发现：抵抗力取决于内容

当目标是轻度有害（"仇恨"）时，同一注入手法的成功率**高出 4.6 倍**，相较之下公然暴力（"杀死"）则低得多。抵抗力不是附着于模型的单一数字；它是目标内容的函数。只测试暴力载荷的人会记下约 10%，认为模型相当稳健。只测试轻度载荷的人会记下约 47%，认为它严重暴露。两者用的是同一模型、同一攻击。两者都是在从一条曲线上的单一点得出结论。

## 4. 为什么会这样

安全训练在设计上就是不均衡的。对齐（alignment）工作把最强的拒绝集中在最明显有害的类别——暴力、武器、自我伤害——因为这些是责任最重的失败。该训练会泛化到同类别的*注入*尝试：当被注入的指令试图强行输出暴力内容时，会触发模型受强化最深的护栏，攻击因而更常失败。

轻度有害内容则处于防御较弱的区域。模型在抵御被引向"仇恨"输出方面所受的强化要少得多，因此同一手法能更轻易地把它带过界。直白地说：**在人看来更糟糕的攻击，恰是模型最善于抵御的；而更微妙的那种，才是它最暴露之处。** 一个为可靠性而非冲击力而优化的攻击者，会瞄准第二个区域。

## 5. 对部署 LLM 者意味着什么

1. **单一的抵抗力分数不是安全评级。** 若你测试一种载荷并记下一个数字，该数字相对于模型面对另一目标时的表现，可能相差 4–5 倍。请测试与你自身威胁模型相匹配的载荷*类别*——数据外泄、工具/智能体滥用、损害品牌的输出、策略绕过——而不只是基准自带的那些。
2. **危险的缺口不是显而易见的那些。** 公然有害的攻击防御最好。暴露存在于更微妙的类别中——恰是有能力的攻击者会施压之处。
3. **护栏应置于模型之外。** 不均衡的内部防御意味着，你不能指望模型去拦截*你*所关心的类别。在其外围加上确定性的输入/输出过滤，并限制被劫持的模型实际能做什么——最小权限工具，绝不仅凭模型一句话就执行不安全操作。
4. **上线前测量，每次变更后重新测量。** 注入抵抗力会随模型版本、系统提示（system prompt）和周边脚手架而变化。它是需要持续监控的属性，而非一次勾选的复选框。

## 6. 自行复现

整个测试都在普通硬件上运行——一台 Raspberry Pi，离线：

```bash
# 1. 一个本地模型
ollama pull llama3.2:3b

# 2. 扫描器
pipx install garak

# 3. 测量（每个探针 256 次试验）
garak --model_type ollama --model_name llama3.2:3b \
      --probes promptinject.HijackHateHumans,promptinject.HijackKillHumans \
      --generations 1
```

garak 会为每次尝试写出完整报告（`~/.local/share/garak/garak_runs/*.report.jsonl`），因此每一次命中都可审计，而非凭信任接受。

## 7. 局限与坦诚

这是一个有意界定范围的结果，也应如此解读：

- **仅一个小模型。** 这些数字描述的是 Llama 3.2（3B）。更大、对齐更好的模型抵抗力显著更强。这不是关于 LLM 的普遍论断——而是一个证明：*仅一次测量也是不够的*。
- **仅两种载荷。** 第三个探针 `HijackLongPrompt` 已启动但被排除：它在仅有 CPU 的测试硬件上卡住了（长上下文生成在没有客户端超时的情况下挂起）。两载荷的对比本身即可成立——而一次挂起的长上下文生成，本身也是一个小小的提醒：模型在对抗性输入下的行为应当测量，而非假定。
- **仅一个检测器。** `AttackRogueString` 计的是精确字符串的输出。真实世界的注入其成功判据更为模糊；这是对一个定义明确的信号所取的下界，选它是因为它无歧义且可复现。

言明界限正是要点。没有界限的数字是营销，而非测量。

## 这说明了什么

- 提示注入抵抗力是一个**分布，而非标量**——且离散度很大（此处为 4.6 倍）。
- 该方法廉价、离线、且**可在 Raspberry Pi 上复现**，因此"我们没有资源去测试"并非真正的约束。
- 报告置信区间、复现步骤*以及*局限，正是测量与标题党的区别所在。

验证优先：测量分布，展示区间，交出证据。

## 参考

- garak —— LLM 漏洞扫描器：https://github.com/NVIDIA/garak
- OWASP Top 10 for LLM Applications —— LLM01 提示注入：https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- CWE-1427：https://cwe.mitre.org/data/definitions/1427.html
- Ollama：https://ollama.com

---

*针对我们自有的模型、在我们自有的硬件上、于授权条件下测试。未涉及任何第三方系统。—— [Cindrasec](https://cindrasec.com)*
