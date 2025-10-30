# Relog-ASM Engine & DSL — LLM Brief (Concise & Complete)

**Goal:** Generate **valid Relog-ASM DSL** that compiles to the current `asm-core` ISA and runs on the VM.

---

## 0) Hard Constraints (must-follow)

* **No prelude.** You **must** `use` every module **before first call**, including built-ins (`log`, `env`, `dotenv`, `b64`, `is`, `kv`, …) and external (`fs`, `http`, `sh`, `rand`, `math`, `str`, `eval`, `llm`, …).
* **No infix operators.** Never write `a + b`, `x * y`, etc. Arithmetic is **command-only** with **in-place** semantics: `add|sub|mul|div|rem {dst,rhs}`, `neg {dst}`.
* **Be strict.** If something would require guessing (e.g., missing `use`, infix), treat it as a **generation error**—rewrite to valid forms.

---

## 1) Runtime Values

Types: `Unit | Bool | I64 | F64 | Str | Bytes(Vec<u8>) | Json(serde_json::Value) | Future(u64) | FutureState`.

* Uninitialized vars → `Unit`.
* Mixed numeric ops promote to `F64`.
* `FutureState`: `Pending | Ready | Canceled | Consumed | NotFound | Failed`.

---

## 2) ISA (instruction set)

* **Vars:** `Let {dst,value}`, `Mov {dst,src}`.
* **Flow:** `Label {name}`, `Jmp {label}`, `JmpIf {cond,label}`, `JmpIfNot {cond,label}`, `Halt`.
* **Calls:**

  * Sync: `Call {name,arg,dst?}`.
  * Async: `CallAsync {name,arg,dst}` → saves `Future`; `Await {future,dst}`.
* **Futures:** `FutureStatus {future,dst}`, `FutureCancel {future}`, `FutureSelect {futures, dst_index, dst_value?, timeout_ms?}`.
  *No timeout ⇒ non-blocking poll; `idx = -1` if none ready. Timeout ⇒ `idx = len(futures)`.*
* **Traps:** `Trap {handler_label, err_dst?}`, `Untrap`. On error: jump; store error string if `err_dst` set.
* **JSON:** `JsonParse {dst,src}`, `JsonStringify {dst,src}`, `JsonGet {dst,path,json}`, `JsonLen {dst,json}`.
* **Arithmetic:** `Add|Sub|Mul|Div|Rem {dst,rhs}`, `Neg {dst}` (in-place on `dst`).
* **Compare:** `Cmp {dst,op,a,b}`, `op ∈ {Eq,Ne,Lt,Le,Gt,Ge}` → `Bool`.

### Semantics (arithmetic & compare)

* `I64`×`I64` ops wrap on edges (`I64::MIN / -1 → I64::MIN`).
* Mixed `I64`/`F64` ⇒ compute as `F64`.
* `add(Str,Str)` concatenates; other cross-type arithmetic is an error.
* Comparisons: all numeric types (mixed coerced to `F64`); `eq/ne` also for `Bool`, `Str`, `Bytes`, and `Unit==Unit`. `FutureState` supports `eq/ne`.

### JSON intrinsics

* `json.parse`: accepts `Str | Json` ⇒ `Json`.
* `json.stringify`: any value ⇒ **JSON text** `Str`.
  `Bytes` → base64 string.
  **If input is a `Str` that itself is valid JSON, it is parsed then re-emitted as normalized JSON text** (not quoted). Use a raw string for literal body text when needed.

---

## 3) Module System

* **Import required:** `use <module> [{…}] [as <alias>]`. Options are a JSON object; `as` sets a short prefix.
* **Name resolution:** longest-prefix before last dot. `a.b.c.m` tries `a.b.c`, then `a.b`, then `a`.
* **Default method:** calling a prefix without method uses `"call"`.
* **Always has an argument:** if omitted in DSL it becomes `Unit`.

### Core built-ins (in `asm-core`)

* `log`: `trace|debug|info|warn|error <Value> -> Unit`
* `env`: `get <Str> -> Str|Unit`
* `dotenv`: `load <Unit|StrPath> -> Bool`, `get <Str> -> Str|Unit`
* `b64`: `encode <Str> -> Str(base64)`, `decode <Str> -> Str(utf-8)`
* `is`: `unit|str|bool|i64|f64|bytes <Value> -> Bool`
* `kv`: `set "key=value" -> Bool(true)`, `get "key" -> Str|Unit`

*(External crates expose additional modules: `fs`, `http`, `sh` (method `run`), `rand`, `math`, `str`, `eval`, `llm`, …—still require `use`.)*

---

## 4) DSL (surface syntax you must emit)

### Imports & structure

