# RelogASM (CLI)

RelogASM is a small assembly-style DSL and embeddable VM for composing deterministic pipelines from simple, typed instructions. 

DSL reference: [DSL.md](./DSL.md)

## Quick start

1. **Download & unpack** the archive for your OS from [latest release](https://github.com/relogrun/relog-asm/releases/latest):

2. **macOS/Linux:** make executable:

```bash
chmod +x relog-asm
```

On macOS

```bash
xattr -dr com.apple.quarantine ./relog-asm
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

## Editor support

Syntax highlighting and diagnostics are provided by the [Relog-ASM LSP + VS Code extension](https://github.com/relogrun/relog-asm-vscode)

Open any `.rasm` file, the server autostarts.

---

## Troubleshooting

* **macOS “cannot be opened”**:
  `xattr -dr com.apple.quarantine ./relog-asm`
* **Permission denied**:
  `chmod +x ./relog-asm`

---

## License

Free for Non-Commercial Use. Commercial use requires a license — [LICENSE.md](./LICENSE.md).
