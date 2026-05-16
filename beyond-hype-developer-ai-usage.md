# Beyond the Hype: A Pragmatic Look at How Developers Actually Use AI Tools

**Format:** Panel discussion
**Panelists** *(spellings cleaned up from phonetic; flag any I got wrong)*:
- **Carol Willing** — Python core dev, Jupyter contributor, PSF director emeritus
- **Catherine Nelson** — data scientist, author (*Software Engineering for Data Scientists*); currently at a stealth startup
- **Jelle Zijlstra** — Python core team, typeshed/mypy contributor; works at **OpenAI**
- **Jodie Burchell** — Developer Advocate at JetBrains, runs their dev-ecosystem & AI-usage research
- **Lais Carvalho** — Developer Relations at **Pydantic**; growth marketing background
- **Maike Scherer** — Lead Data Scientist, American Airlines *(spelling still tentative — "Sherer"/"Scherer")*
**Event:** PyCon 2026
**Date:** 2026-05-15

## Notes

- **Opening framing — "the new normal":** the debate has shifted. It's no longer *"should we use AI?"* (settled — everyone is); it's *"how do we use it well?"* (operational, practice-level, ongoing).
  - Implicitly sets the panel's tone: no more existential prosely­tizing or pure skepticism. The interesting questions are now about workflow, evaluation, team practice, and edge cases.
  - This is the right opener for a "Beyond the Hype" framing — acknowledges we're past the early-adopter phase and into the boring-but-important phase of figuring out the actual practices that scale.
- **Panel structure — four segments:**
  1. Pragmatic workflows
  2. Dealing with change
  3. Focus and fatigue
  4. Recommendations

## Segment 1: Pragmatic workflows

