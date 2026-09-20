# CLAUDE.md

## Project

homerun2-core-catcher — Go microservice that consumes messages from Redis Streams, resolves full payloads from Redis JSON, and supports three operating modes: log, CLI (interactive TUI), and web (HTMX dashboard).

## Tech Stack

- **Language**: Go 1.24+
- **Consumer**: Redis Streams via `redisqueue` (consumer groups)
- **Library**: `homerun-library` for shared types, Redis JSON, helpers
- **TUI**: Bubble Tea + Lip Gloss (CLI mode)
- **Build**: ko (`.ko.yaml`), no Dockerfile
- **CI**: Dagger modules (`dagger/main.go`), Taskfile
- **Deploy**: KCL manifests (`kcl/`), Kustomize, Kubernetes
- **Infra**: GitHub Actions, semantic-release, renovate

## Git Workflow

**Branch-per-issue with PR and merge.** Every change gets its own branch, PR, and merge to main.

### Branch naming

- `fix/<issue-number>-<short-description>` for bugs
- `feat/<issue-number>-<short-description>` for features
- `test/<issue-number>-<short-description>` for test-only changes
- `chore/<issue-number>-<short-description>` for infra/CI changes

### Workflow

1. Branch off `main`: `git checkout -b fix/<N>-<desc> main`
2. Make changes, commit with `Closes #<N>` in the message
3. Push: `git push -u origin <branch>`
4. Create PR: `gh pr create --base main`
5. Merge: `gh pr merge <N> --merge --delete-branch`
6. If multiple issues are closely related (e.g., same file), combine into one branch with multiple `Closes #N`

### Commit messages

- Use conventional commits: `fix:`, `feat:`, `test:`, `chore:`, `docs:`
- End with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>` when Claude authored
- Include `Closes #<issue-number>` to auto-close issues

## Code Conventions

- No Dockerfile — use ko for image builds
- Config via environment variables, loaded once at startup
- Tests: `go test ./...` — unit tests must not require Redis; integration tests run via Dagger with Redis service
- Catcher interface pattern: pluggable backends (Redis consumer, FileCatcher for dev)
- Pluggable message handlers: LogHandler, StoreHandler

## Architecture

```
Redis Stream ──► RedisCatcher ──┬──► LogHandler (structured slog)
                                └──► StoreHandler (in-memory store)
                                          │
                           ┌──────────────┤
                           ▼              ▼
                      CLI mode        Web mode
                    (Bubble Tea)      (HTMX UI)
```

### Operating Modes (`CATCHER_MODE`)

| Mode | Description |
|------|-------------|
| `log` (default) | Structured JSON logging with severity-aware levels |
| `cli` | Interactive Bubble Tea TUI with search, sort, detail view |
| `web` | HTMX web dashboard (planned) |

### Backends (`CATCHER_BACKEND`)

| Backend | Description |
|---------|-------------|
| `redis` (default) | Consumes from Redis Streams, resolves payloads via Redis JSON |
| `file` | Reads messages from a JSON file (dev/testing) |

## Key Paths

- `main.go` — entrypoint, mode selection, signal handling
- `internal/catcher/catcher.go` — RedisCatcher with JSON.GET payload resolution
- `internal/catcher/file.go` — FileCatcher for dev/testing
- `internal/catcher/handlers.go` — LogHandler, StoreHandler, severity mapping
- `internal/store/store.go` — thread-safe in-memory MessageStore
- `internal/models/models.go` — CaughtMessage struct
- `internal/tui/` — Bubble Tea TUI (app, table, search, detail)
- `internal/config/` — env-based config loading
- `internal/banner/` — animated TUI startup banner
- `dagger/main.go` — CI functions (Lint, Build, BuildImage, ScanImage, BuildAndTestBinary, IntegrationTest)
- `kcl/` — KCL deployment manifests
- `tests/kcl-deploy-profile.yaml` — generic KCL deploy profile
- `tests/kcl-movie-scripts-profile.yaml` — movie-scripts cluster profile
- `Taskfile.yaml` — task runner for build/test/deploy/release
- `.ko.yaml` — ko build configuration
- `.github/workflows/` — CI/CD (build-test, build-scan-image, release, lint-repo)

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CATCHER_MODE` | `log` | Operating mode: `log`, `cli`, `web` |
| `CATCHER_BACKEND` | `redis` | Backend: `redis` or `file` |
| `REDIS_ADDR` | `localhost` | Redis host |
| `REDIS_PORT` | `6379` | Redis port |
| `REDIS_PASSWORD` | *(empty)* | Redis password |
| `REDIS_STREAM` | `messages` | Redis stream to consume from |
| `CONSUMER_GROUP` | `homerun2-core-catcher` | Consumer group name |
| `CONSUMER_NAME` | hostname | Consumer name within the group |
| `REDIS_STARTUP_TIMEOUT` | `120s` | How long startup retries Redis before exiting (Go duration) |
| `MAX_MESSAGES` | `10000` | Max messages in memory store (cli/web) |
| `CATCHER_FILE_PATH` | `messages.json` | JSON file path (file backend) |
| `CATCHER_FILE_INTERVAL` | `1s` | Replay interval (file backend) |
| `LOG_FORMAT` | `json` | Log format: `json` or `text` |
| `LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |

## Testing

```bash
# Unit tests (no Redis needed)
go test ./...

# Integration test via Dagger (spins up Redis)
task build-test-binary

# Lint
task lint

# Build + scan image
task build-scan-image-ko

# CLI mode with file backend (local dev)
CATCHER_MODE=cli CATCHER_BACKEND=file CATCHER_FILE_PATH=tests/smoke-test-messages.json go run .
```

## Reference Project

`homerun2-omni-pitcher` is the sibling producer service — same patterns for infra, CI, and deployment.
