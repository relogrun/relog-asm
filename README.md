# RelogASM (CLI)

RelogASM is a small assembly-style DSL and embeddable VM for composing deterministic pipelines from simple, typed instructions. 

## Quick start

1. **Download & unpack** the archive for your OS from [latest release](https://github.com/relogrun/relog/releases/latest):

2. **macOS/Linux:** make executable:

```bash
chmod +x relog-asm
```

1. **Run a file**:

```bash
./relog-asm path/to/script.rasm
```

### Hello world

`hello.rasm`:

```asm
use log
call log.info "hello from RelogASM"
```

Run:

```bash
relog-asm hello.rasm
```

### One more (async shell)

```asm
use sh
use json
use b64

let out = await call sh.exec {"cmd":"bash","args":["-lc","echo hi"]}
let s = call json.get {"json": out, "path":"stdout_base64"}
let txt = call b64.decode s
```

Full DSL reference: [DSL.md](./DSL.md)

---

## Config (optional)

Place `relog.config.toml` next to your script (or the current working dir):

```toml
[vm]
step_limit = 100000

[log]
filter = "debug"
```

CLI flags (e.g., `--step-limit`) override config values.

---

## Built-in modules (short list)

`log`, `env`, `dotenv`, `is`, `b64`, `str`, `fs`, `http` (async), `sh` (async), `rand`, `math`, `eval` (nested DSL), `llm_chat` (async), `json`.

A compact cheat-sheet lives in the repo (see **Quick reference**).

---

## Troubleshooting

* **macOS “cannot be opened”**:
  `xattr -dr com.apple.quarantine ./relog-asm`
* **Permission denied**:
  `chmod +x ./relog-asm`

---

## License

Free for Non-Commercial Use. Commercial use requires a license — [LICENSE.md](./LICENSE.md).

```


