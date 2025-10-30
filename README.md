# RelogASM (CLI)

RelogASM is an experimental, assembly-style DSL and embeddable VM.
It composes prompts from files, calls an LLM for strict JSON output, and can execute the returned program in the same DSL.

DSL reference: [docs/dsl.md](./docs/dsl.md)

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

`log`, `env`, `dotenv`, `is`, `b64`, `str`, `fs`, `http`, `sh`, `rand`, `math`, `eval`, `llm`, `json` core ops.

---

## Editor support

Syntax highlighting and diagnostics are provided by the [RelogASM LSP + VS Code extension](https://github.com/relogrun/relog-asm-vscode)

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
