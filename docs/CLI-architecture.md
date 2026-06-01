# ARchetipo CLI — Architecture Document

## 1. Executive Summary

The ARchetipo CLI is a deterministic Go application that implements the ARchetipo workflow operations (backlog management, spec planning, status transitions) against pluggable backends. It is designed to be:

*   **Deterministic**: given the same input and config, it produces the same output and side-effects.
*   **Testable**: all I/O is injected; the CLI can be driven from unit tests without touching the filesystem or network.
*   **Pluggable**: new storage backends (connectors) can be added without modifying the command layer.
*   **Machine-friendly**: every command emits a versioned JSON envelope on stdout (success) or stderr (failure), making it ideal for consumption by AI coding agents.

**Tech stack:** Go 1.26, `spf13/cobra`, `gopkg.in/yaml.v3`, `fsnotify`, `gh` CLI (for the GitHub backend).

---

## 2. Module Structure

```
cli/
├── cmd/archetipo/main.go          # Minimal entry point
├── internal/
│   ├── cli/                       # Cobra command tree (public surface)
│   ├── config/                    # .archetipo/config.yaml loader
│   ├── connector/                 # Abstract interface + registry
│   │   ├── builtin/               # Side-effect import to register all connectors
│   │   ├── conformance/           # Shared behavioural test suite
│   │   ├── filefs/                # Local filesystem / YAML backend
│   │   ├── github/                # GitHub Issues + Projects v2 backend
│   │   └── inmemory/              # Reference in-memory backend
│   ├── domain/                    # Canonical data types
│   ├── iox/                       # JSON envelope + typed errors
│   ├── version/                   # Build-time version + update notifier
│   └── web/                       # Local Kanban HTTP viewer
├── go.mod
└── .goreleaser.yaml               # Release configuration
```

The module path is `github.com/techreloaded-ar/ARchetipo/cli`.

---

## 3. The CLI Surface (Cobra)

### 3.1 Entry Point & Execution Model

`cmd/archetipo/main.go` is intentionally thin:

```go
func main() {
    os.Exit(appcli.Execute(os.Args[1:], os.Stdin, os.Stdout, os.Stderr))
}
```

`Execute` in `internal/cli/root.go` accepts **all I/O streams as parameters**. This is the single most important testability decision: tests invoke `Execute` with `bytes.Buffer` instances instead of real OS streams.

### 3.2 Root Command Construction

`newRootCmd` builds the Cobra tree. It sets:

*   `SilenceUsage: true` and `SilenceErrors: true` — the CLI handles its own error rendering.
*   `Version: version.Version` — injected at link time.
*   A `streams` struct bundling `stdin/stdout/stderr` passed to every leaf command.

Commands are grouped by noun (`spec`, `prd`, `task`, `config`, etc.) and registered via `cmd.AddCommand(...)`.

### 3.3 The `withConnector` Plumbing Helper

Most business commands do not interact with Cobra directly after parsing. They delegate to `withConnector`, a higher-order function that:

1.  Loads `.archetipo/config.yaml` from the current working directory (walking up the tree).
2.  Builds the selected connector via the registry.
3.  Invokes the operation callback with a `context.Context` and the connector.
4.  On success, writes the JSON envelope to stdout.
5.  On failure, returns the error so `Execute` can serialize it to stderr and map it to an exit code.

This pattern keeps command handlers focused on argument validation and business orchestration.

### 3.4 Exit Code Contract

```
0  ok
1  generic error
2  input / validation error
3  connector error (auth, network, gh failure)
4  precondition missing (e.g. backlog not found)
```

Errors implement an optional `ExitCode() int` interface. `iox.CodedError` is the concrete type used throughout.

---

## 4. Domain Model

### 4.1 Core Types (`internal/domain`)

The `domain` package defines connector-agnostic types:

