# Distributing AI with Python in the Browser: Edge Inference Flexibility Without Infra

**Speaker:** Fabio Pliger
**Event:** PyCon 2026
**Date:** 2026-05-15

## Background (pre-loaded while listening)

- **Speaker:** Fabio Pliger — creator/lead of **PyScript** at Anaconda; previously core contributor to Bokeh.
- **PyScript / Pyodide stack** (the substrate this talk lives on):
  - **Pyodide** = CPython compiled to WebAssembly. Real Python interpreter in the browser, including most of the stdlib and a curated set of pre-built scientific packages (numpy, pandas, scipy, scikit-learn, etc.).
  - **PyScript** = the framework on top of Pyodide that makes it ergonomic to embed Python in a web page — `<py-script>` tags, lifecycle, DOM bindings, package management.
- **Why the timing is right for this talk (2026):**
  - **WebGPU shipped by default across Chrome, Firefox, Edge, Safari on 2025-11-25** — ~82.7% global coverage. The single biggest enabler for client-side AI; no more flag-flipping.
  - Browser LLM runtimes have matured this year — see below.
- **Browser ML runtime ecosystem (the things the talk will likely reference or compare to):**
  - **WebLLM** (MLC AI) — full LLM inference in browser, WebGPU-accelerated, ships an **OpenAI-compatible API** so you can swap a hosted endpoint for a local one with minimal code changes. ~17.6k GitHub stars.
  - **Transformers.js v4** (Feb 2026) — Hugging Face's browser port; rewritten in C++ with a WebGPU backend, **3–10× faster than v3**. 200+ model architectures, 1,200+ converted models.
  - **ONNX Runtime Web** — multi-backend (WebGPU / WebGL / WebNN / WASM); the generalist option for arbitrary ONNX models.
- **Performance numbers worth knowing:**
  - WebGPU ≈ **3–5× faster than WebGL** for transformer workloads.
  - WebGPU ≈ **10–15× faster than WASM-only** for the same.
- **Two product framings the talk is likely to make** (per the PyCon abstract):
  1. **Edge inference** — model runs in the browser, no server round-trip, no infra bill, data never leaves the device (privacy + offline capable).
  2. **Agentic workflows** — Python in the browser as a *secure sandboxed runtime* where the LLM lives on an external API (OpenAI/Anthropic), but the browser hosts the agent loop, tool calls, preprocessing, and UI glue.
- **Key tension to watch for:** Python's strength is the data/ML ecosystem (numpy, pandas, transformers, etc.); the browser's strength is JS-native ML runtimes (transformers.js, WebLLM). The talk will probably argue Python-in-browser as the *glue/orchestrator* over JS-native runtimes, rather than competing with them — i.e. Python does preprocessing, agent logic, UI; JS does the heavy GPU inference.

