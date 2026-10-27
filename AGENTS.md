# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

`jira-cli` is an interactive Jira CLI (issues, epics, sprints, boards, projects, releases) heavily inspired by GitHub CLI. Go 1.26 module `github.com/ankitpokhrel/jira-cli`, built on cobra (commands) + viper (config) + tview (interactive TUI). README and the agent skill at `skills/jira-cli/SKILL.md` document the product behavior.

## Development commands

Go version is pinned via `go.mod` (`toolchain go1.26.5`); CI uses `^1.26.5`. Dependencies are vendored with `vendor/` gitignored.

```sh
make deps install     # vendor deps, then build & install the binary (documented workflow)
make build            # build the binary
make ci               # lint + test — run before submitting a PR
make lint             # golangci-lint v2.12.2 (auto-installed if missing): golangci-lint run ./...
make test             # go clean -testcache, then CGO_ENABLED=1 go test -race ./...
make jira.server      # docker compose up — local Jira instance for manual testing
```

Run a single package/test without the full `-race ./...` sweep:

```sh
go test ./internal/query/ -run TestIssue -v
go test ./pkg/jira/ -run TestCreate -v -race   # CGO_ENABLED=1 if -race
```

Lint expectations (`.golangci.yml`): code must be formatted with `gofumpt`/`goimports`/`gofmt`. Enabled linters include `errcheck`, `gocritic`, `gocyclo` (min complexity 20), `godot`, `revive`, `staticcheck`, `unparam`, `unused`, `bodyclose`, `mnd`. `mnd` (magic-number detector) flags numeric literals other than 0–3 — use named constants. `package-comments` and `testpackage` are disabled.

Build/version info (`git commit`, `version`, `SOURCE_DATE_EPOCH`) is injected at link time by Makefile `LDFLAGS` into `internal/version`; `GOCACHE` is set to a local `.gocache`.

## Architecture

Top-level layout:

- `cmd/jira/` — the only `main` package. `main()` calls `root.NewCmdRoot()` then `ExecuteC()`.
- `api/` — thin wrapper over the Jira REST client: a process-global singleton client plus `Proxy*` helpers.
- `internal/` — application code: `cmd/` (commands, one package per parent/subcommand), `cmdutil/`, `config/`, `query/`, `view/`, `version/`.
- `pkg/` — reusable packages: `jira` (REST client + domain models), `jql` (JQL builder), `tui` (tview primitives), `netrc`, `adf`, `md`.

### How a command flows (issue list as the reference)

1. `internal/cmd/issue/list/list.go`: `SetFlags(cmd)` registers all cobra flags; the `Run` handler reads viper keys and flags.
2. `internal/query.NewIssue(project, cmd.Flags())` parses flags into typed `IssueParams` and constructs the JQL string via `pkg/jql`.
3. `api.ProxySearch(api.DefaultClient(debug), jql, from, limit)` performs the HTTP call.
4. `view.IssueList` (or `EpicList`/`SprintList`) renders: interactive `pkg/tui` table, or machine-readable `--plain`/`--raw` (JSON)/`--csv` output.

Every command package exposes `NewCmdXxx()` and registers subcommands by plain `cmd.AddCommand`. Root command registration lives in `internal/cmd/root/root.go::addChildCommands`. Root also owns config loading (`cobra.OnInitialize`) and the token/config-file pre-check.

Commands: `init issue epic sprint board project release open me serverinfo completion version man`.

### Cross-cutting patterns (require reading multiple files)

