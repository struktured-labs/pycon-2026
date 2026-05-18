# DRAFT — discuss.python.org post

> **Status:** unsubmitted draft. Intended target: [discuss.python.org → Ideas](https://discuss.python.org/c/ideas/6). Tone modeled on the well-received PEP 750 / PEP 661 announcement threads. Do not paste verbatim — reread the day of posting to remove any mention that's gone stale.
>
> **Working title for the thread:** *"Building on PEP 661: a syntactic marker for omissible keyword parameters?"*

---

Hi all,

PEP 661 (Sentinel Values) landed in 3.15, which is great — we finally have a blessed way to construct sentinels with sensible repr, identity semantics, and pickling. I've been using `sentinel()` in a few of my own projects and it's a real improvement over `_MISSING = object()`.

But there's one specific use of sentinels — distinguishing *"the caller did not pass this argument"* from *"the caller passed None"* — where the library-level primitive doesn't quite close the loop. I wanted to share a small proposal and a working CPython reference implementation, and ask for early feedback before deciding whether to push it toward a PEP.

## The gap

In a typical partial-update API today:

```python
_UNSET = sentinel("_UNSET")

def patch_user(user_id: int, *, name=_UNSET, email=_UNSET, age=_UNSET):
    if name is not _UNSET:
        db.set_name(name)
    if email is not _UNSET:
        db.set_email(email)         # None is a real value — clears the field
    if age is not _UNSET:
        db.set_age(age)
```

This works, but two things still bother me:

1. **There's no enforcement that callers can't pass `_UNSET` themselves.** `patch_user(1, name=_UNSET)` is indistinguishable from omission, which means `name is _UNSET` in the body isn't actually a reliable signal of caller intent — it's a convention.

2. **Every library reinvents its own `_UNSET`.** SQLAlchemy has one, Django has one, Pydantic has one, dataclasses has `MISSING`, and so on. They're all the same shape; none of them interop. PEP 661 standardizes the *factory* but the per-library naming and discipline isn't going away.

The second point is mild and arguably fine. The first one is the load-bearing complaint: today's idiom relies on caller goodwill for a property the signature *wants* to guarantee.

## The proposal in one snippet

```python
from inspect import UNSET   # explicit import per file

def patch_user(user_id: int, *, name?: str, email?: str, age?: int):
    if name  is not UNSET: db.set_name(name)
    if email is not UNSET: db.set_email(email)
    if age   is not UNSET: db.set_age(age)
```

Where `x?: T` means:

- Keyword-only parameter that the caller may omit
- Implicit default: `inspect.UNSET` (a stdlib sentinel constructed via PEP 661's `sentinel()` factory)
- Passing `UNSET` *explicitly* (`patch_user(1, name=UNSET)`) raises `TypeError` — the runtime guarantee that today's library-level idiom can't offer
- Body sees `T | type(UNSET)`; narrows on `is`/`is not UNSET`

**Crucially, `UNSET` lives in `inspect`, not `builtins` and not `typing`.** That's deliberate — putting it in `builtins` would create a de-facto universal sentinel and undermine PEP 661's "per-purpose sentinels" position. The `inspect` location signals limited intent and adds enough import friction that nobody reaches for it casually.

## What's deliberately not in this proposal

- **Positional `?:`** — out of scope. Forwarding ambiguities with `*args`. Maybe a future PEP if demand emerges.
- **`x?: T = default`** — SyntaxError. Omissibility *is* the default specification.
- **`case UNSET:` as a bare-name value pattern** — no. PEP 634's "bare names always capture" rule is intentional. Use `case inspect.UNSET:` (dotted form is a value pattern, works today, consistent with how enums are matched).
- **Lambda support** — structurally impossible (lambdas have no annotations).
- **A new builtin predicate (`given()`, `was_passed()`)** — considered, doesn't clear the new-builtin bar.

## Reference implementation

Working CPython fork against 3.16-main: [link to your fork]

Total patch: ~80 lines across `Grammar/Tokens`, `Grammar/python.gram`, `Parser/action_helpers.c`, `Python/ceval.c::initialize_locals`, and `Lib/inspect.py`. Full test suite passes. The implementation desugars `?:` to `(annotated arg, default=UNSET)` at the AST level and adds the per-call runtime check to reject explicit `UNSET` arguments.

To try it:

```
git clone <fork> && cd cpython && ./configure --with-pydebug && make -j
./python -c "from inspect import UNSET; ..."
```

Draft PEP text: [link to draft] (deliberately minimal — kw-only only, ~140 lines, references PEP 661 as Final, no speculative scope).

## Questions I'd like feedback on

I'm not asking for sponsorship or a Steering Council read yet. Just trying to figure out if this is worth shaping into a formal PEP. Specifically:

1. **Is the "runtime guarantee" property worth syntax?** If `_UNSET` with discipline is good enough, this proposal collapses. Curious whether anyone running ORMs / PATCH APIs / SDK clients has felt the lack of enforcement bite in practice, or whether convention has held up fine.

2. **Is `inspect.UNSET` the right home?** I picked it specifically to avoid building a universal sentinel against PEP 661's intent. Alternatives: `typing.UNSET`, a new module (`unset`?), or a deliberately-private location like `inspect._UNSET`. Open to all of these.

3. **Type checker maintainers — is asymmetric body-vs-call-site typing implementable cleanly?** The body sees `T | UnsetType`, the call site rejects `UnsetType`. I've sketched it for mypy/pyright but haven't filed issues yet — would love to know if there's a fundamental obstacle before going further.

4. **Library maintainers (SQLAlchemy, Pydantic, attrs, dataclasses, FastAPI, Django ORM, OpenAPI codegen folks)** — if this shipped, would your library adopt it? What would it replace? "We're happy with our existing sentinel" is a totally valid answer and useful data.

5. **The `?` token reservation.** Is burning `?` on omissibility the right call? Competing potential uses: None-coalescing (`a ?? b`), optional chaining (`obj?.attr`), etc. I'd argue omissibility is the highest-value use because the alternatives can be expressed acceptably with existing operators (`a if a is not None else b`, getattr with default), but I'm interested in pushback.

## Honest priors

I'm aware that new-syntax PEPs have a steep acceptance hill. PEP 671 (late-bound defaults) and PEP 505 (None-coalescing) are both deferred-indefinitely despite real problems and clean designs. I'd put this PEP's realistic odds of acceptance — even with serious polish, sponsor engagement, and library endorsements — at single-digit percentages. That's fine. I'd still rather get the design clearly stated and the implementation in front of people who can poke at it than leave it as a private itch.

Happy to take this in any direction the discussion goes — including "this is what `from __future__ import annotations` was supposed to enable, just write `x: Optional[T] = UNSET`" or "this should be a `Sentinel` library convention not syntax." Both reasonable. Just want the conversation to happen with concrete material on the table.

Thanks for reading,
Carmelo

---

## Posting checklist (for me, not the post)

- [ ] Replace `[link to your fork]` with the actual GitHub URL
- [ ] Replace `[link to draft]` with the PEP file URL (https://github.com/struktured-labs/pycon-2026/blob/main/pep-unset-kwarg-syntax.md)
- [ ] Re-read the day-of for tone, drop any dated phrasing
- [ ] Pick the category: **Ideas** (not PEPs — that requires sponsor first)
- [ ] Tag with `typing` if appropriate
- [ ] Don't @-mention specific core devs in the OP — let them self-select. Reply-mention is fine once they've engaged.
- [ ] Have the CPython fork pushed and `./configure && make` instructions verified before posting (people will try; broken repro = thread dies)
- [ ] Be available to respond within the first 24 hours — discuss.python.org threads cool fast if the author goes quiet
- [ ] Steel yourself: someone will say "just use a sentinel" without reading the runtime-guarantee section. Don't take it personally; politely point at the relevant paragraph.
- [ ] If response is positive, the natural next step is "would anyone be willing to sponsor this as a PEP?" — but only after substantive technical engagement, not as the first ask.
- [ ] If response is negative, capture the strongest objections in the PEP draft's "Rejected Ideas" section before letting the thread die. The feedback is the deliverable.

## What success and failure look like

- **Best plausible outcome:** 1-2 core devs engage substantively, 1+ library maintainer says "we'd consider adopting," someone files a "feasibility from a type checker perspective" reply. Thread runs for a week. You leave with a clearer PEP and a real sense of where the sponsorship conversation could go.
- **Median outcome:** modest engagement (5-20 replies), some bikeshedding on syntax, no sponsor, no commitments. PEP draft gets a few useful refinements. Thread cools after 2-3 days.
- **Worst case:** "use a library" pile-on, no substantive engagement, 5 replies, dead in 24 hours. Disappointing but cheap — you've lost a Saturday, learned where the resistance is, and your CPython fork still exists for the next time the question comes up.

All three are worth the cost.
