# Musings: A First-Class "Absent Argument" Concept for Python

**What this is:** a side-doc capturing a hallway-track-style musing from PyCon 2026.
**Triggered by:** Koudai Aono's "Beyond Optional" talk ([notes](beyond-optional.md)) — specifically his framing of `UNSET` as a "workaround" for a missing language primitive, and his disclosure during Q&A that he had t-string syntax proposals (the rejected alternatives in [PEP 750](https://peps.python.org/pep-0750/#rejected-ideas)) cut during the PEP process.
**Date:** 2026-05-16
**Status:** Speculative / unlikely to be acted on. Recorded for future reference, not as a serious roadmap.

## The seed idea

Even after Python 3.15's `sentinel()` lands (PEP 661), Python still has no **first-class concept of an "absent argument"** at the function-signature level. The fix Koudai walked the room through — sentinel-typed unions like `T | None | _UnsetType` everywhere — works, but the *type-union pollution* it introduces is conspicuously ugly compared to what other languages give you for free:

- **OCaml**: `let f ?(foo : 'a option) = ...` — `foo` inside is `'a option`, language handles the absence
- **TypeScript**: `function f(x?: T)` — `x` inside is `T | undefined`
- **Scala / Kotlin / Swift / Rust**: optionality is encoded in the type system at the parameter level, not via sentinel values

In Python, "did the caller pass this argument?" remains a question the language cannot answer statically. We encode the answer in value-space (sentinels) rather than type-space (call-protocol metadata). Every PATCH handler reinvents the dance.

## The current state (as of Python 3.15)

| Feature                              | PEP | Python | What it does                                          |
|--------------------------------------|-----|--------|-------------------------------------------------------|
| `TypedDict`                          | 589 | 3.8    | Typed dict shape                                      |
| `Required` / `NotRequired`           | 655 | 3.11   | Per-field key-presence at TypedDict level             |
| `type X = ...` (and generics)        | 695 | 3.12   | Modern type-alias syntax                              |
| `ReadOnly[T]`                        | 705 | 3.13   | TypedDict-field-level read-only marker                |
| `TypeIs[T]`                          | 742 | 3.13   | Bidirectional narrowing predicates                    |
| `sentinel(...)`                      | 661 | 3.15   | Standard sentinel-value factory                       |

**What's solved:** the absent/null/value distinction at the **dict / wire boundary**. TypedDict + NotRequired + `in` checks + `TypeIs` predicates handle it cleanly.

**What's *not* solved:** the same distinction at the **function-call boundary**. `def f(x=UNSET)` still requires value-space sentinels because Python's call protocol doesn't expose "which named args were passed" in a type-checker-friendly way.

## The shape of a hypothetical follow-on PEP

Three strawman directions, from least-invasive to most-invasive:

### Option A — syntax-light: stronger type-checker support for the sentinel pattern

Make `Unset` (or `Missing`) a standard import from `typing`. Add narrowing rules so that `def f(x: T | Unset = Unset)` narrows correctly in `if x is Unset` branches across all mainstream type checkers.

- **No grammar change.** Reuses PEP 661's runtime sentinel + adds typing-side blessing.
- Probably 80% of the practical benefit. The "ergonomic" gap closes mostly because there's a *canonical* name everyone agrees on, not because the syntax is shorter.
- Could ship in a typing PEP, no CPython parser work.

### Option B — typing-only: a `typing.NotPassed[T]` / `Maybe[T]` annotation

A typing construct that *looks* like a normal annotation but carries call-protocol metadata. The type checker knows that a parameter annotated `Maybe[T]` may not be passed, and narrows accordingly inside the function. At runtime it's still a sentinel — but the syntax in user code stays clean:

```python
def patch_user(*, nickname: Maybe[str | None], bio: Maybe[str]) -> None:
    if nickname.is_set:
        ...
```

- Smaller scope than full syntax change.
- Requires a wrapper object at runtime, which changes function-call ergonomics slightly.
- Type checkers need new narrowing rules for the wrapper's `.is_set` / `.value` properties.

### Option C — syntax-heavy: TypeScript-style `?:` parameter modifier

```python
def patch_user(*, nickname?: str | None, bio?: str) -> None:
    if nickname is Unset:  # builtin
        ...
```

- Requires CPython parser change, call-protocol change, `inspect.signature` updates.
- Cleanest at the call site; matches the ergonomic destination other languages reached.
- **Major lift.** Every Python release has incrementally added typing primitives but new *syntax* is a much higher bar.

## Realistic obstacles

### PEP 661 cycle time

PEP 661 (the `sentinel()` factory the talk celebrated as the "language solved it") was originally proposed by Tal Einat in **2021**. Final acceptance and shipping in 3.15 means a ~4-5 year cycle from proposal to landing. That's the *small* version of what this would propose. A syntax-level follow-on is bigger and would presumably take longer.

### PEP 750 reference case (Koudai's experience)

