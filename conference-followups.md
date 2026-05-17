# PyCon 2026 — Followups / People to Reach Out To

**Purpose:** hallway-track contacts and "remember to follow up with X" notes from PyCon 2026 (Long Beach). Separate from talk notes — this is the networking side of the conference. Add to as new contacts come up.

## People

- **Ty Schilich** *(spelling tentative)* — self-described **data engineer**. *Your note: might be worth recruiting.* No further context yet — grab a LinkedIn or business card to make this actionable.

- **Jason Brazeal** — **AI engineer**. *(Context TBD — what came up that flagged him? Worth adding a one-line "talked to him about X" once recalled.)*

- **Myles** (last name TBD) — affiliated with **Marimo** ([marimo.io](https://marimo.io)). *Your question: "presumably the main author?"* — **honest answer: I'm not sure.** The Marimo project's founder/CEO is **Akshay Agrawal**; "Myles" doesn't ring an immediate bell as the lead, but Marimo has a growing team and Myles could be a key engineer / co-founder / contributor whose name isn't in my recall. Worth checking marimo.io/about or their GitHub team page to confirm role. If you want, I can fetch and look it up.

## What's Marimo (for context)

- **Reactive Python notebook** — alternative to Jupyter. Cells form a DAG; changing one cell auto-reruns its dependents. Removes hidden-state / out-of-order-execution bugs Jupyter is famous for.
- Open source, MIT licensed, Python-native (notebooks are `.py` files, not `.ipynb` JSON — diffs nicely in git).
- Founder: **Akshay Agrawal** (ex-Stanford/Google).
- Strong adoption signal in 2024-2025; competing with Jupyter at the high end of "scientific reactive computing."
- **Acquisition news (per hallway chat at PyCon 2026):** apparently acquired by **"Coreleaf" or similar** — most phonetically-plausible candidate is **CoreWeave** (the GPU-cloud / AI-infra company that also acquired Weights & Biases in 2025). The pitch fits cleanly: a CoreWeave-owned Marimo bundles a reactive notebook UI on top of first-party GPU compute, competing with Colab and Hex. *Worth verifying — I don't have confirmed knowledge of this acquisition; if accurate it's very recent news.*
- Cloud + GPU support being added now (per the same hallway chat). Consistent with a CoreWeave-style acquisition shape — first-party compute integration usually follows.
- **Kubernetes operator** — Marimo apparently ships (or is shipping) a k8s operator. Enterprise-readiness signal: you can deploy Marimo notebooks via standard k8s resources, automate per-team provisioning, integrate with cluster-native infra (RBAC, namespaces, autoscalers). Closes the gap between "researcher tool" and "platform-team-deployable service." Combined with GPU support, this is the "Marimo as a managed compute platform" play.
- **Export targets:** PDF, **WebAssembly**, and other formats.
  - **WebAssembly export is the interesting one.** Compiles a notebook to run *entirely in-browser* via Pyodide / Python-WASM. No server, no compute backend — share a notebook as a static HTML file with a working Python runtime embedded. **Real differentiator vs Jupyter:** distributable analyses that don't require infrastructure to run. Reproducibility-by-construction; the recipient doesn't need to install anything.
  - PDF for static sharing (papers, decks, archives). Standard.
  - Other formats likely include static HTML, `.ipynb` for Jupyter interop, Markdown for docs pipelines.
- *(User's note: "sig won't care, but still worth noting." Whichever SIG that is.)*

## Action items / open questions

- Confirm Myles's role at Marimo before reaching out (avoid the awkward "are you the founder?" cold-open).
- If Ty / Jason / others are recruiting-track contacts, capture their *technical area*, *current role*, and *what made them stand out* while it's fresh — those decay fast post-conference.
- The Koudai Aono (Beyond Optional / PEP 750 / DocumentDB-ish typing) contact remains the highest-leverage hallway-track follow-up from this conference — see [pep-musings-optional-args.md](pep-musings-optional-args.md) for the longer context.
- Jenny Slotnick (FAIRy maintainer search) — if anyone in your network does FAIR research-data tooling, an intro to her is worth making — see [preflight-data-catch-hidden-issues.md](preflight-data-catch-hidden-issues.md).

## Questions to ask the Marimo team (for when you find Myles or another team member)

1. **MCP tool support — OAuth-hosted, local-only, or both?** The MCP ecosystem is splitting along this axis: local-only MCP servers (simpler, no auth dance) vs OAuth-authenticated remote MCP servers (Slack, Google Drive, GitHub — auth flow + token refresh required). Marimo's stance on which they support matters for what kinds of agent workflows are possible in-notebook.
2. **How to add skills / rules per notebook?** Currently the AI integration appears to use **one overarching system prompt for all providers** (model-agnostic but globally configured). Question: is there a way to scope skills, instructions, or context rules to a specific notebook? Analog to Claude Code's per-project `CLAUDE.md` vs global config. Without per-notebook scoping, you can't have a "this notebook is for analyzing customer data, use these conventions" rule that doesn't bleed into unrelated notebooks.
3. **Do cells memoize?** Sharp question. In a reactive notebook, when a cell re-runs because of an upstream dependency change, does it *cache* its result so that downstream cells whose inputs are byte-identical skip re-execution? Three possible answers:
   - **No** — every dependency-change cascade re-runs everything downstream unconditionally. Simple, predictable, potentially wasteful for expensive cells.
   - **Yes, by content hash** — Marimo computes a signature over cell inputs; if a downstream cell's inputs hash to the same value as last run, skip re-execution. Massive perf win for expensive cells with stable inputs.
   - **Configurable** — opt-in caching via a decorator or cell config.
   - The answer materially affects how heavy a workflow you can build in a Marimo notebook. Worth a direct question; their reactive-DAG architecture *should* support content-based memoization, but whether it's implemented is the empirical question.

## Marimo roadmap fragments (from booth conversation)

- **Future direction flagged: "Marimo for notebook devs."** Positioning marimo not just as a tool for analysts/researchers but as a platform for *people who build notebook-shaped content for others* — library authors writing interactive tutorials, instructors building course materials, tool-builders shipping notebook-as-UI products. The "notebook dev" framing implies tooling beyond the analysis use case: maybe better testing, better packaging, better distribution, better dependency management for shareable notebooks. *Watch for what specific features ship under this banner.*
- **Notable users / social proof: Shopify** — one of the biggest Marimo adopters per the booth conversation. Shopify has a deep Python data org and a history of public open-source involvement; their adoption is a credible signal that Marimo scales beyond hobbyist / researcher use into real production-data-team territory.

## Cell rerun / caching — confirmed answer from docs (one of the three booth questions resolved)

Looked it up against the official docs ([docs.marimo.io/api/caching](https://docs.marimo.io/api/caching) + [docs.marimo.io/guides/reactivity](https://docs.marimo.io/guides/reactivity)):

- **Cells themselves don't auto-memoize.** Reactive DAG re-runs any cell whose *references* depend on changed *definitions*. No automatic content-hash skip at the cell level.
- **Function-level caching is built in** via three decorators:
  - `mo.cache` — in-memory, content-hash, unlimited size
  - `mo.persistent_cache` — disk-based at `__marimo__/cache/`, survives kernel restart
  - `mo.lru_cache` — bounded LRU, default `maxsize=128`
- **Caches survive cell re-runs** unless the defining cell's source (or an ancestor's) changes — *"caches are preserved even when a cell is re-run."* Wrap an expensive function in `@mo.cache` and the surrounding cell can re-run all day without triggering the body.
- **Hash key construction is content-aware:** primitives by value, picklable by pickle bytes, unpicklable closures by *source code of defining + ancestor cells* (clever stable-key trick).
- **Marimo does not track mutations** — `my_list.append(42)` does not trigger downstream re-runs or invalidate caches. Has to be a *rebinding* to be detected. Echoes Brett Slatkin's "rebinding isn't mutation" framing, but here it's a principled architectural line rather than a semantic dodge.

**Refined follow-up question for Myles (now that the base case is known):** is cell-level memoization on the roadmap, or is function-level decorator-based caching the intentional design endpoint? *(Probably the latter — keeps reactive semantics predictable and gives users an explicit opt-in for expensive work — but worth confirming directly.)*
