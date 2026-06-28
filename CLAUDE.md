# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Go service that scrapes V2EX and Hacker News, then pushes posts to a Telegram bot/channel. HN posts additionally get a Chinese AI summary via DeepSeek. Ported from an earlier Rust version (see "legacy parity" notes below).

## Commands

```bash
go mod tidy                                   # sync deps
go build -o bin/news-notify ./cmd/news-notify # build (binary name = directory name)
./bin/news-notify -c config.toml              # run; -c / --config selects the TOML file

go test ./...                                 # all tests
go test ./internal/monitor                    # one package
go test ./internal/monitor -run TestFilterNewTopic -v  # single test
go vet ./...
gofmt -l .                                    # list unformatted files
```

Requires Go ≥ 1.22 (relies on the Go 1.22 loop-variable semantics, though some code still defensively shadows — see `cmd/news-notify/main.go:127`).

## External runtime dependency

HN body extraction is **not** done in Go. It goes through the `digest.Fetcher` interface; the current implementation (`digest.Python`, `internal/digest/python.go`) calls a separate Python sidecar over HTTP at `http://127.0.0.1:50051/digest` ([hacker-news-digest](https://github.com/cheedonghu/hacker-news-digest)). Without it running, HN posts fall back to a "网页摘要获取失败" placeholder and skip the AI step. The `Dockerfile` clones and runs both the sidecar and this binary in one container; `docker-compose.yml` is the intended deployment.

## Architecture

`main.go` wires concrete implementations together, then runs each monitor as a goroutine under a single signal-cancellable `context.Context`. If any monitor returns a non-context error, `main` calls `cancel()` to bring all of them down together (`cmd/news-notify/main.go:130`).

The design is interface-based dependency injection — business code (`monitor`) depends only on interfaces, so swapping channels/providers means adding an implementation, not editing callers:

- **`monitor.Monitor`** (`internal/monitor/monitor.go`) — `Run(ctx, cfg) error`, blocks until ctx cancelled. Implementations: `V2EX`, `HackerNews`. Contract: return `ctx.Err()` for normal shutdown; handle transient errors internally (log + continue) rather than returning them.
- **`notify.Notifier`** (`internal/notify/notify.go`) — `Notify` / `NotifyBatch`. Implementation: `Telegram`. `NotifyBatch` sends serially with a 1.5s gap to dodge Telegram rate limits.
- **`ai.Helper`** (`internal/ai/ai.go`) — `Summarize`. Implementation: `DeepSeek` (uses `go-openai` SDK pointed at `api.deepseek.com`, since DeepSeek is OpenAI-protocol compatible).
- **`digest.Fetcher`** (`internal/digest/digest.go`) — `Fetch(ctx, originURL)` for HN body extraction. Implementation: `Python` (calls the Python sidecar over HTTP). Abstracted as an interface so an "agent" channel can be added later without touching `monitor`. Injected into `HackerNews` via `NewHackerNews`.
- **`model`** (`internal/model/model.go`) — shared DTOs (`NotifyBase`, V2EX `Topic`/`Node`/`Member`), separated into its own package to avoid import cycles.
- **`tools`** (`internal/tools/markdown.go`) — `EscapeMarkdownV2` (every Telegram message must be escaped before send) and `TruncateUTF8` (rune-aware truncation; never slice strings by byte).

Monitors share a single `*http.Client` (one connection pool) created in `main`. They poll on a `time.Ticker` (V2EX every 2 min, HN every 5 min) and `select` on `ctx.Done()` vs `ticker.C`.

### Dedup

Each monitor keeps an in-memory `map[id]yyyymmdd` guarded by an `RWMutex`, pruned by a date window (V2EX 5 days, HN 15 days). **This state is lost on restart** — a known, intentional tradeoff (no persistence). After a restart, recently-pushed posts may be re-sent.

### Config

TOML (`internal/config/config.go`), mapped via struct tags. `[telegram]`, `[features]` (per-source on/off switches + V2EX keyword/node filters + HN count/time-gap), `[deepseek]`. `chat_id` is stored as a string and `ParseInt`'d in `main` to avoid TOML int issues. `config.toml` is the template; `myconfig.toml` is a local (gitignored-style) variant.

## Conventions specific to this repo

- **Bilingual teaching comments.** Almost every line carries a Chinese comment explaining Go semantics (the original author was learning Go). When editing, match this density and style rather than stripping comments — they are the house style here, not noise.
- **AI/digest failures never abort a push.** `DeepSeek.Summarize` and the digest fetch return fallback strings (`"大模型返回异常"`, etc.) with `nil` error on purpose, so the post still goes out. Preserve this behavior unless explicitly changing it.
- **Legacy parity.** `formatHNMessage` (`internal/monitor/hackernews.go:320`) reproduces the old Rust output format byte-for-byte (spacing, newlines). `judgeNewsDate` only accepts the `"N hours ago"` form, matching Rust. Don't "clean up" these formats casually.
- **Known intentional quirk:** `filterNewTopic` combines keyword/node filters with **OR**, even though `config.toml` comments say "AND". A test in `internal/monitor/v2ex_test.go` locks the current OR behavior — changing it will (correctly) turn that test red.
- Logging is mixed: structured `slog` (JSON to stderr) in most places, but some flows still use `fmt.Printf` to stdout. Prefer `slog` for new code.
- Docker env var is named `RUST_CONFIG_PATH` for backward compatibility with the Rust deployment — not a typo.
