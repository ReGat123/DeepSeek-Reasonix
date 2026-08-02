# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Reasonix (module `reasonix`) is a DeepSeek-native AI coding agent: a config- and
plugin-driven harness built as a single static Go binary. It ships as a CLI/TUI,
a Wails-based desktop app (`desktop/`, a separate Go module), and a VS Code
extension (separate repo, `SivanCola/reasonix-vscode`) that talks to the local
`reasonix acp` backend. This repo notably contains its own agent-memory file,
`REASONIX.md` — Reasonix's analog of `CLAUDE.md` — read it; it holds the
project's standing conventions and is kept up to date by the team.

## Commands

```bash
go build ./cmd/reasonix     # build the CLI binary directly
make build                  # -> bin/reasonix(.exe) + bin/reasonix-plugin-example
make cross                  # -> dist/, cross-compiled for 6 darwin|linux|windows × amd64|arm64 targets
make vet                    # go vet ./...
make fmt                    # gofmt -w .
make test                   # go test ./...
make hooks                  # git config core.hooksPath .githooks (pre-push runs go vet)

go test ./...                                    # full suite
go test ./internal/agent/ -v                     # one package, verbose
go test ./internal/tool/builtin/ -run TestGrep   # one test
go test -race ./internal/agent/... ./internal/plugin/...  # race-checked packages (see ci.yml for the full list)

cd desktop && go test .                          # desktop module (separate go.mod)
```

Run this before every commit — it catches the fastest CI failures locally:

```bash
gofmt -w .
go vet ./...
go test ./internal/tool/builtin/ ./internal/boot/
```

CI additionally runs `golangci-lint` (not runnable locally in this environment)
and `govulncheck`. gofmt + vet already block ~80% of fast-fail scenarios.

### Isolated dev environment

A source-built binary must not touch a real install's config/credentials/sessions.
Set `REASONIX_HOME` to sandbox everything under one directory:

```bash
REASONIX_HOME=/tmp/reasonix-dev go run ./cmd/reasonix
```

## Architecture

```
cmd/reasonix/main.go   entry point; blank-imports built-in providers/tools to trigger init() registration
internal/
├── cli/          subcommand routing, flags, assembly, exit codes, TUI, setup wizard
├── control/      transport-agnostic Controller — shared logic behind CLI/TUI, HTTP/SSE serve, and desktop
├── config/       TOML loading: flag > project reasonix.toml > user config.toml > defaults
├── agent/        Session + harness loop (Run: stream -> execute tool calls -> repeat, bounded by maxSteps)
├── provider/     Provider interface + kind->factory registry
│   └── openai/   OpenAI-compatible impl (DeepSeek and most vendors are config instances of this kind)
├── tool/         Tool interface + Registry (canonicalized schemas)
│   └── builtin/  compile-time built-ins: read_file, write_file, edit_file, bash, ls, glob, grep, ...
├── plugin/       MCP client (stdio + http/streamable-http + sse transports), JSON-RPC 2.0 wire protocol
├── remote/       SSH transport for Remote-SSH (forward/, sftpfs/, bootstrap/)
├── memory/       REASONIX.md/AGENTS.md/CLAUDE.md hierarchy + fact-based auto-memory (remember/forget/memory tools)
├── permission/   per-call Policy: allow/ask/deny -> Decision
├── command/      custom slash commands from .reasonix/commands/*.md
├── skill/        skill discovery from Markdown
├── hook/         shell hooks (PreToolUse, ...)
├── event/        typed event stream
├── checkpoint/   snapshot-based rewind
├── serve/        HTTP/SSE server frontend
└── sandbox/      OS-level process sandboxing
desktop/          Wails desktop app; separate Go module (`replace reasonix => ../`) + Vite/TS frontend in desktop/frontend
docs/             SPEC.md is the engineering contract — read it before changing core abstractions
```

**Dependency direction is acyclic and enforced by convention:**
`cli → {agent, plugin, config} → {tool, provider}`. Built-in subpackages
(`provider/openai`, `tool/builtin`) import their parent to self-register via
`init()`; parents never import children. `remote` and its subpackages never
import `cli`, `agent`, or `serve` — all interactivity flows through callbacks
so the desktop module can consume the same surface.

**Before importing a new internal package from a non-test file**, check
whether the target package's `_test.go` files already import back to you —
`go test ./path/to/target/` reports `[setup failed]` if an import cycle exists.
Detect this before pushing, not in CI.

### Core abstractions

