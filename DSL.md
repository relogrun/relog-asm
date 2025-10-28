# RelogASM DSL — quick reference

## Core DSL

- `use <module> [as <alias>]` — import a module, optionally with an alias.
- Labels & jumps: `:ENTRY`, `jmp -> :LABEL`, `jmpif <cond> -> :LABEL`, `jmpifnot`.
- Comparisons: `cmp <cX> <op> <a> <b>` → `<op>`: `gt|ge|lt|le|eq|ne`.
- Variables: `let x = <value>`.
- Arithmetic (in-place): `add|sub|mul|div|rem|neg x <y>`.
- Calls: `call <mod>.<method> <arg>`.
- Async: `async call …` returns a `Future`; await with `await <future>`.
- Comments: `// …`.
- Strings: `"..."` and raw multi-line `r" ... "`.
- Value types: `Unit, Bool, I64, F64, Str, Bytes, Json, Future`.
- File imports: `import "path/to/file.rasm" as <alias>` — include another `.rasm`.
- Module declaration: `mod <name> { ... }` — define an inline module. Access via `<name>::<var or label>`.
- JSON ops:  `json.parse <str> -> Json`, `json.stringify <value> -> Str`, `json.get {"json": <val>, "path": "<dotpath>"}`, `json.len <json|array|map> -> I64`.

---

## Modules & methods

### `log`

- `call log.trace|debug|info|warn|error <Str>`

### `env`

- `call env.get "KEY"` → `Str|Unit`
- `call env.set {"KEY":"value"}` → `Unit`
- `call env.exists "KEY"` → `Bool`

### `dotenv`

- `call dotenv.load` or `call dotenv.load ".env"`

### `is` (type checks)

- `call is.unit|str|bool|i64|f64|bytes <Value>` → `Bool`

### `b64`

- `call b64.encode <Str|Bytes>` → `Str` (base64)
- `call b64.decode <Str>` → `Bytes`

### `str` 

- `call str.len <Str>` → `I64`
- `call str.lower|upper|trim <Str>` → `Str`
- `call str.replace {"text":"t","from":"a","to":"b","all":true}` → `Str`
- `call str.split {"text":"a,b","by":","}` → `Json(Array)`
- `call str.join {"parts":["a","b"],"sep":","}` → `Str`

### `fs`

- `call fs.read_text <path>` → `Str`
- `call fs.write_text {"path":"...","text":"...","create_dirs":true}` → `Unit`
- `call fs.read_bin <path>` → `Bytes`
- `call fs.write_bin {"path":"...","bytes":<Bytes>,"create_dirs":true}` → `Unit`
- `call fs.exists <path>` → `Bool`
- `call fs.mkdirs <path>` → `Unit`
- `call fs.list_dir <path>` → `Json(Array<Str>)`

### `http` (async)

- `await call http.get {"url":"...","headers":{...},"timeout_ms":5000}`
- `await call http.post {"url":"...","json":{...}}`
- `await call http.download {"url":"...","to":"path"}`

  - Responses → `Json({"status":I64,"headers":{...},"text":Str,"bytes_base64":Str})`

### `sh` (async; policy set at init)

- `await call sh.exec {"cmd":"ffmpeg","args":["-v","error"],"cwd":".","env":{...},"timeout_ms":60000}`

  - Returns → `Json({"code":I64,"stdout_base64":Str,"stderr_base64":Str})`

### `rand`

- `call rand.i64 {"min":0,"max":100}` → `I64`
- `call rand.f64` → `F64`
- `call rand.bytes {"len":32}` → `Bytes`
- `call rand.uuid` → `Str`

### `math`

- `call math.add|sub|mul|div {"a":X,"b":Y}` → `I64|F64`
- `call math.round {"x":F,"digits":2}` → `F64`

### `eval` 

- `await call eval.dsl {"code": <Str>, "exports":["x"], "allow":["log"], "strict_await": true}` → `Json`
- `await call eval.file {"path":"script.rasm","exports":[...],"allow":[...]}`

### `llm` (async)

- `await call llm.chat {"model":"...","messages":[{"role":"user","content":"..."}],"temperature":0.7}`

  - Returns → `Json({"text":Str,"usage":{...}})`

