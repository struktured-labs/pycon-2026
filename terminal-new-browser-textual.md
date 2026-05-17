# The Terminal Is the New Browser: Building Rich TUIs with Textual

**Speaker:** **Andres Pineda** — handle **[@ajpinedam](https://github.com/ajpinedam)**. Senior Application Developer · **RealPython Core Team** (writes tutorials / contributes to one of the most-read Python publications) · **Microsoft MVP** · involved with **PyCascades** and **Python San Diego** (PySD) regional communities. Strong Python-community profile; not a Textualize founder/team member but a heavily-engaged community contributor and educator.
**Event:** PyCon 2026 (Long Beach)
**Date:** 2026-05-16
**Mode:** TLDR / lightweight capture.

## Notes

- **Opening line: *"The terminal is not dead."*** Classic contrarian opener. Setting up the talk against the implicit assumption that browsers and native GUIs have superseded TUIs.
- **Thesis slide: *"The terminal is the new browser."*** The reframe — not just defending the terminal as a survival case, but positioning it as the *forward-looking* surface for a class of applications. Strong claim; the rest of the talk is presumably the evidence for it. Expected pillars: zero-install distribution, ubiquity (every dev box has one), uniform across OS, low resource overhead, no JS / no auth / no DNS, and now (with Textual) rich UI ergonomics that historically only belonged to browsers.
- **UI-paradigm progression slide:** punch cards → GUI → voice → AI in the browser. Setting up the historical context — each era added a modality. The terminal as a *surface* has persisted across all of them. Likely framing: while each generation of UI promised to replace what came before, the terminal kept growing as the *workhorse for technical work* through every era, and is now poised for another cycle of relevance precisely because the latest era (AI / agents / dev tooling) generates work that's better expressed in text-first terms than in browser-native ones.
- **Killer one-liner:** *"The terminal is the most automation-friendly UI ever created."*
  - **Argument structure** (compressed into the one line): text in, text out. Every operation is scriptable; every output is parseable; pipes compose natively; no accessibility/computer-use hacks needed because *text IS the API*.
  - **Sharp connection to the AI era:** LLMs are *also* text-in / text-out. Terminal I/O is structurally aligned with how language models work. Driving a browser requires Anthropic's Computer Use API + screenshots + pixel-coordinate clicks; driving a terminal requires *just text*. **AI agents operating in the terminal are operating in their native substrate.** This is the strongest version of the "terminal is the new browser" argument — the terminal isn't catching up to the browser, it's a *better fit* for the dominant new workload.
  - Lift verbatim — it's the talk's single most quotable line for the AI-era reframe.
- **CLI vs TUI distinction** *(speaker's exact framing):*
  - **CLI** — *command-driven, typically stateless.* You issue a command, it runs, it exits. Examples: `ls`, `grep`, `git status`.
    - **Your "sorta" pushback is fair:** CLIs aren't truly stateless — env vars, working dir, files on disk, shell history all carry state. But each *invocation* is independent; the *process* doesn't persist across commands. That's the comparative-with-TUI sense the speaker is reaching for.
  - **TUI** — *interactive: maintains state, responds to events.* Persistent process, event loop, widgets, focus, async I/O. Examples: `htop`, `vim`, `lazygit`, `k9s`, Textual apps.
  - **Textual lives firmly in TUI territory.** The "terminal is the new browser" reframe applies specifically to TUIs — browsers are inherently *applications* (stateful, event-driven), and TUIs are their terminal-side analog. CLIs aren't competing with browsers; *TUIs* are.
  - The event-driven / state-maintaining shape also makes TUIs architecturally the same family as GUI apps and SPAs — same event loop, same widget tree, same reactivity. Textual's reactive programming model lands here naturally; it's not novel, it's just *the standard application architecture, applied to the terminal*.
- **Python TUI landscape — from oldest/lowest to newest/highest:**
  - **`curses`** — stdlib. Thin Python wrapper over C ncurses. Manual screen buffer management, integer color codes, painful API. Always available, rarely chosen by new projects.
  - **`urwid`** — pre-Textual incumbent. Widget-based, event-driven, mature. Used by `bpython`, older sysadmin tools.
  - **`prompt_toolkit`** — Jonathan Slennders'. Specialized for *input-heavy* interfaces (REPLs, autocomplete, fuzzy search). Powers IPython 5+, `pgcli`, `mycli`, `ptpython`. Less "general app framework," more "rich input widget."
  - **`rich`** — Will McGugan's foundational library. Not a TUI framework per se — for *output* formatting: tables, syntax highlighting, progress bars, markdown rendering, traceback prettification. **Most-installed terminal-prettification library in Python by far.** The substrate Textual is built on.
  - **`textual`** — McGugan's full application framework, built on Rich. CSS-style theming, async, web deploy. The talk's centerpiece. **Modern default for new TUI applications in Python.**
  - **The progression is one of escalating abstraction:** curses gives you the keyboard and a character grid; urwid gives you widgets but feels dated; prompt_toolkit gives you input flows; rich gives you beautiful output; textual gives you everything you'd want from a GUI framework, in the terminal. Each addresses what its predecessor couldn't.
  - **Speaker scopes the rest of the talk to Rich + Textual** for the *presentation* focus. Older / lower-level options (curses, urwid, prompt_toolkit) get acknowledged-and-skipped. Sensible scoping for a modern audience — most new Python TUI work in 2026 is on the Rich/Textual stack.
- **Rich pitch: *"everything looks pretty now in your CLI."* Selling point: *"more beautiful, no more complexity required"* — or, per the refinement, *"just a little more"* complexity if you want the extra features.
  - Rich's killer feature is **zero-config value** — `from rich import print` and your existing `print()` calls become syntax-highlighted, color-aware, smart-wrapped, with tracebacks auto-prettified. The cost is essentially zero; the visual improvement is large.
  - Beyond the import-and-go default, Rich also gives you **tables, layouts, columns, panels, progress bars, markdown rendering, JSON pretty-printing** — opt-in widgets for richer output without committing to a full TUI. **Tables and layouts** specifically called out: the gap between "I want pretty output" and "I want a TUI" gets filled by Rich's compositional primitives — you can build CLI tools that look almost-application-shaped without an event loop.
  - This is the entry-point case: even teams that won't adopt full TUIs *will* drop Rich into their CLIs because the migration cost is one import and the result is immediately visible to every user of the tool. **Highest-leverage cheap upgrade in the Python CLI ecosystem.**
- **Textual pitch: *"game changer — fully fledged API."*** The step up from Rich:
  - Rich is for **output**; Textual is for **applications**.
  - Where Rich asks "how do I make my prints pretty?" Textual asks "how do I build an *interactive app* with widgets, layouts, focus, events, mouse, CSS, async?"
  - **"Fully fledged API"** = the same surface area a GUI framework gives you (widgets, layout containers, signal/slot-style events, themes, dev tools, test harness), just running in a terminal instead of a window manager.
  - The progression Andres is walking the audience through: *plain `print`* → *Rich `print`* → *Rich components* → *Textual app*. Each step is a small jump in complexity and a large jump in capability; the last step crosses the line from "prettier CLI" to "actual terminal application."
- **Textual key-features slide** (verbatim list):
  1. **Rich widget library** — buttons, inputs, tables, trees, text-areas, etc., out of the box.
  2. **Flexible layout manager** — grid, vertical, horizontal, dock; CSS-style sizing.
  3. **Reactive attributes** — declarative state that auto-updates the UI on change. Vue/Svelte-shaped mental model.
  4. **"CSS-like" styling** *(speaker's exact wording — qualifier deliberately added to not scare the audience)* — `.tcss` files. A subset/dialect of CSS adapted for terminal rendering: selectors, properties, pseudo-classes, live-reload. Not real-CSS; you don't need a frontend skillset to use it.
  5. **Async event handling** — built on `asyncio`; handlers can be sync or async without ceremony.
  6. **Command palette** — Ctrl+P fuzzy command search, extensible per-app. Modern dev-tool ergonomic baseline (VS Code, Sublime, etc.).
  7. **Accessibility features** — screen-reader-friendly, focus management, keyboard navigation as first-class.
  8. **Cross-platform** — Linux / macOS / Windows; the framework abstracts terminal capability differences.
  9. **Remote app support** — `textual serve` / `textual-web` to deploy your TUI as a web app without code changes. The "terminal is the new browser" pillar.
  10. **Integration with Rich** — Textual builds on Rich; you can drop Rich renderables (tables, panels, markdown) directly into Textual widgets. The two libraries are explicitly designed to compose.
  - Solid checkbox-sweep of "what does a serious application framework need?" Every item is something a 2020-era TUI couldn't claim cleanly. Textual is positioned not as a niche tool but as the equivalent of *PySide / Qt / Electron* for the terminal.
- **Smallest Textual app — hello world.** Standard pedagogical move. Canonical shape:
  ```python
  from textual.app import App
  from textual.widgets import Label

  class HelloApp(App):
      def compose(self):
          yield Label("Hello, World!")

  if __name__ == "__main__":
      HelloApp().run()
  ```
  ~7 lines. Shows the three core primitives: subclass `App`, yield widgets in `compose()`, call `.run()`. **Lower bar than a Flask "hello world."** Brett's "the cost of starting is essentially nothing" pitch made tactile.
- **Layouts & widgets — the screen-level architecture.** Three-tier hierarchy:
  - **`App`** — top-level container. Runs the event loop, holds global state, owns the screen stack. One per process.
  - **`Screen`** — a "page" within the app. You push/pop screens like a stack (or a browser-history shape). Each Screen has its own composed widget tree and event handlers.
  - **`ModalScreen`** — a Screen that *overlays* the current one without unmounting it. Dialog boxes, confirm prompts, settings modals. The underlying screen keeps state and gets refocused on dismiss.
  - **Analog to web shapes:** `App` ≈ the SPA shell; `Screen` ≈ a route/page; `ModalScreen` ≈ a `<dialog>` element. Same compositional pattern, terminal-native. Pushing/popping screens is how you build multi-view apps (e.g., a settings screen, a detail view, a login flow) without re-running the whole app.
- **Container types (layout primitives):**
  - **`Container`** — generic container; children arranged by CSS rules.
  - **`Horizontal`** — children laid out left-to-right (≈ flexbox row).
  - **`Vertical`** — children laid out top-to-bottom (≈ flexbox column).
  - **`Grid`** — grid-style 2D layout (≈ CSS Grid). For dashboards, forms, tabular UIs.
  - **`ScrollableContainer`** — overflow handling; content larger than viewport scrolls.
  - **Mental model:** these are the terminal-native analogs of HTML's `<div>` + flex / grid CSS. You nest containers to compose any layout; CSS rules govern sizing, gaps, alignment. Same skill that ports cleanly from frontend work.
- **Built-in widget catalog** (selected highlights):
  - **`DataTable`** — sortable, scrollable tabular data display.
  - **`Tree`** — hierarchical / collapsible tree widget. File browsers, taxonomy navigators.
  - **`ListView` / `ListItem`** — vertical list of selectable items; the "menu" primitive.
  - **`Markdown`** — render Markdown inline (uses Rich underneath for formatting).
  - **`Pretty`** — renders any Python object via Rich's pretty-printer. Great for debugging tools / data inspectors.
  - **`Log`** — append-only scrolling log view; useful for tail-style outputs.
  - **`ProgressBar`** — animated progress indicator with percentage / ETA support.
  - Plus the standard input set (`Input`, `Button`, `Switch`, `Checkbox`, `RadioSet`, `Select`, `TextArea` etc., not enumerated this slide). **Together it's a complete enough widget library to build a real app without reinventing primitives.** The "rich widget library" feature-list bullet from earlier made concrete.
- **"What is a widget" in Textual: *the fundamental building block of the user interface.*** Standard definition. Mental-model parallel: same role as React components, Vue components, or HTML elements — composable units with their own state, styling, events, and child widgets.
- **`ListView` is hard to implement** (speaker's aside). Sounds offhand but worth marking — scrollable, selectable, keyboard-navigable lists with custom item rendering are *consistently* one of the harder widgets to build well in any UI framework (web, native, or TUI). Edge cases pile up: focus management, partial visibility, large-dataset virtualization, dynamic item heights. **Having it built-in is one of the points where Textual saves real time** over rolling your own; the same widget in raw curses is genuinely a week of work.
- **`ContentSwitcher`** also mentioned. Widget that toggles between alternative child views in the same region. Use cases: tab content without visible tab bar, wizard step containers, conditional renderings driven by reactive state. The "show one of N views" primitive — same role as React Router outlets or Vue `<KeepAlive>` + dynamic component.
- **Events in Textual** — async signals triggered by user actions (keypress, click, button-pressed) or system changes (mount, resize, focus). Handled via **`on_<event_name>`** method prefixes — convention-driven auto-wiring:
  ```python
  class MyApp(App):
      def on_button_pressed(self, event: Button.Pressed) -> None:
          self.log("Button was clicked")

      def on_input_changed(self, event: Input.Changed) -> None:
          self.log(f"Input is now: {event.value}")

      async def on_key(self, event: Key) -> None:    # can be async
          if event.key == "q":
              self.exit()
  ```
  - Handlers can be **sync OR async** — no decoration required, the framework introspects and awaits appropriately.
  - **Event bubbling**: events from child widgets propagate up to parents (like the DOM). You can intercept at any level.
  - Convention-based wiring is the *default*; the explicit `@on(Button.Pressed, "#my-id")` decorator is the alternative for fine-grained filtering (e.g., react only to a specific button by CSS selector). Same pattern as Qt's signal/slot system, just with Python idioms.
  - Standard events worth knowing: `Button.Pressed`, `Input.Changed`, `Input.Submitted`, `Key`, `Mount`, `Unmount`, `Resize`, `Focus`, `Blur`, `Click`. Each widget defines its own event types as inner classes — `Button.Pressed` is the "Pressed event of a Button," type-checked when you destructure `event`.
- **`@on(Button.Pressed)` decorator example.** Andres now shows the explicit alternative to convention-based handlers:
  ```python
  from textual.app import App, on
  from textual.widgets import Button

  class MyApp(App):
      def compose(self):
          yield Button("Save",   id="save")
          yield Button("Cancel", id="cancel")

      @on(Button.Pressed, "#save")
      def handle_save(self, event: Button.Pressed) -> None:
          self.log("Saved")

      @on(Button.Pressed, "#cancel")
      def handle_cancel(self, event: Button.Pressed) -> None:
          self.log("Cancelled")
  ```
  - **When you reach for `@on` over `on_button_pressed`:**
    - Multiple buttons (same event type), need to dispatch by ID / class / selector
    - Want the handler name to describe the action (`handle_save`) instead of the event (`on_button_pressed`)
    - Need to register multiple handlers for one event without method-overloading gymnastics
  - **CSS-selector filtering is the key feature.** `"#save"` matches by ID; you can also use classes (`".primary"`) and types (`Button.warning`). Same selector syntax as Textual's CSS — one mental model for both styling and event routing.
- **Reactivity in Textual — data binding.** Now on Textual's reactive-state model.
  - **Reactive attributes** declared as class-level descriptors. Change the attribute, the UI updates automatically. No manual repaint logic, no `self.refresh()` plumbing for state changes.
  ```python
  class Counter(Widget):
      count = reactive(0)
      def render(self) -> str:
          return f"Count: {self.count}"
  ```
  - **Hooks for reacting to changes:**
    - `watch_<name>(old_value, new_value)` — called automatically when a reactive attribute changes. Side-effect / logging / cascading update territory.
    - `compute_<name>()` — derived/computed values; re-runs when dependencies change. Vue's `computed` analog.
    - `validate_<name>(value)` — coerce or reject incoming values before they're set.
  - **Mental model = React/Vue/Svelte/SwiftUI** — declarative state, framework owns the diffing and rendering. Once the audience sees this, the "TUI = browser-shaped app architecture" thesis fully lands.
- **CSS in the terminal** — Textual's styling system, deeper this time.
  - **`.tcss` files** — Textual CSS, lives separately from your Python code. Or **`DEFAULT_CSS`** as a class attribute (string) for inline styling.
  - **Selectors:** widget types (`Button`), IDs (`#save`), classes (`.primary`), pseudo-classes (`:hover`, `:focus`, `:disabled`), nesting / descendant combinators. Same vocabulary as web CSS minus some web-specific bits.
  - **Properties** that work: `color`, `background`, `border`, `padding`, `margin`, `width` / `height` (in cells or percentages), `dock` (anchor to a side), `layout` (vertical / horizontal / grid), `align`. Subset of CSS adapted for the cell-grid coordinate system of a terminal.
  - **Variables**: `$primary`, `$accent`, etc. — same `$name` syntax as Sass. Theme tokens that flow through the stylesheet.
  - **Specificity** rules borrowed from CSS too — id > class > type, with `!important` available. If you've done CSS, the rules transfer.
  - **Live reload** during development — change the `.tcss` while the app is running and the UI re-styles instantly. Same dev ergonomics as a hot-reloading frontend.
  - **The "CSS-like" qualifier from earlier explained:** it's a real, expressive subset, but it's not pixel-perfect web CSS. Some web properties don't make sense in a cell grid (e.g., float, gradients with full pixel control), and some terminal-specific properties don't exist in web (e.g., `dock` directly anchors a widget to a screen edge). **Same skill, slightly different vocabulary.**
- **Synthesis slide: *"What makes Textual so powerful."*** Five pillars:
  1. **Reactive state** — declarative UI bound to attribute changes; framework owns rendering.
  2. **Async support** — `asyncio`-native; handlers can be sync or async; workers for background tasks.
  3. **Keyboard-first UX** — navigation, focus, command palette, keybindings as first-class concerns. Terminal users expect to drive everything from the keyboard; Textual delivers.
  4. **Composable widgets** — small primitives nest into complex UIs; `compose()` yields children; CSS controls layout. Same compositional logic as React/Vue.
  5. **Testable logic** — snapshot tests, simulated input, event injection. TUIs historically had no test story; Textual ships one. Big maturity signal.
  6. **Mouse support** — yes, even mouse. Clicks, scrolls, drags are first-class events. Surprises people who assume "terminal = keyboard-only."
  7. **Web deploy** — `textual serve` runs your TUI as a web app. Same code, different surface. The "terminal is the new browser" mechanic in two words.
  8. **SSH deployment** — SSH into a server, run the app remotely; the terminal as a *naturally remote* substrate. No web stack required for hosted apps.
  9. **Database access** — async DB drivers integrate cleanly via the asyncio event loop. `asyncpg`, `aiomysql`, `aiosqlite` drop straight into a Textual app. Real data layer in a TUI.
  10. **Auth-capable apps** — reactive state + screens + modals make login flows, session management, credential dialogs tractable. Textual apps can do real auth, not just toy demos.
  - **Cross-cutting observation:** every bullet maps to something a modern web/native framework also offers. Textual is positioning itself not as a *terminal-specific* niche tool but as a *general-purpose application framework that happens to render to the terminal*. The "what makes it powerful" answer is essentially "everything you expect from a serious UI framework — and your runtime is a terminal."
  - The mouse + web-deploy + SSH bullets together cement the "new browser" reframe most explicitly: you have **all the same interaction modalities** (keyboard, mouse, network) and **all the same deployment surfaces** (local, web, remote) — without the browser tax.
