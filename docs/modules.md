# RelogASM Modules — Compact API

Type notation: `Unit | Bool | I64 | F64 | Str | Bytes | Json`.
Calls: synchronous via `call`, async via `async call` + `await`.
Module errors look like `module error: <mod>: ...` and can be caught with `trap/untrap`.

---

## fs — sandboxed filesystem (async)

### Use

```
use fs {
  root: ".",              // jail root
  read_only: false,       // forbid writes if true
  max_read_bytes: 2097152,
  max_write_bytes: 2097152,
  max_entries: 100000,
  default_timeout_ms: 10000,
  max_timeout_ms: 30000
}
```

### Methods

* `await call fs.read {path, encoding?, max_bytes?, timeout_ms?}` →
  `Str` if `encoding:"utf8"`, otherwise `Bytes`.
* `await call fs.write {path, data_utf8? | data_base64?, append?, create?, truncate?, timeout_ms?}` → `Bool`.
* `await call fs.list {path, glob?, max?, sort?, timeout_ms?}` →
  `Json {entries:[{name,is_dir,size,mtime}], truncated}` where `sort` is `"name"` (default) or `"mtime"`.
* `await call fs.stat {path, timeout_ms?}` → `Json {exists,is_dir,size,mtime}`.
* `await call fs.exists {path, timeout_ms?}` → `Bool`.
* `await call fs.mkdirp {path, timeout_ms?}` → `Bool`.
* `await call fs.remove {path, is_dir?, timeout_ms?}` → `Bool`.

### Policy / constraints

* Paths must be relative; absolute paths and `..` are rejected.
* Timeouts are clamped to `[default_timeout_ms, max_timeout_ms]`.

**Example**

```
// Write -> Read (utf8)
use fs { root: ".", read_only: false }
let _ = await call fs.write {"path":"tmp/hi.txt","data_utf8":"Hi","create":true}
let s = await call fs.read {"path":"tmp/hi.txt"}
```

---

## http — HTTP client (async)

### Use

```
use http
```

### Method

* `await call http.req { ... }` → `Json {status, headers:{..}, body_base64, url?}`

**Args**

