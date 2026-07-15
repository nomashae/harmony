# Harmony "Cheat Sheet" (Publication 1)

Everything here reflects what the compiler actually does, not the aspirational
full-language spec. If something isn't listed, assume it's not implemented yet.

---

## Running the compiler

```
node bin/harmony.js build <file.harm> --out <file.html>    # compile & write a runnable app
node bin/harmony.js tokens <file.harm>                     # dump the raw lexer output
node bin/harmony.js ast <file.harm>                        # dump the parsed AST as JSON
```

`tokens` and `ast` are the debugging tools, if `build` gives a confusing error,
run `tokens` first to check the lexer saw what you expected, then `ast` to check
the parse tree shape.

---

## File shape

Every `.harm` program is some combination of `store` blocks, `component` blocks,
and exactly one `app` block. Order doesn't matter except that the `app` block
needs whatever component it mounts to already be declared somewhere in the file.

```harmony
store MyStore:
    ...

component MyView:
    ...

app MyApp:
    window(title: "...", width: 400, height: 300):
        mount MyView
```

---

## Indentation & comments

- 4 spaces per level (tabs are a hard error).
- `#` starts a line comment; blank lines are always ignored.
- A block always follows `:` + newline + indent, same shape everywhere
  (`if x:`, `store S:`, `Column:`, `action foo:`).
- An **unclosed** `(`, `[`, or `{` lets you spread a call across multiple
  physical lines without breaking anything:

```harmony
StepSlider(
    value: step,
    on_change: (v) => emit CounterStore.set_step(v)
)
```

---

## Variables

```harmony
let name = "Harmony"        # inferred
let age: Int = 0            # optional type annotation (documentation only, not enforced)
const PI = 3.14159          # parses, but nothing currently stops reassignment at runtime
```

Only usable **inside `action` bodies** right now (see "Where can I use statements?" below).

---

## Literals

| Kind | Syntax | Notes |
|---|---|---|
| Number | `42`, `3.14` | one numeric type, JS `number` under the hood |
| String | `"hi"`, `'hi'` | supports `\n \t \\ \" \'` escapes |
| F-string | `f"Count: {value}"` | `{expr}` can be any expression, not just a name |
| Bool | `True`, `False` | |
| Nil | `Nil` | compiles to JS `null` |
| List | `[1, 2, 3]` | parses; no indexing (`list[0]`) yet |
| Map | `{name: "Ana", age: 29}` | parses; no `map.get(...)` helpers yet |

---

## Operators

```
or                     # lowest precedence
and
== != < > <= >=
+ -
* /                    # highest precedence
not x    -x            # unary
```

Compound assignment: `= += -= *= /=` only legal as a statement (`x += 1`), not
mid-expression.

---

## Control flow **is only valid inside `action` bodies**

```harmony
if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
else:
    grade = "C"

for item in items:
    print(item)          # 'print' isn't wired to anything yet, it parses, doesn't run

while i < 5:
    i += 1
```

Warning **Component bodies cannot contain `if`/`for`/`while`/plain statements.** A
component body is only ever a `bind` list followed by a tree of UI elements.
This isn't a missing feature so much as the core rule of the language: view
code can't branch on or loop over anything you'd call "logic", that's what
`store`/`action` is for. (Conditional *rendering* — e.g. an `if` around an
element — is a known Phase 2 gap.)

## Functions

```harmony
fn add(a: Int, b: Int) -> Int:
    return a + b
```

This **parses correctly** but is currently a dead end in codegen — top-level
`fn` declarations don't get emitted into the compiled output yet. Don't rely
on calling a top-level `fn` from a store or component; put the logic straight
into an `action` for now.

---

## `store` — state + the only place mutation is legal

```harmony
store CounterStore:
    state count: Int = 0
    state step: Int = 1

    action increment:
        count += step

    action set_step(new_step):
        if new_step > 0:
            step = new_step
```

