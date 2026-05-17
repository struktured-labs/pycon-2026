# Brett Slatkin — Immutable Data Structures

**Speaker:** Brett Slatkin — author of *Effective Python* (Addison-Wesley), longtime Google engineer, frequent PyCon speaker.
**Event:** PyCon 2026 (Long Beach)
**Date:** 2026-05-16
**Mode:** TLDR / lightweight capture — terse notes, no speculative deep-dives.

## Notes

- **Goals slide** *(struck-through ones are explicit disclaimers — what the talk is NOT about):*
  - ~~Learn to write Python in the style of Haskell.~~
  - ~~Literally translate functional idioms into Python.~~
  - **Understand the value of functional idioms.** *(kept)*
  - Pragmatic, not dogmatic. The talk isn't "Python should be Haskell"; it's "here's *why* FP idioms have value, even if you're writing plain Python."
- **Agenda** *(partial — moved on too quickly):*
  - Pure functions
  - Spooky action at a distance *(mutation / side effects)*
  - ... *(rest missed)*
- **Pure functions — example.** A recursive sum from 0 to n. Standard pedagogical pick.
  - Caveat from your OCaml background: this may be sophomoric for you. Watch for whether he pushes past intro material into something less obvious.
- **Pure functions are easy to write data-driven tests for.** Standard FP-evangelism point. No setup/teardown/global state — just input → expected output. Tests become declarative tables.
- **"Functional sandwich":** **input phase → logical phase → output phase.**
  - This is **functional-core / imperative-shell** (Gary Bernhardt's framing). Or in different communities: onion / hexagonal architecture, "I/O at the edges, pure logic in the middle."
  - The sandwich metaphor: imperative bread (I/O), pure-functional filling (transformations). Read once at the top, transform purely, write once at the bottom.
  - **The practical payoff for Python**: the pure middle is testable, refactorable, parallelizable, cacheable. The imperative ends are where complexity is unavoidable, so you minimize their surface area.
  - Speaker's own gloss confirmed: *"functional part is pure in the middle, outsides are side effects."*
- **Python caveat: no tail-call optimization.** Brett flags that the recursive-sum example will blow the stack at large `n`. CPython explicitly does *not* implement TCO (Guido's design call — preserves stack traces for debugging). So FP idioms imported wholesale via deep recursion fail in Python where they'd work in OCaml/Haskell. Practical Python FP uses iteration or `functools.reduce` for the same shape.
  - Brett also mentions **Stackless Python** as a historical aside — alternative CPython implementation that *did* support deep recursion / coroutines without OS-stack growth. **Not maintained.** Lives on in spirit via `greenlet` / `gevent` / asyncio's coroutine model, but not as a TCO solution. Worth knowing it existed; don't expect to use it.
- **Brett's claim: imperative is more readable than functional** *(in Python)*.
  - **Your pushback** *(from OCaml background)*: not so clear; you prefer functional style.
  - Honest read: this is **audience-relative**. Brett is speaking to a primarily-Python audience whose default eye-training is imperative for-loops; in that context, `for x in xs: result.append(f(x))` *is* faster to read than a pointfree composition. For someone with OCaml-shaped intuitions, functional is more readable because *data flow is explicit* and there's no mutable state to track. Both are right in their respective contexts; neither is universally right.
  - Python comprehensions (`[f(x) for x in xs]`) are a useful **middle ground** — declarative *and* Python-idiomatic. Brett may be conflating "deeply functional Python" (reduce chains, partials, pointfree) with "any FP idiom" — the former *is* harder to read in Python; the latter (comprehensions, generators, map filter on a single line) is well-loved.
  - Refinement from you: Brett scopes the claim to *"the average person."* That's the defensible form — the median Python reader hasn't trained on OCaml/Haskell, so imperative *is* easier for them. Fair enough; your FP comfort is the outlier, not the norm. **The claim survives if framed as "imperative is more readable for the average Python programmer," not "imperative is more readable, full stop."**
- **Now: imperative loops in depth — "if you unroll, is it really mutable?"** Brett's setup question. Take a simple-looking loop:
  ```python
  total = 0
  for x in xs: total += x
  ```
  Unrolled, it's:
  ```python
  total = 0
  total = total + xs[0]
  total = total + xs[1]
  total = total + xs[2]
  ...
  ```
  Even though the underlying `int` is immutable (so `+=` creates a new int), the *binding* `total` is rebound on every iteration. **The loop is mutable at the binding level** — the name `total` does not refer to the same value across the loop. **Imperative loops carry mutation by construction**, even when the values themselves are immutable. Brett is presumably setting up the FP alternative (`sum(xs)`, `reduce(add, xs, 0)`, or `functools.reduce`) where the binding is computed once from a single expression — no rebinding, no per-iteration name mutation.
- **Brett's actual landing: rebinding variables is NOT mutation in Python.** Your reaction: *"fishy argument."* Agreed.
  - **What's technically true in Brett's claim:** Python `int` is immutable. `total = total + 1` doesn't mutate the integer object — it creates a *new* integer and rebinds the name `total` to it. If you held a reference to the old `total`, you'd still see the old value. So at the *object-identity* level, there's no mutation, only rebinding.
  - **Why the argument is fishy anyway:**
    1. **Referential transparency is broken either way.** The whole point of FP's hostility to mutation is that you can't substitute equals for equals — `total` at line 5 doesn't mean what it means at line 10. Whether that's "the object mutated" or "the name was rebound" is *academic*; in both cases the variable is no longer a stable reference to a value. The reader still has to track its evolution.
    2. **FP languages don't accept this dodge.** OCaml's `let` bindings are immutable in exactly the sense Brett is dismissing — you can't rebind a name to a new value within the same scope. Haskell same. **If "rebinding is fine" were a valid FP position, ML-family languages would allow it. They don't.** That's strong evidence the FP community doesn't consider the distinction meaningful for purity reasoning.
    3. **The claim only holds for immutable value types.** For `list`, `dict`, custom mutable objects, `+=` *does* mutate in place — `list.__iadd__` extends the original list, no rebinding happens. So Brett's frame ("rebinding isn't mutation") only saves the simple `int`-sum case; the moment your loop accumulates into a list, the object itself is being mutated and Brett's defense evaporates.
    4. **Practical consequences are mutation-shaped.** Parallel-safety, cache-friendliness, equational reasoning for tests, refactor confidence — all these benefits *of immutability* are lost by rebinding regardless of object identity. If the practical effects are mutation-shaped, calling it "not mutation" is just terminology.
  - **The honest reframing:** rebinding *is* mutation in every sense that matters for FP-style reasoning; the object-identity distinction is a Python-implementation detail that has no payoff for the reader or the verifier. Brett may be making this move to soften the "but Python is imperative" objection from his audience — but doing so by redefining mutation away from the meaning FP languages actually use it for is a bait-and-switch. Your skepticism is justified.
- **Next section: "spooky action at a distance."** Your guess: stateful / non-deterministic effects. Likely right. The phrase is borrowed (from Einstein on quantum entanglement) and in programming usually names:
  - mutating a parameter inside a function and the caller silently seeing the change
  - global variable changes affecting distant code
  - mutable default arguments (the classic Python gotcha)
  - thread-safety violations from shared state
  - hidden side effects in nominally-pure functions
- **Demo example: a dataclass that mutates in a `for` loop for Fibonacci, then returns `self` for chaining.** Sketch:
  ```python
  @dataclass
  class FibState:
      a: int = 0
      b: int = 1
      def step(self) -> "FibState":
          self.a, self.b = self.b, self.a + self.b
          return self     # chaining
  ```
  - **This is the anti-pattern.** Returning `self` after mutating it is a *fluent-interface trap* — looks like FP method-chaining (`obj.step().step().step()`), behaves like imperative mutation. Caller can't tell from the chain alone that `obj` is being mutated; if anyone else holds a reference to `obj`, they see the changes.
  - Common pattern in Python ecosystems (pandas, ORM query builders, some HTTP clients) — convenient but actively misleading about its semantics. **The visual grammar is FP; the runtime behavior is OO state mutation.**
  - Brett is presumably about to contrast this with a *truly* functional alternative: `step()` returning a *new* `FibState` rather than mutating self. Then chaining is honest — each step produces an independent value.
- **`fib_up()` and `fib_down()`** — two functions both operating on (and mutating) the same `Fib` dataclass. Brett is building the spooky-action example: two functions that both reach into the same shared object. The order of calls now matters; the state of `obj` depends on which functions were called and in what sequence — none of which is visible from a call site like `fib_up(obj); fib_down(obj)`. **Classic shared-mutable-state setup; the bug-class he's about to demonstrate.**
- **The bug demo:**
  ```python
  result  = fib_up(8, fib)
  result2 = fib_down(3, fib)   # WRONG — meant to pass `result`, not `fib`
  ```
  - **The intended code** was `fib_down(3, result)` — take 3 steps back from where we ended up.
  - **The actual code** passes `fib` (the original). In an *immutable* world, `fib_down(3, fib)` would operate on the original starting state — totally wrong answer; bug is obvious.
  - **In the mutable world:** `fib_up(8, fib)` *mutated* `fib` while computing `result`. After the first line, `fib` and `result` refer to the same mutated object (or `fib is result`, depending on whether it returned self). So `fib_down(3, fib)` happens to produce the same value as `fib_down(3, result)` would have. **The bug is masked by the aliasing.**
  - **This is the killer punch on mutation:** sharing state makes some bugs *invisible*. The typo (`fib` vs `result`) produces the right answer for the wrong reason. Anywhere else in the program where `fib` is referenced, the reader expects the original state; it now silently carries the post-`fib_up(8)` state. **Mutation hides programmer errors that immutability would surface loudly.**
  - Spooky-action-at-a-distance landed exactly: a write through one alias (`fib`) is observable through another (`result`), without any indication at the call site that the two are linked. **Reasoning about either variable requires reasoning about both, simultaneously, in temporal order — which is the whole reason FP rejects this pattern.**
  - **Brett's prescription:** the dataclass should be **`frozen=True`**. Then `step()` can't mutate `self`; you'd have to return a new instance via `dataclasses.replace(self, a=self.b, b=self.a+self.b)`. The aliasing bug above becomes impossible because there's no shared mutable state to alias *into*.
  - **Cross-talk match:** this is the exact same prescription Koudai's "Beyond Optional" gave for the *domain model* layer (frozen dataclass, FP-style updates via `replace`). **Two speakers, same answer:** when you want shared-aliasing safety in Python, the answer is `frozen=True` dataclasses + `replace()`. The pattern has crystallized into a Python-community standard regardless of which talk you walked into.
- **`dataclasses.replace()` for frozen classes.** Brett's followup — the canonical "update" mechanism for a frozen dataclass:
  ```python
  from dataclasses import replace
  new_fib = replace(fib, a=fib.b, b=fib.a + fib.b)
  ```
  Better than the explicit constructor (`Fib(a=fib.b, b=fib.a + fib.b)`) because it **carries forward all unchanged fields automatically**. Add a field to the dataclass later, every explicit constructor call breaks; `replace()` keeps working. The FP-style record-update pattern, in stdlib.
- **Bug fixed via the immutable rewrite.** Brett now shows the same pipeline with `frozen=True` + `replace()`-based `fib_up`/`fib_down`. The `result = fib_up(8, fib); result2 = fib_down(3, fib)` aliasing typo from before *can't produce the right answer for the wrong reason anymore* — `fib` stays at its original `(0, 1)`, so `fib_down(3, fib)` produces a visibly wrong value. **The bug becomes loud instead of silent**, which is the whole point. Immutability isn't "less mutation" — it's *fail-loudly-on-this-class-of-mistake*. Same code shape, different failure mode, dramatically higher signal at the bug-catching layer.
  - **Your earlier pushback validated.** Brett's "rebinding isn't mutation in Python" claim is now retroactively shown to be insufficient: the *fix* requires `frozen=True` to prevent rebinding/mutation of the underlying state. If rebinding really weren't mutation, you wouldn't need `frozen=True` — but you do, because the practical effects are mutation-shaped regardless of the Python-implementation framing. **The talk's own prescription contradicts the earlier semantic dodge.** Your skepticism scored.
- **Next section: referential transparency.** The formal version of "pure function" — an expression has referential transparency if you can replace it with its value (anywhere it appears) without changing the program's meaning. Equivalently: same inputs → same outputs, no hidden state, no side effects observed by anyone else. The whole conceptual frame for why purity *enables* substitution-based reasoning, memoization, parallelism, etc.
- **Memoization with `fib`** — the canonical payoff demo. Pure `fib(n)` is safely memoizable:
  ```python
  @functools.cache    # or @lru_cache
  def fib(n):
      if n < 2: return n
      return fib(n-1) + fib(n-2)
  ```
  Transforms exponential recursion into linear. **The `@cache` decorator works only because `fib` is pure.** Memoize an impure function and you'd get stale cached results when the hidden state changed underneath. Purity makes the caching transformation *semantically safe by construction*. Brett's likely punchline: *this whole free speedup is locked behind a property you only get if you avoid mutation and side effects.*
- **`functools` caching preserves object identity.** Subtle but worth catching: a cache hit returns the *same object* (`is`-identical), not just an equal one. Implications:
  - **For immutable returns (frozen dataclasses, tuples, ints):** beneficial — no duplicate allocations, `is` checks work across cache hits, references can be shared widely with zero risk.
  - **For mutable returns:** *dangerous* — the caller could mutate the cached object and affect every future cache-hit recipient. **Identity-preservation only composes safely with immutability.** Brett's chain of recommendations is consistent: frozen dataclasses + pure functions + memoization stack together correctly precisely because each layer respects the others.
- **Partial application via `functools.partial`.** `partial(f, x)` returns a new callable with `x` pre-bound. Standard ML-family currying-lite, in stdlib. Useful for specializing generic functions, building callbacks, cleaner than lambdas for many cases.
- **Function composition.** Brett's now on `f ∘ g`-style composition. Python doesn't have an infix compose operator (no `.` like Haskell, no `>>` like F#). Options:
  - Manual: `f(g(x))` — inline, no abstraction.
  - Hand-rolled helper: `def compose(*fns): return lambda x: reduce(lambda acc, f: f(acc), reversed(fns), x)`
  - `toolz.compose` if you reach for a library
  - Method-chaining on a wrapper class (pandas-style fluent interfaces) — different ergonomics but composes-ish.
  - **Honest take:** composition is the roughest FP edge in Python. Without an infix operator, deep pipelines either inline-nest awkwardly or import a library. Most Python codebases settle for nested calls and stop short of "everything is a pipeline" Haskell-style design.
- **The `|>` pipe operator — wished for, PEP blocked.** Brett laments that Python could have had clean pipe syntax (`x |> f |> g` for `g(f(x))`) but the proposal stalled / was rejected. Recurring `discuss.python.org` topic with no acceptance.
  - **Yes — it's the OCaml pipe operator.** Also F#, Elixir, Elm. Same `x |> f ≡ f(x)` shape across the ML / ML-influenced family.
  - **Why Python doesn't have it:** mostly Guido / steering-council resistance — `|` is already bitwise-or, the syntactic ambiguity is real, and the typing-checker story for "argument-position-injection" semantics (Elixir-style `x |> f(y)` ≡ `f(x, y)`) is non-trivial. The pure single-arg form (OCaml-style `x |> f` ≡ `f(x)`) is the cleanest and probably what a future PEP would target if anyone got it past the steering council.
  - **Workarounds in the wild:** `toolz.pipe`, the `pipe` library, `returns`'s pipeline operators via `__or__` overloading (pandas-style trick). All ergonomic-ish but not language-native.
  - **Connects to your "submit a PEP" musing from earlier today:** the pipe operator is *another* example of an ML-family idiom Python community keeps wanting and the steering council keeps not landing. Same shape as your "optional argument syntax" musing — a clean idea, real-world precedent in other languages, blocked in Python for reasons combining "explicit is better than implicit" + parser-cost + "you can do this in a library." Worth knowing as a precedent for what gets stuck in this space.
- **`map()` builtin vs list comprehensions.** The standard Python comparison.
  - `map(f, xs)` — FP-native, lazy (returns an iterator), composes with other iterator pipelines.
  - `[f(x) for x in xs]` — Pythonic, eager (materializes a list), more readable to most Python devs (Guido's stated preference).
  - `(f(x) for x in xs)` — generator-expression analog of `map`, lazy like `map`, but in Python's "comprehension grammar."
  - **Community consensus has been comprehensions** for most cases — readability + filter/condition syntax built in (`[f(x) for x in xs if pred(x)]`). `map`/`filter` survive mainly for callable composition (e.g., `map(int, strings)` is more concise than `[int(s) for s in strings]`).
- **Your prediction (reduce / itertools / more-itertools) is solid for what comes next.** Brett's almost certainly on a stdlib-FP-primitives tour: `functools.reduce`, then `itertools` (`accumulate`, `chain`, `groupby`, `takewhile`/`dropwhile`, `tee`), then maybe `more-itertools` for the pieces stdlib left out. The whole tour is "Python *does* have FP primitives, you just have to know where to look."
- **"Everything is map" + (foreshadowed) bind.** Brett's now pushing the Functor/Monad reframe: most loops decompose into `map` (apply a function per element, preserve structure) and `bind` / `flatMap` (apply a function that returns a wrapped value, then flatten the structure).
  - In Python:
    - **map** — `map(f, xs)` or `[f(x) for x in xs]`.
    - **bind** (a.k.a. `flatMap` / `>>=` in Haskell) — generator expression with multiple `for` clauses: `(y for xs in xss for y in xs)` is exactly `flatten(map(f, ...))`. Or explicit: `itertools.chain.from_iterable(map(f, xs))`.
    - **filter** is a *special case* of bind: `bind(xs, lambda x: [x] if pred(x) else [])`.
    - **reduce** is *not* map/bind — it's a fold (catamorphism); collapses a structure to a single value.
  - **Why Brett is selling this:** "everything is map" is the FP-evangelism wedge for imperative-trained programmers. Once you see `for x in xs: do_thing(x)` as a map, the next step is seeing `for x in xs: for y in g(x): do_thing(y)` as bind. From there, monads are just "containers with a bind operation." The whole conceptual hierarchy lights up if you swallow the first step.
  - **Your OCaml-background frame:** this is exactly the `List.map` / `List.bind` distinction from any ML book's chapter 3. The expected payoff is `Option.bind` and `Result.bind` for error-handling — which is where this is going if Brett hits monads. Watch for whether he calls them "monads" or stays in the more-accessible "containers with bind" framing; senior Python audiences sometimes get spooked by the M-word.
- **`Executor.map` — free parallelism payoff.** Brett's strongest practical argument lands:
  ```python
  from concurrent.futures import ThreadPoolExecutor
  with ThreadPoolExecutor() as ex:
      results = ex.map(f, xs)   # same shape as map(f, xs), but parallel
  ```
  - Same function `f`, same input `xs`, same result shape — drop-in swap from sequential to parallel. **Trivially safe *only because* `f` is pure.** Add hidden state and the parallel version races.
  - This is the *concrete payoff* for everything that came before. Purity → mappable → free parallelism. The case for FP-flavored Python isn't "be more elegant"; it's "unlock this API safely."
  - `ThreadPoolExecutor` for I/O-bound; `ProcessPoolExecutor` for CPU-bound (GIL bypass, pickle cost). In 3.13+'s free-threaded build, ThreadPool can do real CPU parallelism too — *exactly* the kind of language change that pays off for FP-disciplined code more than imperative.
- **`immutable == sharing`** — Brett's punchline. **Lock-free data sharing across threads.**
  - Immutable data can be shared between threads with zero synchronization. No locks. No atomic operations. No memory-fence dance. **Race conditions only exist on writes; remove writes and the entire class disappears.**
  - This is *the* concurrency-correctness payoff for immutability and the strongest pragmatic argument in the whole talk. Erlang, Clojure, Haskell — every language with a strong concurrency story leans on this property. Python inherits it the moment you commit to `frozen=True` dataclasses + pure functions.
  - **The talk's three pillars now snap together:**
    - Pure functions (no side effects)
    - Immutable data (no mutation)
    - → Parallel execution becomes *safe by construction* (no shared mutable state to race on)
  - **Buy one, get the rest for free.** This is the coherent design space the FP camp has been selling for decades, and Brett is landing why it matters *for Python specifically* — not as language-level abstraction but as a discipline that unlocks `Executor.map` and free-threaded 3.13+ without rewriting your concurrency primitives.
  - **Term Brett used: "free threading"** — i.e., PEP 703's GIL-less Python build (`python3.13t`, expanding in 3.14). The community-canonical name. Worth knowing for searching docs.
- **Next slide preview: other built-in immutable data structures.** Brett's about to cover the rest of Python's immutable toolkit beyond `frozen=True` dataclasses. Expected coverage:
  - `tuple` — the ur-immutable sequence
  - `frozenset` — immutable set
  - `types.MappingProxyType` — read-only view of a dict
  - `str` / `bytes` — already immutable
  - Possibly: `pyrsistent` library for persistent/immutable collections (lists, maps, vectors with structural sharing — the Clojure-style data structures)
- **`frozenset` + `frozendict` "coming soon."** Brett confirms.
  - **`frozenset`** is already in Python — hashable immutable set, fully stdlib.
  - **`frozendict`** has been *long-discussed but not in stdlib*. PEP 416 (2012) was rejected; PEP 603 (frozenmap) was deferred. The workaround today is `types.MappingProxyType` (a read-only *view* over a dict; still mutable through the original reference). A real frozen dict — hashable, value-immutable, fully owned — has been on the wishlist for years.
  - **"Coming soon" is Brett-promising** — could be a renewed PEP, a steering-council nod, or a 3.15/3.16 work item I don't have visibility into. Worth following up on what specifically he's referencing if you want the most current status. Either way, it's the long-missing piece in Python's immutable-collections lineup.
- **`tuple` is frozen by definition** — Brett's reminder. The original immutable Python container; no special opt-in needed.
  - Worth knowing the gotcha: tuples are **shallow-immutable.** `(1, [2, 3])` is a "tuple" but the inner list remains mutable. Holds true for `frozenset` / `frozendict` too — they immutabilize the *container*, not the *contents*. Real deep-immutability requires every element also being immutable, recursively. Common newcomer trap.
- **Payoffs of immutability — the comprehensive list slide.** Brett enumerates everything immutability buys you. Decoded from rapid capture:
  - **Stable hashing** — immutable values have stable hashes; safe as dict keys / set members.
  - **Cache keys** — same property; `functools.cache` keys work correctly.
  - **Idempotence** — operations on immutable data don't accumulate state; rerun yields the same result.
  - **Deduping** — equality is well-defined and stable; detect duplicates by value, not identity.
  - **Tracking lineage** — old versions aren't destroyed; the history of how a value was derived can be preserved.
  - **Data versioning** — each "update" produces a new version; old versions remain reachable.
  - **Reversibility** — no destructive operations; can recover prior states.
  - **Sharing** (across threads/processes without locks — covered earlier).
  - **Serialization** — no aliasing surprises, no cycles, no concurrent-modification races. Ship the value out cleanly.
  - **Network communication** — immutable values are the natural unit on the wire (messages, events, snapshots).
  - **Memory** — *structural sharing* via persistent data structures (Clojure-style hash-array-mapped tries, `pyrsistent`-style structures). Two "different" versions can share most of their internal state without copying.
  - **One slide, eleven distinct wins.** Each is something imperative-only code either misses entirely or fakes via significant complexity (locking, defensive copying, audit trails, version columns in DBs). Immutability turns "expensive to maintain" properties into "free by construction" properties. **The talk's whole case for immutability fits on this slide.**
- **"Immutable diffs" — a graph of shared structure; diffs are the nodes.** Persistent-data-structures territory.
  - Each "update" creates a new version sharing most of its underlying structure with the previous one. The **diff** between versions becomes the first-class artifact; the version space is a directed graph (or DAG, with branching) where nodes represent versions and edges encode differences.
  - **Same shape as git's commit graph.** Each commit is a diff against its parent(s); the full history is a graph of shared structure (file contents deduped via content-addressed storage). git is *literally* this idea applied to source code.
  - Also how **Clojure's persistent collections**, **immutable.js**, **Datomic** (Rich Hickey's database), and **Datalog-style fact stores** work — diffs as first-class entities, structural sharing for memory efficiency, the version graph as queryable history.
  - **What this unlocks beyond the eleven-win list:** branching (multiple successors of a version), merging (with conflict resolution), arbitrary-version comparison, time-travel queries, efficient persistence of the full history. Most Python apps don't reach for this layer, but knowing it exists clarifies what immutability *fully* enables — and why "immutable" isn't a limitation but a *foundation* on which dramatically richer data primitives can be built.

- **Final takeaways:**
  1. **Immutable >> mutable.** Direct and opinionated. Default to immutable; reach for mutation only when there's a clear reason.
  2. **Tell your AI to use immutability as much as possible.** Notable 2026-shaped advice — Brett is acknowledging that LLM coding assistants now write a large fraction of production Python, and that *how you prompt them* materially affects code quality. The recommendation: include "prefer immutable patterns, frozen dataclasses, pure functions" in your system prompt / project rules / coding-agent instructions. **Codifies the discipline at the agent-context layer, not just the human-author layer** — increasingly load-bearing as more code passes through LLM hands.
  3. **Use higher-order functions to DRY your code.** Standard FP closer. Compose small pure functions instead of copy-pasting imperative blocks. `functools` + `itertools` + comprehensions cover most needs; reach for a library when stdlib runs out.

- **Honest take on the talk overall** (your call, your notes):
  - The case for immutability lands cleanly *if you didn't already believe it*. Each section was a different angle on the same answer.
  - For an OCaml-trained reader, most of this was first-chapter FP material wrapped in Python idioms. The actually-novel-to-you parts were probably:
    - The 2026 *"tell your AI to use immutability"* meta-takeaway.
    - `functools.cache` preserving object identity (subtle and not always called out).
    - `Executor.map` as the concrete-payoff slide.
    - Free-threading (PEP 703) intersecting with FP discipline.
    - `frozendict` "coming soon" (worth tracking — if it lands, it closes the last big immutable-collection gap).
  - Skippable for you in retrospect; potentially useful if you're explaining FP-style Python to a junior teammate.