PEP 750 (t-strings) became *Final* on April 10, 2025. Its *Rejected Ideas* section ([link](https://peps.python.org/pep-0750/#rejected-ideas)) documents what got cut during the process:

- Arbitrary string literal prefixes (JS tagged-template style)
- Delayed evaluation via implicit lambdas
- Protocol-based approach for `Template` / `Interpolation`
- Custom equality and hashing
- Alternate interpolation symbols (`${name}`)
- Binary template strings (`tb"..."`)

When Koudai mentioned his rejected syntax proposals during Q&A, he was pointing at items in this list. **Even successful PEPs leave significant syntax on the floor.** A new PEP for argument-absence syntax would need an equivalent "Rejected Ideas" section as part of due diligence.

Also worth noting: PEP 750 builds on PEP 501 ("i-strings"), which was *deferred* circa 2015 alongside PEP 498 (f-strings) and only resumed in 2023. **The end-to-end timeline from first attempt to acceptance was about a decade.** Syntax PEPs in this neighborhood are slow.

### Team composition required

PEP 750's author list: Jim Baker, **Guido van Rossum**, Paul Everitt, Koudai Aono, **Lysandros Nikolaou** (CPython parser specialist), Dave Peck. Six co-authors including BDFL emeritus and the parser specialist. A new syntax PEP would presumably need similar bench depth — at minimum a CPython parser-and-eval-loop expert plus a typing-council connection. **Not a solo undertaking.**

### "Two ways to do similar things" is a soft PEP-rejector

`def f(x=None)` already works as a poor-man's "optional." Selling new syntax over the existing pattern requires demonstrably better ergonomics, not just "this is more correct in pedantic terms." Steering Council has been clear that adding parallel ways to express almost-the-same-thing is a hard sell.

## The honest verdict

**Submitting a PEP for argument-absence syntax is unlikely to be a good use of effort for someone outside the typing council's regulars.** Reasons in priority order:

1. **The syntax-change version (Option C) is roughly a decade of effort** with a high rejection probability.
2. **The typing-only versions (Options A and B) are smaller wins** — they close the ergonomic gap by ~30-50%, not 100%, so the value proposition is narrower.
3. **PEP 661 already shipped.** The momentum for a follow-on is reduced by the fact that the most-recent language change in this area just resolved one slice of the problem; arguing for another change immediately is hard.
4. **The audience for the win is small.** PATCH handlers and partial-update flows are common in API design, but most teams have already settled into Pydantic conventions that "work fine" without language-level help.

## The better path: library-first, language-second

The destination most likely to actually ship:

1. **Build a small library** that wraps `dataclasses` with first-class "absent field" support:
   - `field(absent_default=True)` — declares a field whose default means "not provided"
   - `partial_replace(existing, update)` — applies an update, skipping absent fields
   - Clean integration with `TypeIs` predicates for narrowing
2. **Ship it on PyPI.** Get adoption. Document the patterns.
3. **If/when adoption is significant**, the case for stdlib/language standardization writes itself, because you can point at concrete usage data.

This is how `attrs` influenced `dataclasses`, how `requests` shaped `urllib`, how `pydantic` shaped typing decisions. **Library first; language second.** It's the realistic path for a not-yet-on-the-typing-council person to influence the language.

## Followup / who to talk to

If this idea ever does get pursued seriously:

- **Koudai Aono** ([@koxudaxi](https://github.com/koxudaxi)) — most sympathetic ear at PyCon, has lived through PEP 750 review, would be the natural mentor / sponsor / co-author. He explicitly didn't disagree with the framing during Q&A.
- **Lysandros Nikolaou** — CPython parser specialist, PEP 750 co-author. Anyone proposing syntax needs a parser collaborator; he's the precedent.
- **Tal Einat** — PEP 661 author. Has firsthand experience with the cycle time and the steering-council dynamics for "we need a sentinel" proposals.
- **`discuss.python.org` → Ideas category** — first-stop for pre-PEP discussion. *Search for prior threads on "optional arguments" or "absent arguments" before posting* — there's almost certainly old discussion to build on rather than re-tread.

## Worth reading

- [PEP 750 — Template Strings](https://peps.python.org/pep-0750/) — especially the *Rejected Ideas* section as a model for due-diligence on alternatives.
- [PEP 661 — Sentinel Values](https://peps.python.org/pep-0661/) — the shipped precedent for "let's standardize a sentinel pattern."
- [PEP 655 — Marking individual TypedDict items as required or potentially-missing](https://peps.python.org/pep-0655/) — the closest existing PEP to what this would propose (per-field absence at the dict layer rather than per-argument absence at the function layer).
- [PEP 742 — Narrowing types with TypeIs](https://peps.python.org/pep-0742/) — the narrowing primitive any new sentinel pattern would compose with.

---

*Filed as: probably-not-actionable, but preserved so future-me can either pick it up or find it as the moment I remembered which talk this came from.*
