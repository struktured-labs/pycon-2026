# PEP XXXX — Unset Parameter Marker Syntax (`?:`)

| Field         | Value                                                                  |
| ------------- | ---------------------------------------------------------------------- |
| **Author**    | (your name) <struktured@strukturedlabs.com>                            |
| **Status**    | Draft                                                                  |
| **Type**      | Standards Track                                                        |
| **Created**   | 2026-05-16                                                             |
| **Python-Version** | 3.16 (target)                                                     |
| **Post-History**   | —                                                                 |
| **Discussions-To** | discuss.python.org/c/peps                                         |

## Abstract

Introduce a dedicated syntactic marker — a `?` placed between a parameter name and its annotation colon — that designates the parameter as **omissible**: callers may leave it out entirely, and the function body can distinguish "the caller did not supply this argument" from any value the caller might have supplied (including `None`, `0`, `""`, sentinels, etc.).

```python
def patch_user(user_id: int, name?: str, email?: str, age?: int) -> User: ...
```

When the caller omits an omissible parameter, the parameter binds to a new language-level singleton, `typing.Unset`. The declared annotation (`str`, `int`, ...) describes the type *when the argument is present*; the static type of the parameter inside the body is `T | Unset`. The `Unset` value **cannot** be passed explicitly by a caller — it can only arise from omission — and it is propagated cleanly through `**kwargs` forwarding without leaking into downstream calls.

## Motivation

Python has no first-class way to express *"the caller may omit this parameter, and I need to tell the difference."* Every existing workaround sacrifices one of: ergonomics, type-checker fidelity, runtime cost, or forwarding correctness.

### The `None` ambiguity problem

The single most common workaround is `Optional[T] = None`:

```python
def update_profile(name: str | None = None, bio: str | None = None) -> None:
    if name is not None:
        db.set_name(name)
    if bio is not None:
        db.set_bio(bio)
```

This conflates two semantically distinct caller intents:

1. *"I am explicitly clearing this field — set it to NULL."*
2. *"I am not touching this field — leave it alone."*

In a PATCH-style API, REST update, ORM partial save, GraphQL mutation, dataclass `replace`, configuration merge, or any function that forwards a subset of received arguments, the difference matters and is currently unrepresentable in a single signature. Every team reinvents one of the workarounds below.

### Existing workarounds (and why they each fall short)