* `method?`: `"GET"|"POST"|...` (default `GET`)
* `url`: `https://...` (**http:// is blocked** unless `allow_insecure:true`)
* `headers?`: `{"K":"V"}`
* `body_utf8?` | `body_base64?`
* `timeout_ms?` (default 10000), `max_redirects?` (default 5)
* `accept_non_2xx?` (default `false` → non-2xx is an error)
* `allow_insecure?` (default `false`)
* `max_body_bytes?` (error if body larger)

**Example**

```
// GET and decode body
use http
use b64
let r   = await call http.req {"url":"https://httpbingo.org/json","accept_non_2xx":true}
let b64 = call json.get {"json": r, "path": "body_base64"}
let body = call b64.decode b64
```

---

## llm — chat completions (async)

### Use

```
use llm
```

### Methods

* `await call llm.chat { ... }` → `Str` (extracted text).
* `await call llm.chat_raw { ... }` → `Str` (raw HTTP response text).

**Core args**

* `endpoint`: base URL; normalized to `.../chat/completions` if needed.
* `model?`: string (required unless `body_override_b64` is provided).
* `messages?`: `[{role, content? | content_b64?}]` **or** `messages_b64` (base64(JSON array)).
* `temperature?`: `f32`.

**HTTP / auth**

* `method?` (default `POST`), `timeout_ms?` (default `30000`), `headers?`.
* `api_token?` (direct token) | `api_env?` (env var name).
* `auth_header?` (default `"Authorization"`), `auth_prefix?` (default `"Bearer "`).

**Extras**

* `body_override_b64?`: base64(JSON) — replaces request body entirely.
* `response_text_path?`: dot-path to extract text (default `"choices.0.message.content"`).
* `response_format?`: JSON (OpenAI-compatible passthrough).

**Behavior**

* Non-2xx → error. Cooperative cancel supported.

**Example**

```
// Minimal OpenAI-compatible call
use llm
let ans = await call llm.chat {
  "endpoint": "https://api.openai.com/v1",
  "model": "gpt-4o-mini",
  "messages": [{"role":"user","content":"hello"}],
  "api_env": "LLM_API_KEY",
  "timeout_ms": 15000
}
```

---

## math — numeric helpers (sync)

### Use

```
use math
```

### Methods (sync)

* Unary:
  `call math.abs (I64|F64)` → `I64|F64` (`I64` uses `wrapping_abs`)
  `call math.sqrt (I64|F64)` → `F64` (error on negative `I64`)
  `call math.floor|ceil|round (I64|F64)` → `I64|F64` (`I64` is no-op)
  `call math.sin|cos|tan (I64|F64)` → `F64` (radians)
* JSON args:
  `call math.min {"a":num,"b":num}` → `I64|F64`
  `call math.max {"a":num,"b":num}` → `I64|F64`
  `call math.clamp {"x":num,"lo":num,"hi":num}` → `I64|F64` (error if `lo>hi`)
  `call math.pow {"x":num,"p":num}` → `F64`

---

## rand — randomness (sync)

### Use / config

```
use rand {
  "engine": "thread" | "chacha",  // default "thread"
  "seed": 42,                      // u64 | "123" | "0xFF"; forces deterministic ChaCha
  "max_bytes": 1048576             // cap for rand.bytes
}
```

### Methods (sync)

* `call rand.i64 {"lo":I64,"hi":I64,"inclusive":Bool?}` → `I64`
  (`inclusive:false` requires `lo < hi`; `true` allows `lo <= hi`).
* `call rand.f64 {"lo":F64,"hi":F64}` → `F64` in `[lo, hi)`.
* `call rand.bytes {"len":usize}` → `Bytes` (error if `len > max_bytes`).
* `call rand.choice {"items": JsonArray}` → `Value` (element converted to `Value`).
* `call rand.shuffle {"items": JsonArray}` → `JsonArray` (shuffled copy).
* `call rand.seed (I64|Str)` → `Bool` (reseeds **only** for `engine:"chacha"`).

---

## sh — sandboxed shell (async)

### Use / policy

```
use sh {
  "cwd_root": ".",                 // jail for cwd
  "allowed_cmds": ["echo","cat"],  // names only; no slashes/spaces
  "max_output_bytes": 1048576,     // cap per stream
  "default_timeout_ms": 10000,
  "max_timeout_ms": 30000,
  "search_paths": ["/usr/bin","/bin","/usr/local/bin"],
  "freeze_path": true              // ignore PATH from request env
} as shell
```

### Method

* `await call shell.run { ... }` →
  `Json {exit, stdout_base64, stderr_base64, stdout_truncated?, stderr_truncated?}`

**Args**

* `cmd`: basename (must be in `allowed_cmds`)
* `args: [Str]?`
* `cwd?`: relative path under `cwd_root`
* `env?`: `{K:V}` (if `freeze_path:true`, PATH is overridden)
* `stdin_utf8?` | `stdin_base64?`
* `capture_stdout?` / `capture_stderr?` (default `true`)
* `timeout_ms?` (clamped by policy)

**Security**

* Strict allow-list; executable resolved via `search_paths`.
* `cwd` and paths are jailed under `cwd_root`.

---

## str — string utilities (sync)

### Use

```
use str
```

### Methods (sync)

* Unary on `Str`:
  `call str.len Str` → `I64` (Unicode scalar count)
  `call str.lower Str` → `Str`
  `call str.upper Str` → `Str`
  `call str.trim Str` → `Str`
* JSON args:
  `call str.contains {"s":Str,"pat":Str}` → `Bool`
  `call str.starts_with {"s":Str,"pat":Str}` → `Bool`
  `call str.ends_with {"s":Str,"pat":Str}` → `Bool`
  `call str.replace {"s":Str,"from":Str,"to":Str}` → `Str`
  `call str.split {"s":Str,"delim":Str}` → `JsonArray(Str)` *(non-empty `delim`)*
  `call str.join {"items":[Str...],"sep":Str}` → `Str`

---

## eval — nested DSL execution (async)

### Use

```
use eval
```

### Method

* `await call eval.dsl { ... }` → `Json {<exported vars...>}`

**Args**

* `code`: `Str` (DSL source)
* `vars?`: `JsonObject` (initial VM variables for the inner run)
* `exports?`: `[Str]` (variable names to return)
* `step_limit?`: `u64` (VM step cap)
* `allow?`: `[Str]` (module allow-list for the inner code; `"*"` allows all)

**Behavior**

* Registers modules referenced by the inner `use` statements, guarded by `allow`.
* Returns an object with requested exports (missing vars → `null`).
* `vars` are converted from JSON into VM `Value`s.

**Example**

```
// Nested DSL execution with export
use eval
let inner = r"
use log
let x = 2
add x 5
"
let res = await call eval.dsl {
  "code": inner,
  "exports": ["x"],
  "allow": ["log","math","str","*"]
}
```

---

## Quick cheats

* **Async only**: `fs.*`, `http.req`, `sh.run`, `eval.dsl`, `llm.chat*`.
* **Sync**: `math.*`, `rand.*`, `str.*`.
* **Safety**:

  * `http`: `http://` is blocked by default; use `allow_insecure:true` consciously.
  * `fs`: jailed root; no absolute paths or `..`.
  * `sh`: allow-listed commands only; jailed `cwd`; output/time caps.
* **JSON args**: when a module expects `Json`, pass an actual DSL JSON literal/object, not a stringified JSON (use `{"k":"v"}` or build via `json.parse/stringify`).