**Sources:**
- [PyCon US 2026 — Distributing AI with Python in the Browser](https://us.pycon.org/2026/schedule/presentation/126/)
- [WebGPU Browser AI Inference (2026)](https://www.buildmvpfast.com/blog/webgpu-browser-ai-inference-cost-savings-2026)
- [WebLLM — MLC AI](https://github.com/mlc-ai/web-llm)
- [Pyodide](https://pyodide.com/)
- [PyDev of the Week: Fabio Pliger](https://blog.pythonlibrary.org/2023/01/23/pydev-of-the-week-fabio-pliger/)

## Notes

- **Opening definition: agent = 4 parts.**
  1. **Model** — the LLM doing the reasoning
  2. **Tools** — external capabilities the model can invoke (web fetch, code exec, file I/O, API calls)
  3. **Context** — state carried across turns (conversation history, scratchpad/memory, system prompt)
  4. **Loop** — the run cycle: model proposes → tool executes → result feeds back → repeat until done
  - This is the now-canonical industry framing of what an agent is (Anthropic, OpenAI, etc. have converged here).
  - Set-up for the talk: each of the four parts can live in the browser or on a server, and the talk is presumably going to show that **all four can live in the browser** with PyScript + WebLLM + local storage + a Python event loop. Browser-as-agent-runtime is the thesis.
- **In-browser implementation of each part:**
  - **Model:** **WebLLM**, GPU-accelerated via WebGPU, **Llama-family models** as the workhorse (and other open weights). All model weights and inference live client-side.
  - **Tools:** **MCP / Skills.** Notable choice — Fabio is leaning on **Anthropic-flavored tool abstractions** (Model Context Protocol + Skills) rather than rolling his own. Means browser agents can plug into the same tooling ecosystem as server-side ones.
  - **Memory (context):** quoting Fabio — *"just a big array of things."* Deliberately deflationary. He's pushing back on the mystique around memory/RAG/vector DBs. In a browser context, memory genuinely can be a JSON-serializable list in local storage. No vector DB required unless your scale demands it.
  - **Loop:** *"self-explanatory."* Fabio signaling that the loop isn't the interesting part. Standard `while not done: r = model(history); run_tools(r); update(history)` shape — same in browser as anywhere else.
- Implicit thesis: the agent stack is *not* mystified. With these four mappings, **the entire agent runs in the browser** — no server, no infra, no per-token API bill. The talk is essentially demystifying what looks like a serious infrastructure undertaking down to four boxes you can wire together in PyScript.
- **Important conceptual point:** *"The LLM does not call tools, it asks the agent to provide them."*
  - Why the distinction matters (where he's going):
    - The LLM is a pure text-in/text-out function. It just emits a *request* like "I want to call `read_file('foo')`." It never actually executes anything.
    - The **agent** (the surrounding orchestration code) is what receives that request, decides whether/how to fulfill it, runs the actual work, and feeds the result back.
    - This is a **security/policy seam**. Because the model only requests, the agent gets to be the policy layer — it can validate, sandbox, refuse, or transparently redirect every tool call.
    - It's also a **decoupling seam**: the model doesn't know or care *where* the tool runs (browser DOM, server API, user prompt, local file). The agent decides per-call.
  - **Why this lands hard in the browser context specifically:** the browser is already a sandboxed runtime with a tight permission model. Putting the agent inside the browser means *every tool call passes through two policy layers* (the agent's logic + the browser's sandbox). That's a uniquely strong security story you don't get with server-hosted agents.
  - Almost certainly the setup for: *"...and that's why the browser is the ideal agent host — it's the place where you can intermediate every tool call between the model and the outside world."*
- **Demo time — two example browser agents:**
  1. **CSV agent** — load a CSV, ask questions, get summaries. Pure in-browser: pandas (via Pyodide) does the parsing/aggregation, model (WebLLM or API) does the natural-language layer.
  2. **Folder agent** — operates over a folder of files. Likely using the **File System Access API** (modern browsers) or directory-picker for read access.
- Why these two demos are well-chosen:
  - **CSV agent** is the *Python differentiator demo*: pandas in the browser is a real edge — JS-only agents don't have anything as fluent for tabular work. This is exactly the "Python glues, JS does inference" thesis paying off.
  - **Folder agent** is the *agency demo*: shows browser agents can touch real file systems (within sandboxed user-granted permission), not just chat. Counters the "browser agents are toy" perception.
  - **Both demos hit the privacy angle hard** — sensitive CSV / personal folders never leave the device. No upload, no server logs, no per-token bill. The data-locality value prop becomes tangible.
  - User confirmation: **both demos run entirely in-browser.** No server-side hop anywhere in the loop — model, tools, data, agent loop all running client-side via PyScript + WebLLM. This is the talk's thesis made concrete on screen.
- **Next demo beat: "get agent dynamically, run agent in browser."** Distribution pattern — agent definitions can be **fetched at runtime** (from a URL/manifest), loaded into PyScript, and executed client-side without any prior install.
  - New distribution model for agents:
    - Hosted agents → require infra and per-call billing
    - CLI agents → require install + local trust
    - **Browser-dynamic agents → visit URL, agent runs in browser sandbox over user's local data, nothing uploaded**
  - This is the "edge inference flexibility without infra" subtitle of the talk in one concrete shape: agents become as portable and frictionless to distribute as web pages.
- **Demo: Anaconda Desktop running a local model, with the browser agent talking to it.** This section was a bit hard to parse mid-talk — losing the thread on what the *point* was. Two readings:
  1. **Architectural point (probable real reason):** the agent lives in the browser, but the **model can live anywhere** — WebLLM (in-browser), Anaconda Desktop / Ollama / LM Studio (localhost), or a remote API (Anthropic/OpenAI). Same agent code, swappable model. Anaconda Desktop specifically unlocks "bigger model than fits in browser" while still keeping data local. This is the **flexibility** half of the talk's subtitle made concrete.
  2. **Honest reading:** Fabio works at Anaconda; Anaconda Desktop is an Anaconda product. Part of this is product showcase. Not all demos are pure architecture.
  - Both can be true at once. The architectural point is real (model-pluggable agents are a legitimate design), but the choice of *Anaconda Desktop specifically* over the more generic Ollama or LM Studio is product-positioning.
- **Chrome demo: mounting a local folder into the browser** — almost certainly the **File System Access API** (Chrome/Chromium-only well-supported; Firefox/Safari lag). Browser prompts the user to grant access to a local directory; once granted, the page can read *and write* files in that folder directly.
  - Why it's a "powerful model-building feature":
    - **No upload** — training data, documents, model artifacts stay on disk; the agent reads them in place.
    - **Bidirectional** — agent can write outputs back (fine-tuned weights, RAG indices, generated artifacts) into the same folder.
    - **Persistent across sessions** — permission can be re-granted on reload; the agent can resume work on the same project folder.
    - **Sandboxed** — user grants per-folder; the browser still mediates everything.
  - Combined with the earlier "folder agent" demo, this is the substrate for non-trivial workflows: build a RAG index over a local docs folder, fine-tune on local CSVs, run a code-analysis agent over a local repo — all without any data ever leaving the device.
  - Caveat to flag: Chrome-only-ish today (Safari support partial, Firefox behind a flag). The "browser agent" story still has cross-browser friction unless you stay within the lowest-common-denominator APIs.

## Q&A

- **Q:** *"Can you rely on user machine capability to run local models?"*
- **A:** *"It depends."* Short version: works fine if (a) the problem is small enough to fit any modern machine, or (b) you know what hardware you're targeting (enterprise scenario where IT controls the fleet). Otherwise, **add guardrails**.
  - What "guardrails" probably means in practice:
    - Capability detection — check WebGPU availability, GPU memory, RAM before loading the model
    - Graceful fallback to a smaller local model, or to a remote API (Anthropic/OpenAI) if the device can't handle it
    - Clear UX disclosure about what's running locally vs. remotely
    - Possibly model auto-selection based on detected hardware tier
  - **Honest read on the answer:** this is a partial walk-back on the "edge inference without infra" pitch. The "no infra" win is real, but the constraint has *shifted* from servers you don't have to clients you don't control. For consumer products that means heterogeneous hardware, and your great architecture meets the reality that 30% of your users are on a 6-year-old Chromebook. For enterprise (Cap One-style controlled fleets) the story is much cleaner — IT picks the hardware, you pick the model that fits. The talk's pitch is strongest in the enterprise context for exactly this reason.

## The main demo: "Hybrid AI Router Demo"

- **What the name means:** browser-hosted agent that **routes model calls between local and remote** depending on context.
  - **Hybrid** — not all-local, not all-remote; both, conditionally
  - **Router** — the agent decides per-request which model to use
- **Likely routing inputs:**
  - Privacy classification of the input (sensitive → local, generic → remote)
  - Required model capability (local 7B can't handle this → bump to remote frontier model)
  - Cost (local = free, remote = per-token)
  - Latency (small/fast → local, heavy reasoning → remote)
  - Availability (no local GPU? → fall back to remote)
- **This demo answers the prior Q&A's "guardrails" point.** The router IS the guardrail — capability detection drives routing, and the agent code stays the same regardless of where the model actually runs.
- **Why this is the right capstone for the talk:** it ties every earlier thread together —
  - **4-part agent** (model/tools/context/loop): the router is just policy *inside* the model slot
  - **Model can live anywhere**: the router proves it by demoing the choice in action
  - **Privacy/edge inference**: routing local-by-default preserves the data-locality story
  - **Flexibility without infra**: the router is what makes the "without infra" claim defensible — if local can't handle it, remote takes over, no infra needed on the developer's side
  - **Anaconda Desktop demo**: now contextualized — it's one of the routes the router can pick
  - **MCP/Skills tooling**: tool calls likely route similarly (browser-side vs. server-side execution)
- This is genuinely the architectural punchline. The talk isn't "run models in your browser" (that's old news); it's "use the browser to orchestrate where models run, dynamically." That's the actually novel claim.
- **The "turn off WiFi" reveal:** Fabio killed WiFi mid-talk and ran his local-model demo — two persona agents (**Barney Rubble** and **Snoop Dev**) chatting in the browser. Kinda cool. Solid showmanship beat:
  - The WiFi-off moment is the strongest possible visual proof that inference is actually happening client-side. No "trust me, it's local" — you can see the network is dead and the agent still responds.
  - Multiple personas in one browser = same underlying model, different system prompts/context per agent. Demonstrates the "context" slot of the 4-part agent in a playful way.
  - PyCon audience-appropriate: silly characters + dramatic infrastructure-disable reveal = applause moment. Doesn't hurt the credibility either; it earns it.
- **Closing reveal: the demo UI itself is PyScript.** Surprise. The whole stack on screen — agent loop, model orchestration, AND the rendered UI — was end-to-end PyScript/Python in the browser. Doubly self-referential and a clean punchline: not "Python can run AI in the browser," but "Python *is* the entire app — UI, agent, model orchestration, no JS scaffolding at all."
  - **Vibe-coded.** Fabio admitted/revealed the UI was thrown together with AI assistance — no frontend engineer required, no design system, just iterate with the model until it looks right.
  - **Direct DOM/CSS** — the UI is PyScript driving the **DOM and CSS directly**, not riding on a JS framework (React/Vue/Svelte/etc.). Python is reaching into the browser primitives and rendering from there.
  - Combined punchline gets sharper: Python in the browser can **vibe-code a real DOM-manipulating UI** on top of an AI agent stack, no JS framework, no server. That's a meaningful productivity claim — the gap between "data scientist with an idea" and "shipped browser app" just got dramatically smaller, because the data scientist doesn't have to learn React.