*   `Spec` — the unit of work (user story, bug, feature).
*   `Task` — an implementation item inside a spec.
*   `Epic`, `Priority`, `Status`, `Scope`, `TaskType` — strongly-typed strings for determinism.
*   `SetupInfo`, `BacklogSummary`, `WriteResult`, `PlanInput`, `SelectQuery`, `ReorderAnchor`, `SpecUpdate` — operation payloads.

All struct fields carry both `json` and `yaml` tags so the same types can be used for API envelopes and on-disk persistence.

### 4.2 The Connector Interface (`internal/connector`)

The `Connector` interface mirrors the public CLI operations and is divided into three groups:

*   **Setup**: `InitializeConnector`
*   **Read**: `FetchBacklogItems`, `SelectSpec`, `ReadSpecDetail`, `ReadSpecTasks`, `ReadExistingBacklog`
*   **Write**: `SavePRD`, `SaveInitialBacklog`, `AppendSpecs`, `SavePlan`, `TransitionStatus`, `CompleteTask`, `MoveBoardCard`, `UpdateSpec`, `PostComment`

Every method takes `context.Context` so timeouts and cancellation propagate correctly.

### 4.3 Registry Pattern (`internal/connector/registry.go`)

Connectors self-register via `init()` functions using a global map:

```go
type Builder func(cfg config.Config) (Connector, error)

func Register(name string, b Builder)
func New(cfg config.Config) (Connector, error)
```

`internal/connector/builtin/builtin.go` imports every concrete connector package for side-effects:

```go
func init() {
    filefs.Register()
    inmemory.Register()
    github.Register()
}
```

This means the `cli` package never imports a concrete connector directly, keeping the dependency graph clean.

---

## 5. Connector Implementations

### 5.1 `filefs` — Local Filesystem Backend

**Responsibilities:**
*   Stores backlog as YAML files (`.archetipo/backlog.yaml` + `.archetipo/specs/*.yaml`).
*   Stores plans as YAML (`.archetipo/plans/{code}-plan.yaml`).
*   Supports **legacy migration** from markdown (`BACKLOG.md`, `planning/*.md`) via HTML-comment markers.
*   Implements board column mapping from configured workflow statuses.

**Key files:**
*   `connector.go` — implements the `Connector` interface.
*   `store.go` — `yamlStore`, `loadStore`, `writeStore`, normalization, legacy fallback.
*   `markers.go` — parses and renders `<!-- archetipo:... -->` HTML-comment markers.
*   `parser.go` — legacy markdown backlog/plan parsers (GFM task tables).
*   `writer.go` — deterministic markdown renderers for backlog and plans.

**State management:** `filefs.Connector` is **stateless** apart from the injected `config.Config`. It reads the full store on every operation and writes it back atomically (no intermediate files, but `writeYAML` uses `os.WriteFile` which is atomic on POSIX).

### 5.2 `github` — GitHub Issues + Projects v2

**Responsibilities:**
*   Maps specs to GitHub issues and project board items.
*   Maps tasks to sub-issues.
*   Manages labels (`archetipo-backlog`, epic labels).
*   Synchronizes workflow statuses with Project v2 custom fields.

**Key architectural decisions:**
*   **Runner abstraction (`gh.go`)**: all shell-outs to `gh` go through the `Runner` interface. The real implementation uses `exec.CommandContext`; tests inject a mock.
*   **State caching**: `Connector.state` caches repo info, project metadata, and the issue→itemID map for the lifetime of the process. This avoids repeated `gh` calls in a single CLI invocation.
*   **GraphQL isolation (`templates.go`)**: all GraphQL queries/mutations live in one file, making them easy to audit and snapshot-test.
*   **Board resolution (`board_resolver.go`)**: auto-detects or creates the GitHub Project board, aligns status options, and persists resolved IDs back into `config.yaml`.

**Idempotency:** `SaveInitialBacklog` refuses to run if issues with the `archetipo-backlog` label already exist. `AppendSpecs` skips duplicate issue numbers.

### 5.3 `inmemory` — Reference Backend

A map-backed implementation used exclusively by tests. It provides:
*   A **shared conformance target** (`connector/conformance`) that every real connector must satisfy.
*   A fast fixture for CLI-level tests that need a connector but do not want filesystem side-effects.

