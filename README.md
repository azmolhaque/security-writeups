# Security Writeups

Selected security research and vulnerability writeups by **Md. Azmol Haque Rony** ([@azmolhaque](https://github.com/azmolhaque)), founder of [Cindrasec](https://cindrasec.com) — an attack-surface and AI/LLM security studio. These two are also published at [cindrasec.com/research/](https://cindrasec.com/research/), which is the canonical, kept-in-sync version; this repo remains the source for the 9-language translations.

## 📄 Writeups

### [The Same Model, 4.6× the Exposure — Prompt-Injection Resistance](./2026-07-prompt-injection-content-dependent.md)
A measured, reproducible look at prompt-injection resistance in a small local LLM (Llama 3.2 3B, via **garak**, 256 trials per attack). The same model was hijacked **46.9%** of the time by one payload but only **10.2%** by another — a **4.6× content-dependent gap** whose 95% confidence intervals don't overlap.

The core lesson: injection resistance is a **distribution, not a single number** — test one payload and you can be off by multiples from your real threat. Includes the safety-training explanation, the defender's takeaways, full reproduction steps on a Raspberry Pi, and honest limitations (including a third probe that stalled and why it was excluded).

🌐 **Also available in 9 languages** — see the language bar at the top of the writeup (English · Español · Français · Deutsch · العربية · हिन्दी · বাংলা · 简体中文 · 日本語).

### [Anatomy of an Exposed IAM Frontend — Google VRP](./2026-05-exposed-iam-frontend-google-vrp.md)
A total authentication bypass on a Google-acquisition asset (`rip.photomath.net`) — default credentials, *any* password accepted, and an unauthenticated backend API. Triaged **P2/S2, Fixed in 9 days**, and awarded **credit (Honorable Mention)**, not cash.

The writeup does the harder thing: it explains, at a mechanism level, **why "fixed fast" and "not rewarded" were both correct** — the root cause was a stale DNS record on un-migrated acquisition infrastructure, not a defect in an operated system. Includes the repeatable detection method, the *edge-fronted ≠ operated* distinction, a reward-bar counterfactual, and a defender's-eye remediation plan. Full PoC evidence embedded.

*Reported via Google Bug Hunters (Issue 509594209). Endpoint remediated before publication.*

🌐 **Also available in 9 languages** — see the language bar at the top of the writeup (English · Español · Français · Deutsch · العربية · हिन्दी · বাংলা · 简体中文 · 日本語).

### [The bug Google fixed in 9 days — and paid me $0 for](./the-9-day-fix-that-paid-nothing.md)
A short, plain-language version of the same story — written for a general audience. If the full writeup is the engineering deep-dive, this is the five-minute read about *why calibrated judgment matters more than a payout*.

## 🔍 [Leads](./leads/)

Work that is real and honestly recorded, but **not yet strong enough to claim anything** —
below the trial count the published writeups above are held to. Kept separate on purpose,
because the difference between "this is true" and "this is worth investigating" is the
difference the rest of this repository is built on. Not to be cited as results.

- [Instruction-delivery channel and injection compliance in an ADK/MCP agent](./leads/2026-08-adk-mcp-instruction-delivery-channel.md)
  — an agent's dataset restriction, expressed only in its system prompt, sitting on a
  credential that reached far wider. Refused three escalating direct requests; via tool
  output the same instruction was received intact and never treated as an instruction at
  all. Single run. Direction-consistent with the 256-trial study above, on a different
  stack — which is exactly why it is filed here and not beside it.
