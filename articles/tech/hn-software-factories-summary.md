---
tags:
  - result-machine-learning
  - software-factories
  - code-review
  - model-improvement
  - verification/result
---

# HN Discussion — "Why Software Factories Fail (or: harness engineering is not enough)"

Summary of the Hacker News comment thread on Dex Horthy's article, which argues that heavily automated AI coding pipelines ("software factories") fail because tooling/harness engineering alone isn't enough.

- **Thread:** https://news.ycombinator.com/item?id=49023019
- **Article:** github.com/humanlayer/advanced-context-engineering-for-coding-agents (wsff.md)
- **State when summarised:** small, fresh thread (~15 comments, ~1 hour old, 35 points)

## Main points being discussed

| Point being discussed | The argument | Who |
|---|---|---|
| **Model improvement is under-credited** | The author's failed "lights-off" experiment was July 2025, but models took a step-change in usefulness around late 2025/early 2026 — so pre-that experience is stale, and the piece waves away the improvement in a way that doesn't match commenters' own results | fishtoaster |
| **Route around the weakness** | Since agents are good at coding but bad at maintainability, build a modular/domain-specific architecture *first*, then unleash agents on the isolated modules where maintainability matters less | 2001zhaozhao |
| **PR review UX is the real bottleneck** | Code review tooling (esp. GitHub's) is painful; a small model grouping changes by theme, adding commentary, and sorting by importance (as Linear now does) cuts reviewer load for free. Sub-debate: just review/merge from the agent window instead — and isn't "grouping changes by theme" just what commits should already be? | rglynn; 2001zhaozhao; xorcist |
| **It's a training/RL problem** | The model's limits trace back to how labs RL them; debate quality would improve if people understood training. One dissent: training big models on noisy data then using RL to walk it back is a wasteful use of compute | vanuatu; vkaku |
| **Too early for "standards"** | These aren't "factories," they're stitched-together Rube Goldberg machines; it's premature to codify methodologies — teams should experiment against their own quality metrics rather than parrot standards | AIorNot |
| **Rigor, not skill, is the differentiator** | Success with LLMs is an effort/discipline issue — teams already doing high-hygiene engineering win. "Code is data," so bad code yields bad output; spec-driven dev without deep/formal verification falls short because prompts can't steer to the needed precision | _doctor_love; stellar_jay (offers a constraint taxonomy: generative/interpretive/elicitative) |
| **What the StrongDM "dark factory" outcome actually means** | A consulting spin-off (diffusion.io) suggests it went well — countered by: consulting arms usually appear when the *product* can't stand alone (cf. Palantir, Salesforce). Added context: StrongDM was sold ~a year after the dark-factory announcement, and the ex-CTO left to continue the idea | _doctor_love; edot; sythe2o0 (ex-employee) |
| **Author credibility / tone** | Dismissals that the piece is self-promotion or a rehash of LeCun's argument — one flagged, with a moderator note about shallow dismissals and personal attacks | syndacks; M4R5H4LL (flagged); dang (mod) |

## The two most substantive threads

**PR review UX** (rglynn) — the most concrete contribution: code review tooling is the pain point, and lightweight model-assisted triage of diffs already beats existing tools.

**Rigor / verification** (_doctor_love) — largely *agrees* with the article's thesis but reframes it: the problem isn't the harness, it's that teams without pre-existing engineering discipline can't get good results regardless of tooling. Prompts can't steer to the required precision without deep/programmatic verification.

The remainder is either skeptical of the framing (too early to standardise, clickbait title) or arguing the author under-weights how much the models themselves have improved.