It uses a `sync.Mutex` to make it safe for parallel sub-tests.

---

## 6. Configuration Management

### 6.1 Loading (`internal/config/config.go`)

`config.Load(startDir)` walks up the directory tree looking for `.archetipo/config.yaml`. If none is found, it returns **defaults** rooted at `startDir`.

Defaults include:
*   `connector: file`
*   Paths: `docs/PRD.md`, `docs/mockups/`, `docs/test-results/`
*   File paths: `.archetipo/backlog.yaml`, `.archetipo/plans/`
*   Workflow statuses: TODO, PLANNED, IN PROGRESS, REVIEW, DONE

### 6.2 Validation

`validate()` checks that configured paths are writable by probing the parent directory with a temporary file. It only checks paths relevant to the **active** connector.

### 6.3 YAML Manipulation

`Config.Save()` uses `yaml.Node` (not struct marshalling) to patch the existing config file **in-place**, preserving comments and key order. This is critical because the shipped template contains human-readable comments that must not be destroyed when the GitHub connector writes back auto-detected `owner` and `project_number`.

---

## 7. Protocol & Errors

### 7.1 JSON Envelope (`internal/iox/iox.go`)

All stdout output is wrapped:

```json
{"schema":"archetipo/v1","kind":"...","data":{...}}
```

All stderr errors are wrapped:

```json
{"schema":"archetipo/v1","kind":"error","error":{"code":"E_*","message":"...","hint":"..."}}
```

The `schema` field allows consuming skills to detect breaking changes.

### 7.2 Typed Errors

`iox.CodedError` carries:
*   `Code` — stable string for machine branching (e.g. `E_INVALID_INPUT`).
*   `Message` — human-readable.
*   `Hint` — actionable remediation.
*   `Exit` — process exit code.
*   `Cause` — underlying Go error (exposed via `Unwrap()`).

Helper constructors (`NewInvalidInput`, `NewConnector`, `NewPrecondition`, etc.) enforce consistent mapping between error codes and exit codes.

---

## 8. Web Viewer (`internal/web`)

The `archetipo view` command starts a local HTTP server that renders the backlog as a Kanban board.

### 8.1 Server (`server.go`)

*   Registers REST routes on `http.ServeMux` (Go 1.22 pattern: `GET /api/board`).
*   Serves static assets from an embedded FS.
*   Serves design mockups from the configured `paths.mockups` directory.
*   Binds to `127.0.0.1:8080` by default; no authentication.

### 8.2 Optional Connector Interfaces

The viewer probes the connector at runtime for extra capabilities:

*   `planBodyReader` — returns the prose body of a plan (filefs only).
*   `prdReader` — returns raw PRD markdown.
*   `mockupLister` — lists design mockup folders.
*   `boardOrderReader` — exposes the global drag-and-drop ordering.

This keeps the viewer loosely coupled: connectors implement only what they support.

### 8.3 Real-Time Updates

*   **Broker (`broker.go`)**: in-memory pub/sub with buffered channels. Non-blocking send means slow SSE clients coalesce events.
*   **Watcher (`watcher.go`)**: `fsnotify.Watcher` observes `.archetipo/`. It debounces bursts (200 ms) and ignores editor swap files / hidden files.
*   **SSE Handler (`handlers.go`)**: `GET /api/board/stream` flushes `event: board_changed` on every filesystem change, with a 25-second heartbeat comment to keep proxies happy.

---

## 9. Testing Strategy

### 9.1 Conformance Suite (`connector/conformance`)

A pure Go test suite that exercises every `Connector` method in a realistic workflow order. Each concrete connector provides a `Factory` and calls `conformance.Run`:

*   `filefs` uses a temporary directory.
*   `inmemory` uses a fresh `inmemory.Connector`.
*   `github` uses a mock `Runner` with recorded `gh` output.

This guarantees behavioural parity across backends.

### 9.2 CLI Tests (`internal/cli/cli_test.go`)

