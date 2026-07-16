# Harmony

# **Developed by The Goverment of Nomashae and The Ministry of Technology of The Republic of Nomashae**

https://github.com/user-attachments/assets/fa028d62-53a9-431b-b3f0-b102f1432d8f

# Description

**A small, indentation-based language for building GUI apps with strict state/view separation, compiled to real running HTML/JS.**

Harmony is a from-scratch language: its own lexer, parser, AST, and semantic
analyzer, feeding a pluggable codegen backend. This repo is the **Phase 1
reference implementation** with a real compiler, not a mockup and it is verified by
compiling example apps and testing the output with simulated clicks.

```harmony
store CounterStore:
    state count: Int = 0

    action increment:
        count += 1

component CounterApp:
    bind count from CounterStore.count

    Column(padding: 16, spacing: 12):
        Text(text: f"Count: {count}", style: "heading")
        Button(label: "+", on_click: emit CounterStore.increment)

app CounterDemo:
    window(title: "Counter", width: 320, height: 200):
        mount CounterApp
```

Compile it, open the result in a browser, click the button. That's the whole loop.

---

## Why Harmony

Most UI frameworks bolt state management onto a general-purpose language as a
library or convention (hooks, stores, reducers, all opt-in, all
enforceable-by-discipline-only). Harmony makes the separation a **grammar
rule**:

- `store` / `action` is the only place state can change.
- `component` bodies can only *read* state (via `bind`) and *declare*
  layout, there is no statement syntax reachable from inside a component,
  so "the view mutated global state" isn't a bug you can write, even by accident.
- `emit Store.action(...)` is the one bridge between the two and it's
  checked at compile time: reference a store or action that doesn't exist
  and you get a `SemanticError` with a line number, not a runtime crash.

---

## Quickstart

**Requirements:** Node.js 16+

```bash
git clone https://github.com/nomashae/harmony/
cd harmony_source
node bin/harmony.js build examples/counter_app.harm --out counter_app.html
open counter_app.html   # or just double-click it
```

Compiles in well under a second there's no bundling, no type inference, no
optimization passes. It's a straightforward tree-walk over a small AST.

---

## CLI

```bash
node bin/harmony.js build <file.harm> --out <file.html>   # compile to a runnable app
node bin/harmony.js tokens <file.harm>                     # dump the lexer's token stream
node bin/harmony.js ast <file.harm>                        # dump the parsed AST as JSON
```

`tokens` and `ast` exist for debugging your own `.harm` files if `build`
throws a confusing error, `ast` usually makes it obvious which node the
parser built wrong.

---

## How it works

```
 .harm source
      │
      ▼
   Lexer          hand-written scanner; converts indentation into
      │           explicit INDENT/DEDENT/NEWLINE tokens
      ▼
   Parser         recursive-descent + Pratt expression parser
      │
      ▼
     AST
      │
      ▼
Semantic Analyzer  builds the "State Ownership Graph" maps every
      │            state field to its owning store, enforces that
      │            components never write to one directly
      ▼
  Codegen Backend   walks the *validated* AST only; currently one
      │             backend implemented: HTML/JS (Backend #2
      │             "transpile to JS for web/Electron deployment")
      ▼
 self-contained .html file, runnable in any browser
```

The AST is the contract between stages, the codegen backend never sees raw
Harmony syntax, only validated, ownership-checked nodes. That's what makes a
second backend (native runtime, WASM, whatever) additive rather than a rewrite.

## Project structure

```
harmony_source/
├── bin/
│   └── harmony.js          CLI entry point
├── src/
│   ├── lexer.js             tokenizer + indentation handling
│   ├── parser.js            recursive-descent/Pratt parser → AST
│   ├── semantic.js          State Ownership Graph + validation
│   └── codegen_html.js      AST → HTML/JS backend + embedded runtime
├── examples/
│   ├── counter_app.harm
│   └── calculator.harm
└── README.md
```

---

## Language at a glance

| Concept | Keyword(s) | Example |
|---|---|---|
| State container | `store`, `state`, `action` | `store S: state x: Int = 0` |
| View | `component`, `bind` | `component V: bind x from S.x` |
| State → view bridge | `emit` | `on_click: emit S.increment` |
| Entry point | `app`, `window`, `mount` | `app A: window(...): mount V` |
| Built-in elements | - | `Text`, `Button`, `Column`, `Row`, `Slider`, `Input`, `Image` |

Full syntax reference, operator precedence table, and the complete list of
built-in element props: see [`harmony_cheat_sheet.md`](./harmony_cheat_sheet.md).

---

## Status: Phase 1

This is an early, functional slice, not a complete language. What's real:

- Full lexer with correct indentation and multi-line bracket continuation
- Parser covering `store`/`component`/`app`, control flow, f-strings, lambdas
- Compile-time enforcement of the state-ownership rule
- A working HTML/JS backend with a reactive `Store` (subscribe/publish)
- Verified end-to-end: generated apps tested with simulated DOM events

What's not built yet:

- No native/Electron renderer, only the browser/HTML backend exists
- No conditional rendering inside components (`if` around an element)
- No list/map indexing
- Top-level `fn` declarations parse but aren't wired into codegen
- Re-render is whole-subtree replacement, not a fine-grained VDOM diff
- No static type checking, no module system

See the cheat sheet's "Known Phase 1 gaps" section for the full list.

---

## Roadmap

1. Conditional rendering + list rendering inside components
2. A real VDOM diff instead of subtree replacement
3. A second codegen backend (native or WASM) to validate the "pluggable
   backend" architecture end-to-end
4. Multi-file module system
5. Optional static type checking

---

## Contributing

This is a Phase 1 reference implementation, so the most useful contributions
right now are: example `.harm` programs that find parser/codegen gaps, and
small, focused PRs against a single pipeline stage (lexer, parser, semantic
analyzer, or codegen) rather than cross-cutting changes.

## License
MIT - Refer to License.