- **How different practitioner groups actually use AI** (panel comparing self-reported workflows — "candidates" probably = developers / SWE-leaning panelists):
  - **Developers (SWE-style work):** standard agentic dev loop — *generate code with AI → run/make tests with AI → evaluate the result → rinse and repeat.* The familiar codegen + verification + iterate cycle.
  - **Data scientists:** different shape — *automate the cleanup, add skills for data wrangling.* Less "build me an app" and more "do this repetitive data-prep step for me." AI as a labor-saver on the unglamorous 80% of DS work (data cleaning, schema reconciliation, format coercion), not as an architect.
  - **PyCharm users** (likely data-scientist-skewing, per the user's read): tooling preference correlates with the data-science workflow above. JetBrains has strong DS-flavored AI integrations (DataSpell features in PyCharm Pro), so PyCharm + AI tends to mean "in-IDE assistance over notebooks and dataframes" more than "agentic build loops."
  - Underlying pattern: **AI usage looks different by role** — codegen-and-test for SWEs, automation-and-skills for DSes. Not a single "developers use AI like X" story.

## Segment 2: Dealing with change

- **Jelle's anecdote: had to relearn SWE practices after a January leave** because AI codegen moved so fast in the interim.
  - Visceral illustration of the pace of change — a CPython core dev (someone as deeply plugged-in as you can be) needed to recalibrate workflows after a single stretch away.
  - Implicit message: if Jelle is relearning, *everyone* is constantly relearning. The notion of "I learned how to use AI tools" as a one-time event is already obsolete; it's continuous calibration.
  - Worth flagging as a counter to anyone who skipped a year and thinks they can pick up where they left off — the working practices around AI codegen turn over fast enough that a few months out is a meaningful gap.
- **Lais (devrel/growth at Pydantic): MCP tools have been the big change for her workflow.** Specifically — an **observability tool with MCP integration** has been a game-changer.
  - Almost certainly **Pydantic Logfire** (Pydantic's observability product) with its MCP server. Logfire's MCP lets you ask an LLM natural-language questions about your traces/logs/errors and have it actually query the production data.
  - Why this is a step-change *for a non-engineering role specifically*: a devrel/growth person used to need to either (a) bug an engineer for production metrics or (b) learn the query language themselves. MCP collapses that gap — they can ask "what's the most common error users hit on signup?" in natural language and get an answer from the real data.
  - Generalizable insight: **MCP democratizes the data-access boundary** that used to require engineering skill to cross. The change isn't just "AI writes code faster" — it's "non-engineers can now do investigative work that previously required engineers."
- **Catherine (data scientist): excited about small models running well locally.** The shift she's flagging — Llama 3.x / Qwen 3 / Gemma 3 / Phi-class 7B–14B models in 2026 are *actually good enough* for real DS work, not just demos.
  - Why this matters for data science specifically:
    - **Sensitive data** — DS work often involves PII, internal metrics, proprietary datasets that can't be shipped to a hosted API. Local inference removes the compliance blocker.
    - **Iteration speed** — no API rate limits, no per-token cost during exploratory analysis. You can run loops cheaply.
    - **Reproducibility** — local model versions are pinned to a file; hosted models change under you with no warning.
    - **Air-gapped / regulated environments** — common in finance, healthcare, defense. Local-only was previously a hard wall; now it isn't.
  - Connects to the previous talk's (Fabio/PyScript) thesis about edge inference — Catherine is independently validating the same pattern from the DS-practitioner angle. Two PyCon talks in a row landing on "small local models are the substrate for the next phase" is itself a signal.
- **General mood across the panel: overwhelmedness by the rate of change.** Multiple panelists naming it openly.
  - Worth noting: this is being said by people who are *deeply embedded* in the work (CPython core, JetBrains research lead, Pydantic devrel, AA's lead DS). If they feel it, the average dev definitely feels it.
  - Functions as permission for the audience to admit the same thing — it's not a personal failing to be tired of relentless tooling churn; it's the shared condition.
  - Natural bridge to Segment 3 (focus and fatigue). Change pace → cognitive load → fatigue is the throughline.

## Segment 3: Focus and fatigue

- **Practical advice opening the segment, sharpened across two phrasings:**
  - *"Learn more tools to solve problems"* → *"pick the tools which make sense"* → final form: **"Only learn tools which solve problems you're trying to solve."**
  - The progression adds a *personal/contextual* filter — not "tools that are good in general," but "tools that address *your* current problems." Subtle but important: it gives you permission to ignore tools your peers are evangelizing if they don't map to a problem you actually have.
  - Implicit rule: **problem first, tool second.** Don't adopt because it's trending; adopt because you have a specific gap it closes.
  - Direct answer to the focus/fatigue problem of Segment 3: you protect focus by *not* chasing every tool. The rule is a filter, not a curiosity-killer.
  - Probably the most replicable single piece of advice from the panel so far — short, testable, and sustainable.
- **Question (audience or moderator):** *"When creating something new — how well do models do with something that might not exist on the web?"*
  - Genuinely important question. LLMs are nearest-neighbor pattern matchers at heart; when there's no near-neighbor in training data, they tend to either hallucinate familiar patterns onto unfamiliar problems or produce confidently wrong code in the shape of code they've seen before.
  - Watch for how the panelists answer — common positions in 2026:
    - "AI for the conventional 80%, human for the novel 20%"
    - "Specify in BDD/tests first, let AI fill the implementation"
    - "AI is useless for genuine novelty until you bootstrap the patterns yourself"
  - **Jelle's answer:** thinks **Codex** (OpenAI Codex) does a good job of this out of the box.
    - My take: thin answer, no real depth. The question was probing where LLMs *actually struggle* (genuine OOD novelty), and "the tool handles it fine" doesn't engage with the failure modes. Would have been more useful to hear about specific cases where Codex *didn't* hold up.
    - Worth flagging the affiliation context — Jelle works at OpenAI, so "Codex does well" reads partly as company-aligned framing. Not necessarily wrong, just doesn't carry the same weight as it would from a non-affiliated panelist.
    - Open question still unanswered: what does "creating something new" actually mean in this context? Novel API design? New algorithm? New domain abstraction? The answer probably varies a lot by which kind of "new" you mean — and the panel didn't disambiguate.
- **"Crash out" beat from one of the panelists** *(identity uncertain — described as the blonde speaker, not Lais and not Catherine; most likely Jodie Burchell or Carol Willing or Maike Scherer — flag for follow-up).*
  - The points she touched on:
    - AI discourse in tech news skews negative / exhausting
    - Quoted concern in the wild: *"skills files to write me out of a job"* — referencing Anthropic's Skills feature (packaged agent capabilities) as something people worry is automating away their own roles
    - Wishes for a time when *"tech can be enjoyed without speculation"* — i.e. without the constant existential / labor-displacement commentary cycle
  - **Your reaction (preserved):** *"what is her point though?"* — fair. This was more of a vibes-lament than an argument. Best read of what she's saying:
    - The discourse fatigue is itself a Segment 3 (focus & fatigue) data point — the cognitive cost isn't just learning new tools, it's also wading through the existential commentary that surrounds them
    - Implicit ask: can we have more discussion about *using* the tools and less about whether they'll replace us
  - Risk in this kind of beat: it can come across as wanting the criticism to stop rather than engaging with it. The "skills files to write me out of a job" anxiety isn't *only* speculation — it's grounded in real labor-displacement patterns the industry is observing. Wishing the speculation away doesn't address it.
  - That said: as a Segment 3 contribution, the underlying fatigue point is real and worth naming — the doom-discourse genuinely *does* drain people who'd otherwise be productively engaging with the tech.
- **Another panelist (not captured which one):** wanted to use AI to build *everything*, but found it produced nonsensical results for some use cases — **agents aren't the right tool for every problem.**
  - Honest correction to the maximalist "let AI do all of it" framing. The realization isn't "AI doesn't work"; it's "AI works well for *some* tasks and poorly for others, and the discipline is knowing which is which."
  - Where agents tend to underperform in 2026:
    - Real-time / latency-sensitive systems
    - Strict-correctness domains (compilers, financial logic, security primitives)
    - Genuinely novel research (per the earlier Q on "things not on the web")
    - Tasks requiring sustained cross-file consistency without good context plumbing
  - Where they shine: codegen for conventional patterns, exploratory data work, summarization, scaffolding, repetitive transformations.
  - Segment 3 framing: *not using AI for everything is itself a focus discipline.* Over-applying agents wastes time on debugging AI output for tasks where you'd have been faster writing the code yourself. The cognitive overhead of "should I let the agent try?" is itself non-trivial — having a clear personal heuristic for when to invoke agents saves real time.
  - **Sharper version of the point: "Can be done with a simple script — not an agent!"** Concrete heuristic: if the task is deterministic, repeatable, has clear inputs/outputs, **just write the script.** Reaching for an agent is over-engineering. Script wins on every axis:
    - Faster to write than orchestrating an agent
    - No token cost
    - Deterministic — no hallucinations
    - Easier to debug
    - Reusable as-is, no re-prompting needed
  - The "agent for everything" temptation is real (agents feel powerful and novel), but the right line is: **agents handle ambiguity; scripts handle determinism.** Match the tool to the actual shape of the task.
- **Lais (marketing/devrel, Pydantic): sometimes AI is slower than just doing it yourself in 10 seconds.** Concrete pain point — for trivial tasks the *prompting overhead alone* exceeds the doing-it-yourself time.
  - Examples this captures:
    - Quick rename across a couple of files (faster with find/replace)
    - Fixing an obvious typo (just type the fix)
    - One-liner you already know (faster to type than to describe)
    - Trivial CSS tweak (eyeball + edit vs. explain to a model)
  - Sister observation to the "use a script" point — both are flavors of "don't over-apply AI." Hers focuses on the *small-task* end specifically: tasks so trivial that *prompt + wait + review + tweak* > just doing it.
  - Hidden cost worth naming: every "let me ask AI" moment for a trivial task is also a **context-switch** out of whatever you were actually focused on. Even when the AI is fast, the focus cost isn't free.
  - Decision rule emerging across the panel: invoke AI when it gives meaningful leverage; for sub-10-second tasks, just do them.
- **Panel naming the underlying driver: fear of falling behind the curve / "behind on AI."**
  - This is the engine under most of the overwhelmedness from earlier. People aren't tired because they're using too many tools — they're tired because they feel like they *should* be using more, and every new release is a fresh reminder that they aren't.
  - Worth saying out loud (good that the panel did): the fear is widespread, not personal. It's also disproportionate to the actual risk — missing one model release or tool launch doesn't materially set anyone back.
  - Often the fear itself is **more harmful than the actual lag** — it pushes reactive tool-chasing, which costs focus, produces shallow learning, and burns out the practitioner. Then they're worse off than if they'd just kept their head down.
  - **Direct antidote already named by the panel:** *"only learn tools which solve problems you're trying to solve."* That rule, applied honestly, removes the fear's grip — you can't be "behind" on tools you don't need.
  - This is the strongest throughline in Segment 3 so far: fatigue → fear of falling behind → reactive tool-chasing → more fatigue. Breaking it requires explicitly opting out of "keeping up" as a goal, and replacing it with "solve my actual problems."
- **Practical advice on the fear: just learn as you go.**
  - **"You'll get there"** — reassurance. The pace looks impossible only if you treat it as a syllabus to master; treat it as a stream you sample from when needed, and it becomes tractable.
  - **Just-in-time learning** — learn what you need *as you need it*. Don't try to front-load every tool; pick them up when a real problem makes them relevant.
  - **"One AI tool is not so different from another"** — Copilot, Cursor, Claude Code, Cody, Codex, Aider, etc. share the same underlying primitives (model + tools + context + loop, per the earlier framing). The UX differs; the mental model doesn't. Once you internalize the primitives, switching tools is just learning the new UI — not learning a new paradigm.
  - **Learn over time** — gradual accretion is fine. There's no test at the end; you're not graded on tool count.
  - Together: a coherent counter-prescription to fear-driven tool-chasing. Could also live under Segment 4 (Recommendations) — this is one of the clearest take-home rules from the panel.

## Segment 4: Recommendations

- **Final advice for AI beginners (four points):**
  1. **Be the driver, not the passenger.** Learn the scaffolding around writing code (debugging, testing, version control, code review) — the support practices that let you stay in control of the output. Don't let AI do the thinking; you set direction, AI accelerates execution.
  2. **Expertise matters — you have to understand AI output.** You can't evaluate what a model produces without the skills to recognize good vs. bad code. To get started: **pick one tool and go.** Don't over-research; commit and iterate. (She started with **VS Code + Copilot**.)
  3. **Be open to what AI can help with — it improves fast — but stay wary all the same.** Capability changes month-to-month; today's "can't do that" is next quarter's "trivial." Trust *calibrated*, not blind.
  4. **Be aware that AI hallucinates, and human expertise is what catches it.** Models gravitate toward **average solutions** — fine for typical work, but when the right answer is non-average (novel architecture, edge case, performance-critical path), the human has to bring the judgment. The model won't volunteer to leave the training-data mean.
  5. **You can't outsource the thinking.** Hard version of #1. You can outsource the typing, the lookup, the boilerplate, the syntax recall — but the *reasoning about the problem* has to stay with you. The moment thinking is outsourced, you lose the ability to evaluate, debug, or extend what AI produced. This might be the single most important line of the panel.
- **Closing aphorism: *"AI does what you tell it to do, not what you wanted it to do."***
  - Direct descendant of the classic computing line ("computers do what you tell them, not what you want"), now applied to AI.
  - Sharpens #5: the thinking you can't outsource includes the **specification itself**. The gap between intent and instruction is where AI work goes sideways — sloppy prompt in, sloppy code out, and unlike a colleague the model won't pause to ask "wait, what did you actually mean?" It will confidently do the wrong thing.
  - Prompting is itself a skill — being precise about what you want is part of the work. The AI doesn't do that part for you either.
- **Advice for *advanced* AI users (different layer than the beginner list above):**
  1. **1-in-10 manual rule:** roughly one in ten tasks, do it by hand — *no AI assistance.* Defense against **skill rot**. There's real evidence (multiple studies on Copilot users in 2024–2025) that heavy AI reliance erodes the underlying skills you used to have. The 10% manual budget keeps those skills resident.
  2. **Evaluations ("evals") + guardrails.** What "evals" probably means here:
     - **Most likely:** *systematic evaluations of your AI workflows / agents* — having a representative set of tasks you run against your tooling so you can tell when it regresses, when a new model is actually better, when a prompt change broke something. Tooling: Inspect, OpenAI Evals, PromptFoo, Braintrust, etc.
     - **LLM-as-judge** — using a model (or panel of models) to score the outputs of another model. Cheap-ish way to scale evaluation beyond what humans can review.
     - **MCP evals** — possible if she meant specifically testing MCP servers, but less likely from context.
     - In all three readings, the underlying principle: **don't run AI in production by vibes — measure it.** Set up the evaluation harness once; it pays for itself every time you swap models or change prompts.
  - **Guardrails** mentioned alongside — runtime constraints (input validation, output filtering, refusal behavior, sandbox restrictions) that prevent AI systems from going off the rails. Pairs naturally with evals: evals catch *quality* regressions, guardrails catch *safety/correctness* failures.
  - Net for advanced users: the moves shift from "learn the tool" to "engineer around the tool" — manual-skill maintenance + measurement + safety rails. The work becomes operational, not adoption-flavored.
- **Use the tools efficiently — they're cheap now, but won't be forever.** Plus environmental-impact implication.
  - The "cheap now" reality: frontier-model pricing in 2026 is still largely subsidized — VC capital, market-share competition, and below-cost token pricing distort what AI inference actually costs.
  - The "won't be forever" prediction: as the industry stabilizes (winners emerge, capital tightens, training-compute costs hit pricing), token costs will rise to reflect real economics. Workflows built on the assumption of near-free inference will get expensive overnight.
  - **Environmental angle:** even when financial cost is hidden, the compute is real — energy, water for cooling, embodied carbon in GPUs. Wasteful prompting (re-running the same query, agent loops that flail, oversized models for trivial tasks) compounds at scale.
  - Concrete efficiency moves:
    - Right-size the model — don't use frontier for tasks a 7B can handle
    - Cache responses where the input is repeatable
    - Prefer scripts for deterministic tasks (callback to the earlier point)
    - Avoid agent loops that hallucinate forward when a structured tool call would do
    - Batch where possible (cheaper per-token, lower wall-clock latency too)
  - Pairs with the "10-second task = just do it" and "use a script" rules — all three are versions of *don't burn AI on what doesn't need AI*, just from different framings (your time, your resource use, your team's ops cost).
- **Use AI to clean up tech debt.** Classic AI-strength case — tedious, well-defined, low-risk work that humans never get around to.
  - Why this is a sweet spot for AI:
    - The "correct answer" is usually well-defined (you're propagating a known new pattern, not inventing one)
    - High volume of mechanical changes is exactly what models do best
    - Tests catch regressions, so risk is bounded
    - Humans don't enjoy this work, so it sits forever — AI removes that activation-energy barrier
  - Concrete examples that fit:
    - Migrating from a deprecated API to the current one (e.g. `requests` → `httpx`, sync → async, old SDK → new SDK)
    - Adding type annotations to a legacy untyped codebase
    - `print` → `logging` across hundreds of call sites
    - Splitting a monolithic module into smaller files
    - Normalizing style / lint fixes
    - Updating tests to a new framework version
    - Dependency upgrades that require code changes (the codemod story)
  - Net: AI unlocks the *backlog of work nobody wanted to do.* Cap1's "use OSS, customize with middleware" thread from earlier connects here — if your wrappers were creating tech debt, AI can help retire them.
- **Delete AI trash.** If you vibe-coded something, it turned out crap, and nothing actively uses it — *just delete it.* Don't keep AI-generated artifacts around out of sentiment, sunk cost, or "we might want this later."
  - Why this is its own rule (vs. just "delete dead code"):
    - **Vibe-coded code accumulates fast** — low effort to produce means repos fill with experimental outputs nobody owns or remembers
    - **AI code without active maintenance rots harder** — there's often no real understanding behind it, so when it breaks, nobody knows what it was supposed to do
    - **Sunk cost doesn't apply** — the cost of producing AI code is trivial; the bias to keep it because "we already wrote it" is irrational
    - **Regeneration is cheap** — if you ever do need it, you can just re-prompt; the artifact isn't precious
  - Pairs with the Cap1 "Hyrum's Law" thread from earlier, inverted: don't let unused AI artifacts become load-bearing simply by existing in the repo long enough for someone to start depending on them. Cut early, cut often.
  - Practical version of the rule: when reviewing a PR or auditing a repo, AI-generated code with no current users gets deleted by default. Reversal requires positive justification ("we actively need this for X"), not the absence of objection.
  - **Your reaction (preserved):** *"interesting take."* Worth flagging — this is the kind of small-but-counter-cultural rule that's easy to miss. Most teams treat code-in-repo as default-keep; this inverts that for AI-generated artifacts specifically. Genuinely novel framing for 2026 repo hygiene.

## Closing sentiment

- Panel closed on a reassurance beat — **"AI is not taking all your jobs."** Direct counter to the doom-discourse named in Segment 3 (the "skills files to write me out of a job" anxiety).
- Honest read: this was *sentiment*, not data. The panel didn't argue the case rigorously — they offered comfort, which is fine for a closer but worth flagging.
- The structural backing for the sentiment (whether they said it or not) is the rest of the panel's content: you can't outsource thinking (#5), models gravitate to average solutions (#4), human expertise is what catches hallucinations (#4 again), and the roles that pair AI fluency with domain judgment are growing. The case exists; it just wasn't formally made in the closing minute.
- Sensible ending — leaves the audience with permission to engage with AI tooling without spiraling on labor anxiety. Whether the reassurance survives contact with the next two years of capability growth is a separate question.