Tests are black-box: they call `cli.Execute` with injected buffers.

```go
func runCLI(t *testing.T, stdin string, args ...string) result
```

Each test creates a temp directory (`t.Chdir`) so the file connector sees an empty project. Tests assert on:
*   Exit codes.
*   JSON envelope `kind`.
*   Specific fields inside `data` or `error`.

This pattern proves the entire stack (parsing → config → connector → I/O) without mocking internals.

### 9.3 Mocking External Dependencies

*   **GitHub connector**: the `Runner` interface allows replaying recorded `gh` stdout/stderr without network access.
*   **Version notifier**: uses a real HTTP client but against the npm registry; disabled in CI via `ARCHETIPO_NO_UPDATE_NOTIFIER=1`.

---

## 10. Build & Release

### 10.1 GoReleaser (`.goreleaser.yaml`)

*   Cross-compiles for `darwin/linux/windows` × `amd64/arm64`.
*   `CGO_ENABLED=0` for static binaries.
*   `-ldflags` injects `version.Version` at build time.
*   Archives are **bare binaries** (no tar/zip), named `archetipo-<os>-<arch>`.

### 10.2 NPM Distribution

The binary is wrapped by an npm package (`@techreloaded/archetipo`). A Node.js shim:
1.  Resolves the platform-specific sub-package (`@techreloaded/archetipo-<os>-<arch>`).
2.  Sets `ARCHETIPO_DATA_DIR`.
3.  Spawns the Go binary.

Skills are bundled inside the npm package under `skills/` and copied into the target project by `archetipo init`.

### 10.3 Update Notifier (`internal/version/notifier.go`)

A background goroutine (started in `Execute`) fetches the latest npm version once per day. If a newer version exists, a banner is printed to stderr at the end of the command. It is:
*   Non-blocking (50 ms timeout on the done channel).
*   Cached on disk.
*   Disabled by `ARCHETIPO_NO_UPDATE_NOTIFIER=1` or non-TTY stderr.

---

## 11. Key Architectural Patterns

| Pattern | Implementation | Benefit |
|---|---|---|
| **Dependency Injection** | `Execute` takes `stdin/stdout/stderr`; connectors take `config.Config`; GitHub connector takes `Runner`. | Full testability without monkey-patching. |
| **Registry / Plugin** | Connectors self-register in `init()`; `builtin` package imports them for side-effects. | The CLI does not import concrete backends. |
| **Interface Segregation** | The viewer uses small optional interfaces (`planBodyReader`, etc.) probed at runtime. | Clean separation between mandatory and optional connector capabilities. |
| **Stable Error Codes** | `iox.CodedError` with machine-readable `Code` and documented exit codes. | AI agents can branch reliably on failure. |
| **Deterministic Output** | JSON envelope with `schema`; sorted YAML keys; canonical markdown rendering. | Enables snapshot testing and idempotent tooling. |
| **Context Propagation** | Every connector method accepts `context.Context`. | Timeouts, cancellation, and tracing work end-to-end. |
| **Higher-Order Functions** | `withConnector` centralizes config load, connector build, and envelope writing. | Sub-command handlers stay small and focused. |

---

## 12. How to Use This as a Blueprint

If you are building a similar Go CLI:

1.  **Adopt the stream-injection pattern** in `main.go` and `Execute`. It costs nothing and makes testing trivial.
2.  **Define a domain package first**. Let commands, connectors, and the web layer all depend on it, but not on each other.
3.  **Use a registry for backends**. It removes import cycles and keeps the command layer agnostic.
4.  **Write a conformance suite** before implementing the second backend. It forces you to clarify edge cases early.
5.  **Version your CLI protocol**. The `schema` field in the JSON envelope is cheap insurance against breaking changes.
6.  **Keep Cobra handlers thin**. Validate args, then delegate to a helper like `withConnector` that handles I/O consistency.
7.  **Use `yaml.Node` for config patching**. It preserves comments and ordering, which users appreciate.
