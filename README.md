# AI Context Manager - AICM

> 🚧 **Work in progress — not yet buildable.** The core Rust source (`src/`) is not
> published in this repository yet, so `./install.sh` / `cargo build` will fail until it
> is added. This README documents the tool's intended design and usage.

**A high-performance CLI proxy that snips command output before it hits your AI.**

Zap (binary: `zap`, also invoked as `rtk`) sits between your AI coding agent and your
shell. When the agent runs a noisy command like `git status` or `cargo test`, Zap runs
it, filters and compresses the output, and returns only what matters — saving **60–90%
of the tokens** that would otherwise be wasted on your LLM context.

> Single-source-of-truth design: all filtering logic lives in the Rust binary. The
> per-agent hooks are thin delegates that just call `rtk rewrite`.

## Why

A typical week of AI-assisted development (~1,200 commands) is dominated by verbose tool
output — test logs, build chatter, `git` dumps, linter noise. Most of it is irrelevant to
the model but burns tokens, money, and context window. Zap strips it down before it ever
reaches the agent — **transparently, with zero workflow changes**.

## Features

- **Transparent command rewriting** — `git status` → `rtk git status`, automatically, via
  hooks. No change to how you or the agent work.
- **60–90% token savings** across common dev commands.
- **Wide agent support (tiered):**
  | Agent | Mechanism |
  |---|---|
  | Claude Code, Cursor, Copilot (Chat/CLI), Gemini CLI | Full hook (shell / Rust binary) |
  | OpenCode, Hermes, Pi | Plugin (TS / Python) |
  | Cline / Roo, Windsurf, Codex CLI | Prompt-level rules file |
- **Broad command registry** — test runners (vitest, pytest, cargo/go test, playwright),
  build tools (cargo, npm, pnpm, make), VCS (git), linters (eslint, ruff, biome),
  language servers (tsc, mypy), package managers, file ops, and infra (docker, kubectl,
  aws, terraform).
- **Compound-command aware** — handles `&&`, `||`, `;`, `|`, `&` correctly
  (e.g. `cargo fmt && cargo test` → `rtk cargo fmt && rtk cargo test`).
- **Built-in analytics** — `rtk gain` shows your token savings over time.
- **Safe by design** — hooks are non-blocking: on any error they exit 0 so your command
  still runs raw. `unsafe_code = "deny"`, no `async`, single-threaded.

## Install

> Note: requires the Rust source to be present in the repo (see the WIP notice above).

```bash
git clone https://github.com/Madhusshree13/AI_Context_Manager.git
cd AI_Context_Manager
./install.sh                      # builds release + installs to ~/.cargo/bin
# or a custom prefix:
PREFIX=/usr/local sh install.sh   # may require sudo
Verify:


zap --version
rtk gain          # token-savings analytics
which rtk         # confirm the right binary (avoid name collisions)
Usage
Meta commands (run directly)

rtk gain              # show token-savings analytics
rtk gain --history    # command usage history with savings
rtk discover          # analyze agent history for missed opportunities
rtk proxy <cmd>       # run a raw command without filtering (debugging)
rtk rewrite "<cmd>"   # print the rewritten form of a command
Everyday commands
After installing the hook for your agent, just work normally. The agent's commands are
rewritten transparently — filtered output reaches the model, raw behavior reaches you.

Overrides

RTK_DISABLED=1 git status         # run a single command raw
Or list commands to never rewrite in ~/.config/rtk/config.toml:


exclude_commands = ["git push", "^docker .*--follow"]   # ^ = regex
How it works

Agent runs:  cargo test --nocapture
  → Hook intercepts (PreToolUse / plugin event)
  → Calls: rtk rewrite "cargo test --nocapture"
  → Registry matches → "rtk cargo test --nocapture"
  → Agent executes the rewritten command
  → Filtered output reaches the LLM (~90% fewer tokens)
Hooks are thin delegates: parse the agent's JSON, call rtk rewrite, format the
agent-specific response. All rewrite decisions come from the single registry in the Rust
binary — one source of truth for every pattern.

Repository layout

Cargo.toml / Cargo.lock / build.rs   Rust crate (binary: zap)  [src/ not yet published]
install.sh                           build + install script
hooks/                               per-agent hook delegates
  claude/  cursor/  copilot/  codex/
  cline/   windsurf/ opencode/ pi/  hermes/
marketing/                           showcase HTML/SVG assets
CONTRIBUTING.md                      dev loop, filter authoring, code style
Project status
Area	State
Hooks (per-agent delegates)	- Present
Build config (Cargo.toml, build.rs, install.sh)	- Present
Marketing assets	- Present
Core Rust source (src/)	❌ Not yet published — required to build
Contributing

cargo fmt --all && cargo clippy --all-targets && cargo test --all
All three must pass before a PR. New filters live in src/cmds/<ecosystem>/ and must
demonstrate ≥60% token savings on a fixture. See CONTRIBUTING.md.

Code style: anyhow::Result with .context("…")? on all error paths, regexes in
lazy_static!, no unwrap() in production paths, no async.

License
Apache-2.0.


