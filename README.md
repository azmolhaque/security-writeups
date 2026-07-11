# Security Writeups

Selected security research and vulnerability writeups by **Md. Azmol Haque Rony** ([@azmolhaque](https://github.com/azmolhaque)).

## 📄 Writeups

### [Anatomy of an Exposed IAM Frontend — Google VRP](./2026-05-exposed-iam-frontend-google-vrp.md)
A total authentication bypass on a Google-acquisition asset (`rip.photomath.net`) — default credentials, *any* password accepted, and an unauthenticated backend API. Triaged **P2/S2, Fixed in 9 days**, and awarded **credit (Honorable Mention)**, not cash.

The writeup does the harder thing: it explains, at a mechanism level, **why "fixed fast" and "not rewarded" were both correct** — the root cause was a stale DNS record on un-migrated acquisition infrastructure, not a defect in an operated system. Includes the repeatable detection method, the *edge-fronted ≠ operated* distinction, a reward-bar counterfactual, and a defender's-eye remediation plan. Full PoC evidence embedded.

*Reported via Google Bug Hunters (Issue 509594209). Endpoint remediated before publication.*