- **`Provider`** (`internal/provider`): `Name()` + `Stream(ctx, Request) (<-chan Chunk, error)`, registered by `kind` (e.g. `"openai"`) via `init()`. Adding an OpenAI-compatible vendor is a config edit, not a code change.
- **`Tool`** (`internal/tool`): `Name()`, `Description()`, `Schema()`, `ReadOnly()`, `Execute(ctx, args) (string, error)`. Built-ins self-register into a process-global set; a runtime `*Registry` is assembled per run from enabled built-ins plus plugin-provided tools — the agent only ever sees the `*Registry`. The documented contract lives in `docs/TOOL_CONTRACT.md`, generated from the same canonical schema path the runtime uses (`go test ./internal/tool -run TestBuiltinToolContractDocumentation`).
- **`Coordinator`** (`internal/agent`): optional two-model collaboration (`agent.planner_model`). Planner and executor run in **separate sessions** so neither model's prompt prefix is disturbed by the other's turns — mixing them in one session would break DeepSeek's prefix cache.
- **`control.Controller`**: the one place to add new behavior so the TUI, HTTP/SSE server, and Wails desktop all inherit it, instead of duplicating logic per frontend.

For anything beyond this summary — plugin/MCP transport details, context
compaction tiers, memory/fact semantics, two-model routing policy — read
`docs/SPEC.md` (870 lines, the authoritative engineering contract: change the
contract first, then the code) rather than guessing from code alone.

## Conventions

- **English is the primary language for all code**: comments, user-facing strings, tool descriptions, system prompts, and docs. `README.md`/docs are bilingual (`*.zh-CN.md` counterparts); code and CI-facing text are English-only.
- **Cache-first is a hard constraint, not a style preference.** The provider-visible system-prompt prefix (base prompt + tool schemas + memory) must stay byte-stable across turns within a session so DeepSeek's automatic prefix cache stays warm. Never mutate it mid-session. Changes touching provider-visible system prompt construction, memory prefix, output styles, skill index behavior, default tool surfaces, tool schemas, provider request serialization, compaction, or MCP/tool registration are **cache-sensitive** (see `scripts/check-cache-impact.sh` for the exact path list) and require, in the PR body:
  ```
  Cache-impact: <none|low|medium|high> — <reason>
  Cache-guard: <focused guard test/command, or why an existing guard covers it>
  ```
  Add `System-prompt-review: <note>` too when touching `internal/config/`, `internal/memory/`, `internal/outputstyle/`, `internal/skill/`, or `internal/boot/`. CI (`cache-impact.yml`) enforces this metadata; `n/a`/`none`/`todo`/`tbd` are rejected. `scripts/cache-guard.sh` is the broader release-level cache-hit check.
- **Library code never calls `os.Exit` or prints to stdout/stderr.** Only `internal/cli` and `main` decide exit codes and user-facing output. Wrap errors with `fmt.Errorf("...: %w", err)`. Exported identifiers need doc comments.
- **`gofmt` is enforced by CI** — format before committing.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/) (`feat(scope): ...`, `fix: ...`, `test(scope): ...`, `docs: ...`, `ci: ...`).
- PRs target `main-v2` (the default branch), not `main`.
- **PR hygiene**: one force-push per round of review feedback; amend rather than adding new commits for review feedback; keep the diff scoped to the PR's stated purpose.

### Adding a new built-in tool

1. `internal/tool/builtin/mytool.go` implementing `tool.Tool` (`Name`, `Description`, `Schema`, `ReadOnly`, `Execute`)
2. `func init() { tool.RegisterBuiltin(myTool{}) }`
3. Tests in `internal/tool/builtin/builtin_test.go` or a new `mytool_test.go`
4. It's automatically available — `main` blank-imports `builtin`

### Adding a new model provider

(For MCP tool servers, use `internal/plugin` instead — a different layer.)

1. `internal/provider/myprovider/` implementing `provider.Provider` (`Name`, `Stream`)
2. `func init() { provider.Register("mykind", New) }`
3. Available from config via `kind = "mykind"`

### Adding i18n strings

1. Add the field to the `Messages` struct in `internal/i18n/i18n.go`
2. Add values in both `internal/i18n/messages_en.go` and `messages_zh.go`
3. `TestCatalogsComplete` fails if a locale is missing the new field

## Repo layout beyond `internal/`

- `cmd/`: `reasonix` (CLI), `reasonix-plugin-example` (reference stdio MCP server, used by an end-to-end test), plus release/migration/bench tooling (`reasonix-launcher`, `reasonix-legacy-migrator`, `e2ebench`, `remote-protocol-gen`, `signpath-contract`).
- `desktop/`: Wails app, its own `go.mod` (`reasonix/desktop`, `replace reasonix => ../`), frontend under `desktop/frontend` (Vite/TS, `pnpm`).
- `workers/`: Cloudflare Workers (`accounts`, `crash-report`, `forum`) backing hosted services, each deployed by its own workflow.
- `site/`: Astro-based marketing/docs website (`esengine.github.io/DeepSeek-Reasonix`).
- `scripts/`: release automation, cache-impact checking, desktop build/packaging — mostly invoked by CI, not by hand.
- `.reasonix/`: this repo's own Reasonix project config (commands, hooks) — separate from `internal/` runtime code.