- `state` fields are the store's persistent data.
- `action` blocks are the **only** code path allowed to assign to a `state`
  field — the compiler checks this (`SemanticError`) if a component ever tries
  to write to a field name it recognizes as owned by a store.
- Every `state` write auto-triggers a re-render of any component subscribed
  to that store (see `bind` below) — you never call anything like `render()`
  yourself.

---

## `component` — view only

```harmony
component CounterDisplay(value):
    Text(text: f"Count: {value}", style: "heading")

component CounterApp:
    bind count from CounterStore.count

    Column(padding: 16, spacing: 12):
        CounterDisplay(value: count)
        Button(label: "+", on_click: emit CounterStore.increment)
```

- `component Name(params):` — params can have defaults: `fn(value, step = 1)`.
- `bind local from Store.field` — the *only* way a component reads store
  state. Read-only; there's no write-equivalent.
- The body after any `bind` lines is a tree of **elements**: either a
  built-in (table below) or another `component` you've declared, called the
  same way (`CounterDisplay(value: count)`).
- Elements that take children use a `:` block, same as everything else:

```harmony
Row(spacing: 8):
    Button(label: "-", on_click: emit CounterStore.decrement)
    Button(label: "+", on_click: emit CounterStore.increment)
```

- An element with no props can drop the parens entirely: `Row:` on its own.

---

## `emit` — the only way a view triggers a state change

Two forms, and they mean different things:

```harmony
on_click: emit CounterStore.reset              # bare — a REFERENCE, fires when clicked
on_click: (v) => emit CounterStore.set_step(v)  # call form — fires IMMEDIATELY when the lambda runs
```

Rule of thumb:
- **Zero-arg action, used directly as a prop** → bare form, no lambda needed.
- **Action that needs an argument from the event** → you must wrap it in a
  lambda (`(v) => emit Store.action(v)`), because `emit Store.action(v)` on
  its own evaluates *right now*, at render time, not on click.

---

## Built-in UI elements

| Element | Props | Notes |
|---|---|---|
| `Text` | `text`, `style` | `style: "heading"` adds CSS class `h-heading` |
| `Button` | `label`, `on_click` | |
| `Column` | `padding`, `spacing` | children stack vertically |
| `Row` | `spacing` | children stack horizontally |
| `Slider` | `min`, `max`, `value`, `on_change` | `on_change` value is coerced to a **Number** |
| `Input` | `value`, `on_change` | `on_change` value stays a **String** |
| `Image` | `src` | |

Any prop named `on_<event>` with a function value gets wired as a real DOM
event listener (`on_click` → `click`, `on_change` → `input`, anything else →
the literal suffix, e.g. `on_hover` → `hover`).

---

## `app` — the entry point

```harmony
app MyApp:
    window(title: "My App", width: 360, height: 240):
        mount MyView
```

- Exactly one `window(...)` per `app`, exactly one `mount <ComponentName>`.
- `title` is read at compile time to become the browser tab / titlebar text.
  `width`/`height` are currently cosmetic in the generated HTML — the window
  is sized by CSS, not enforced by any real OS-level window.

---

## Error types you'll actually see

| Error | Cause | Example |
|---|---|---|
| `LexError` | tabs, unterminated string, bad character | mixing tabs and spaces |
| `ParseError` | grammar violation | `Row:` block with mismatched indent |
| `SemanticError` | broken reference or illegal mutation | `bind x from GhostStore.y`, or `emit Store.no_such_action` |

All three include a line number. If `build` exits with code `1`, the message
on stderr tells you exactly which of the three stages caught it.

---

## Known Phase 1 gaps (so you don't chase phantom bugs)

- No conditional rendering inside a component (`if` around an element).
- No list/map indexing (`items[0]`, `myMap.key`).
- Top-level `fn` parses but isn't emitted by codegen yet.
- Re-render is "replace the whole component subtree," not a fine-grained diff.
- `const` doesn't actually prevent reassignment at runtime yet.