- **Two Jira API generations**: Cloud uses REST v3 (`/rest/api/3`), on-prem/Local uses v2 (`/rest/api/2`); agile endpoints use v1 (`/rest/agile/1.0`). The split is driven by the `installation` config value (Cloud/Local). `pkg/jira/client.go` exposes `Get`/`GetV2`/`GetV1`, `Post*`, `Put*`, `DeleteV2` over hardcoded base paths; the `Proxy*` helpers in `api/client.go` (`ProxySearch`, `ProxyGetIssue`, `ProxyCreate`, `ProxyAssignIssue`, `ProxyUserSearch`, `ProxyTransitions`, `ProxyWatchIssue`) dispatch to the right variant so command code rarely handles the v2/v3 difference.
- **Config is ambient viper, not a struct.** Loaded once in `internal/cmd/root/root.go` with precedence `--config/-c` > `JIRA_CONFIG_FILE` env > default `~/.config/.jira/.config.yml`; env prefix is `JIRA` (`JIRA_API_TOKEN`, `JIRA_AUTH_TYPE`). Commands call `viper.GetString("server" | "project.key" | "board.id" | "timezone" | ...)` directly. Config regeneration lives in `internal/config` (`jira init`).
- **JQL value semantics** (`pkg/jql` + `internal/query`): value `x` → `IS EMPTY`, prefix `~` → negation (`~x` → `IS NOT EMPTY`); ordering/date filters (`--created today|week|month|year|-10d`), `--order-by`/`--reverse`, `--paginate from:limit` are all handled by the query layer.
- **Rendering split**: `pkg/tui` is the interactive tview layer (tables, key bindings `q/v/m/c`, help modal); `internal/view` is the renderer layer (per-domain lists + plain/JSON/CSV output modes). `sprint`/`epic` "explorer" views nest a sidebar + issue table.
- **Client acquisition is a package-level singleton**: commands call `api.DefaultClient(debug)` / `api.Client(jira.Config{...})`; there is no DI. `api.GetAPIToken` resolves the credential from `JIRA_API_TOKEN` env → `.netrc` (`pkg/netrc`) → OS keyring (`zalando/go-keyring`).
- **Auth types** (`auth_type` config): `basic` (default), `bearer` (PAT), `mtls` (client certs under `mtls.*`), `cf-access` (cloudflared `Cf-Access-Token` round-tripper).
- **Output/error helpers**: `internal/cmdutil` provides `Info`/`Warn`/`Success`/`Failed` (spinner + output) and `ExitIfError`; `Failed` and `ExitIfError` call `os.Exit`, so command bodies chain them after each fallible step.
- **Custom help**: `internal/cmd/root/help.go` replaces cobra's help. Commands group under MAIN/OTHER via the `cmd:main` annotation; an `ARGUMENTS` section is rendered from the `help:args` annotation. Mark the primary commands `cmd:main`.

## Tests

- Unit tests live next to the code (`*_test.go`). `make test` cleans the test cache and runs with `-race` and `CGO_ENABLED=1`.
- `pkg/jira` tests drive the real `*jira.Client` against `net/http/httptest.NewServer` with inline handlers; response fixtures live in `pkg/jira/testdata/` (`issue.json`, `issue-2.json`, `issue-link-types.json`, `fields.json`). Assertions use `testify`.
- `internal/query` tests are table-driven, using a hand-rolled fake `query.FlagParser` (the interface wraps `pflag.FlagSet` — the one deliberate testability seam; there are no client interfaces or mocks elsewhere).

## Operating the built CLI against real Jira

When a task involves querying/updating real Jira through the built binary, `skills/jira-cli/SKILL.md` is the authoritative guide (also referenced from the README). Key points:

- List commands are interactive by default. Prefer machine-readable output: `--raw` (JSON), `--plain --columns key,summary --no-headers`, or `--csv`.
- The default config `~/.config/.jira/.config.yml` holds the project, issue types and custom-field definitions. Inspect `issue.types` and `issue.fields.custom` before `create`/`edit`/`--custom`; never re-run `jira init` (it rewrites local config) without explicit user confirmation.
- Custom fields: `--custom key=value` where `key` is the configured field name lowercased with hyphens; arrays are comma-separated. Don't guess field names/IDs — use the config or another authoritative source.
- Use `jira issue list -q "..."` for raw JQL beyond the built-in flags; a JQL with its own project clause overrides `--project`.
- Never print `JIRA_API_TOKEN` or config credentials; `--debug` may dump auth headers.
