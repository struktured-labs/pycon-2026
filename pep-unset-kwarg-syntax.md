# PEP XXXX — Omissible Parameter Marker (`?:`)

| Field              | Value                                         |
| ------------------ | --------------------------------------------- |
| **Author**         | Carmelo Piccione <carmelo.piccione@gmail.com> |
| **Status**         | Draft                                         |
| **Type**           | Standards Track                               |
| **Created**        | 2026-05-17                                    |
| **Python-Version** | 3.17 (target)                                 |
| **Discussions-To** | discuss.python.org/c/peps                     |
| **References**     | PEP 661 (Sentinel Values, Final)              |

## Abstract

Introduce a keyword-only parameter marker `?:` that means *"the caller may omit this argument; if omitted, the parameter binds to the stdlib sentinel `inspect.UNSET`, distinct from every other value (including `None`)."* Callers cannot pass `inspect.UNSET` explicitly to a `?:`-marked parameter — the call raises `TypeError`.

```python
from inspect import UNSET

def patch_user(user_id: int, *, name?: str, email?: str, age?: int) -> User: ...

patch_user(1)                       # name, email, age all bound to UNSET
patch_user(1, name="alice")         # email, age bound to UNSET
patch_user(1, email=None)           # explicit clear; name and age bound to UNSET
```

## Motivation

Python functions cannot today distinguish "the caller did not supply this argument" from "the caller passed `None`." Every workaround — `_MISSING = object()`, `dataclasses.MISSING`, `**kwargs` + `in`, `Optional[T] = None` with documented "None means omitted" conventions — sacrifices at least one of: type-checker fidelity, runtime guarantee, or call-site ergonomics.

### The default-forwarding bug

The most pervasive symptom is what happens when a wrapper forwards its own arguments to a callee whose default differs:

```python
def f(x: int | None = 5):       # f's default is 5
    return x

def g(x: int | None = None):    # g's default is None
    return f(x=x)               # always passes x — overrides f's default

g()                             # returns None, NOT 5
```

Every wrapper, middleware, factory, and convenience constructor in Python that re-calls a target function with received keyword arguments has this bug latent in it. The current correct pattern requires either (a) per-argument sentinel checks before forwarding, or (b) `**kwargs` plus manual `in`-tests — both verbose, both easy to get wrong silently.

With this PEP:

```python
def g(*, x?: int | None):
    if x is UNSET:
        return f()              # don't forward; f's default of 5 wins
    return f(x=x)               # forward, including explicit None

g()                             # returns 5 — correct
g(x=None)                       # returns None — explicit clear
```

Or with the forwarding helper (see `inspect.drop_unset` below):

```python
def g(*, x?: int | None):
    return f(**drop_unset(x=x))
```

### Other affected surfaces

The same omission-vs-None distinction matters in PATCH-style HTTP endpoints (RFC 7396 JSON Merge Patch), ORM partial updates, `dataclasses.replace`, configuration overrides, and any code that forwards a subset of received keyword arguments to a downstream callee.

PEP 661 standardizes a `sentinel()` factory at the language level, but leaves the call-site ergonomics, the runtime guarantee (caller cannot fabricate the omission marker), and the forwarding helper unsolved. This PEP supplies all three, while deliberately scoping the new sentinel to its specific purpose.

## Specification

### Syntax

```
parameter := NAME '?' ':' annotation
```

The `?` marker is permitted on **keyword-only parameters only** (after `*` or `*args`). Mixing `?:` with an explicit default (`x?: int = 5`) is a `SyntaxError`.

```python
def f(*, x?: int): ...                    # OK
def f(*, x?: int = 5): ...                # SyntaxError
def f(x?: int): ...                       # SyntaxError (positional)
def f(*args, x?: int): ...                # OK (kw-only after *args)
```

(Positional `?:` is excluded from this PEP. See "Rejected Ideas".)

### The `UNSET` sentinel

A new module-level constant `inspect.UNSET` is added, constructed once at interpreter startup via `sentinel("UNSET", "inspect")` (PEP 661 C API). It is **deliberately not added to `builtins`** and **not re-exported from `typing`** — see Rationale.

```python
from inspect import UNSET
isinstance(UNSET, type(UNSET))   # True; type is `sentinel`
UNSET is UNSET                   # True
```

### Semantics

For a parameter `x?: T`:

1. **Implicit default.** If the caller omits the argument, `x` binds to `inspect.UNSET`.
2. **Runtime guarantee.** Passing any value `v` where `v is inspect.UNSET` explicitly to a `?:`-marked parameter raises `TypeError`. This is the property a library-only sentinel cannot provide.
3. **Body-side type.** Static type checkers SHOULD treat the in-body type of `x` as `T | type(inspect.UNSET)`. Narrowing via `is`/`is not UNSET` works as expected.
4. **Call-site type.** Static type checkers SHOULD reject `inspect.UNSET` as a call-site argument to a `?:`-marked parameter.
5. **Forwarding-safety check.** Static type checkers SHOULD reject expressions of type `T | type(UNSET)` as call-site arguments to parameters whose annotated type is `T` (without `UnsetType` in the union). This prevents the default-forwarding bug shown in Motivation — wrappers cannot accidentally pass a possibly-`UNSET` value through a parameter that doesn't expect it. The wrapper must either narrow first or use `inspect.drop_unset` to strip omitted values from the forwarded mapping.

### Forwarding helper

A new function `inspect.drop_unset(**kwargs) -> dict` is added:

```python
def drop_unset(**kwargs):
    """Return a dict containing only kwargs whose values are not UNSET.
    Intended for forwarding subsets of received omissible parameters."""
    return {k: v for k, v in kwargs.items() if v is not UNSET}
```

Canonical forwarding pattern:

