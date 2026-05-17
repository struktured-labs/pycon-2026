# Beyond Optional

**Speaker:** **Koudai Aono** — GitHub: [@koxudaxi](https://github.com/koxudaxi). Heavy-hitter in the Python typing / codegen ecosystem. Notable work:
- **[datamodel-code-generator](https://github.com/koxudaxi/datamodel-code-generator)** — the canonical Python tool for generating Pydantic models (and dataclasses, TypedDicts, msgspec, etc.) from JSON Schema / OpenAPI / GraphQL specs. ~3k+ stars; widely used in OpenAPI-first Python shops.
- **PyCharm plugins for Pydantic and Ruff** — IDE integration for two of the most-used Python ecosystem tools. Real authority on what works at the IDE/type-checker boundary.
- **Co-author of PEP 750** (template strings / t-strings — the new templated-literal syntax in Python). Means he's literally writing the typing-adjacent language spec, not just consuming it.
- **t-linter** — a linter specifically for PEP 750 t-strings.

Credibility implications for this talk: he's been staring at the gap between "what type signatures can express" and "what real API specifications actually require" for years, in the tool that converts the latter into the former. If anyone has seen every shape of "Optional doesn't carry enough information" pain at scale, it's him.

**Event:** PyCon 2026 (Long Beach)
**Date:** 2026-05-16

## Notes

- **Opening code on the slide — a PATCH endpoint with a hidden bug:**
  ```python
  def patch_endpoint(payload: dict) -> None:
      nickname = payload.get("nickname")
      bio      = payload.get("bio")
      patch_user(..., )  # both values passed in
  ```
- **The bug** *(speaker said he missed it the first time and only saw it after "switching sides" — likely meaning he moved from API-author to API-consumer perspective, or from writing the handler to debugging the production incident)*:
  - `.get("nickname")` returns `None` **in two distinct cases that the code can no longer tell apart**:
    1. The key was **absent** from the payload — i.e., the client didn't try to update this field. Correct behavior: *leave the existing value alone.*
    2. The key was **present with value `None`** — i.e., the client explicitly wants to clear this field. Correct behavior: *set the field to null.*
  - In a PATCH (partial update) handler these two cases must produce **different** outcomes. The code above collapses them into the same `None` and then passes it downstream, where `patch_user` has no way to distinguish "the client wants to clear this" from "the client didn't send this." Result: **the handler silently clears fields the client never touched** every time a partial update comes in without that field set.
- **"A gap" / "contract violation":**
  - The API contract (HTTP PATCH semantics, especially **RFC 7396 JSON Merge Patch** — where `null` means *delete* and *absent* means *leave alone*) requires distinguishing these two states.
  - The code's type signature (`payload: dict`) doesn't encode that distinction. `dict[str, Any]` flattens "absent" and "null-valued" into the same observation once you call `.get()`.
  - That's the *gap*: between **what the contract requires** (three states: absent · null · value) and **what the type signature can express** (two states after `.get()`: `None` · value). The handler is *physically incapable* of honoring the contract with this code shape — not because of a typo or off-by-one, but because the **type isn't expressive enough** to carry the information the contract depends on.
- **Why this is the perfect "Beyond Optional" opener:**
  - `Optional[str]` (i.e., `str | None`) only has two states. It cannot encode "absent." To honor PATCH semantics you need a **three-state shape**: `Missing | None | str`.
  - This is exactly the territory the talk title points at — `Optional[T]` is a Python typing primitive most devs reach for, and it is **not enough** for any API design where "didn't say" and "explicitly said null" need to remain distinguishable. The whole talk is presumably about the patterns that go beyond it.
  - **Predicted next moves the speaker may walk through** (just listening aids, wait for actual content):
    - **Sentinel pattern**: a singleton `UNSET` value and a type like `str | None | Literal[UNSET]`.
    - **Pydantic's `model_fields_set` / `exclude_unset=True`** — Pydantic models track which fields were explicitly set by the client at parse time, so you can recover the absent-vs-null distinction even though both look like `None` on the resulting object.
    - **`NotRequired` from `TypedDict`** (PEP 655) — lets you encode "this key may or may not be present" as a *type-level* statement.
    - **Tagged union / discriminated union** — `Absent | Null | Value[T]` explicitly modeled as a sum type.
    - Possibly: **`Annotated[T, ...]`** to attach the semantic distinction via metadata.
- **The proposed pipeline — three distinct typed stages, not one end-to-end shape:**
  > `payload → normalized update → domain model`
  - **Payload** — the raw wire input. JSON-shaped, untyped beyond `dict[str, Any]`. Three possible states per field (*missing · null · valued*) are all observable here if you read the dict carefully (e.g., `"key" in payload`).
  - **Normalized update** — an intermediate, *typed* representation of "what the client asked us to change." This is where the absent-vs-null-vs-set distinction gets **promoted into the type system** — so downstream code can't accidentally collapse it. Likely shape: a dataclass / Pydantic model with each updatable field typed as `Missing | None | T` (or via Pydantic's `model_fields_set` discipline + `exclude_unset=True`).
  - **Domain model** — the application's full representation of the entity. *Does not carry the "absent" concept* — a fully-realized user has a nickname or doesn't, no third state. The update is **applied** to the domain model by reading the normalized-update and mutating only the fields that were explicitly set.
  - **Why three stages, not two?** You could imagine going straight `payload → domain model` and "patching" in one step. The problem is that the domain model's type system can't express "I don't want to update this field" — every field is either set or not. Sticking the *update intent* into a separate typed object means you can pattern-match / branch on each field's three-way state cleanly, in code the type checker can verify. **The intermediate type exists to carry information that neither the wire format nor the domain model is natively shaped to hold.**
  - This is a **typing pattern, not a Python pattern** — it's the same shape you'd see in Rust (`Option<Option<T>>` for "is the field present, and if so is it null"), or in Haskell (`Maybe (Maybe T)`), or in TypeScript (`T | null | undefined`). nikkie's broader argument: the typing system can solve this, but only if you stop trying to make one type carry all the information.
- **Speaker's crisp imperatives — the talk's design rules:**
  - **"Stop conflating *missing*, *none*, *unset*."** — Three distinct concepts that the Python ecosystem habitually flattens:
    - **Missing** — the key wasn't in the payload at all. Semantic meaning: *the client didn't address this field.*
    - **None** — the key was in the payload with the literal value `null`. Semantic meaning: *the client wants this field cleared.*
    - **Unset** — a sentinel/default state inside your own data model (e.g., a Pydantic field whose default was used because the parser didn't see it). Semantic meaning: *internal — "we haven't yet recorded an intent here."*
    - Each maps onto a different *layer* of the pipeline (missing/none live at the wire boundary; unset lives in the normalized-update layer). Treating them as interchangeable is what produces the bug from the opening slide. The fix isn't "be careful with `None`" — it's **give each concept its own representation** and never let one stand in for another.
  - **"Model *boundary shape* before *domain meaning*."** — Design principle for the order of operations.
    - First: figure out **what shape the data has at the boundary** (the API/wire format). What states does each field have on the wire? Three? Two? With what discriminator?
    - Then: figure out **what those states mean in your domain**. (Map "field present with null" → "clear the field"; map "field absent" → "leave alone"; map "field with value" → "update to value.")
    - Inverting this order is the classic mistake — domain-driven design starts from the domain, but at API boundaries that approach **loses information the wire carries**. If your domain model says "a user has a nickname or doesn't" (two states), and you try to PATCH it with a wire payload that has three states, the wire's third state gets discarded silently. The boundary needs its *own* type that's faithful to the wire shape, *then* you translate into the domain.
    - This is the same idea as "make illegal states unrepresentable" (Yaron Minsky / OCaml community), applied at the API boundary: **the boundary type has to be expressive enough to represent every state the wire can produce**, including the awkward third one, even if your domain doesn't care.
  - Together these two rules generate the three-stage pipeline above. Rule 1 says don't flatten the three states; rule 2 says respect the wire shape at the boundary before translating. The pipeline is the operationalization of both rules.
- **More crisp imperatives from the slide:**
  - **"Narrow raw input before business logic uses it."** — Don't let `dict[str, Any]` flow past the boundary. The handler's *first* job is to convert the raw payload into a precisely-typed object; only the typed object reaches the domain code. This is **type narrowing as an architectural pattern**, not just a type-checker convenience.
    - Practical effect: business logic only ever operates on objects where every field's state is unambiguously knowable from its type. The three-state ambiguity (missing/null/value) is **resolved at the boundary** and represented faithfully in the typed object — never propagated as a `dict` for downstream code to interpret on its own.
    - Why this matters: every place in the codebase that handles a raw payload is a place where the missing-vs-null bug could recur. Narrowing once, at the boundary, gives you *one* place to get this right rather than N. Same logic as input validation in general, but applied at the *type* level rather than the *value* level.
  - **"Use a sentinel for safe partial updates."** — The concrete typing pattern. Define a singleton (commonly named `UNSET`, `MISSING`, `_unset`, or `_NOT_GIVEN`) with its own type, and use `T | None | Literal[UNSET]` to represent the three states.
    - `Optional[T]` (= `T | None`) has two states: handles "valued" vs "null." Can't represent "absent."
    - `T | None | Literal[UNSET]` has three states: handles "valued" vs "null" vs "absent" cleanly. The `UNSET` sentinel makes the third state a *first-class typed value*, not an implicit absence.
    - In Pydantic-land the equivalent is `model_dump(exclude_unset=True)` paired with `model_fields_set` — Pydantic tracks which fields were *actually set during parsing* and lets you recover that information later. Same idea, different surface syntax.
    - Other libraries' conventions: `httpx` uses an `UNSET` sentinel; `openai-python` uses `NOT_GIVEN`; `typing.NotRequired` (PEP 655, TypedDict) does it at the *type level* rather than the *value level*. All three solve the same problem with slightly different ergonomics.
  - **Slide question / setup:** *"Build the pattern with typing and …?"* — Cliffhanger. He's about to reveal the second ingredient. Predictions for what fills the blank (just listening aid):
    - `typing + dataclasses` (the simplest stdlib-only version — a `@dataclass` with sentinel-typed fields)
    - `typing + Pydantic` (using `model_fields_set` discipline)
    - `typing + TypedDict + NotRequired` (PEP 655 — the most "type-purist" version, no runtime construct needed)
    - `typing + msgspec` / `typing + attrs` / `typing + dataclasses.MISSING`
    - Given his work on **datamodel-code-generator** (which targets all of these), he may walk through several and compare.
- **Section 1: "The Real Bug."** — Slide header transition. He's structuring the talk into numbered sections, and section 1 is named *"The Real Bug,"* implying everything so far (the obvious-looking `.get()` mistake) was the **surface bug**, and he's about to reveal the **real** one underneath.
  - **Predicted framing of "the real bug":** it isn't that the developer used `.get()` carelessly. It isn't that `None` is ambiguous. It's that the **type signature `payload: dict` allowed the ambiguity to exist in the first place.** The bug isn't a careless mistake by a programmer — it's a **type-system failure**: a too-permissive boundary type that let semantically-distinct states (*missing · null · value*) be observationally identical inside the function body.
  - That reframing is the bridge into the "Beyond Optional" thesis: stop treating these bugs as "developer mistakes to be more careful about" and start treating them as "type signatures that aren't expressive enough." The fix is at the *type design* level, not at the *be-more-careful* level. Imperatives ("narrow before business logic uses it") follow from this — they're operationalizations of "make illegal states unrepresentable."
  - Watch for him to label the sections that follow: sections 2, 3, etc. probably walk through *the wrong fixes* (e.g., "just be careful with `None`", "add a runtime check") and then *the right fix* (sentinel + narrow + three-stage pipeline). Numbered structure is doing argumentative work — he's foreshadowing that "the real bug" is one of several layers.
- **Slide: *"No value" changes meaning by layer.*** — a table/diagram showing how the same concept ("the caller didn't send a value") has *different representations* at every layer of the stack the request traverses:
  - **Python function arguments** — "no value" = the caller omitted the argument entirely. Caller's call site says `f()` (no value) vs `f(x=None)` (explicit `None`). **Inside `f`, both look like `x is None` if `x` defaults to `None`.** Python has no built-in "this argument wasn't passed" sentinel that survives into the function body.
  - **HTTP** — "no value" = the field doesn't appear in the request body (or in query/form params). The HTTP wire format has a clean distinction here: a field present in the body is observably different from a field absent from the body.
  - **JSON** — "no value" = either the key isn't in the object **or** the key is present with the literal `null`. JSON has *two* representations of "no value," and they're observably distinct on the wire — but both flatten into Python's `None` after `json.loads()` + `.get()`.
  - **JSON Schema / OpenAPI** — "no value" distinguishes *required vs not-required* (is the field allowed to be absent?) from *nullable* (is the value allowed to be `null`?). Two orthogonal axes that the schema language *forces* you to model separately — and that **datamodel-code-generator** (his project!) has to translate into Python types that preserve the distinction.
  - **The killer observation:** every time you cross a layer boundary, the "no value" concept may **lose information** or **gain ambiguity** depending on which direction. Wire-format → Python `dict` is a *lossy* transition: three distinct on-wire states (*absent · null · valued*) collapse into two `dict`-level states (*key absent · key present*) after `.get()`. The bug from the opening slide lives in that lossy transition.
  - This slide is the *systemic* version of the talk's argument: the problem isn't a Python issue or an HTTP issue or a JSON issue — it's a **layer-translation issue** that recurs at every stack boundary. Solving it requires being deliberate about which representation you use at which layer.
- **The talk's main motivating question — single most important framing:**
  > *"Did the caller omit a value, or pass `value=None`?"*
  - This is the question every PATCH handler, every partial-update endpoint, every "merge these settings" function in Python is silently asking and silently getting wrong. **The talk exists because this question is unanswerable from inside the function with stdlib types alone.**
  - Why it's so insidious: at the Python call-site, `f()` and `f(value=None)` are clearly distinct keystrokes by the caller — they *mean* different things. But by the time you're inside `f`, both produce a parameter that satisfies `value is None`. The language collapses the call-site distinction into a single internal observation, and there's no language primitive to recover it.
  - **The same shape recurs at every layer:**
    - Python: `f()` vs `f(x=None)` — collapsed to `x is None` inside.
    - JSON: `{}` vs `{"x": null}` — collapsed to `data.get("x") is None` after `.get()`.
    - HTTP form: no `x=` field vs `x=` (empty value) — collapsed to `request.form.get("x")`.
    - It's the same bug at three layers. The Pythonic instinct to "just use `Optional[T]`" papers over all three identically — which is precisely why it produces silent wrong behavior at all three.
  - The whole "Beyond Optional" thesis is right here: **`T | None` cannot answer this question.** Until you have a type that can distinguish "the caller didn't address this" from "the caller addressed this with `None`," your code is structurally incapable of doing the right thing on a partial update. Being more careful won't fix it; only a richer type will.
- **Why `T | None` is not enough — formal version:**
  - `Optional[T] = T | None` has a domain of two states: *values of type `T`* + *the singleton `None`*. It can represent "field has a value" vs "field is null." It cannot represent "field is unaddressed."
  - The boundary needs a domain of **three states**: `Missing | None | T`. The `Missing` state has no representation in `Optional[T]`.
  - You can sometimes *infer* `Missing` from out-of-band information (e.g., "I called `payload.get(key)` and it returned `None`, but then I did `key in payload` and it returned False, so it must have been missing") — but at that point you've **left the type system** and are doing runtime introspection of a `dict[str, Any]`. The whole point of typing is to make the language enforce the distinctions for you; relying on `in` checks moves the burden back onto the programmer's diligence.
  - **Therefore: `Optional[T]` is one state short.** The talk's titular move is to add the missing third state (via sentinel, `NotRequired`, `model_fields_set`, etc.) and let the type system carry the information that `Optional[T]` lost.
- **Locating the bug precisely — "the bug starts on the boundary between HTTP and Python world, in his example."**
  - Important pinpointing: the patch-endpoint bug doesn't live inside `patch_user`, and it doesn't live in the JSON shape on the wire. It lives at the *transition point* — the moment the HTTP body crosses into Python and becomes a `dict`. That's where information loss occurs, and that's where the type signature `payload: dict` *allows* the loss to be invisible.
  - Reinforces "model boundary shape before domain meaning" — the boundary is a concrete location in your code, and it's the *highest-leverage place* to spend your typing budget. Get the type right at the boundary and downstream code inherits the precision; get it wrong at the boundary and no amount of downstream care can recover the lost distinction.
- **Solution #1 — Boundary models with TypedDict.** *"Data arrives as runtime JSON, not Python — use `TypedDict` for the user payload."*
  - **The pitch:** since the payload is *runtime* data (parsed from JSON, not constructed in Python), you don't need a runtime-validating object like a Pydantic model. You need a **type-checker-only** description of what shape the dict is supposed to have. That's exactly what `TypedDict` is — a static type for a `dict[str, Any]` with named, typed fields. **Zero runtime overhead**: TypedDict is purely a typing construct; at runtime, a TypedDict instance *is* an ordinary dict.
  - **Why TypedDict specifically:** it preserves the dict-ness of the wire data while attaching type information. You're not transforming the parsed JSON into something else; you're *naming* what the parsed JSON's shape is supposed to be, so the type checker can verify your accesses. Cheap, lightweight, faithful to "the data is JSON, treat it as JSON."
  - **The critical pairing — `NotRequired` (PEP 655):**
    ```python
    from typing import TypedDict, NotRequired

    class UserPatchPayload(TypedDict):
        nickname: NotRequired[str | None]
        bio:      NotRequired[str | None]
    ```
    - `NotRequired[T]` says **"this key may be absent from the dict."** Distinct from `T | None`, which says "this key is present but its value may be `None`." Stacked together — `NotRequired[str | None]` — encodes all three states: *absent · null · valued*. **This is the type system finally able to express what the PATCH wire format actually carries.**
    - PEP 655 (accepted into Python 3.11) is doing the load-bearing work here. Before PEP 655, you couldn't express "this key is optional in the dict" at the type level — all TypedDict fields were required by default, and the only flexibility was `total=False` (everything optional) or `total=True` (everything required). PEP 655 added per-field granularity. This is the language change that makes the whole pattern viable.
  - **How the handler uses it:**
    ```python
    def patch_endpoint(payload: UserPatchPayload) -> None:
        if "nickname" in payload:
            new_nickname = payload["nickname"]   # type: str | None  (could be None = explicit clear)
            # apply update — the user addressed this field
        # else: leave nickname alone — the user didn't address it

        if "bio" in payload:
            ...
    ```
    - The `if "nickname" in payload` check is now **type-aware**: modern type checkers (mypy 1.x, pyright) narrow the type inside the branch so `payload["nickname"]` is known to exist. The absent-vs-present distinction is checked statically, and forgetting the check is a *type error*.
    - Critically: there is **no `.get()`** in this handler. `.get()` is what flattened the distinction in the original buggy code; using `in` + indexed access keeps the three states distinct. The `if "key" in payload` pattern is the *type-safe* analog of `.get()` for partial-update handlers.
  - **Why TypedDict beats Pydantic *for this specific job***: Pydantic has its own way to solve this (`model_fields_set` + `exclude_unset=True`), and it's a fine tool — but Pydantic costs you a runtime model construction and a validation pass. If the *only* job at the boundary is "preserve the absent/null/value distinction so the handler can branch correctly," TypedDict + NotRequired does it for free at the type level. **Pay for runtime validation when you need it; use static types when you don't.** This is the principled version of "use the simplest tool that solves your problem."
  - **Foreshadowing other solutions** (he framed this as "one solution"): there are presumably more to come. Likely candidates:
    - **Pydantic with `model_fields_set`** — when you need runtime validation in addition to the absent-vs-null distinction.
    - **`@dataclass` with an `UNSET` sentinel** — when you want a typed object instead of a `dict`-like, but still no validation overhead.
    - **`msgspec.Struct`** — fast runtime decoding with sentinel support.
    - Each makes a different tradeoff between *typed precision*, *runtime cost*, *ergonomics at the call site*, and *integration with frameworks like FastAPI*. The fact that he's flagging "one solution" suggests he'll walk through several and let the audience pick.
  - **Reinforcing the "static-only" point:** *"the runtime value is still a plain dict."*
    - This is a major selling point and worth making louder. With TypedDict you get a complete typing-level description of the payload — every type-check on the handler verifies the shape, every IDE autocompletes the keys correctly, every mypy run catches missing-key/wrong-type bugs — and **at runtime, the object is indistinguishable from a `dict`**. No subclassing, no `__init__`, no validation pass, no serialization round-trip, no extra memory. `isinstance(payload, dict)` returns True. `json.dumps(payload)` works directly.
    - Contrast Pydantic: a `BaseModel` instance is *not* a dict; it's a separate object with field descriptors, validation hooks, and serialization machinery. That's appropriate when you want runtime validation, but it's overhead you don't need if the only goal is *typing-level precision*.
    - This is the **"pay only for what you use"** principle applied to typing: the type system is a static-analysis tool, so use it for static-analysis benefits, and don't drag runtime overhead in unless you specifically need runtime semantics. TypedDict + NotRequired is the minimum-cost solution for the missing/null/value problem.
  - **`total=False` — the alternative way to express optionality (coarser-grained than `NotRequired`):**
    ```python
    class UserPatchPayload(TypedDict, total=False):
        nickname: str | None
        bio:      str | None
    ```
    - `total=False` says **every key in this TypedDict may be omitted** — i.e., all fields are implicitly `NotRequired`. The default is `total=True`, where every key is implicitly `Required`.
    - **For a pure PATCH payload, `total=False` is actually the cleaner default.** Partial updates are *defined* by "every field is optional," so flipping the class-level switch once is more concise than annotating each field with `NotRequired[T]`. Saves visual noise when the whole class is uniformly optional.
    - **When you'd prefer per-field `NotRequired` instead:** mixed-shape payloads. If some fields are mandatory (e.g., the request always has a `user_id`) and others are optional (the actual update fields), you can't use `total=False` cleanly — you'd have to mark the required fields explicitly with `Required[T]`. At that point you've inverted the defaults and the class becomes harder to read. PEP 655's per-field markers (`NotRequired`/`Required`) let you express any mix without flipping defaults.
    - **Historical note:** `total=False` is older (PEP 589, Python 3.8); `NotRequired`/`Required` came later (PEP 655, Python 3.11). For a long time `total=False` was the *only* way to express any optionality at all in a TypedDict. The per-field PEP 655 syntax is the modern refinement, but `total=False` is still the right tool when everything is uniformly optional — like in a pure PATCH payload.
    - Subtle gotcha: `total=False` doesn't *recurse* — nested TypedDicts have their own `total` setting. If your patch payload contains a nested object that itself has required fields, the inner shape stays required-by-default unless you also pass `total=False` to the inner class.
  - **The realistic case — mixed-shape payloads.** *Speaker confirms: real payloads mix required and omittable keys.* Example called out from the slide: `bio: NotRequired[str]`.
    - In a typical update endpoint, *some* fields are mandatory (auth tokens, resource IDs, user identifiers) and *some* are omittable (the actual fields being updated). `total=False` is too coarse for that — it would make *every* key optional, including the mandatory ones, which weakens your type-check coverage on the required fields.
    - **`bio: NotRequired[str]`** (not `NotRequired[str | None]`!) is worth pausing on. Note what this type does NOT allow: it doesn't permit the value to be `None`. So the wire contract this expresses is:
      - The `bio` key may be **absent** (caller didn't address this field — leave alone), OR
      - The `bio` key is **present with a string value** (set bio to that string).
      - **There is no third "explicitly null to clear" state.** If your API doesn't support clearing bio (e.g., bio is allowed to be empty-string but never null), this is the right shape. If your API does support null-clears, you'd write `NotRequired[str | None]` instead.
    - This is the talk's design principle made tactile: **`NotRequired` and `| None` are orthogonal axes you compose deliberately, per field, to match the contract.** Four combinations per field:
      - `bio: str` — required key, must be string, no null. *(Required field.)*
      - `bio: str | None` — required key, must be present, value may be null. *(Required-but-nullable, e.g., for full PUT replace.)*
      - `bio: NotRequired[str]` — key may be absent, but if present must be a non-null string. *(PATCH semantics: address or don't, but never null-clear.)*
      - `bio: NotRequired[str | None]` — key may be absent, and if present may be a non-null string OR null. *(Full PATCH with explicit-null-clears.)*
    - Each combination expresses a different real-world contract. The naïve `Optional[str]` (= `str | None`) can only express the second of these four — and it's the *least useful* option for PATCH. The talk's "Beyond Optional" lesson made operational: **pick the combination that matches your wire contract**, don't reach for `Optional[T]` by reflex.
    - **Mental shortcut for choosing**: ask two questions per field, in order:
      - *"May the caller omit this key entirely?"* → if yes, wrap in `NotRequired[...]`.
      - *"If the caller sends this key, may they send `null` as a value?"* → if yes, include `| None` in the inner type.
      - Independent decisions, four combinations, one type per field that says exactly what the API does. **This is the typing discipline the rest of the talk is in service of.**
  - **The full example slide** — *worth lifting verbatim*:
    ```python
    class UserPayload(TypedDict, total=False):
        id:       Required[str]
        email:    Required[str]
        nickname: str | None
        bio:      str
    ```
    - **Walk-through, field by field:**
      - `id: Required[str]` — under `total=False`, you'd otherwise default to omittable; `Required[...]` overrides that. Must be present. Must be a string. No null.
      - `email: Required[str]` — same shape as `id`. Mandatory identifier.
      - `nickname: str | None` — under `total=False`, this is implicitly `NotRequired[str | None]`. **All three PATCH states allowed**: absent (leave alone) · null (explicitly clear the nickname) · string (set to value). API supports null-clearing for nickname.
      - `bio: str` — under `total=False`, this is implicitly `NotRequired[str]`. **Two PATCH states**: absent (leave alone) · string (set to value). **No null allowed.** API does not support null-clearing for bio — probably treats empty-string as "no bio" instead.
    - **Why this design is elegant** — note the **inverted defaults pattern**: `total=False` + explicit `Required` for the mandatory fields. This is the *opposite* convention from the standard `total=True` + explicit `NotRequired` for the optional fields. **For a PATCH-shaped payload, the inversion is actually clearer**, contra what I claimed earlier:
      - PATCH payloads are *defined* by partial updatability — the optional case is the *common* one, the required case is the *exception*.
      - Declaring `total=False` puts the "PATCH-ness" of the whole shape on the class line, where a reader sees it first.
      - The mandatory fields get visually marked with `Required[...]`, sticking out as the exceptions to the rule.
      - The optional fields stay clean — no per-field `NotRequired` wrapper noise.
    - **Compare the same shape with the opposite convention** (`total=True` default + per-field `NotRequired`):
      ```python
      class UserPayload(TypedDict):  # total=True
          id:       str
          email:    str
          nickname: NotRequired[str | None]
          bio:      NotRequired[str]
      ```
      Same semantics, more visual noise on the optional fields. For a payload where most fields are optional, the noise piles up. For a payload where most fields are required (e.g., a POST body), `total=True` + per-field `NotRequired` reads better. **Pick the convention based on which set — required or optional — is the larger one** in the payload. The four-combination matrix is the same; only the visual layout changes.
    - **Heterogeneous PATCH semantics across fields** — this example also makes a subtle point: *not all fields in a PATCH need the same wire contract*. `nickname` accepts null-clears; `bio` doesn't. The type signature carries that distinction; the handler can branch on it without runtime introspection. **This is the typing system actually doing real work** — encoding API-level decisions in a checkable form instead of trusting the developer to remember each field's quirks.
  - **The four-quadrant matrix — also presented as its own slide.** The speaker explicitly laid out the four combinations with field-name examples:
    | Field shape                          | Required? | Nullable? | What the wire allows                                    |
    |--------------------------------------|-----------|-----------|---------------------------------------------------------|
    | `id: str`                            | required  | no        | key present, string value, no null                      |
    | `nickname: str \| None`              | required  | yes       | key present, value is string OR null *(no omission)*    |
    | `bio: NotRequired[str]`              | optional  | no        | key may be absent; if present, must be a string         |
    | `xxx: NotRequired[str \| None]` *[4th — you noted missing it]* | optional  | yes       | **all three states**: absent · null · string |
    - **The 4th quadrant is `NotRequired[str | None]`** — the only remaining combination on the two-axis grid. It's the full PATCH-with-null-clears shape that the talk's opening bug was structurally unable to express. Worth marking as the one to reach for whenever your wire contract supports "address-or-don't, and if address-then-may-be-null" semantics.
    - **The matrix is the talk's central artifact.** Every other piece (the opening bug, the layer-translation diagram, the boundary-models-with-TypedDict prescription, the worked example) leads to or follows from this grid. If you remember nothing else, remember: **`Required` ⊥ `nullable` — they're orthogonal axes, and Python's typing system can express all four cells if you compose `Required`/`NotRequired` with `T`/`T | None` deliberately.**
    - **The flat naïve `Optional[T]`** only addresses *one cell* of this grid (required + nullable). Reaching for it as a default is, in this framing, picking one of four cells without checking which one your API actually needs. The "Beyond Optional" thesis: **stop reaching for `Optional[T]` as a default and instead pick the cell that matches your wire contract per field.**
  - **How `total=...` interacts with the four-quadrant matrix** *(slide showed both class-level settings side by side):*
    - The class-level `total` setting **picks the default** for required-ness; per-field `Required[...]`/`NotRequired[...]` markers **override** that default. The four cells of the grid are reachable under either convention, but the syntax flips.
    - **Under `total=True` (the default)** — required is the default; mark optional fields explicitly:
      ```python
      class UserPayload(TypedDict):  # total=True implicit
          id:       str                            # required, no null
          nickname: str | None                     # required, nullable
          bio:      NotRequired[str]               # optional, no null
          xxx:      NotRequired[str | None]        # optional, nullable
      ```
    - **Under `total=False`** — optional is the default; mark required fields explicitly:
      ```python
      class UserPayload(TypedDict, total=False):
          id:       Required[str]                  # required, no null
          nickname: Required[str | None]           # required, nullable
          bio:      str                            # optional, no null
          xxx:      str | None                     # optional, nullable
      ```
    - **Same four-cell semantic grid in both versions** — only the *syntax cost per cell* differs. `total=...` is purely about which convention is cheapest to express; it doesn't add or remove any of the four possible shapes.
    - **The subtle gotcha the slide is implicitly warning about:** the same field declaration `id: str` means **different things** depending on the class's `total` setting. Under `total=True`, it's required + no null. Under `total=False`, it's optional + no null. **Without seeing the class line, you can't read a TypedDict field correctly.** This is a real readability hazard if your code uses both conventions inconsistently across the codebase.
    - **Practical takeaway:** pick *one* convention per project (probably `total=False` for PATCH-shaped payloads, `total=True` for everything else) and stick with it. Mixing the two leaves every reader doing extra mental work to figure out which default is in effect for any given class.
  - **`ReadOnly[T]` for boundary safety — PEP 705 (Python 3.13).** Adds a *third orthogonal axis* to the typing grid: mutability.
    - **What PEP 705 introduced:** `ReadOnly[T]` as a TypedDict field marker. A `ReadOnly` field is *one the consumer must not mutate*. The type checker will flag any attempt to assign to it after the dict is constructed. **Pure static enforcement** — at runtime the dict is still an ordinary mutable Python dict; the prohibition lives only in the type system.
    - **Why this matters at the boundary:** the payload coming in from the HTTP layer is *shared state*. If your handler does something like `payload["bio"] = payload["bio"].strip()` to clean up input, you've now mutated the dict that came from the parser — and any other code reading from the same dict (logging, metrics, audit trails) now sees the cleaned-up version, not the original. **You've silently rewritten history.** Worse, if the parser cached or memoized the parsed payload, subsequent reads see your mutation too.
    - **The `ReadOnly`-at-boundary discipline:** wrap every field in the boundary TypedDict with `ReadOnly[T]`. The compiler now refuses any in-place mutation. If you need to transform a field, you do it by **constructing a new typed object** (the *normalized update* layer in the pipeline) — never by mutating the input. This is the static-typing analog of "treat function parameters as immutable" or Python's preference for immutable data at boundaries.
    - **Example:**
      ```python
      from typing import TypedDict, Required, ReadOnly

      class UserPayload(TypedDict, total=False):
          id:       ReadOnly[Required[str]]
          email:    ReadOnly[Required[str]]
          nickname: ReadOnly[str | None]
          bio:      ReadOnly[str]
      ```
      Now `payload["bio"] = ...` is a type error. Handler code is *forced* to build a new typed object (the normalized-update layer) and let the original input dict travel untouched to any downstream consumers.
    - **Three orthogonal axes now:**
      - *Required vs. NotRequired* (key present or absent)
      - *Nullable* (`T` vs. `T | None`) (value may be null)
      - *ReadOnly vs. writable* (consumer may mutate, or may not)
      - 2 × 2 × 2 = **eight cells** in the full grid. Each axis is independently composed by wrapping types. Real boundary types use all three.
    - **Boundary discipline at full strength:** if you adopt all three axes consistently, the boundary TypedDict for any external input becomes a *complete static contract* — it expresses what the caller may send, what the caller may legally omit, what values may be null, and that the consumer is forbidden from rewriting any of it. **All of that is static-only, zero runtime cost, and the type checker enforces it.** This is the "static-only" promise of TypedDict at its strongest — you're getting close to the type-safety guarantees of a language like Rust at the API boundary, in plain Python, with no Pydantic-style runtime validation overhead.
    - **PEP 705 acceptance/availability:** accepted, included in Python 3.13. If you're on 3.12 or earlier, `ReadOnly` is available via `typing_extensions`. Mypy ≥ 1.11 and recent pyright versions support it.
    - **Connects to nikkie's broader ecosystem work:** as a `datamodel-code-generator` author, the eight-cell grid is exactly the design space he has to *emit code into*. When OpenAPI says "this field is `nullable`, `required`, and `readOnly`," the generator has to produce a Python type that captures all three axes faithfully. PEP 705 closes the last gap that previously had no Python-level representation — before 705, the `readOnly` flag in OpenAPI had to be smuggled into Python via comments or convention, because the type system literally could not say it. This is the kind of language change that has *direct* downstream consequences for codegen tools and the contracts they can preserve end-to-end.

- **Recap slide on the TypedDict pattern** *(speaker walked through ~5 bullets; user caught 2 verbatim and noted 3 were missed; reconstructing the missed three based on the talk so far):*
  - ✅ **Captured: TypedDict is a *static* contract, not a runtime one.** The runtime object is still a plain `dict`. All the type information evaporates at runtime — it lives only in the type checker. Zero overhead, full static enforcement.
  - ✅ **Captured: `NotRequired[T]` models missing keys.** The PEP 655 marker that closes the absent-state gap `Optional[T]` couldn't express.
  - 🔄 **[reconstruction — likely missed bullet]**: **`Required[T]` to opt back into mandatory under `total=False`**, or equivalently, **`total=False` to flip the default** when most of your fields are optional. The class-level setting and per-field markers compose to reach any of the four (Required, NotRequired) × (T, T | None) cells.
  - 🔄 **[reconstruction — likely missed bullet]**: **`T | None` is for *nullable values*, kept separate from the *missing-key* axis.** The two are orthogonal — `Optional[T]` (= `T | None`) only addresses the nullable dimension; you compose it with `NotRequired[...]` to express both axes independently per field.
  - 🔄 **[reconstruction — likely missed bullet]**: **`ReadOnly[T]` (PEP 705) for boundary safety** — prevents handler code from mutating the input dict, forcing transformations to produce *new* typed objects rather than rewriting the boundary payload in place. Third orthogonal axis after required-ness and nullability.
  - *(If these reconstructions are off, the most-likely alternatives are: "separate missing from null from unset" as one bullet, the four-quadrant matrix as another, and "use `in` not `.get()` to access NotRequired fields" as the third. Flagging all of these in case any rings closer to what was on the slide.)*

- **New section: `default=None` subtleties with PATCH commands.** Section transition — having shown the TypedDict-at-boundary solution, he's now moving to the *Python-side model* (dataclass / Pydantic) and the parallel trap that lives there.
  - **The antipattern being warned against:**
    ```python
    class UserUpdate(BaseModel):     # or @dataclass — same problem
        nickname: str | None = None
        bio:      str | None = None
    ```
    - Looks innocent. Every field is `Optional[str]` with a default of `None`. This is the canonical Python idiom for "optional parameter." **It is exactly the wrong shape for PATCH input.**
  - **Why `default=None` is uniquely toxic for PATCH:**
    - The default value `None` is **observationally indistinguishable** from "the user explicitly sent `null`." Once the model is constructed, `instance.nickname is None` is True in *both* of these cases:
      - The wire payload had no `nickname` key at all → the default kicked in → field is `None`.
      - The wire payload had `"nickname": null` → field is `None`.
    - The default *creates* the ambiguity the wire format was trying to preserve. By time the handler reads `instance.nickname`, the absent-vs-null distinction has been collapsed by the model's own default value. **The model is the lossy translator** — not the JSON parser, not the `.get()` call, but the *dataclass/Pydantic model itself*, via its default.
    - Worse: this is the *default* idiom Python developers reach for when writing an "optional update" model. Pydantic tutorials, FastAPI examples, and most blog posts all recommend `field: Optional[T] = None`. **The community-standard pattern is the bug.**
  - **The fix — sentinel default instead of `None`:**
    ```python
    from typing import Final
    from enum import Enum

    class _UnsetType(Enum):
        UNSET = "UNSET"
    UNSET: Final = _UnsetType.UNSET

    class UserUpdate(BaseModel):
        nickname: str | None | _UnsetType = UNSET
        bio:      str | None | _UnsetType = UNSET
    ```
    - Now `instance.nickname` has three distinguishable states at the type level:
      - `is UNSET` → field was absent on the wire (caller didn't address it).
      - `is None` → field was present with `null` on the wire (caller wants to clear).
      - `isinstance(_, str)` → field was present with a value (caller wants to set).
    - Handler code can branch on these three cleanly: `if instance.nickname is not UNSET: apply(nickname=instance.nickname)`. The type checker can verify exhaustiveness; the runtime can answer the question "did the caller address this?" without out-of-band introspection.
    - **The `Enum` singleton trick** (instead of a bare `class _Unset: pass` instance) gives you a static `Literal[_UnsetType.UNSET]` type that narrows cleanly in conditionals — the type checker can prove "after this `if is not UNSET` check, the value can only be `str | None`." Cleaner than ad-hoc sentinel classes.
  - **The Pydantic-native alternative — `model_fields_set` + `exclude_unset=True`:**
    ```python
    class UserUpdate(BaseModel):
        nickname: str | None = None
        bio:      str | None = None

    def patch_endpoint(update: UserUpdate) -> None:
        if "nickname" in update.model_fields_set:
            apply(nickname=update.nickname)   # None here means "clear"
        # else: caller didn't address nickname
    ```
    - Pydantic tracks at parse time *which fields were actually present in the input*. The set is available as `update.model_fields_set`. Checking membership recovers the absent-vs-null distinction even though the field types still look like the buggy `Optional[T] = None` shape.
    - Trade-offs vs the sentinel approach:
      - **Sentinel: type system carries the distinction; can't forget the check.** The compiler reminds you the value might be `UNSET`. You can never "just use the field" without acknowledging the three states.
      - **`model_fields_set`: type system *doesn't* carry the distinction; you must remember to check.** A junior dev who writes `if update.nickname is not None: apply(...)` introduces the same bug back — Pydantic can't help, because the type still says `str | None`. The distinction exists only as a runtime attribute on the model, not in the field's type.
      - Sentinel is *safer*. `model_fields_set` is *more convenient* if you don't want to wrap every type in `| _UnsetType`. Pick by which mistake you're more worried about: forgetting to handle `UNSET` (sentinel forces you to) vs. forgetting to check `model_fields_set` (no compile-time pressure).
  - **The conceptual move this section makes:** the talk has now solved the missing/null/value problem at *both* of the relevant abstraction layers:
    - **Wire layer**: TypedDict + `NotRequired` + `Required` + `ReadOnly` (per the earlier section).
    - **Object layer**: sentinel-typed fields + sentinel defaults (this section), OR Pydantic's parse-time-tracking model_fields_set.
    - Either layer's solution alone is insufficient if the *other* layer collapses the distinction. **You need both**: TypedDict-shaped boundary parse, then either sentinel-typed normalized-update objects or Pydantic with `exclude_unset` discipline downstream. The three-stage pipeline from earlier (`payload → normalized update → domain model`) is what knits these layers together.

- **Speaker explicitly labels this a "workaround": *"UNSET marker."*** Worth pausing on the framing — he didn't call it "the solution," he called it the **workaround**. That word choice is doing real argumentative work.
  - **Why he's flagging it as a workaround, not a solution:** the `UNSET = object()` (or `_UnsetType.UNSET` enum) sentinel is ad-hoc, project-local, and incompatible across libraries. Every project that needs the three-state distinction invents its own UNSET; **`httpx`'s `UNSET`** is not **`openai`'s `NOT_GIVEN`** is not **`google-api-core`'s `_MethodDefault`** is not **your-project's `UNSET`**. Each is a different singleton at a different import path with a slightly different type. They can't interoperate; a function that takes `httpx`'s `UNSET` can't accept yours. Every project that needs this pattern *reinvents the same wheel*, slightly incompatibly.
  - The "right" solution from a language-design standpoint would be a **standard library sentinel** for "this argument/field was not provided" — something like `typing.Unset` or `typing.Missing` as a single canonical singleton with a single canonical type. That doesn't exist yet. Until it does, you patch the gap with a project-local UNSET, knowing you can't quite share it.
  - **Sharp meta-observation about the speaker:** nikkie is a **PEP author** (PEP 750, t-strings). PEP authors are *language designers*. From that perspective, telling people "use a project-local sentinel as a workaround for a missing language primitive" must be *deeply unsatisfying* — these are exactly the kinds of papered-over gaps that PEP authors exist to close. The framing "workaround" (not "solution") is leaking that dissatisfaction.
  - **Predicted unstated subtext / Q&A direction:** there's a strong chance nikkie either has thoughts on, or would welcome thoughts on, what a *standard library sentinel for "absent"* should look like. The PEP 655 (`NotRequired`) and PEP 705 (`ReadOnly`) precedents suggest the typing community is actively closing exactly these gaps via PEPs; a future PEP for `typing.Unset` (or similar) would be a natural next step. Could be a great Q&A prompt: *"Is there an active proposal to make `UNSET` a standard library construct, or are we stuck with per-library sentinels for the foreseeable future?"*
  - **Pattern recognition for the rest of the talk:** if "TypedDict + NotRequired + Required + ReadOnly" was framed as the *real* solution at the wire-format layer (using language primitives), and "UNSET marker" is framed as the *workaround* at the object layer (using a hacked singleton), the asymmetry itself is the argument. **The wire layer is solved by Python's typing system; the object layer isn't.** Worth watching whether nikkie closes the talk by pointing at that asymmetry as future work — it would be very on-brand for a PEP author to wrap up with "and here's the language hole that's still open."
  - **The actual code on the slide — the workaround in full:**
    ```python
    class _UnsetType: pass
    UNSET: Final = _UnsetType()
    ```
    - Three lines of declarations to create *one sentinel value*. A type for the singleton (`_UnsetType`), an instance of that type (the `()` call constructs the singleton), and a `Final` annotation so the type checker treats `UNSET` as a single fixed value (otherwise mypy would type it as `_UnsetType` generally and would not be able to narrow `x is UNSET` reliably).
    - Then every type that admits the absent-state has to bolt `| _UnsetType` onto the union: `str | None | _UnsetType`. Every field gets visually polluted with the workaround's type, repeated everywhere it appears in the codebase.
    - And `_UnsetType` has *no singleton enforcement* — nothing prevents `another_unset = _UnsetType()` from existing as a second instance that breaks `is`-equality with the canonical `UNSET`. If you're paranoid you'd add `__new__` plumbing to make it a true singleton; more lines, more cost.
  - **Verbatim user reaction:** *"wow the boilerplate is awful."*
    - Honest, correct, and exactly the speaker's own dissatisfaction surfacing in the audience. Three lines to declare a single language-missing concept, then `| _UnsetType` polluting every type union forever. **The pattern works but the ergonomics are openly bad.**
    - Contrast a hypothetical first-class form (this doesn't exist, but it's the obvious destination):
      ```python
      # Hypothetical Python with first-class Unset:
      from typing import Unset

      class UserUpdate(BaseModel):
          nickname: str | None | Unset = Unset
          bio:      str | None | Unset = Unset
      ```
      No `_UnsetType` declaration. No `Final` annotation. No singleton dance. One name imported from `typing`, used everywhere, statically narrowable by every checker. **That's what a language solution looks like, and it's what the current workaround openly isn't.**
    - This is the moment the talk's underlying argument crystallizes most visibly: **the bug from slide 1 is in fact a *language-design hole*, not a developer-discipline issue.** The recommended workaround is itself ugly enough that any honest observer can see the language *should* close this gap — and nikkie, as a PEP author, is presumably the right person to be saying so from the stage at PyCon.

- **Plot twist — Python 3.15 has a built-in `sentinel()` function.** The talk just landed exactly where the previous note's "hypothetical first-class form" prediction was pointing. **The language closed the gap.**
  - **What's new:** Python 3.15 ships a `sentinel()` factory function (the user reports it's on `builtins`, so no import needed; this is consistent with **PEP 661** — Sentinel Values — which proposed exactly this construct after years of discussion in the Python community).
  - **Likely shape of the new API** *(reconstructing from the user's live reporting + PEP 661's design)*:
    ```python
    # Python ≥ 3.15:
    UNSET = sentinel("UNSET")

    class UserUpdate(BaseModel):
        nickname: str | None | type(UNSET) = UNSET
        bio:      str | None | type(UNSET) = UNSET
    ```
    Or, if 3.15's API exposes a usable type for the sentinel directly:
    ```python
    UNSET: Sentinel = sentinel("UNSET")
    ```
    The point is **one line replaces three**, the singleton-enforcement and `Final` semantics are handled by the language, and the type that goes in unions is a standard, importable, single-vendor name — not a project-local `_UnsetType` class.
  - **What this fixes from the previous workaround:**
    - **No more `class _UnsetType: pass` boilerplate.** The sentinel factory creates the class for you internally.
    - **No more `Final` annotation needed.** The factory's return type is already a singleton.
    - **Cross-library compatibility, finally.** If `httpx`, `openai-python`, your-project, and `google-api-core` all switch to `sentinel(...)` (or one shared canonical instance — TBD on whether 3.15's API supports a *single* universal absent-marker or many named sentinels), the inter-library interop problem dissolves. Either way it's an order of magnitude less painful than today's per-project ad-hoc singletons.
    - **Singleton enforcement is the runtime's responsibility, not yours.** No need to add `__new__` plumbing to prevent accidental second-instance construction.
  - **Why this is the speaker's payoff slide:** nikkie has just walked the audience through *the bug*, *the workaround's awfulness*, and now *the language-level fix that just landed in 3.15*. As a PEP author this is the arc he'd most want to tell — a real bug in production code, a community workaround that smells, and a CPython release that finally puts a clean primitive in the language to replace the smell. **The "workaround → first-class language feature" trajectory is the PEP-author hero story**, and this talk is one telling of it.
  - **Caveat from my end** *(not the speaker's)*: I'm relying on the user's live reporting that 3.15 has `sentinel()` on `builtins`. Worth verifying the exact API shape post-talk — `builtins.sentinel` vs `typing.Sentinel` vs `dataclasses.MISSING`-style sentinel-per-purpose; whether sentinels created with the same string are `is`-equal or are fresh singletons; whether there's a single canonical "absent" sentinel or you're meant to create your own. PEP 661 (the proposal) gave it a value-creation-by-name API (`sentinel("MISSING")` returns the same singleton for the same name, like `intern`), but the final accepted form may differ.
  - **Practical migration takeaway:** if you're on Python ≥ 3.15, you can drop the `class _UnsetType: pass` + `UNSET: Final = _UnsetType()` pattern and replace it with one `sentinel(...)` call. Type unions still need to mention the sentinel's type in every PATCH-shape field, but the *declaration cost* per project drops from 3 lines to 1, and the *cognitive cost* of "is this the same singleton across libraries" goes away. If you're on < 3.15, the workaround in the previous note still applies — and the `typing_extensions` backport (if any ships) would be the right way to get 3.15-shaped sentinels in older code.
  - **Vindicates the talk's title.** *"Beyond Optional"* turns out to mean, in part, "use a sentinel for the absent state — and as of 3.15, the sentinel is finally a standard-library primitive instead of a per-project hack."

- **My pushback / cross-language observation** *(triggered by the 3.15 `sentinel()` reveal — keeping the meta-thread alive):*
  - **3.15's `sentinel()` solves the boilerplate, but Python *still* doesn't have a first-class concept of "an unset argument" at the function-signature level.** The sentinel approach is value-space machinery: a magic value gets passed (or not), and inside the function you check `if x is UNSET`. The language itself doesn't *know* the argument is optional-with-absence-semantics — you're still encoding optionality by convention.
  - **Languages that *do* have this at the type level**, for comparison:
    - **OCaml.** Write `let f ?(foo : 'a option) = ...` and `foo` is a real optional-keyword-argument whose type *inside the function* is `'a option`. The caller's "I didn't pass foo" is preserved through the call into the parameter's type as `None`; "I passed foo with value x" becomes `Some x`. The language preserves the distinction at the call boundary — **no sentinel needed because the type itself carries the absence.**
    - **TypeScript** is the closest mainstream analog. `function f(x?: T)` makes `x` an optional parameter whose type *inside the function* is `T | undefined`. TypeScript also has `null` (distinct from `undefined`), so it can naturally express the three-state PATCH problem: a field typed `T | null | undefined` has all three states with no workaround. JSON parsing into TypeScript types preserves the absent/null distinction by construction.
    - **Scala / Kotlin / Swift / Rust** all express optionality at the type level (`Option[T]` / `T?` / `Optional<T>` / `Option<T>`), though most don't independently distinguish "absent" from "null" the way TypeScript and OCaml do.
  - **What Python is still missing, even post-3.15:**
    - A way for the *function signature* to say "this argument is optional, and inside the body you can ask `was_provided(x)` to know whether the caller passed it." Sentinels are a value-space stand-in for this. They work, but they leak everywhere: the type union has to mention the sentinel type, the caller has to know which sentinel to import, the function body has to write `is UNSET` checks. The language doesn't natively *understand* that this is an "absent argument" pattern — it just sees a default value that happens to be magic.
    - The OCaml-style "optionality baked into the function signature" approach would look something like:
      ```python
      # Hypothetical syntax — does NOT exist in Python 3.15:
      def patch_endpoint(*, nickname?: str | None, bio?: str) -> None:
          if nickname is Unset:  # or: if "nickname" not in passed_args:
              ...
      ```
      where `?` in the parameter list explicitly marks the parameter as "may be omitted, and the omission is observably distinct from any value the caller could pass."
    - **Why this would be cleaner than sentinels:** the type `str | None` for `nickname` would *not* include the sentinel type — the absence would be a property of the *call*, not a value in the parameter's type. The mental model becomes: types describe values; argument-presence is metadata about the call. Right now Python conflates them (the "absence" has to be encoded as a magic value of a magic type that pollutes the parameter's type union forever).
  - **Why this gap is unlikely to close soon:** Python's design strongly favors "everything is a value." Magic absence-of-value as a call-level concept would require either (a) extending the function-call protocol to expose "which args were passed" (which it already does via `inspect.signature(...).bind()` introspection, but not in a static-type-friendly way), or (b) some kind of dependent typing where the parameter's type changes based on whether it was passed. Neither is in keeping with Python's design philosophy.
  - **What this means for the talk's title in retrospect:** *"Beyond Optional"* lands at "use a richer type union with a sentinel value for absent." It's the best Python can do with its current language model. It's a *real* improvement over `Optional[T]`. But it's still working *around* a conceptual gap that other languages don't have — and that gap is unlikely to close. **The full destination isn't `Optional[T] + UNSET`; the full destination is a Python where "did the caller pass this argument" is a question the language exposes statically.** That's a different (and bigger) PEP than 661 / 655 / 705, and isn't on anyone's roadmap that I'm aware of. Worth flagging as the *real* future-work direction underneath the sentinel patch.

- **The application-site pattern — the second source of ugliness.** Speaker now shows what consuming an `UNSET`-typed field looks like in practice:
  ```python
  # for each field in the update:
  return current if update is UNSET else update
  # where  update: T | _UnsetType
  ```
  - Reads as: "if the caller didn't address this field, return the current value; otherwise return what they sent." This is the **apply** step — the bridge from *normalized update* into *domain model* in the three-stage pipeline.
  - User's verbatim reaction: *"still looks kinda ugly imho — I hope he offers something better."* — and the reaction is exactly right.
  - **Why this is the *second* source of ugliness:**
    - The declaration-site ugliness (the `class _UnsetType: pass` + `UNSET: Final = _UnsetType()` boilerplate) was solved by 3.15's `sentinel()`.
    - The *application-site* ugliness — having to write `current if update is UNSET else update` for **every field of every PATCH-able model** — is **not** solved by 3.15. You still have N ternaries for an N-field model, scattered across whatever function applies the update. **A model with 20 fields has 20 of these.** Forgetting one is a silent bug (the corresponding field never gets updated). Adding a new field requires touching the apply-function in another file.
    - This pattern doesn't *scale* — and the user's instinct that "there must be something cleaner" is correct.
  - **Cleaner alternatives I expect the speaker to offer next** *(if he doesn't, these are the natural follow-ups):*
    - **Pydantic's `model_copy(update=...)` + `model_dump(exclude_unset=True)`.** Pydantic-native idiom: convert the update model to a dict that omits unset fields, then pass it as the `update` to `model_copy` on the existing instance. One line of glue, scales to N fields automatically:
      ```python
      new_user = existing_user.model_copy(update=update_payload.model_dump(exclude_unset=True))
      ```
      No per-field ternaries; the field-wise merge is library-handled.
    - **`dataclasses.replace()` with a dict comprehension.** stdlib analog for dataclass-based models:
      ```python
      new_user = dataclasses.replace(
          existing_user,
          **{k: v for k, v in vars(update).items() if v is not UNSET}
      )
      ```
      Slightly more boilerplate than Pydantic's version, but stdlib-only.
    - **A generic `apply_update(current, update)` helper.** Write the ternary once, parameterize over the field set, never write it again. A few-line utility module that any project can adopt.
    - **`msgspec.Struct` with `__post_init__` / `merge`** patterns, if you're optimizing for runtime speed.
  - **What the user's reaction is really asking for:** *the merge step should be declarative, not per-field.* You shouldn't have to write a ternary for each updatable field — the data model itself should know which fields are PATCH-shaped, and a single library call should walk the model and apply the present-only updates. That's what Pydantic's `model_copy + exclude_unset` provides; that's what the rest of the talk should probably head toward. **If the talk ends without naming this declarative-merge step, that's the biggest piece of unfinished business in the prescription.**
  - **Predicted next slide:** either (a) Pydantic-based merge, (b) a "putting it all together" example showing the full pipeline end-to-end, or (c) the speaker acknowledging the per-field ternary is awful and pointing at a higher-level abstraction. Worth watching for which framing he picks.

- **Section transition — moving on to Section 4: "Safe extraction."** *(Sections 2 and 3 we encountered without explicit numbering; presumably they were the TypedDict-at-boundary work and the UNSET-marker / `default=None` discussion. The numbered structure is: 1. The Real Bug → 2. TypedDict at the boundary → 3. UNSET markers for the object layer → 4. Safe extraction.)*
  - **User's observation in the transition:** *"guess that's it"* — meaning the per-field-ternary application-site ugliness is **not** directly resolved before moving on. The speaker is leaving the `return current if update is UNSET else update` pattern as-is and pivoting to extraction. **The declarative-merge step (Pydantic `model_copy + exclude_unset`, dataclasses `replace`, or a custom helper) does not appear to be on the slide deck.** Honest gap in the prescription — the talk gives you typed *representations* of the three-state distinction but leaves the *applying-an-update-to-a-domain-model* pattern hand-rolled.
  - **What "safe extraction" likely covers in section 4** *(predictions ahead of content — listening aid):*
    - **Type-narrowing patterns:** using `is UNSET` / `is None` / `isinstance(_, str)` checks to narrow the three-state union down to a single concrete type in each branch. Modern mypy/pyright narrow on these patterns, so once you've checked `if update is UNSET: return; ...` the downstream code can rely on the narrower type without explicit casts.
    - **Pattern matching (`match` / `case`):** Python 3.10+'s structural pattern matching is a natural fit for the three-state union. A clean form:
      ```python
      match update:
          case _UnsetType():
              # caller didn't address this field
          case None:
              # caller wants to clear this field
          case str(value):
              # caller wants to set this field to `value`
      ```
      Exhaustive, type-checker-narrowable, no `if/elif/else` chain.
    - **`TypeIs` / `TypeGuard` (PEP 742 / PEP 647):** if you write a helper like `def is_set(x: T | _UnsetType) -> TypeIs[T]`, the type checker narrows after `if is_set(x)` calls — gives you a *named* extraction predicate that documents intent at the call site and propagates type information.
    - **Pydantic's `model_dump(exclude_unset=True)` revisited:** the natural escape hatch from per-field ternaries, if you went the Pydantic route at the object layer. Probably gets a mention here as the declarative analog to the manual extraction patterns.
    - **Avoiding `cast()` and `# type: ignore`:** the *unsafe* extraction patterns. If your code uses `cast(str, payload.get("nickname"))` to satisfy the type checker, you've moved the bug from compile time to runtime. "Safe extraction" probably means using narrowing primitives the language gives you, not lying to the type checker.
  - **Why "extraction" deserves its own section in the talk's structure:** the rest of the talk has been about *representing* the three states correctly (TypedDict with NotRequired, sentinel-typed unions). But *representing* the three states is only half the job — *consuming* them safely (so you don't accidentally pass `UNSET` to a function that expects a `str`, or treat `None` as "no value sent") is the other half. Section 4 is the answer to "okay, I have the right types — now how do I actually use the values without re-introducing the bug?"
  - **Speaker's framing line for section 4:** *"Extraction turns raw dicts into meaning."*
    - Lovely framing. The boundary TypedDict is **shape only** — it tells the type checker "this dict has these keys with these types." It doesn't tell anyone what those values *mean* in the application's domain. Extraction is the bridge from *shape-typed* (a `dict` that satisfies a schema) to *semantically-typed* (a domain concept like "the new nickname value, or 'leave alone', or 'clear it'").
    - This is the same distinction as parsing vs. validating, or in DDD terms, the difference between a *Data Transfer Object* and an *Entity / Value Object*. The boundary type is the DTO; the extracted type is the domain concept. They have different lifetimes (boundary type lives at the edge of the system; domain type lives in business logic), different concerns (boundary cares about wire format; domain cares about invariants), and different mutability rules (boundary should be read-only via `ReadOnly[T]`; domain may be mutable as the business needs).
    - **Implication for the three-stage pipeline framing:** the *normalized update* layer in `payload → normalized update → domain model` is precisely the artifact extraction *produces*. Section 4 is the talk's most concrete treatment of that middle layer — how do you produce it, and what does it look like?
  - **Recommendation #1 — "Name payload shapes."**
    - Don't write inline anonymous types like `str | None | _UnsetType` scattered throughout your code. Give the recurring patterns **named types**. e.g., `PatchField[T]`, `NullableUpdate[T]`, `OptionalNullable[T]`. Names turn ad-hoc patterns into reusable vocabulary.
    - Why this matters more than it sounds: when the same `T | None | _UnsetType` shape appears in fifteen places across your codebase, fifteen developers each have to *decode* what that union means semantically (is it PATCH-shape? full-replace? something custom?). A named type — say, `type PatchField[T] = T | None | _UnsetType` — encodes the *intent* alongside the shape. Reading `bio: PatchField[str]` is immediately understood; reading `bio: str | None | _UnsetType` is a puzzle to be re-solved each time.
    - Also crucial for refactoring: if you decide later to switch from a project-local `_UnsetType` to `builtins.sentinel()` (per Python 3.15), or to change the null-handling semantics, **you change the named type in one place** and every usage migrates with it. With anonymous inline unions, every usage is a manual edit.
    - The "name the pattern" instinct also surfaces *which patterns* you actually have. If you start naming, you'll discover your project has 2-3 recurring shapes (PATCH with null-clear, PATCH without null-clear, optional input) — and that's a useful taxonomy for any new code that needs to fit one of them.
  - **Recommendation #2 — "Type defs using PEP 695."** Use the modern type-alias syntax (Python 3.12+).
    - **PEP 695** ("Type Parameter Syntax," accepted into Python 3.12) added two big things:
      1. A new `type Name = ...` statement for declaring type aliases (replaces the older `Name: TypeAlias = ...` from PEP 613).
      2. New generic-parameter syntax with `[T]` brackets directly on classes, functions, and type aliases (replaces the older `Generic[T]` / `TypeVar` ceremony).
    - The clean way to name the three-state shape with PEP 695:
      ```python
      # Python 3.12+:
      type PatchField[T] = T | None | _UnsetType   # nullable PATCH field
      type SetField[T]   = T | _UnsetType          # non-null PATCH field

      class UserPatch(TypedDict, total=False):
          nickname: PatchField[str]    # = str | None | _UnsetType
          bio:      SetField[str]      # = str | _UnsetType
      ```
    - Compare the pre-695 way:
      ```python
      # Pre-3.12:
      from typing import TypeAlias, TypeVar
      T = TypeVar("T")
      PatchField: TypeAlias = T | None | _UnsetType  # ❌ doesn't even work — TypeVar can't be used like this
      # You'd need: PatchField = Union[T, None, _UnsetType]  with the TypeVar boilerplate
      ```
      PEP 695's `type X[T] = ...` is dramatically more ergonomic — no separate `TypeVar` declaration, no `TypeAlias` annotation, no generic-base-class trick. Generic type aliases finally feel like a first-class language feature.
    - **Bonus advantage — lazy evaluation.** PEP 695's `type` statement creates a `typing.TypeAliasType` instance whose right-hand-side is *not evaluated at definition time*. Forward references work without strings: `type Foo = Bar` where `Bar` is defined below in the same file is fine. The old `Foo: TypeAlias = "Bar"` (using a string) was the workaround; PEP 695 makes it natural.
    - **For nikkie's audience specifically:** as a maintainer of **datamodel-code-generator** he's almost certainly thinking about *what code generators should emit*. Pre-695 generated code was full of `TypeVar` declarations and `TypeAlias` annotations; post-695 generated code can use the modern syntax. **Tools that emit Python type definitions now have a much cleaner target.** This is a recurring theme of his ecosystem work — the language has been gradually catching up to what codegen tools need, and PEP 695 closes one of the bigger gaps.
  - **Where this lands the talk so far:** putting together everything from sections 2-4, the recommended pattern is:
    ```python
    # 1. Define named patterns once, with PEP 695:
    type PatchField[T] = T | None | _UnsetType
    type SetField[T]   = T | _UnsetType

    # 2. Use them in your boundary TypedDict:
    class UserPatchPayload(TypedDict, total=False):
        id:       Required[ReadOnly[str]]
        email:    Required[ReadOnly[str]]
        nickname: ReadOnly[PatchField[str]]  # absent OR null OR string
        bio:      ReadOnly[SetField[str]]    # absent OR string (no null-clear)

    # 3. Extract safely in the handler:
    if "nickname" in payload:
        # type narrows; payload["nickname"] is str | None here
        new_nickname = payload["nickname"]
        # apply: if None, clear; if str, set
    # else: leave alone — caller didn't address it
    ```
    Every part of this signature carries semantic meaning: required-vs-optional, nullable-or-not, readable-or-writable, "this is a PATCH field" vs "this is an identity field." **The type signature is now a complete static description of the API contract.**
  - **Recommendation #3 — `TypeIs` for safe extraction predicates** *(PEP 742, Python 3.13).* The speaker is showing this as the right way to write extraction helpers — the "what does it actually mean to *check* if a field was set" question, answered at the typing level.
    - **What `TypeIs[T]` does:** it's a return-type annotation for a *user-defined type narrowing function*. When you write `def foo(x: A) -> TypeIs[B]`, the type checker treats `if foo(x):` as narrowing `x` to `B` (more precisely: `A & B`) in the true branch — *and* to `A & ~B` in the false branch. Bidirectional narrowing, which is exactly what you want for two-state unions like `T | _UnsetType`.
    - **The canonical extraction predicate for PATCH fields:**
      ```python
      from typing import TypeIs

      type PatchField[T] = T | _UnsetType

      def is_set[T](value: PatchField[T]) -> TypeIs[T]:
          return not isinstance(value, _UnsetType)
      ```
      Now in the handler:
      ```python
      if is_set(update.nickname):
          # type checker knows: update.nickname is str | None here
          apply_nickname(update.nickname)
      else:
          # type checker knows: update.nickname is _UnsetType here
          pass  # leave field alone
      ```
      The `is_set` predicate is **named, reusable across every field of every PATCH model**, and the type checker proves both branches handle the field correctly. The earlier per-field-ternary ugliness (`return current if update is UNSET else update`) gets factored into a single typed predicate plus a single typed apply step.
    - **`TypeIs` vs the older `TypeGuard` (PEP 647) — important distinction:**
      - `TypeGuard[T]`: narrows *only in the true branch* to `T`. The false branch retains the original type (no narrowing). Often surprising and rarely what you want for two-state unions.
      - `TypeIs[T]`: narrows in *both branches* — true branch is `A & T`, false branch is `A & ~T`. Symmetric and intuitive. **`TypeIs` is what you want for `is_set`-style predicates over PATCH fields.**
      - Reach for `TypeGuard` only when the predicate's truth doesn't logically imply anything about the falsity case (e.g., "is this a strict subtype" where the false case could be either the supertype or a sibling subtype). For "is this UNSET / is this set," `TypeIs` is correct.
    - **Why `TypeIs` over inline `isinstance` / `is UNSET` checks:**
      - Inline `if value is UNSET:` works and narrows correctly, but it doesn't *name* the concept. Every field that needs this check duplicates the same comparison; reading the handler is a noise-vs-signal problem.
      - `if is_set(value):` makes the question being asked *part of the function's vocabulary*. Future readers understand the intent immediately; future maintainers swapping in `builtins.sentinel()` (per 3.15) only need to update one definition.
      - For more complex unions (`T | None | _UnsetType`), you may want multiple predicates (`is_set`, `is_clearing`, `is_value`) — each with its own `TypeIs` shape. Inline checks make this hard; named predicates make it natural.
    - **Pattern recognition — this is the "rich union types" idiom from typed FP, finally landing cleanly in Python.** Languages like Haskell and OCaml have always had bidirectional narrowing on sum types via pattern matching. Python's recent additions (`match`/`case` from PEP 634, `TypeIs` from PEP 742, PEP 695 generic type aliases) compose into a workflow where Python can handle ADT-style data the way ML-family languages have for decades. **Section 4's "safe extraction" recommendations are really "do ADT-style discriminated-union code in Python, using the recent typing primitives."** That's the talk's lasting takeaway in one sentence: **Python's type system has grown the primitives to do this properly — start using them instead of papering over with `Optional[T]`.**
    - **Speaker's slide title for the `TypeIs` example: *"One shape predicate."*** Crisp framing of the recommendation — *one* function names the "is this field actually set" check, and *every* place in the codebase that asks the question calls that one predicate. The bug from slide 1 (scattered `payload.get("nickname")` calls that all silently mishandle the same edge case) is fundamentally a **predicate-duplication problem**: every site reinvents "is this field set?" and any of them can get it wrong. Centralizing into one shape predicate makes the check **uniformly correct or uniformly broken** — and uniformly broken is fixable with a one-line edit, while scattered subtly-wrong is the bug class he opened the talk with.
    - **The full Section 4 discipline, in one phrase:** **name your shapes, name your predicates.** Combined:
      - "Name payload shapes" → PEP 695 `type PatchField[T] = T | _UnsetType` (named *types* for the recurring patterns).
      - "One shape predicate" → `def is_set[T](v: PatchField[T]) -> TypeIs[T]` (named *predicates* for the recurring checks).
      - Together: the boilerplate exists *once*, the rest of the codebase reads as direct application of named domain concepts. No more scattered `is UNSET` checks, no more inline `T | None | _UnsetType` unions polluting every signature.
  - **Handler shape — "use `TypeIs` with a simple `if` guard before currying to other functions."**
    - The recommended handler structure is dead simple — a sequence of `if is_set(...): forward(...)` lines, one per updatable field:
      ```python
      def handle_user_patch(user: User, update: UserUpdatePayload) -> None:
          if is_set(update.nickname):
              # update.nickname narrowed to str | None — pass directly downstream
              set_nickname(user, update.nickname)
          if is_set(update.bio):
              # update.bio narrowed to str — pass directly downstream
              set_bio(user, update.bio)
          # ... one line per field ...
      ```
    - **"Currying" in this context** *(loosely meant)*: the downstream functions like `set_nickname(user, value)` are *partial applications* — `user` is the contextual arg, the update value is what varies. The handler is responsible for the binding-the-context-and-forwarding-the-narrowed-value step; the domain functions are clean unary-ish callees that don't know `_UnsetType` exists.
    - **Why this shape is the right destination:**
      - **Domain functions stay clean.** `set_nickname(user, value: str | None)` takes the *domain-meaning* type (`str | None` = nickname value or None to clear), not the boundary-meaning type (`PatchField[str]` = absent or value-or-null). The "absent" concept does not propagate past the handler. Domain layer is unburdened.
      - **The handler is the *only* place that knows about the boundary type.** All `_UnsetType` and `PatchField[T]` mentions are concentrated in the handler module. The rest of the codebase operates on clean domain types. This is precisely the *narrow at the boundary, propagate domain types downstream* discipline from earlier in the talk made concrete.
      - **The structure is uniform across all PATCH fields.** Every updatable field gets the same two-line treatment (`if is_set(x): forward(x)`). A new field added to the model requires adding one of these blocks to the handler — mechanical, hard-to-forget, and impossible to subtly mishandle in the wrong-way-this-time fashion that produced the opening-slide bug.
      - **No per-field ternaries needed in the apply-functions.** The earlier ugliness (`return current if update is UNSET else update`) is gone: the handler decides whether to call the apply-function at all, and the apply-function just does its job. The "merge" step has been pushed up to the handler and uniformized via the predicate.
    - **Compare against the alternative — declarative Pydantic `model_copy(update=...)`:** the if-guard pattern is *more verbose* than the Pydantic dict-merge approach but *more flexible* — each field can have a custom downstream call, including ones that aren't simple value-assignments (e.g., "if nickname is set, also send a notification" or "if email is set, trigger re-verification"). For pure data-mapping cases, `model_copy` wins on conciseness; for cases with **side effects per field**, the if-guard pattern is the right shape. Speaker is showing the more general one, which makes sense for a generic teaching context.

- **Section 5: "Domain model with dataclasses."** Now arriving at **stage 3 of the three-stage pipeline** (`payload → normalized update → domain model`). The domain model is the application's full representation of the entity — the *fully-realized* object. With sections 2–4 covered, section 5 names the concrete tool for the domain layer.
  - **The architectural picture, now complete across all three pipeline stages:**
    | Stage             | Layer purpose                  | Recommended representation                   | Sentinel/UNSET present? |
    |-------------------|--------------------------------|----------------------------------------------|--------------------------|
    | 1. Payload        | Wire shape, parsed input       | `TypedDict` with `Required`/`NotRequired`/`ReadOnly` | No (states observable via `key in dict`) |
    | 2. Normalized update | In-flight intent              | dataclass / Pydantic with `PatchField[T]` sentinel-typed fields | **Yes** — the absent state is the whole point of this layer |
    | 3. Domain model   | Fully-realized entity          | `@dataclass` (stdlib, frozen optional)        | **No** — UNSET never crosses this boundary |
    - The clear pattern: **`_UnsetType` exists only in stage 2.** It enters the type system when extraction-with-narrowing happens at the boundary, and it *exits* before the domain layer ever sees it. The handler's `if is_set(...)` guards from section 4 are precisely what strip the UNSET state away — once a value is past the guard, it's a clean domain-meaning type that a dataclass can accept without further encoding.
    - This is the **information lifecycle for "did the caller address this field":** captured at the boundary (TypedDict + `in` checks), carried through the intermediate layer (sentinel-typed `PatchField`), discarded at the handler-domain interface (via `is_set` narrowing), absent from the domain layer entirely. Each layer represents only the distinctions *it* needs to make.
  - **Why dataclasses for the domain layer specifically:**
    - **Stdlib.** No additional dependency. `from dataclasses import dataclass` and you're done.
    - **Lightweight runtime.** Unlike Pydantic models, dataclasses don't re-validate on every field assignment. The domain object trusts that its construction site (the handler, which already did boundary validation) produced valid input. **Validation belongs at the boundary, not in the domain.**
    - **Frozen-ness is one keyword away.** `@dataclass(frozen=True)` gives you an immutable entity — updates produce *new* entities via `dataclasses.replace(existing, nickname="new")`, which is the FP-style "everything is a transformation" pattern that pairs beautifully with the rest of the typed-boundary discipline.
    - **`replace()` is the natural partial-update primitive.** `dataclasses.replace(user, **changes)` produces a new instance with the listed fields changed and all others copied — which is exactly what a PATCH means once you've already extracted the change-set. Clean, declarative, no per-field code needed beyond constructing the `**changes` dict.
    - **Type-checker friendly.** Dataclasses participate fully in mypy/pyright narrowing, attribute access, etc. No magic descriptor classes, no runtime introspection trickery — it's just a regular class with autogenerated `__init__` / `__repr__` / `__eq__`.
  - **Example completing the three-stage pipeline end-to-end:**
    ```python
    from dataclasses import dataclass, replace

    # Stage 3: domain model.
    @dataclass(frozen=True)
    class User:
        id:       str
        email:    str
        nickname: str | None   # domain meaning: nickname value, or None = no nickname
        bio:      str          # domain meaning: bio text, never null

    # Handler stitches stages 1 → 2 → 3:
    def handle_user_patch(existing: User, payload: UserPatchPayload) -> User:
        changes: dict[str, Any] = {}
        if "nickname" in payload:
            changes["nickname"] = payload["nickname"]    # str | None
        if "bio" in payload:
            changes["bio"] = payload["bio"]              # str
        return replace(existing, **changes)
    ```
    - Returns a *new* `User` reflecting the partial update. The original `existing` is untouched (frozen dataclass enforces this). The `payload` is untouched (`ReadOnly[T]` enforces this). **Pure transformation: input → input → new output.** Easy to reason about, easy to test (`assert handle(...) == User(...)`), easy to log (you have both before and after).
    - Compared to the in-place mutation pattern that opens the talk (`patch_user(...)` mutating somewhere), this is **dramatically safer**. Every layer of the stack has a clear input → output contract, no hidden state mutation, no opportunity for the boundary type's "did the caller address this field" question to be silently mis-answered.
  - **Why this is the talk's natural landing point:** sections 1–4 built the typing infrastructure. Section 5 finally shows what to *do* with it — produce clean, immutable, domain-typed values via stdlib dataclasses. The pipeline is now end-to-end legible:
    - **In**: raw JSON via FastAPI / your framework's parser.
    - **Out**: a new typed `User` instance.
    - **In between**: TypedDict at the boundary, sentinel-typed update intent in the middle, named predicates for narrowing, dataclasses for the domain.
    - Every layer uses the right Python typing primitive for its job. Every "boundary" between layers is a place where information is either preserved (states stay distinct) or deliberately discarded (UNSET is consumed and not propagated). **The pipeline is the discipline; the typing primitives are the parts.**
  - **Honest caveat the speaker may not address:** this pattern is *more code* than the naive Pydantic-default approach. The opening bug's solution (`if "nickname" in update.model_fields_set: ...`) is a single conditional; the full pipeline above is dozens of lines of typing declarations + handler logic + dataclass definitions across three layers. **For small projects or simple endpoints, the full pipeline may be overkill.** The talk's value is in showing the *destination* and the principles; teams should adopt as much of it as their codebase complexity warrants. A 200-LOC service with one PATCH endpoint can live with `model_fields_set` discipline; a 100k-LOC service with hundreds of PATCH endpoints across dozens of resource types needs the full pipeline. **Adoption is incremental.**

- **Speaker slide: *"Wire shape and domain invariants are different contracts."*** This is the *generalization* of the three-stage pipeline into a single design principle. Yes — your instinct is right — it's an argument to **split your data types**, and serialization boundaries are exactly the canonical place this matters.
  - **What's being split:** the *types representing data at the wire boundary* from the *types representing fully-realized domain entities*. Even when they describe "the same thing" (e.g., a User), they have **different contracts** that pull in different directions, so they shouldn't be the same class doing double duty.
  - **Why the contracts differ:**
    | Concern                        | Wire shape (boundary type)                                              | Domain invariants (entity type)                              |
    |--------------------------------|-------------------------------------------------------------------------|--------------------------------------------------------------|
    | Fields                         | What the client *is allowed to send*                                    | What the entity *must have* to be valid                      |
    | Partial-ness                   | May be partial (PATCH) — absent fields are legitimate                   | Always complete — no "missing field" state                   |
    | Validation                     | Format-level only (right shape, right types, parseable)                 | Full invariants (non-empty email, length limits, foreign-key existence) |
    | Backward compatibility         | Must accept legacy fields the domain has retired                        | Reflects current domain truth — old fields removed           |
    | Forward compatibility          | Should tolerate fields the domain doesn't know about yet                | Strictly typed; unknown fields don't exist                   |
    | Mutability                     | `ReadOnly` — never mutated post-parse                                   | Domain choice; often `frozen=True` for FP-style updates      |
    | Identity                       | None — it's a parse result                                              | Has identity (the `id` field) — it's a *thing* in the world  |
    | What it serializes from / to   | JSON (or whatever wire format)                                          | The internal Python heap; possibly a database row            |
    - The contracts have **conflicting requirements**. Wire types want to be *permissive* (accept everything plausibly valid, so old clients don't break); domain types want to be *strict* (no malformed state representable). Trying to make one type satisfy both gives you a chimera that's wrong for both jobs: validation is too loose for the domain, too strict for forward-compat at the boundary.
  - **Where this argument bites hardest — serialization boundaries:**
    - **HTTP request/response.** Wire shape is "what the API contract says clients send/receive." Domain shape is "what the application reasons about internally." If you use the same class for both, you end up either weakening domain invariants ("nickname can be a sentinel" — no it can't, in the domain it's always a string-or-null) or strengthening wire validation prematurely ("email must be parseable" — fine for domain construction, but the wire layer might want to accept a malformed email and *return* an error rather than 422 it).
    - **Database persistence.** The DB row shape is *another* boundary. Your domain object may have computed properties (e.g., `full_name` derived from `first_name + last_name`) that you don't store; the DB row may have audit columns (`created_at`, `updated_by`) that aren't part of the domain identity. Same split argument: domain ≠ row.
    - **Message queues / event bus.** Events sent to a queue have *their own* wire shape (often versioned, often denormalized for downstream consumers). Treating the event payload type as identical to the producing domain entity will block schema evolution every time.
    - **External-system clients** (calling a third-party API). The third-party's request/response shape is *their* wire shape, not your domain. Roundtripping a third-party type as your internal representation couples your domain to their API forever; introduce a translator at the boundary.
  - **The split's payoff — what you can do once the types are separate:**
    - **Schema evolution is local.** Add a field to the wire type without changing the domain (you have a deserializer that ignores unknown wire fields, or you've planned for the field via `NotRequired`). Add a field to the domain without changing the wire (your serializer chooses what to expose). The two evolution stories are independent.
    - **Multiple wire shapes per domain.** Sometimes the same domain entity is exposed differently to different consumers — e.g., admin API includes audit fields, public API doesn't. Splitting the types lets you have `UserAdminResponse` and `UserPublicResponse` both serialized from the same `User` domain object, each with its own contract.
    - **Versioning at the boundary, not in the core.** V1 and V2 of the wire shape can coexist as different types; both deserialize into the same domain. Without the split, "supporting V1 and V2" means smearing version-aware logic through the domain layer.
    - **Testing boundaries.** You can unit-test the boundary type (`assert UserPatchPayload({"nickname": "x"}).nickname == "x"`) and the domain (`assert User(...).is_valid()`) independently. Without the split, every test couples wire concerns to domain concerns.
  - **The talk's three-stage pipeline is the *implementation* of this principle:**
    - Stage 1 (payload) and Stage 2 (normalized update) are *boundary types* — they answer "what's on the wire, with absence/null distinctions preserved."
    - Stage 3 (domain model) is the *invariant-holding entity* — it answers "what's a valid, complete User."
    - The handler is the translator. Its job *is* the contract translation. By naming this as a deliberate stage rather than a series of accidental coercions, the bug from slide 1 (where the wire type's absence-ambiguity bled into a domain-level write) becomes structurally impossible — those types simply can't be confused for each other.
  - **Common antipattern the principle warns against:** the "one Pydantic `BaseModel` per resource, used everywhere." `User(BaseModel)` for the response. `User(BaseModel)` for the request. `User(BaseModel)` for the ORM row. `User(BaseModel)` for the event payload. **Same class doing all four jobs** is the most common Python web-app shape, and it's exactly what the speaker is arguing against. The class accumulates fields and validators that serve some-but-not-all of its uses, and the contracts silently leak across boundaries. **Split the types; let each have its own contract; translate at the boundaries.**
  - **Connection back to the opening bug:** the patch-endpoint bug *was* a wire-vs-domain conflation. The handler used a `payload: dict` (wire shape — three possible states per field) but then called `patch_user(...)` (domain operation — two states per field, value-or-null). The wire shape's third state ("absent") had no representation downstream, so it got coerced into one of the two domain states (incorrectly mapped to `None`/"clear"). **The principle "wire shape and domain invariants are different contracts"** is the highest-level statement of why that bug exists and why naïvely flattening boundary into domain is structurally unsafe.

- **The concrete glue — `apply_payload(...)` slide.** Speaker now shows the helper that operationalizes the pipeline. Three pieces of detail caught:
  - *"TypedDict → dataclass binding. **Missing keys become UNSET.**"*
  - *"Helper keeps current values."*
  - *"`def apply_payload(...)` — normalized updates at the boundary."*
  - **What the helper is doing — both stage transitions in one function:**
    - **Stage 1 → Stage 2 (binding):** read the TypedDict payload, observe each field via `in`-check, produce a dataclass-typed normalized update where **missing keys are explicitly translated to `UNSET`**. This is the moment the absent-vs-null distinction crosses from "live in the dict's key-presence" to "live in the dataclass field's value." The information doesn't change; its *representation* moves from dict-shape to value-shape.
    - **Stage 2 → Stage 3 (apply):** read the normalized update, for each field decide whether to overwrite the corresponding domain-model field (when the update value is not UNSET) or *keep the current value* (when the update is UNSET). Produce a new domain entity reflecting the merge.
    - Both transitions sit in one `apply_payload` helper because they're typically called together — once you've parsed the wire, you want the resulting domain object. The intermediate dataclass exists transiently inside `apply_payload`'s execution.
  - **The "helper keeps current values" line is the key correctness property** — it's the **default behavior** of the apply step, not an opt-in. Fields the update doesn't address keep their current values *by construction*, not because the developer remembered to handle that case. The opening-slide bug was precisely the inverse: the developer remembered to *write* something for each field, and what they wrote happened to clear it. The helper-based pattern is **"do nothing unless told to"** — the safe default.
  - **Likely shape of the function** *(reconstructing from the three caught fragments):*
    ```python
    from dataclasses import fields, replace

    def apply_payload(existing: User, payload: UserPatchPayload) -> User:
        """Apply a partial-update payload to a User, keeping current values for unaddressed fields."""
        # 1. Bind: TypedDict → normalized update dataclass.
        #    Each field defaults to UNSET if the key is absent from payload.
        update = UserUpdate(
            nickname = payload["nickname"] if "nickname" in payload else UNSET,
            bio      = payload["bio"]      if "bio"      in payload else UNSET,
        )

        # 2. Apply: produce a new User with only the explicitly-addressed fields changed.
        changes = {
            f.name: getattr(update, f.name)
            for f in fields(update)
            if not isinstance(getattr(update, f.name), _UnsetType)
        }
        return replace(existing, **changes)
    ```
    - The binding step is mechanical: one `if "key" in payload` per field, mapping to either the value or `UNSET`. This is exactly the kind of code a *codegen tool* (like nikkie's `datamodel-code-generator`) is naturally suited to emit — given an OpenAPI spec describing the PATCH wire shape and the corresponding domain dataclass, the binding function writes itself. Manual handwriting is fine for a handful of fields; for dozens it's automation territory.
    - The apply step is parametric over the field set — it iterates `fields(update)` and includes any that aren't UNSET. **One implementation works for any pair of TypedDict + dataclass with compatible field names.** This generalizes beautifully — you could write a single library helper that takes any `(update_dataclass, domain_dataclass)` pair and does the right thing:
      ```python
      def apply_normalized[U, D](existing: D, update: U) -> D:
          changes = {
              f.name: v
              for f in fields(update)
              if not isinstance(v := getattr(update, f.name), _UnsetType)
          }
          return replace(existing, **changes)
      ```
      PEP 695 generic syntax + `dataclasses.fields` reflection = one reusable apply function for the whole codebase.
  - **The "normalized updates at the boundary" framing is the talk's bottom-line architectural recommendation.** Read it as: **the normalized update is the contract at your application's boundary — not the raw payload, not the domain model.** Production code should pass `UserUpdate` instances around, not `dict`s and not `User`s. The wire layer parses into `UserUpdate`; the domain layer consumes `UserUpdate` and produces new `User` instances; everywhere in the middle reasons about UNSET-typed *intent*, not value-typed *facts*. The intent type is the load-bearing artifact of the whole architecture.
  - **What's elegant about the speaker's whole pipeline now that all five sections are visible:**
    - Section 2 gave us the *shape* (TypedDict + NotRequired + ReadOnly).
    - Section 3 gave us the *value-space sentinel* for the absent state (UNSET).
    - Section 4 gave us the *predicates* for safely narrowing the union (`is_set` + `TypeIs`).
    - Section 5 gave us the *domain target* (frozen dataclass).
    - This `apply_payload` helper is the **glue that connects all four**. Read in dependency order, each section feeds the next; the apply function is the convergence point. **It's the structure of a well-designed talk — every section was setup for the climactic helper that brings them all together.**

- **Section 5 recap slide.** The speaker's three caught bullets *(plus one cut off):*
  - **"Keep TypedDict at the edge."** *(Probable verbatim — user wrote "at the end" but in context, "edge" is the architectural noun and almost certainly what was on the slide.)* TypedDict belongs at the **boundary** of the application — the moment data crosses from JSON-shaped wire into Python-shaped objects. Don't propagate the TypedDict deeper into business logic; that's where you reach for richer typed objects.
  - **"Apply updates into a dataclass instead."** Convert the TypedDict-shaped payload into a domain dataclass (via the `apply_payload` helper) before business logic touches it. The dataclass — not the dict — is what flows through your domain layer.
  - **"Business logic sees [domain types] …"** *(cut off in capture)* — almost certainly completing as "business logic sees the dataclass, not the dict" or "business logic sees clean domain values" or similar. The point being: **by the time control reaches business logic, the boundary types have been consumed.** Business logic operates on `User`, not on `UserPatchPayload`. The UNSET state, the wire-shape concerns, the absent-vs-null distinctions are all *resolved* at or before the handler boundary — they don't leak into domain code.
  - User's self-assessed read: *"TypedDict → dataclass argument was on another slide, I think I got the point."* — confirming the recap was a compression of an argument already made. The principle generalizes cleanly: **typed wire shapes at the edge, typed domain entities in the core, and a translator between them.** Each layer uses the typing primitives that fit its job; no single class doing all three.

- **The adoption checklist — the talk's operational close.** Five ordered steps for a team to start adopting the pattern:
  1. **Pick one boundary crossing.**
  2. **Write its TypedDict contract.**
  3. **Narrow raw input at the edge.**
  4. **Add UNSET with a sentinel.**
  5. **Apply into a dataclass.**
  - **Why this checklist exists at all** — and why it's the right way to close the talk. Sections 1–5 built the full architectural pattern, which is a lot to swallow whole. Without explicit incremental-adoption guidance, the audience hears "redo your whole codebase to match this five-layer pipeline" and quietly decides it's too much work. The checklist solves that: it scopes the *first commit* down to "one endpoint, one TypedDict, one apply helper." **The recipe is migrate-by-endpoint, not migrate-everything-at-once.**
  - **Walking the steps:**
    - **Step 1 — "Pick one boundary crossing."** Don't try to convert the whole codebase. Pick *one* endpoint (probably the most painful PATCH handler in your service) or *one* external consumer (e.g., a single third-party API call). Limit blast radius, build a working example, validate the approach pays off before committing to a fleet-wide migration. The choice is also a signal of where the value is — pick the spot that has been *biting* you, not an arbitrary one, so the conversion has a concrete bug-it-fixed story attached.
    - **Step 2 — "Write its TypedDict contract."** For the chosen boundary, declare a TypedDict describing the wire shape. Use `total=False` or per-field `Required[T]`/`NotRequired[T]` as appropriate. Wrap individual fields in `ReadOnly[T]` if you want compile-time protection against accidental mutation of the parsed payload. **This is the modeling moment — you're committing the wire contract to typed code.** If you can't write it because the wire shape is ambiguous, you've already found a documentation gap worth fixing before any of the rest.
    - **Step 3 — "Narrow raw input at the edge."** Use `if "key" in payload:` checks (or pattern matching) to narrow the TypedDict at the point where you consume each field. Replace any `.get()` calls — those are what flattened the absent-vs-null distinction in the opening slide. The narrowing is where the bug stops being possible: forgetting the `in` check is a *type error* now, not a silently-wrong runtime behavior.
    - **Step 4 — "Add UNSET with a sentinel."** Bring in an `UNSET` sentinel (project-local class on pre-3.15, `builtins.sentinel(...)` on Python 3.15+) for any normalized-update layer fields that need the absent state to survive past the dict-shape boundary. Use it as the default value of dataclass / Pydantic fields representing the *intent to update.* This is where you move the absent-state information from dict-key-presence into a typed value — and from there it can flow through the rest of the program with the type checker enforcing handling.
    - **Step 5 — "Apply into a dataclass."** Write the `apply_payload`-style helper that produces a new (frozen) domain dataclass with only the explicitly-addressed fields updated. Fields where the update is UNSET keep their existing values *by construction*; the helper's default behavior is "do nothing unless told to." This is the last step because it depends on having all four prior pieces: a typed boundary, a narrowed access pattern, a sentinel for absent, and a domain target to populate. Once it's wired up, the endpoint has gone from "uses `dict.get`" to "uses the full pipeline" — and the absent-vs-null bug it was vulnerable to is structurally impossible.
  - **The ordering is deliberate.** Each step depends on the previous one:
    - You can't write a TypedDict contract (step 2) until you've chosen a boundary (step 1).
    - You can't narrow (step 3) until you have a TypedDict (step 2).
    - Adding a sentinel (step 4) only makes sense once you're moving values from dict-shape to value-shape — i.e., after narrowing (step 3).
    - Applying into a dataclass (step 5) is the consumer of all four prior steps.
    - **You can stop after any step and still ship value.** A team that does only steps 1-2 already gains a documented wire contract. Adding step 3 stops the opening-slide bug class. Steps 4-5 give you the full FP-flavored pipeline. **Each prefix of the checklist is a working improvement** — the team chooses how far up the ladder they want to climb based on the value/cost tradeoff in their context.
  - **The checklist closes the "this is too much code" objection.** Earlier in my notes I flagged that the full pipeline is dozens of lines compared to a simple Pydantic-default approach, and warned that small projects might find it overkill. The adoption checklist is the speaker's answer: **you don't adopt the pipeline as a monolith.** You adopt it one boundary crossing at a time, one prefix of the checklist at a time. **Incremental migration is the design.** The pattern works at any scale because adoption scales with project size — small projects do less, large projects do more, and there's no all-or-nothing cutover.
  - **What's missing from the checklist that's worth noting:** no explicit step for the `ReadOnly[T]` (PEP 705) addition, no explicit step for naming patterns via PEP 695 type aliases, no explicit step for writing `is_set`-style `TypeIs` predicates. These are all *refinements* of the basic checklist — once you've done all five steps for a few endpoints, you'll naturally find yourself wanting to factor out repeated patterns (named types, named predicates) and harden against accidental mutation (`ReadOnly`). The five-step checklist is the *foundation*; the named-types-and-predicates discipline is the polish that comes after the foundation is solid. **Crawl, walk, run.**
  - **Companion slide — "Done when …" criteria.** Pairs with the five-step adoption checklist; each step has a verifiable completion indicator. Lets a team **audit** any boundary crossing against the pattern:
    1. **Done when an endpoint or queue handler is chosen.** *(Concrete file picked; scope is bounded.)*
    2. **Done when missing keys use `NotRequired`.** *(TypedDict expresses absence correctly — no `total=True` hiding optional fields, no `dict[str, Any]` sneaking past.)*
    3. **Done when the raw dict/object is stopped at the edge.** *(No `dict[str, Any]` flows past the handler; downstream code receives narrowed typed values only.)*
    4. **Done when omit / null / value stay separate.** *(The three states are distinct at every layer they pass through — TypedDict's `in`-check at the boundary, UNSET-typed in the normalized update, consumed at the apply step before reaching the domain.)*
    5. **Done when business logic …** *(caught fragment — likely completing as "operates on the dataclass, not the dict" or "sees only clean domain types." The criterion is that domain code never imports the TypedDict and never sees `_UnsetType`.)*
  - **Why "done when" reframes the checklist usefully:** the adoption checklist tells you *what to do*; the done-when criteria tell you *how to know you finished*. Together they make the pattern's adoption **testable** — a code reviewer can mechanically check whether a PR introducing a new boundary crossing satisfies each criterion. Teams can write a linter or a checklist comment template that hits each item. The pattern stops being "a thing one engineer read about and tries to remember" and becomes **a property the codebase can be audited against**.

- **Tools-by-Python-version reference slide.** Last big practical slide before close — *which features land in which Python release?* All the typing primitives this talk has been pushing have specific minimum versions, and teams need to know if they can adopt today or have to wait / backport.
  - **The features named on the slide (caught list):** `TypedDict`, `Required` / `NotRequired`, `type` statement (PEP 695), `ReadOnly`, `TypeIs`, `sentinel()`.
  - **Reference table** (cross-checked against the relevant PEPs):
    | Feature                             | PEP | Python min |
    |-------------------------------------|-----|------------|
    | `TypedDict`                         | 589 | 3.8        |
    | `Required` / `NotRequired`          | 655 | 3.11       |
    | `type X = ...` / `type X[T] = ...`  | 695 | 3.12       |
    | `ReadOnly[T]`                       | 705 | 3.13       |
    | `TypeIs[T]`                         | 742 | 3.13       |
    | `sentinel(...)`                     | 661 | **3.15**   |
  - **Practical implications by Python version:**
    - **On 3.8–3.10:** only basic TypedDict (with `total=True/False`) — you can express "all fields required" or "all fields optional" but not per-field optionality. The PATCH bug from slide 1 *cannot* be cleanly fixed at the type level on these versions; you'd have to drop to `typing_extensions` for the `NotRequired` backport.
    - **On 3.11:** PEP 655 unlocks per-field `NotRequired` / `Required` — the **minimum version** for cleanly typing PATCH payloads with TypedDict. This is where the basic boundary-narrowing pattern from sections 2–3 becomes practical without backports.
    - **On 3.12:** PEP 695's `type` statement makes the named-pattern recommendation (e.g., `type PatchField[T] = T | None | _UnsetType`) ergonomic. Before 3.12 you'd use the awkward `TypeAlias` + `TypeVar` ceremony.
    - **On 3.13:** PEP 705 (`ReadOnly`) and PEP 742 (`TypeIs`) both land. This is the **first Python where the whole talk's pattern is natively expressible** — boundary safety via `ReadOnly`, narrowing predicates via `TypeIs`, all without `typing_extensions`.
    - **On 3.15:** PEP 661's `sentinel()` finally closes the UNSET-boilerplate gap. From 3.15 onward, even the workaround is one line instead of three.
  - **The `typing_extensions` escape valve:** every feature listed has a `typing_extensions` backport that brings the symbol to older Pythons for type-checking purposes (runtime support varies). Teams stuck on 3.10 or earlier can still adopt most of the pattern by importing from `typing_extensions` instead of `typing`. This is the path most production codebases will actually take.
  - **The version table is also a meta-narrative of the typing PEPs.** Read the right column top to bottom: 3.8 → 3.11 → 3.12 → 3.13 → 3.15. **Every recent Python release has added one or more pieces of this pattern.** That's not coincidence — the typing council has been *deliberately* closing the gaps the speaker is pointing at. The talk is a synthesis of "here's what all these PEPs add up to when you compose them," which is genuinely useful: each PEP individually is a small change, but together they describe an entire correctness discipline that didn't have a coherent name before.

