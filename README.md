# RelogASM (CLI)

RelogASM is a small assembly-style DSL and embeddable VM for composing deterministic pipelines from simple, typed instructions. 

DSL reference: [DSL.md](./DSL.md)

## Quick start

1. **Download & unpack** the archive for your OS from [latest release](https://github.com/relogrun/relog-asm/releases/latest):

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

```
use log
call log.info "hello from RelogASM"
```

Run:

```bash
relog-asm hello.rasm
```

### One more (async shell)

```
use sh { allowed_cmds: ["cat"], max_output_bytes: 65536 } as sh
use b64
use log

// Run `cat`, capture stdout/stderr, feed stdin
let req = {"cmd":"cat","capture_stdout":true,"capture_stderr":true,"stdin_utf8":"hello"}
let fut = async call sh.run req
let out = await fut

// Extract base64 stdout via core JSON op, then decode to UTF-8
let out_b64 = json.get {"json": out, "path": "stdout_base64"}
let text = call b64.decode out_b64

// Print decoded text
call log.info text
halt
```

---

## Config (optional)

Place `relog.config.toml` next to your script:

```toml
[vm]
step_limit = 100000

[log]
filter = "debug"
```

CLI flags (e.g., `--step-limit`) override config values.

---

## Built-in modules

`log`, `env`, `dotenv`, `is`, `b64`, `str`, `fs`, `http` (async), `sh` (async), `rand`, `math`, `eval`, `llm` (chat), `json` core ops.

---

## Troubleshooting

* **macOS “cannot be opened”**:
  `xattr -dr com.apple.quarantine ./relog-asm`
* **Permission denied**:
  `chmod +x ./relog-asm`

---

## License

Free for Non-Commercial Use. Commercial use requires a license — [LICENSE.md](./LICENSE.md).