```python
def wrapper(*, name?: str, age?: int, **rest):
    return inner(**drop_unset(name=name, age=age), **rest)
```

### Match statements

`UNSET` is matched the same way as any other dotted singleton (e.g., enum members). The dotted form is a value pattern; the bare form is a capture pattern, per PEP 634.

```python
import inspect

match name:
    case None:           ...     # explicit clear
    case inspect.UNSET:  ...     # omitted (dotted = value pattern, matches by ==)
    case str():          ...     # actual value
```

A bare `from inspect import UNSET` followed by `case UNSET:` will silently capture (per PEP 634). Use a guard (`case _ if name is UNSET:`) or stay with the dotted form.

## Rationale

**Why a marker, not just a sentinel default?** The runtime guarantee that `UNSET` cannot be passed explicitly is what makes `x is UNSET` in the body a reliable signal of caller omission. A library-only sentinel pattern (`def f(x=MISSING)`) cannot enforce this — any caller can pass `MISSING`.

**Why scope `UNSET` to `inspect`, not `builtins` or `typing`?** PEP 661 explicitly rejected the "single global sentinel" idea in favor of per-purpose sentinels. Placing `UNSET` in `builtins` (where it is always in scope) would create a de-facto global sentinel that other code would reuse for unrelated purposes — reopening a decided question. Placing it in `inspect` (the meta-introspection module) requires an explicit import per file, signals the limited intent, and keeps PEP 661's "per-purpose" discipline intact. Library authors who need a sentinel for a different purpose are expected to define their own via `sentinel('THEIR_NAME')`.

**Why keyword-only?** Positional `?:` introduces hard-to-resolve ambiguities with `*args` interleaving and forwarding. The dominant use case (partial-update APIs) is keyword-oriented anyway. Positional may be considered in a future PEP if demand emerges.

**Why `?:` and not `: T?`?** `T?` reads as "nullable T" in TypeScript / Kotlin / Swift / C#. Conflating "may be `None`" with "may be omitted" is exactly the ambiguity this PEP exists to resolve. Placing `?` between name and colon makes it a property of the parameter, not the type.

## Backwards Compatibility

`?` is a `SyntaxError` in this position today. No existing valid Python is affected. `inspect.UNSET` is a new attribute on `inspect`; no realistic naming conflict.

## Reference Implementation

A working CPython implementation against 3.16-main is available at [link to fork]. Total patch size: ~80 lines across:

- `Grammar/Tokens` — adds `QMARK '?'`
- `Grammar/python.gram` — adds `?` alternatives to keyword-only parameter productions
- `Parser/action_helpers.c` — desugars `?:` to `(annotated arg, default=Name(...))`
- `Python/ceval.c::initialize_locals` — rejects `UNSET` arguments to `?:`-marked parameters
- `Lib/inspect.py` — defines `UNSET`

The full CPython test suite passes on the patched build.

## Performance

Per-call overhead in `initialize_locals`: a constant-time check skipped entirely when the function has no `UNSET` defaults. Benchmarked with `pyperformance` (see reference implementation): [TODO: insert numbers].

## Rejected Ideas

- **Putting `UNSET` in `builtins` or `typing`.** Convenient at the call site but creates a de-facto universal sentinel, reopening the PEP 661 debate that already settled against this.
- **Positional `?:` parameters.** Out of scope. Forwarding semantics with `*args` are ambiguous. Future work if demand emerges.
- **`case UNSET:` as a bare-name value pattern.** PEP 634 reserves bare-name patterns for capture. The dotted form `case inspect.UNSET:` works today and is consistent with how every other singleton (enum members, `NotImplemented`) is matched. A general `__match_value__` protocol is more principled future work than special-casing `UNSET`.
- **`def f(x?: T = default)`.** An explicit default contradicts the "omissibility *is* the default" property. Use `def f(x: T = default)` if a non-`UNSET` default is wanted.
- **Allowing callers to pass `UNSET` explicitly.** Defeats the runtime guarantee that makes `is UNSET` in the body meaningful.
- **`T?` (TypeScript/Kotlin-style postfix).** Conflates "nullable" with "omissible." Python already has `T | None` for the former.
- **A new builtin predicate (`given(x)`, `was_passed(x)`).** Considered to avoid exposing any sentinel value to user code, but the bar for adding builtins is high enough that this trade — extra language surface for a single use case — does not clear it. `inspect.UNSET` is a thinner ask.
- **Call-site syntax for conditional forwarding (`f(x?=v)` or `f(x?)`).** Considered for symmetry with the parameter-side marker — would let wrappers write `f(x?=x)` instead of `f(**drop_unset(x=x))`. Rejected because (a) burning the `?` token in a second position doubles the token-budget cost, (b) the `inspect.drop_unset` helper covers the same ergonomics with one helper call per forwarding site, and (c) the type-checker forwarding-safety rule (Specification §5) catches the actual class of bugs the syntax would prevent. Call-site sugar is at best a convenience for the helper pattern; helpers are not painful enough to justify additional syntax.
- **Library-only solution (PEP 661 alone).** Provides the value but not the marker, the implicit default, the runtime guarantee, or the asymmetric type semantics. This PEP complements 661, not competes with it.

## Open Questions

- Type checker maintainer feedback on asymmetric body-vs-call-site typing.

## References

- [PEP 661 — Sentinel Values](https://peps.python.org/pep-0661/) (Final, Python 3.15)
- [PEP 634 — Structural Pattern Matching](https://peps.python.org/pep-0634/)
- [RFC 7396 — JSON Merge Patch](https://datatracker.ietf.org/doc/html/rfc7396)
- `dataclasses.MISSING` — stdlib precedent for an absence sentinel

## Copyright

Public domain / CC0-1.0-Universal.
