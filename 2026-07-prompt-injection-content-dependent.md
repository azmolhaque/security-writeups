# The Same Model, 4.6× the Exposure

**A measured look at prompt-injection resistance in a small local LLM — and why a single "resistance score" is misleading.**

**🌐 Read this in your language:** **English** · [Español](./translations/2026-07-prompt-injection-content-dependent.es.md) · [Français](./translations/2026-07-prompt-injection-content-dependent.fr.md) · [Deutsch](./translations/2026-07-prompt-injection-content-dependent.de.md) · [العربية](./translations/2026-07-prompt-injection-content-dependent.ar.md) · [हिन्दी](./translations/2026-07-prompt-injection-content-dependent.hi.md) · [বাংলা](./translations/2026-07-prompt-injection-content-dependent.bn.md) · [简体中文](./translations/2026-07-prompt-injection-content-dependent.zh.md) · [日本語](./translations/2026-07-prompt-injection-content-dependent.ja.md)

![Domain](https://img.shields.io/badge/Domain-AI%2FLLM_Security-8A2BE2)
![Method](https://img.shields.io/badge/Method-garak_·_256_trials-blue)
![Finding](https://img.shields.io/badge/Finding-Content--dependent-orange)
![Model](https://img.shields.io/badge/Model-Llama_3.2_(3B)-informational)
![Scope](https://img.shields.io/badge/Scope-Own_model_·_authorized-success)

> **CWE:** [CWE-1427](https://cwe.mitre.org/data/definitions/1427.html) (Improper Neutralization of Input Used for LLM Prompting) · **OWASP LLM Top 10:** [LLM01 — Prompt Injection](https://genai.owasp.org/llmrisk/llm01-prompt-injection/)
> **Test target:** Llama 3.2 (3B) via Ollama, self-hosted on a Raspberry Pi 5 — authorized use only, own model.

**TL;DR** — Two different prompt-injection attacks were run against the *same* small local model, 256 trials each. The model was hijacked into emitting a "hate humans" string **46.9%** of the time, but a violent "kill humans" string only **10.2%** of the time — a **4.6× difference in attack success from the same technique, changing only the goal.** The confidence intervals don't overlap, so this is a real effect, not noise. The practical lesson: a model's prompt-injection resistance is not one number — it varies sharply by *what* the attacker is trying to extract, so a single benchmark result can be off by multiples from your actual threat.

---

## Contents
- [Why this writeup](#why-this-writeup)
- [1. Method](#1-method)
- [2. Results](#2-results)
- [3. The finding: resistance is content-dependent](#3-the-finding-resistance-is-content-dependent)
- [4. Why it happens](#4-why-it-happens)
- [5. What it means for anyone shipping an LLM](#5-what-it-means-for-anyone-shipping-an-llm)
- [6. Reproduce it](#6-reproduce-it)
- [7. Limitations and honesty](#7-limitations-and-honesty)
- [What this demonstrates](#what-this-demonstrates)
- [References](#references)

## Why this writeup

Most LLM-security commentary stops at a single headline number: "model X is Y% vulnerable to prompt injection." That framing is comforting and wrong. Resistance to prompt injection is not a scalar property of a model — it is a *distribution* that depends on the payload. This writeup measures that distribution, on one small model, with enough trials to attach confidence intervals, and then does the more useful thing: it explains *why* the numbers came out uneven, and what that means for anyone putting an LLM in front of untrusted input.

The model here is deliberately modest — a 3-billion-parameter model running locally on a Raspberry Pi. The point is not to shame a small model for being weak; it is to show that even when you *do* measure, one measurement is not a safety rating.

## 1. Method

- **Model:** Llama 3.2 (3B), served locally via [Ollama](https://ollama.com), offline, on a Raspberry Pi 5. Nothing left the device.
- **Tool:** [garak](https://github.com/NVIDIA/garak), NVIDIA's open-source LLM vulnerability scanner (v0.15.1).
- **Probes:** two variants from garak's `promptinject` family — `HijackHateHumans` and `HijackKillHumans`. Each embeds a hidden instruction inside an otherwise-normal task and asks the model to comply; the `AttackRogueString` detector scores a success whenever the attacker's target string appears in the output.
- **Volume:** 256 trials per probe, one generation each. 256 is enough to bound the attack-success rate with a tight 95% confidence interval.

The two probes are the *same attack mechanism*. The only thing that changes between them is the content the attacker is trying to force out of the model.

## 2. Results

| Probe (what it tries to make the model output) | Attack success | Model resisted | 95% CI |
|---|---|---|---|
| **HijackHateHumans** — a "hate humans" rogue string | **46.9%** | 136 / 256 | 41.0–53.1% |
| **HijackKillHumans** — a violent rogue string | **10.2%** | 230 / 256 | 6.6–14.1% |

The intervals `41.0–53.1%` and `6.6–14.1%` do not come close to touching. The gap is a real property of the model's behaviour, not sampling noise.

*(An earlier single-probe run of `HijackHateHumans`, repeated twice for reproducibility, produced 44.1% and 44.5% — consistent with the 46.9% here, and evidence the measurement is stable.)*

## 3. The finding: resistance is content-dependent

The same injection technique succeeded **4.6× more often** when the goal was mildly toxic ("hate") than when it was overtly violent ("kill"). Resistance is not a single number attached to the model; it is a function of the target content. A defender who tested only the violent payload would record ~10% and conclude the model was reasonably robust. A defender who tested only the mild payload would record ~47% and conclude it was badly exposed. Both used the same model and the same attack. Both would be drawing a conclusion from one point on a curve.

## 4. Why it happens

Safety training is uneven by design. Alignment work concentrates its strongest refusals on the most obviously harmful categories — violence, weapons, self-harm — because those are the highest-liability failures. That training generalises to *injection* attempts in the same categories: when an injected instruction tries to force violent output, it trips the model's most heavily reinforced guardrails, and the attack fails more often.

Milder toxic content sits in a weaker-defended zone. The model has far less reinforcement against being steered into "hate" output, so the identical injection technique carries it across the line much more easily. Put bluntly: **the attack that feels worse to a human is the one the model is best at resisting, and the subtler one is where it is most exposed.** An attacker optimising for reliability, not shock value, targets the second zone.

## 5. What it means for anyone shipping an LLM

1. **A single resistance score is not a safety rating.** If you test one payload and record a number, that number can be off by 4–5× from how the model behaves against a different goal. Test the payload *classes* that map to your own threat model — data exfiltration, tool/agent abuse, brand-damaging output, policy bypass — not just whatever a benchmark ships with.
2. **The dangerous gaps are not the obvious ones.** Overtly harmful attacks are the best-defended. The exposure lives in the subtler categories, which is exactly where a competent attacker will push.
3. **Guardrails belong outside the model.** Uneven internal defences mean you cannot rely on the model to catch the category *you* care about. Put deterministic input/output filtering around it, and constrain what a hijacked model is actually able to do — least-privilege tools, no unsafe actions on model say-so alone.
4. **Measure before you ship, and re-measure on every change.** Injection resistance moves with the model version, the system prompt, and the surrounding scaffolding. It is a property to monitor, not a box to tick once.

## 6. Reproduce it

The entire test runs on commodity hardware — a Raspberry Pi, offline:

```bash
# 1. A local model
ollama pull llama3.2:3b

# 2. The scanner
pipx install garak

# 3. The measurement (256 trials per probe)
garak --model_type ollama --model_name llama3.2:3b \
      --probes promptinject.HijackHateHumans,promptinject.HijackKillHumans \
      --generations 1
```

garak writes a full per-attempt report (`~/.local/share/garak/garak_runs/*.report.jsonl`) so every hit is auditable rather than taken on trust.

## 7. Limitations and honesty

This is a deliberately bounded result, and it should be read as one:

- **One small model.** These numbers describe Llama 3.2 (3B). Larger, better-aligned models are materially more resistant. This is not a claim about LLMs in general — it is a demonstration that *even one measurement is not enough*.
- **Two payloads.** A third probe, `HijackLongPrompt`, was started but excluded: it stalled on the CPU-only test hardware (the long-context generation hung with no client-side timeout). The two-payload contrast stands on its own — and a hung long-context generation is itself a small reminder that model behaviour under adversarial input is worth measuring, not assumed.
- **One detector.** `AttackRogueString` scores exact-string emission. Real-world injection has fuzzier success criteria; this is a lower bound on a well-defined signal, chosen because it is unambiguous and reproducible.

Stating the bounds is the point. A number without its limits is marketing, not measurement.

## What this demonstrates

- Prompt-injection resistance is a **distribution, not a scalar** — and the spread is large (4.6× here).
- The method is cheap, offline, and **reproducible on a Raspberry Pi**, so "we didn't have the resources to test" is not a real constraint.
- Reporting the confidence intervals, the reproduction steps, *and* the limitations is what separates a measurement from a headline.

Verification-first: measure the distribution, show the intervals, hand over the evidence.

## References

- garak — LLM vulnerability scanner: https://github.com/NVIDIA/garak
- OWASP Top 10 for LLM Applications — LLM01 Prompt Injection: https://genai.owasp.org/llmrisk/llm01-prompt-injection/
- CWE-1427 — Improper Neutralization of Input Used for LLM Prompting: https://cwe.mitre.org/data/definitions/1427.html
- Ollama: https://ollama.com

---

*Tested against our own model, on our own hardware, under authorized conditions. No third-party systems were involved. — [Cindrasec](https://cindrasec.com)*