| Workaround | Problem |
| ---------- | ------- |
| `_MISSING = object()`; `def f(x=_MISSING)` | Type checkers see `object` and infer `T \| object`, losing precision. Forwarding requires `if x is _MISSING:` branches at every call site. Library boundary leakage — the sentinel is per-module. |
| `dataclasses.MISSING` | Dataclass-specific; conventionally reused but not blessed for general signatures. Same type-narrowing problems as `_MISSING`. |
| `**kwargs` + `'x' in kwargs` | Erases the signature: IDEs, doc generators, type checkers, and `inspect` cannot see the parameter at all. |
| [PEP 661 — Sentinel Values](https://peps.python.org/pep-0661/) | Standardizes the *value* but not the *syntax* or the *forwarding semantics*. Still requires `Unset` to appear in the type annotation manually, and offers no runtime guarantee that `Unset` cannot be passed by callers or silently forwarded. **Deferred.** |
| `Optional[T] = None` with documented "None means omitted" convention | The documented convention is invisible to callers using IDE autocomplete, breaks for fields where `None` is a legal value, and provides zero runtime enforcement. |
| Overloads (one per omission combination) | 2ⁿ explosion. |

### Concrete use cases

- **ORM / database update functions** — partial update without overwriting unspecified columns.
- **HTTP PATCH endpoints** — JSON Merge Patch (RFC 7396) is *exactly* the unset-vs-null distinction.
- **`dataclasses.replace()`** — currently relies on `MISSING` sentinel under the hood; user-facing wrappers reinvent it.
- **Configuration overrides** — "use the default unless the user provided one" with `None` being a valid user value.
- **Wrapper / decorator code** that forwards a subset of received arguments — currently `if x is not _MISSING: kwargs['x'] = x` boilerplate at every layer.
- **SDK clients** — distinguishing "field not specified" from "field explicitly set to null" when serializing to JSON.

### Why a syntax change

A library-only solution (PEP 661-style) can offer the *value*, but cannot:

1. Make the omitted state the **runtime default** without the user typing it out (`= Unset`).
2. Reject `f(x=Unset)` calls — under a library-only design, `Unset` is an ordinary singleton that anyone can pass, defeating the safety pitch.
3. Strip `Unset` automatically during `**kwargs` forwarding, which is where the existing patterns are most painful.
4. Give type checkers a structural signal that `T | Unset` is the body type while the call-site type is `T`, without requiring a `cast`/narrowing dance.

The `?:` marker turns omissibility into a property of the *parameter*, not a property of a value the user has to remember to use correctly.

## Rationale

### Why `?:` and not `?` after the type, `=...`, a decorator, or `Unset` in the annotation

| Candidate            | Reason rejected                                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------------------------ |
| `def f(x: int?)`     | Reads as "nullable" to anyone arriving from TypeScript/Kotlin/Swift/C#. Conflates two distinct concepts.     |
| `def f(x: int = ...)` (Ellipsis) | `...` is already overloaded (`typing.Any` stubs, slicing, NumPy). Ambiguous and visually noisy.    |
| `@unset_aware` decorator | No syntactic affordance at the call site; introspection still requires reading the decorator's logic.    |
| `def f(x: int \| Unset = Unset)` | Verbose, opt-in, no runtime forwarding semantics, no protection against `f(x=Unset)`.            |
| `def f(?x: int)` (prefix) | Visually closer to `*args`/`**kwargs` prefix markers but reads awkwardly; the marker is about *the parameter*, not *the name*. |
| `def f(x?: int)` ✅  | Reads left-to-right as "x — may be omitted — typed as int." Visually compact, parses unambiguously, no clash with existing operators in this position. |

The placement *between the name and the colon* mirrors how TypeScript and Swift mark optional parameters/properties, which has the most prior art among mainstream languages. The crucial distinction this PEP draws — and the reason `?:` is not just "borrow TypeScript" — is that here `?` means **may be omitted**, not **may be `None`**. Python already has `T | None` for the latter.

### Why introduce `typing.Unset`

The omitted state needs a runtime representative for three reasons:

1. **`if x is Unset:` is the natural body-side check** — symmetric with `if x is None:`.
2. **Forwarding correctness** — `inspect.unset_kwargs(kwargs)` (a new helper) filters out keys whose value is `Unset`, enabling clean passthrough.
3. **Type checker discrimination** — `Unset` is a distinct type, narrowable with `is`/`is not`.

`Unset` is a singleton, falsy, with a `__repr__` of `Unset`. Its type is `typing.UnsetType`. Crucially, it is **not** publicly constructible — a runtime check in the call machinery raises `TypeError: cannot pass Unset explicitly; omit the argument instead` if a caller attempts `f(x=Unset)` on a parameter marked `?:`. This preserves the invariant: *if the parameter is `Unset` in the body, the caller did not supply it.*

(For parameters not marked with `?:`, `Unset` is just a regular singleton value and the check does not apply — useful as a sentinel default for code that opts out of the syntax.)

## Specification

### Grammar

The current parameter production in `Grammar/python.gram` (simplified):

```
param: NAME annotation?
param_with_default: param default
```

is extended:

```
param: NAME '?'? annotation?
param_with_default: param default
```

The `?` is permitted on positional, positional-or-keyword, keyword-only, and `*`-marker-trailing parameters. It is **not** permitted on `*args` or `**kwargs` (those are already inherently optional and bind to `()`/`{}` when omitted).

### Semantics

For a parameter `x?: T`:

1. **Implicit default.** If no `= default` is given, the implicit default is `Unset`. `def f(x?: int)` is callable as `f()`.
2. **Explicit default conflict.** Providing both `?` and `=` is a `SyntaxError`: `def f(x?: int = 0)` → `SyntaxError: parameter marked '?' cannot have an explicit default`. (Rationale: omissibility *is* the default specification. An explicit default with `?:` is either redundant or contradictory.)
3. **Body type.** Static type checkers MUST treat the in-body type as `T | Unset`. `is`/`is not Unset` narrows in the expected direction.
4. **Call-site type.** Static type checkers MUST treat the call-site type as `T`. `Unset` is not a permitted argument value for a `?:`-marked parameter.
5. **Runtime check.** Passing `Unset` (positionally or by keyword) to a `?:`-marked parameter raises `TypeError` at call time. Implementation lives in the C-level call machinery — `CALL`/`KW_NAMES` bytecode path.
6. **Introspection.** `inspect.Parameter` gains `is_unset: bool` (True when the parameter was declared with `?:`). `inspect.Parameter.default` returns `inspect.Parameter.unset` (a sentinel distinct from `Parameter.empty`) for these.
7. **`__defaults__` / `__kwdefaults__`.** `Unset` appears in the appropriate `__defaults__` / `__kwdefaults__` slots — existing introspection code that walks these tuples continues to work.

### Forwarding helpers (stdlib additions)

```python
from typing import Unset, is_unset
from inspect import drop_unset

def wrapper(name?: str, age?: int, **rest) -> None:
    # Filter Unset values out of an arbitrary mapping before forwarding.
    inner(**drop_unset({"name": name, "age": age, **rest}))

# Equivalent shorthand: ** with Unset-stripping semantics.
# Possible future syntax (not part of this PEP's hard requirement):
#     inner(**?{"name": name, "age": age})
```

`drop_unset(mapping)` returns a new dict with `Unset` values removed. It is a cheap, explicit, type-checker-friendly primitive that covers the dominant forwarding case without any new syntax beyond `?:`.

### Interaction with `*` and `/` markers

`?:` is independent of positional-only (`/`) and keyword-only (`*`) markers:

```python
def f(a: int, /, b?: int, *, c?: str) -> None: ...
```

All four parameter kinds (positional-only, positional-or-keyword, keyword-only, and `*`-marker-trailing) accept `?`. `*args` and `**kwargs` do not.

### Interaction with `dataclasses`

A `@dataclass` field whose annotation uses `?:` is treated as `field(default=Unset)` with `repr=True` and the standard field machinery. `dataclasses.replace(instance, x=Unset)` is now well-defined (and still rejected at the call boundary if `x` was declared `?:`; the `replace` machinery special-cases `Unset` for its own forwarding).

### Interaction with `typing`

- `typing.Unset` — the singleton value.
- `typing.UnsetType` — its type, for annotations.
- `typing.is_unset(x) -> bool` — convenience equivalent to `x is Unset`. (Provided primarily for use in `filter`/`map`/comprehension contexts where `is` doesn't compose.)
- `inspect.drop_unset(mapping)` — forwarding helper described above.

Type checkers (mypy, pyright, pyre, pytype) need updates to:
- Recognize the `?:` marker in function definitions.
- Apply the asymmetric body-vs-call-site typing rule.
- Narrow with `is`/`is not Unset`.
- Reject `Unset` as a call-site argument to `?:`-marked parameters.

### Stub files (`.pyi`)

`?:` is permitted in stubs with the same meaning as in `.py` files. Existing stubs that use `T = ...` to indicate "default exists but is not relevant" are unaffected; teams may migrate to `?:` where the parameter is genuinely omissible.

## Backwards Compatibility

- `?` is currently a `SyntaxError` in the parameter position. No existing valid Python is affected.
- `typing.Unset` and `typing.UnsetType` are new names. Any project that has defined its own `Unset` (a common ad-hoc sentinel name) continues to work — `from typing import Unset` is opt-in.
- `inspect.Parameter` gains a new attribute (`is_unset`) and a new sentinel (`Parameter.unset`). Existing code that compares `Parameter.default` against `Parameter.empty` continues to work; omissible-parameter introspection requires the new attribute.
- The new runtime check in the call machinery (rejecting explicit `Unset` arguments to `?:`-marked parameters) has zero cost for parameters not marked `?:`.

## Performance

- Function definition: one extra bit in the per-parameter flags. Negligible.
- Function call: an extra equality check (against the `Unset` singleton) per `?:`-marked parameter on calls that pass it. Skipped entirely when the argument is omitted (the common case). Benchmarkable but expected to be well under 1% on representative workloads; full numbers will accompany the reference implementation.
- Forwarding helper (`drop_unset`): single dict comprehension. Existing `**kwargs` patterns that filter sentinels manually are no slower.

## Reference Implementation

To be provided alongside the formal PEP submission. Sketch:

1. **`Grammar/python.gram`** — extend `param` to accept an optional `'?'` between `NAME` and the annotation.
2. **`Parser/Python.asdl`** — extend `arg` with a `bool is_unset` field.
3. **`Python/compile.c`** — emit a per-parameter flag into the code object's argument metadata.
4. **`Objects/call.c`** — at call time, for each `?:`-marked parameter that receives an argument, check `arg is Unset` and raise `TypeError` if so. Bind `Unset` as the default when the parameter is omitted.
5. **`Lib/typing.py`** — add `Unset`, `UnsetType`, `is_unset`.
6. **`Lib/inspect.py`** — add `Parameter.is_unset`, `Parameter.unset`, `drop_unset`.
7. **`Lib/dataclasses.py`** — recognize `?:`-marked annotations and translate to `field(default=Unset)` with appropriate `replace()` handling.
8. Type checker liaison: track adoption status in mypy, pyright, pytype, pyre.

## Rejected Ideas

- **Reusing `Optional[T]` / `T | None`.** Conflates "may be None" with "may be omitted." The whole point of this PEP is that these are not the same thing.
- **`def f(x: T = ...)` (Ellipsis as omission sentinel).** `...` is already overloaded; using it for "omitted" would break any function that legitimately accepts `Ellipsis` as a value (NumPy, typing stubs).
- **PEP 661 (Sentinel Values) alone.** Would standardize a `Sentinel` *value* but not the syntax for declaring omissibility, the runtime ban on explicit passing, or the forwarding semantics. PEP 661 is complementary, not a substitute. (If PEP 661 were accepted independently, `typing.Unset` could be defined *in terms of* its `Sentinel` factory — this PEP's contribution remains the syntactic marker and the call-machinery enforcement.)
- **`def f(?x: T)` (prefix marker).** Visually echoes `*args`/`**kwargs` but parses awkwardly and groups oddly with default values. Postfix-on-name (`x?:`) matches the dominant prior art (TypeScript, Swift) and reads naturally.
- **Allowing `def f(x?: T = default)`.** Considered and rejected: an explicit default contradicts the "omissible binds to `Unset`" rule, and silently overriding `Unset` with `default` makes forwarding semantics ambiguous. Users who want a regular default with sentinel-style detection can stick with `def f(x: T = default)` and detect `default` manually — they have not opted into `?:`.
- **Making `Unset` publicly constructible / passable.** Defeats the safety property. Callers who want a "no-op value" can use the existing tools (`None`, custom sentinels). `Unset` exists *only* as the body-side representative of caller omission.

## Open Issues

1. **`**?{...}` syntax for Unset-stripping unpacking.** Highly ergonomic for the forwarding case but adds another syntactic surface. Deferred unless the `drop_unset()` helper proves insufficient in practice.
2. **`match` statement integration.** Should `case Unset:` narrow? (Probably yes, for symmetry with `case None:`.)
3. **Pickling / serialization of `Unset`.** Singleton; pickle support trivial but needs to be specified explicitly.
4. **REPL / `repr` of `Unset`-defaulted parameters in `help()`.** Suggest displaying as `x?: T` rather than `x: T = Unset` to keep the source-level marker visible.
5. **Async / generator function parameters.** No semantic difference expected; explicit confirmation in the reference implementation tests.
6. **PEP 695 generic syntax interaction.** `def f[T](x?: T)` — should "just work"; needs grammar verification.

## References

- [PEP 661 — Sentinel Values](https://peps.python.org/pep-0661/) (Deferred). The closest prior art on the value side.
- [PEP 692 — Using TypedDict for kwargs](https://peps.python.org/pep-0692/). Related forwarding-precision concern; complementary, not overlapping.
- [PEP 750 — Template Strings](https://peps.python.org/pep-0750/). Cited only as recent precedent for a small targeted syntax addition that unblocks a wide library-side ecosystem.
- [RFC 7396 — JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7396). External protocol that requires the unset-vs-null distinction; today's Python code implementing it is forced into the sentinel-object pattern this PEP eliminates.
- TypeScript handbook: [Optional Parameters](https://www.typescriptlang.org/docs/handbook/2/functions.html#optional-parameters). Syntactic prior art for the `name?: T` marker (note: semantics differ — TS optional implies `T | undefined`, here it implies "may be omitted, with `Unset` runtime witness").
- `dataclasses.MISSING` — current de facto in-stdlib precedent for an omission sentinel.

## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.
