# The bug Google fixed in 9 days — and paid me $0 for

*A short, plain-language story about a security finding, and why the most valuable thing I took from it wasn't a payout.*

---

Every so often you find something on the internet that makes you sit up straight.

I was looking through subdomains belonging to companies Google had acquired — the quiet corners of a big company, where infrastructure from the acquired business sometimes lingers for a while before it's fully absorbed. On one of them, `rip.photomath.net`, a login page loaded. Not a product. An **admin panel** — the kind of screen that manages users and their permissions.

I tried the most obvious thing in the world: `admin` / `admin`. It let me in.

Then I did something less obvious. I logged out and tried again with a **completely random password**. It let me in *again*. The login wasn't checking anything — it was a door with a painted-on lock. And when I looked at the API behind it, that answered too, with no login at all.

On the surface, this looks like a five-alarm fire: an unauthenticated admin interface, on a Google-owned domain. My first instinct was to write it up as **critical**.

## The part where I slowed down

Here's the thing I've learned to do before hitting "send": ask *what an attacker could actually do with this*, not what it looks like they could do.

So I looked closer. The panel was empty — no real users, no real data. The interface had French placeholder text and a literal "it works!" debug string left in. Everything pointed to a **forgotten test build** from the acquired company that had never been switched off. It was reachable through Google's network, yes — but "reachable through Google" is not the same as "run by Google." That distinction turned out to be the whole story.

I reported it. The next day, Google accepted it with a friendly "🎉 Nice catch!" and filed it to the product team. Nine days later, they fixed it — not by patching an app, but by simply **removing the stale DNS record** so the host stopped existing. Clean, fast, done.

And the reward? **Nothing.** Credit and an honorable mention — no cash.

## Why "no reward" was the right call

It would be easy to feel shortchanged. I didn't — because once I understood the mechanism, their reasoning was airtight.

A bug bounty pays for **real risk to systems a company actually operates**. What I'd found was a hygiene problem: a dangling record pointing at an empty test box that never held anyone's data. It absolutely deserved to be cleaned up (and it was, quickly). But it wasn't a break-in to something Google runs. Two different questions — "should we fix this?" and "did this expose real risk?" — with two different, and equally correct, answers.

I actually appealed at first, leaning on a *what if an attacker phished someone with it* argument. The panel reconsidered and politely held their ground. They were right, and I was reaching. Recognizing that — before digging in — is the skill the second decision was quietly testing.

## The takeaway I actually keep

The lesson here isn't "I found a bug." It's this:

> **In security, the professional move is stating accurately how much something matters — especially when the honest answer is "less than it first looked."**

Anyone can find something alarming and shout about it. Calibrated judgment — knowing the difference between a scary-looking screenshot and a real, exploitable risk — is what actually makes a security person worth trusting. It's what a triage team does all day. It's what keeps a report honest and an appeal disciplined.

That's the muscle I trained on this one. No payout required.

---

**Want the full breakdown?** I wrote a detailed, evidence-backed technical writeup — the repeatable method for finding this class of forgotten infrastructure, the exact proof, the "edge-fronted ≠ operated" root cause, and a defender's plan to catch it early:

- 📄 **Full writeup:** [Anatomy of an Exposed IAM Frontend — Google VRP](./2026-05-exposed-iam-frontend-google-vrp.md) *(also available in 8 languages)*
- 🔎 **Interactive case study:** the same story as a designed, shareable page

*Reported via Google Bug Hunters (Issue 509594209). The endpoint was remediated before publication. No data was accessed or modified beyond what was needed to confirm the exposure.*

— **Md. Azmol Haque Rony** · [@azmolhaque](https://github.com/azmolhaque) · [LinkedIn](https://www.linkedin.com/in/md-azmol-haque-rony)
