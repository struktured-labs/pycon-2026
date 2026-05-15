# PEP 750: t-strings — Safer and Smarter String Processing

**Speaker:** Vinicus Gubiana Ferreria *(spelling unverified — phonetic)*
**Event:** PyCon 2026
**Date:** 2026-05-15

## Background (pre-loaded while listening)

- **Status:** PEP 750 is **Final / accepted**, resolved 2025-04-10. Shipped in **Python 3.14** (released 2025-10-07). Already in stdlib as `string.templatelib`.
- **Authors:** Jim Baker, Guido van Rossum, Paul Everitt, Koudai Aono, Lysandros Nikolaou, Dave Peck.
- **Syntax:** `t"Hello {name}"` — same shape as f-strings, prefix is `t` instead of `f`.
- **Key difference vs f-strings:** evaluates to a **`Template` object**, *not* a `str`. Processing is deferred — the caller gets structure, not a concatenated string.
- **`Template` structure:**
  - `strings`: tuple of the static literal segments
  - `interpolations`: tuple of `Interpolation` objects, each with `value` (evaluated), `expression` (the original source text, e.g. `"name"`), `conversion` (`!r`/`!s`/`!a`), `format_spec` (e.g. `".2f"`)
- **Why it exists (the safety pitch):** f-strings eagerly stringify, so they're unsafe for SQL / HTML / shell — values get concatenated before any escaping context exists. T-strings hand the structure to a processor function that can apply context-aware escaping per interpolation.
  - SQL injection example: `f"SELECT * FROM users WHERE id={user_id}"` → unsafe. With t-strings, a SQL processor sees the interpolation separately and can parameterize it.
  - XSS: `f"<p>{user_input}</p>"` → unsafe. HTML processor can escape per slot.
- **Ecosystem already shipping (as of 2026):**
  - `ludic` — HTML templating + lightweight web framework on t-strings
  - `tdom` — HTML templating system using t-strings
  - `tstr` — backport / cross-version compat for older Pythons
  - "awesome-t-strings" list curates more (logging, SQL, escaping helpers)
- **Mental model shortcut:** f-strings are *eager rendering*; t-strings are *structured deferred rendering* — like JSX vs. a concatenated HTML string, or parameterized queries vs. raw SQL.

