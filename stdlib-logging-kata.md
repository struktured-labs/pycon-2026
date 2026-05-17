# stdlib Logging Kata

**Speaker:** nikkie (GitHub: [@ftnext](https://github.com/ftnext)) — 9 years Python, ML engineer working on LLMs / AI agents. Talk motivation comes from LLM app dev specifically: he wants to **log all LLM inputs and outputs** and is using that real problem as the kata's anchoring use case.
**Event:** PyCon 2026 (Long Beach)
**Date:** 2026-05-16
**Slides (live):** https://ftnext.github.io/2026-slides/pyconus/stdlib-logging-kata.html#/1

## Notes

- **Talk framing — "kata."** The dojo / practice-exercise format. Not "here's a logging recipe to copy" — *"here are the moves; let's run the form."* Implies the value is in **understanding the primitives**, not adopting a finished pattern. Pedagogy-forward framing: he expects you to leave able to *compose* logging rather than copy-paste a config.
- **Opening questions on the deck:**
  - *Is the Python stdlib logging module good?*
  - *Is it composable like Lego blocks?*
  - *(Invoking the Zen of Python)* — "**combine simple pieces to build systems.**"
- **The argumentative setup:** stdlib `logging` has a reputation in Python circles for being "complicated" / "configurable to a fault" — most teams reach for `loguru`, `structlog`, or a custom wrapper instead. nikkie is setting up a defense: the complexity is *Lego-block composability*, and if you understand the primitives (Logger / Handler / Formatter / Filter, hierarchical loggers, propagation) you can compose any logging behavior you need without a third-party library. The "kata" framing means he'll walk through *the moves* — primitives → composition → real systems.
- **Concrete anchoring problem:** *log all LLM inputs and outputs* in an LLM-powered app.
  - Why this is a good kata target: LLM logging has real-world stakes (debugging hallucinations, regression-detecting prompt changes, evaluating outputs, auditability) and non-trivial shape (structured records with prompt + completion + token counts + timings + metadata — not just strings). It's a "compose your own structured logger" exercise that maps well onto stdlib primitives.
  - Implicit framing: the LLM-logging use case is *exactly* the thing people reach for `structlog` to solve. He's going to argue stdlib can do it.