* `use <module> [{…}] [as <alias>]`
* `import "path/file.rasm" as my` — inlines file; names/labels become `my::…`.
* `mod <name> { ... }` — nested block; internal names are `<name>::…`.

### Labels & jumps

* Define `:LABEL` (inside modules → `ns::LABEL`).
* Jump targets may be written with or without leading `:`.

### Vars & literals

* `let x = <expr>`; atoms: identifiers, numbers, booleans, strings, `null`, JSON objects/arrays.
* **Strings:** normal `"..."` and raw `r"..."`, `r#"..."#`, `r##"..."##`, …
  Raw strings are multiline; no escapes; number of `#` must match.

```rasm
let a = r"line1
line2"
let b = r#"He said: "ok""#
let c = r###"quote + hash: "#""###
```

> // In raw strings, `\n` stays as the two characters `\` and `n`. JSON parsers handle escapes only when you later `json.parse`.

### Calls & async (the only allowed forms)

* Sync: `call mod[.method] <arg?>`
* Async starts a future: `let f = async call mod.method <arg?>`
* Await a future: `let x = await f`
* Inline start+await: `let x = await call mod.method <arg?>`
* Bare `await` waits **$last_future**: `let x = await` (errors if none).

**Await errors:** `TypeError(NonFuture)`, `AwaitMissingFuture`.

### Future selection

```
select (f1,f2,...) -> (idx[,val])        # non-blocking poll if no timeout
select (f1,f2,...) -> (idx[,val]) timeout 5000
```

* No ready ⇒ `idx = -1`. Timeout ⇒ `idx = number_of_futures`.

---

## 5) Patterns (good idioms)

```rasm
# Explicit capture of results
use http
use log

let r = await call http.get {"url":"https://example.org"}
call log.info r
```

```rasm
# Safe JSON body
use http

let r = await call http.post {"url":"…","json":{"a":1,"b":true}}
```

```rasm
# In-place arithmetic (no infix)
use log

let a = 10
let b = 17
let sum = a        // copy a
add sum b          // sum = sum + b
call log.info sum
```

```rasm
# Trap for error handling
use http
use log

trap :ON_ERR -> e
let x = await call http.get {"url":"…"}
jmp -> :CONT
:ON_ERR
call log.error e
:CONT
```

```rasm
# Non-blocking poll of several futures
let i = 0
select (f1,f2) -> (i,v)
cmp none eq i -1
```

```rasm
# Timed wait for any of many
select (f1,f2,f3) -> (i,v) timeout 5000
cmp ok lt i 3
```

```rasm
# Raw JSON text for modules expecting text bodies
use http
let r = await call http.post {"url":"…","body_utf8": r#"{"a":1}"#}
```

---

## 6) Pitfalls (avoid)

* **Never emit infix operators.** Only `add|sub|mul|div|rem|neg` with **in-place dst**.
* **Always import modules.** Any `call X.*` without prior `use X` is invalid.
* Avoid bare `await` when multiple futures are in flight; it binds to `$last_future`.
* `json.len` expects `Json(Array|Object)`.
* `json.stringify` on a `Str` containing JSON **normalizes** to JSON text; use `"body_json"` or a raw string for `"body_utf8"` if you need literal text.

---

## 7) Style Guide

* Group all `use …` at the top (built-ins and externals).
* Prefer explicit `let` bindings for results and futures.
* For arithmetic, use an accumulator (`let sum = a; add sum b`) or mutate the left operand (`add a b`).
* Labels: `:ENTRY` conventional for entry; use UPPER_SNAKE or TitleCase.
* Pass structured args as JSON objects; keep method names precise (`sh.run`, not `exec`).

---

## 8) Output Policy

* When asked to generate code, output **DSL only**; optional comments must be `//` and minimal.
* **Self-check before output:**

  1. Every `call X...` has a matching prior `use X` (or its alias).
  2. No infix operators outside string literals/comments.
  3. Arithmetic uses command form with in-place `dst`.
  4. Async forms are only those listed; `await` targets a real future.
  5. JSON args have correct shapes (e.g., `{"json": …}` or `{"path": …, "json": …}` for `json.get`).

---

## 9) WRONG vs RIGHT (quick reminders)

```rasm
# WRONG: missing use + infix
let a = 10
let b = 17
let sum = a + b
call log.info sum
halt
```

```rasm
# RIGHT: import + command arithmetic
use log
let a = 10
let b = 17
let sum = a
add sum b
call log.info sum
halt
```

```rasm
# WRONG: calling built-in without use
call log.info "hello"
```

```rasm
# RIGHT: explicit use
use log
call log.info "hello"
```

---

This brief reflects the **current** `asm-core` and `dsl` behavior. If runtime behavior diverges, treat it as an implementation bug, not a spec change.