**Sources:**
- [PEP 750 — Template Strings](https://peps.python.org/pep-0750/)
- [What's New in Python 3.14](https://docs.python.org/3/whatsnew/3.14.html)
- [awesome-t-strings (community library list)](https://github.com/t-strings/awesome-t-strings)
- [Python 3.14: t-Strings, annotationlib, and locals() Fixes That Shipped](https://www.pyblog.in/programming/python-3-14-beta-whats-landing-before-the-final-release/)

## Notes

- Acknowledged tension: **PEP 20 ("Zen of Python") violation** — *"There should be one-- and preferably only one --obvious way to do it."* Python now has at least four string-formatting/interpolation paths stacked up:
  1. `%`-formatting (`"%s" % x`) — printf-style, the original
  2. `str.format()` / `"{}".format(x)` — PEP 3101
  3. **f-strings** (`f"{x}"`) — PEP 498, eager
  4. **t-strings** (`t"{x}"`) — PEP 750, structured/deferred
  - Each was justified on its own merits, but the cumulative result is exactly the "many obvious ways" outcome the Zen warns against. Worth watching whether the talk acknowledges this honestly or hand-waves it.
- Speaker pulled out the inevitable **xkcd #327 "Exploits of a Mom" / "Little Bobby Tables"** to motivate the SQL injection angle. Classic and a bit overused at this point — any talk that touches string formatting + safety seems contractually obligated to show it. Effective as a hook, even if the audience has seen it twenty times.
- **Headline pitch for t-strings: templating + security.** Two use cases doing the heavy lifting in this talk's narrative:
  1. **Templating** — give libraries (HTML, SQL, logging, shell, etc.) structured access to the literal/interpolation split so they can render correctly per context.
  2. **Security** — eliminate the eager-stringify foot-gun by making safe escaping/parameterization the natural default when a library accepts a `Template` instead of a `str`.
  - Worth noting: both reduce to the same underlying property — *deferred, structured interpolation* — but the speaker is framing them as the two audience-facing reasons to care.
- **Jinja beef incoming** — speaker is openly not a fan of Jinja templates and is using t-strings as the leverage point for the critique. *(Specific gripes not captured yet — revisit. Usual suspects: separate DSL to learn, logic-in-templates anti-pattern, opt-in escaping, heavy runtime, non-Python context switch. The t-strings angle would be: you can now get HTML templating in plain Python with structural safety, so why import a whole templating engine?)*
- Speaker drove home the type-identity point: **"t-strings are not strings."** Concretely: `t"..."` returns a `string.templatelib.Template`, not a `str`. Practical consequences:
  - `isinstance(t"x", str)` is **False**.
  - You can't pass a t-string anywhere a `str` is expected without a processor function in between — by design, this is the safety/forcing function.
  - Static typing sees a different type, so APIs can require `Template` (e.g. a SQL builder) and refuse plain `str` to block injection at the type level.
- Speaker formally named the per-slot type: **"Interpolation objects"** (`string.templatelib.Interpolation`). Each `Template.interpolations[i]` is one of these — carries `value` (already evaluated), `expression` (source text), `conversion` (`!r`/`!s`/`!a`), `format_spec`. Already covered in the Background section above; flagging that the speaker introduced the term explicitly to the audience here.
- Speaker also called out `Template.values` — convenience property that returns just the **tuple of evaluated values**, equivalent to `tuple(i.value for i in template.interpolations)`. Useful when a processor only cares about the substituted values and not the expression text / format spec / conversion (e.g. handing them straight to a DB driver as parameterized query args).
- **Processing path: don't `str(template)` — write/use a processing function.** Speaker re-emphasized the point I touched on earlier: rendering a t-string is *not* done by stringifying it. You pass the `Template` to a context-aware processor (`html(template)`, `sql(template)`, `log(template)`, ...) that walks `strings` and `interpolations` and produces the safe output.
  - This is the whole forcing-function design: by making `str()` a non-path for rendering, the language refuses to let you accidentally produce a concatenated string the way f-strings would have. *(Exact behavior of `str(template)` — debug repr vs. raises — flag to confirm; the point either way is "not how you render it.")*
- **`+` works on t-strings.** `Template + Template`, `Template + str`, and `str + Template` all return a new `Template` — concatenation preserves structure rather than collapsing to a `str`. Important property: templates **compose**, so libraries can build complex queries/HTML by piecing together smaller templates (e.g. a WHERE clause built from optional sub-templates) without losing the per-slot safety the design hinges on.
- **Speaker's summary slide — t-string advantages over every prior option:**
  - **Lazy** — deferred rendering; structure is preserved until a processor is called.
  - **Safer** — context-aware escaping/parameterization at the slot level; type system can require `Template` to reject raw `str`.
  - **Less verbose** than:
    - `string.Template` (PEP 292, old `$name` / `${name}` substitution) — heavy ceremony for the same outcome
    - f-strings — fine when you don't need safety, but no structural handle once rendered
    - `%s` — printf-style, positional, error-prone
    - `.format()` — verbose call site, no safety
  - ⚠️ Naming collision note: the *old* `string.Template` (capital T) from PEP 292 is **not** the same as the new `string.templatelib.Template` returned by t-strings. Same word, different class. Easy to mix up in transcripts/slides.
- Speaker emphasized: **t-strings are already in use by community libraries.** Validates the PEP's bet that processor-style libraries would emerge — ecosystem covered in the Background section (`ludic`, `tdom`, `tstr`, plus the `awesome-t-strings` curated list). Useful point for any audience member still asking "is this real yet?" — yes, real and shipping.
- **DSLs reframed: "break them down into t-strings."** Speaker's bigger architectural claim — instead of embedding/building a full DSL (Jinja, SQLAlchemy expression language, GraphQL string tags, regex builders, shell-command DSLs, etc.) you implement the "language" as a **processor function over a `Template`**. The t-string is the universal substrate; the DSL is just `myDSL(t"...")`.
  - Direct parallel to JavaScript's **tagged template literals** (which gave us `styled-components`, `sql\`...\``, `graphql\`...\``, `html\`...\``). Python now has the equivalent primitive, ~10 years later, but with explicit `Template` typing baked in.
  - Implication for the Jinja beef: HTML templating doesn't need a separate DSL anymore — `html(t"<p>{user}</p>")` does the escaping + structure in plain Python, no `.j2` files, no Jinja runtime.
  - Caveat the speaker may or may not address: replacing a DSL with `t"..."` + processor is great until your "DSL" needs control flow (loops, conditionals, inheritance). Python's host language handles these fine, but the surface area moves from declarative template files to Python code — different review/edit story.
- **Logging use case: t-string logger as a separate function path.** Speaker is proposing a dedicated entry point (e.g. `logger.tinfo(t"...")` / parallel to the existing `.info`, `.debug`, etc.) that consumes `Template` instead of `str`. Wins:
  - **Lazy formatting for free** — the long-standing reason people are told to write `logger.info("got %s", x)` instead of `logger.info(f"got {x}")` is that f-strings format eagerly, wasting work when the log level filters the message out. T-strings defer rendering until the handler decides to emit, so the f-string-like ergonomics are no longer in tension with cost.
  - **Structured logging falls out naturally** — `Template.interpolations` gives you the values *and* their source expressions, so the logger can emit structured records (key/value pairs keyed by expression text) instead of opaque concatenated lines. Free win for observability pipelines.
  - **Separate function path is the right call** — overloading `.info()` to accept either `str` or `Template` would blur the safety boundary; a distinct method makes the contract explicit.
- **The bad — speaker's honest downsides slide:**
  - **Boilerplate to process the t-string yourself.** Every consumer needs a processor function — there's no built-in "just render it" shortcut, so libraries pay an upfront cost to expose t-string APIs. Trivial for library authors; potentially friction for one-off scripts where an f-string would have been a single line.
  - **"No automatic string output."** Restated as a *con* (it was framed earlier as a *pro / forcing function*): you literally cannot `print(t"...")` and get the rendered text — you must route it through a processor. Same property, opposite spin depending on whether you're the language designer or the person trying to dash off a quick log line.
  - **Performance overhead vs. f-strings — "possibly milliseconds" per the speaker.** T-strings allocate a `Template` + N `Interpolation` objects + the strings/values tuples, whereas f-strings produce one `str`. The speaker's "milliseconds" framing is on the loose side — per-call it's more realistically *microseconds* of allocation overhead, but it adds up in hot paths (tight loops, high-throughput logging, etc.). The right framing: t-strings are not free, so don't reach for them in code that's already on the edge of its perf budget.
  - **Complexity / UX cost.** New concepts to internalize: `Template`, `Interpolation`, `.values`, `.strings`, `.interpolations`, processor functions, the `string.Template` vs. `string.templatelib.Template` naming collision, and the decision of *which* formatting tool to use for *which* job (the PEP 20 violation point compounding into real cognitive load at the call site). For library authors this is upside (new substrate to build on); for everyday users it's one more thing to learn and explain in code review.
  - **Partial standardization.** PEP 750 standardizes the *substrate* (the `Template` and `Interpolation` types, the syntax) but **not the processors**. There's no stdlib `html()` / `sql()` / `log()` — every library invents its own conventions for escaping rules, attribute handling, parameterization style, etc. Practical consequences:
    - The "safety" win is only as good as the third-party processor you happen to pick — t-strings don't automatically make your code safe, they just enable a library to make it safe.
    - Same input → different output across libraries (e.g. two HTML libs may escape attributes differently). Portability between processors isn't guaranteed.
    - Direct rhyme with JavaScript's tagged-template ecosystem (`styled-components` ≠ `sql-template-strings` ≠ `graphql-tag` — each invents its own conventions on the same substrate). Python is signing up for the same fragmentation.

## Final thoughts (speaker's close)

- **Net assessment: positive, but targeted.** Speaker wrapped on two notes:
  1. **Great for library maintainers** — t-strings give framework/library authors a new structured substrate they previously had to fake with hacks (sentinel strings, custom parsers, awkward callable wrappers). The PEP's main constituency is library authors, not application authors.
  2. **String interpolation, broadly, is nice** — endorsement of the general ergonomics. Python adopting another interpolation form is, on balance, a feature even if it adds to the formatting-mechanism count (PEP 20 violation acknowledged).
- **Library adoption is strong.** Speaker called this out explicitly as a third closing point. Concrete signal: less than ~7 months after the 3.14 release, there's already a real ecosystem (`ludic`, `tdom`, `tstr`, the curated `awesome-t-strings` list, plus logging/SQL/escaping helpers) — not a chicken-and-egg situation where the PEP shipped and nobody used it. The bet that "if you build the substrate, library authors will come" has paid off so far.
  - **Follow-on PEP cited: [PEP 787 — Safer subprocess usage using t-strings](https://peps.python.org/pep-0787/)** (authors: Nick Humrich, Alyssa Coghlan; status: **Deferred**; target: Python **3.15**). Extends `subprocess` and `shlex` to natively accept t-strings, adding a `shlex.sh()` renderer for POSIX-shell escaping so command injection becomes structurally hard. The PEP frames itself as a reference implementation of "practical t-string application." Currently deferred — authors prototyping in an experimental library during 3.14 beta before revising. Concrete evidence the substrate is being built on at the language-design level, not just in userspace.
- Implicit takeaway: application developers can keep using f-strings most of the time. T-strings are the right tool when (a) safety/escaping is in the loop or (b) a library you depend on adopts them. Don't go rewriting everything.















