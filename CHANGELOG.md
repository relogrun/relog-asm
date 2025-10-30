# Changelog

## 0.1.1 — 2025-10-30
- Stability and correctness improvements across VM, DSL, and modules.
  
## 0.1.0 — 2025-10-26
- Initial release.
- CLI: `relog-asm` (runs `.rasm`).
- Binaries: macOS (universal), Linux x86_64, Windows x86_64 (+ `.sha256`).
- Modules: log, env, dotenv, is, b64, str, fs, http, sh, rand, math, eval, llm_chat, json.
- Config: `relog.config.toml` → `[vm].step_limit`.
- Flags: `--log`, `--step-limit`, `--version`.
