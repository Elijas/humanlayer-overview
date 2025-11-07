# HumanLayer Codebase Architecture & Organizational Patterns

## 1. PROJECT STRUCTURE

### Monorepo Organization
The codebase is a unified monorepo managed by Turbo, containing two distinct project groups:

#### Project 1: HumanLayer SDK & Platform (Legacy - Superseded)
- `humanlayer-ts/` - TypeScript SDK (removed in PR #646)
- `humanlayer-go/` - Minimal Go client
- `humanlayer-ts-vercel-ai-sdk/` - Vercel AI SDK integration
- `docs/` - Mintlify documentation

#### Project 2: Local Tools Suite (Active)
Primary components handling Claude Code orchestration:
- **`hld/`** - HumanLayer Daemon (Go) - REST API + JSON-RPC interface
- **`hlyr/`** - HumanLayer CLI (TypeScript/Node) - MCP server + approval orchestration
- **`humanlayer-wui/`** - CodeLayer Web UI (Tauri + React) - Desktop approval interface
- **`claudecode-go/`** - Go SDK for programmatic Claude Code session launch

#### Secondary Components
- **`apps/`** - Web application components
  - `daemon/` - Backend daemon
  - `react/` - React application frontend
- **`packages/`** - Shared libraries
  - `contracts/` - Type definitions and interfaces
  - `database/` - Database layer abstraction
- **`scripts/`** - Utility scripts
- **`hack/`** - Build and development utilities

### Directory Tree
```
humanlayer/
├── .claude/                          # Claude Code configuration
│   ├── agents/                       # Specialized sub-agents
│   ├── commands/                     # 30 workflow commands
│   └── settings.json                 # Permissions config
├── .github/workflows/                # CI/CD orchestration
├── apps/                             # Monorepo applications
├── packages/                         # Shared packages
├── hld/                              # Daemon (Go 1.24)
├── hlyr/                             # CLI tool (TypeScript)
├── humanlayer-wui/                   # Desktop UI (Tauri)
├── claudecode-go/                    # Go SDK
├── docs/                             # User documentation
├── hack/                             # Development utilities
│   └── linear/                       # Linear ticket integration
├── scripts/                          # Build scripts
├── Makefile                          # Root orchestration
├── turbo.json                        # Monorepo config
├── package.json                      # Node workspaces
└── y-schema.sql                      # Collaborative notes schema
```

---

## 2. DOCUMENTATION APPROACH

### Documentation Files Structure
HumanLayer uses a comprehensive multi-layered documentation approach:

| File | Purpose | Scope |
|------|---------|-------|
| `README.md` | Product marketing + quick start | Public-facing |
| `CLAUDE.md` | Claude Code guidance (repo root + components) | AI assistant instructions |
| `CONTRIBUTING.md` | Contribution workflow | Open source |
| `DEVELOPMENT.md` | Development environment setup | Developer guide |
| `humanlayer.md` | Legacy SDK documentation | Historical reference |
| Component-specific `CLAUDE.md` | Component guidelines + logs | Per-component guidance |
| Component-specific `README.md` | Component overview | Per-component |
| `PROTOCOL.md` (hld) | JSON-RPC 2.0 API specification | Technical reference |
| `TESTING.md` (hld) | Database isolation + integration tests | Testing guide |
| `RELEASE.md` (wui) | Release process + troubleshooting | Release management |
| `config.md` (hlyr) | Configuration precedence + env vars | Configuration guide |
| `CHANGELOG.md` (hlyr) | Semantic versioning changelog | Version history |

### Documentation Patterns
1. **Component-level CLAUDE.md files**: Technical guidance specific to each component
2. **ROOT CLAUDE.md**: Architecture overview + global conventions
3. **Mermaid diagrams**: Configuration flow, dependency graphs
4. **Inline comments**: Go follows context-first design patterns
5. **TODO annotations**: Priority-based task tracking

---

## 3. DEVELOPMENT WORKFLOW

### Build System Architecture
- **Root Makefile**: 426 lines - orchestrates all components
- **Component-specific Makefiles**: Each major component has own Makefile
- **Turbo**: Monorepo task orchestration with caching
- **Package managers**: Mixed (npm, bun, go modules)

### Core Development Commands
```bash
make setup                   # Full repository setup
make check                   # Linting + type checking (all components)
make test                    # All test suites
make check-test             # Complete CI validation

# Component-specific
make check-hlyr / check-hld / check-wui / check-claudecode-go
make test-hlyr / test-hld / test-wui

# Development environments (parallel isolation)
make daemon-dev              # Dev daemon with persistent DB
make daemon-nightly         # Stable daemon
make wui-dev                # Dev WUI connected to dev daemon
make wui-nightly           # Production-like WUI

# Ticket-based isolation (for parallel work)
make daemon-ticket TICKET=ENG-2114  # Isolated daemon per ticket
make wui-ticket TICKET=ENG-2114     # Isolated WUI per ticket
make codelayer-dev TICKET=ENG-2114  # Both with single command
```

### Parallel Development Environment Strategy
**Problem**: Daemon restarts break active Claude sessions
**Solution**: Dual-environment isolation

```
Production (Nightly)              Development (Testing)
├── daemon.sock                   ├── daemon-dev.sock
├── daemon.db                     ├── daemon-dev.db / daemon-{TICKET}.db
└── daemon.log                    └── daemon-dev-{TIMESTAMP}.log

Usage: Developers maintain stable nightly environment while testing
       breaking changes in isolated dev environment with fresh DB copies
```

### Git Workflow
- **Worktree-based development**: `hack/create_worktree.sh` creates isolated worktrees
- **Pre-push hooks**: `make githooks` installs check/test hooks
- **Branch prefixes**: `local/*` branches blocked from push
- **PR workflow**: Fork → branch → checks → PR → merge

### Development Command Cheat Sheet (from CONTRIBUTING.md)
```
1. /research_codebase     # Research codebase structure
2. /create_plan           # Create implementation plan
3. /implement_plan        # Execute plan
4. /commit                # Create git commits
5. gh pr create --fill    # Create GitHub PR
6. /describe_pr          # Generate PR description
```

---

## 4. COLLABORATION PATTERNS

### Multi-Component Communication Architecture
```
Claude Code → MCP Protocol → hlyr → JSON-RPC → hld → HumanLayer API
                                         ↑         ↑
                                    TUI ─┘         └─ WUI
```

#### Component Integration Points
1. **Claude Code → hlyr** (MCP Protocol)
   - hlyr provides MCP server for approvals
   - Model Context Protocol for tool communication

2. **hlyr → hld** (JSON-RPC 2.0 over Unix sockets)
   - Daemon orchestrates Claude sessions
   - Handles approval workflow management
   - Event streaming via SSE

3. **WUI → hld** (JSON-RPC 2.0 over Unix sockets)
   - Real-time session visualization
   - Approval UI interactions
   - Event subscriptions

### Configuration Precedence (hlyr example)
All components follow this pattern:
1. CLI Flags (highest priority)
2. Environment Variables
3. Configuration Files (XDG spec)
4. Default Values (lowest priority)

Example env vars:
```bash
HUMANLAYER_DAEMON_SOCKET          # ~/.humanlayer/daemon.sock
HUMANLAYER_DATABASE_PATH          # ~/.humanlayer/daemon.db
HUMANLAYER_DAEMON_HTTP_PORT       # 7777
HUMANLAYER_DAEMON_VERSION_OVERRIDE # Custom version string
HUMANLAYER_API_KEY                # Authentication
```

### State Synchronization
- **Event bus pattern** (hld/bus/): Cross-component events
- **Database as source of truth**: SQLite for all state
- **Event streaming**: SSE for real-time updates
- **Session lifecycle**: Parent → child session tree structure

---

## 5. STATE MANAGEMENT

### Database Architecture
**Storage**: SQLite with structured schema

#### Session Schema (implicit from PROTOCOL.md)
```
sessions
├── id (PK)
├── run_id (unique)
├── query
├── model
├── created_at
├── status (running, blocked_on_approval, completed)
├── parent_id (nullable - for child sessions)
└── metadata (JSON)

conversations
├── session_id (FK)
├── message_index
├── role (user/assistant)
├── content
└── tool_calls (array)

approvals
├── id (PK)
├── session_id (FK)
├── status (pending, approved, denied)
└── response_time
```

#### Collaborative Notes Schema (y-schema.sql)
```
notes
├── id (PK)
├── title
└── created_at

notes_operations (Yjs CRDT operations)
├── id (PK)
├── note_id (FK)
└── op (BYTEA - binary operation)

ydoc_awareness (Real-time collaboration state)
├── clientId
├── note_id (FK)
├── op (BYTEA)
└── updated (TTL: 2 minutes)
```

### Session Lifecycle Management
**Session States**:
1. `launching` - Initial Claude Code session creation
2. `running` - Active execution
3. `blocked_on_approval` - Waiting for human approval
4. `completed` - Finished
5. `failed` - Error occurred

**Session Hierarchy**:
- Parent sessions can have child sessions (worktree branches)
- Leaf sessions (terminal in tree) tracked separately
- List API defaults to `leafOnly` (filtering out parents)

### Event Bus Pattern (hld/bus/)
```
EventBus (pub/sub)
├── SessionCreated
├── SessionUpdated
├── ApprovalRequested
├── ApprovalResponsed
└── Error events

Event propagation between:
- Approval system → Session manager
- Session manager → RPC handlers
- RPC handlers → WUI/TUI via SSE
```

---

## 6. ROOTED PATHS & REFERENCE CONVENTIONS

### HumanLayer's Path Conventions
Unlike ACF's `::PROJECT/`, `::WORK/`, `::THIS/`, HumanLayer uses environment-based conventions:

#### Socket/Database Paths
```bash
# Production/Nightly
~/.humanlayer/daemon.sock
~/.humanlayer/daemon.db

# Development
~/.humanlayer/daemon-dev.sock
~/.humanlayer/daemon-dev.db

# Ticket-isolated
~/.humanlayer/daemon-{PORT}.sock
~/.humanlayer/daemon-{TICKET}.db

# Logs
~/.humanlayer/logs/daemon-*.log
~/.humanlayer/logs/wui-{branch}/codelayer.log
```

#### Environment Variables as "Rooted Paths"
HumanLayer achieves isolation through environment variables rather than path patterns:
```bash
# Override default locations
HUMANLAYER_DAEMON_SOCKET=~/.humanlayer/daemon-dev.sock
HUMANLAYER_DATABASE_PATH=~/.humanlayer/daemon-{TICKET}.db
HUMANLAYER_DAEMON_HTTP_PORT=7777  # Computed from ticket number

# Version tracking
HUMANLAYER_DAEMON_VERSION_OVERRIDE=$(git branch --show-current)
```

#### Worktree Isolation (closest to ACF concepts)
```bash
hack/create_worktree.sh branch_name
# Creates: .git/worktrees/{branch_name}/

# Each worktree gets isolated environment:
- Separate node_modules (bun install per worktree)
- Separate Go build artifacts
- Separate daemon/WUI instances
```

### No Explicit "Checkpoint" System
Unlike ACF, HumanLayer doesn't use rooted path hierarchies. Instead:
- **Linear tickets** track work units (ENG-2114)
- **Git branches** organize code changes
- **Environment variables** provide isolation
- **ACF checkpoint experiment** exists in `humanlayer_acf_comparison_v1_01/` (experimental comparison)

---

## 7. VERSIONING STRATEGY

### Semantic Versioning (hlyr example)
- Format: `MAJOR.MINOR.PATCH`
- Keep a Changelog format
- References Linear ticket numbers in changelog

Example changelog entry:
```
## [0.12.0] - 2025-10-07
### Added
- New `humanlayer claude init` command (ENG-1784)
- Interactive wizard-style UX
- Arrow key navigation

### Changed
- Improved UX from number-based selection

### Fixed
- Edit tool false 'failed' display (ENG-1727)
```

### Build Version Strategy
**Release workflow** (`release-macos.yml`):

1. **Tag-based triggers**: `v[0-9]+.[0-9]+.[0-9]+` pattern
2. **Nightly builds**: `0.1.0-{TIMESTAMP}-nightly`
3. **Auto-increment**: Patch version auto-bumped if no explicit version given
4. **Build info injected via ldflags**:
```go
go build -ldflags "\
  -X github.com/humanlayer/humanlayer/hld/internal/version.BuildVersion=$(VERSION) \
  -X github.com/humanlayer/humanlayer/hld/config.DefaultSocketPath=~/.humanlayer/daemon.sock"
```

### Version Tracking in Code
- **hld**: `hld/internal/version/version.go`
- **hlyr**: `hlyr/package.json` version field
- **WUI**: `humanlayer-wui/src-tauri/tauri.conf.json` version

---

## 8. QUALITY GATES

### Code Quality Checks
All components use this pattern:

```bash
# Format checking
prettier --check "**/*.{ts,tsx,md}"  # TypeScript/React
go fmt ./...                          # Go

# Linting
biome lint                            # JavaScript/TypeScript
golangci-lint run                     # Go

# Type checking
tsc --noEmit                          # TypeScript
cargo check                           # Rust (Tauri)

# Testing
npm run test (vitest)                 # TypeScript
go test ./...                         # Go
```

### CI/CD Workflows (`.github/workflows/`)

#### main.yml - Standard CI
- Runs on: push to main, PR open/sync/reopen
- Checks: format, lint, type checking, all tests
- Caches: Node modules, Go modules, Rust, Tauri deps

#### linear-research-tickets.yml - Work Prioritization
- WIP limit: 5 tickets max in "research in review"
- Fetches from Linear: highest priority backlog
- Auto-assigns to LinearLayer (Claude) bot
- Status transitions: "research needed" → "research in review"

#### linear-create-plan.yml - Planning
- Triggered by Linear workflow
- Creates implementation plans for research backlog

#### linear-implement-plan.yml - Execution
- Triggered by Linear workflow
- Auto-implements tickets with approved plans

#### release-macos.yml - Release Management
- Tag push trigger: Auto-creates release artifacts
- Workflow dispatch: Manual version control
- Nightly builds: Scheduled (currently disabled)
- Output: DMG, daemon binary, install instructions

#### claude-code-review.yml - Code Review
- Triggered on PRs
- Claude Code-powered review assistance

### Test Structure
**hld (Go daemon)**:
- Unit tests: `./...` (non-integration)
- Integration tests: `-tags=integration` flag
- Race detection: `-race` flag on all
- E2E tests: REST API validation with real daemon

**hlyr (TypeScript CLI)**:
- Framework: Vitest
- Command: `npm run test`
- Watch mode: `npm run test:watch`

**humanlayer-wui (Tauri + React)**:
- Framework: Bun's test runner
- Command: `bun test`
- Focus: Store (state management) tests critical
- Constraint: No Tauri integration tests (desktop-specific)

### Database Isolation (Critical for Testing)
**Required for all integration tests**:
```go
// In-memory (fastest)
t.Setenv("HUMANLAYER_DATABASE_PATH", ":memory:")

// Persistent test DB
_ = testutil.DatabasePath(t, "test-name")

// Manual temp file
dbPath := filepath.Join(t.TempDir(), "test.db")
t.Setenv("HUMANLAYER_DATABASE_PATH", dbPath)
```

### Error Codes & Standards
- JSON-RPC 2.0 error codes
- Custom error types standardized in `rpc/types.go`
- Consistent error response formatting

---

## 9. DEVELOPMENT CONVENTIONS

### TODO Priority System
Enforced across codebase:

```
TODO(0): Critical - never merge
TODO(1): High - architectural flaws, major bugs
TODO(2): Medium - minor bugs, missing features
TODO(3): Low - polish, tests, documentation
TODO(4): Questions/investigations needed
PERF: Performance optimization opportunities
```

### Go Style Guidelines (from hld/CLAUDE.md)
- Async/long-running goroutines accept `context.Context` as first parameter
- Context and CancelFuncs never stored on structs (always parameters)
- Interface-based testing with mockgen

### TypeScript Guidelines
- Modern ES6+ features
- Strict TypeScript configuration
- CommonJS/ESM compatibility
- React 19 (no forwardRef - use `ref` prop directly)
- Zustand for global state management
- ShadCN components + Tailwind CSS for UI

### Architecture Guidelines
- Component isolation through sockets/HTTP
- Event-driven communication via event bus
- Database as source of truth
- Context-first API design

---

## 10. LINEAR INTEGRATION & WORK TRACKING

### Linear as Work Hub
HumanLayer deeply integrates with Linear for work management:

#### Workflow Status Transitions
```
backlog → ready → research needed → research in review → implementation planned → 
implementing → in review → done
```

#### Claude-Powered Automation
- **LinearLayer (Claude)**: Automated research agent
- **Research WIP limit**: 5 concurrent research tasks
- **Implementation**: Auto-triggered from approved plans
- **Tracking**: Each ticket tracks associated PRs and checkpoints

#### Slash Commands Integration
```bash
/research_codebase    # AI analyzes codebase
/create_plan          # Claude creates implementation plan
/implement_plan       # Auto-implements from plan
/commit               # Create structured git commits
/describe_pr          # Generate comprehensive PR descriptions
```

#### Local Slash Commands (from .claude/commands/)
30+ Claude Code workflow commands including:
- Planning: `/create_plan`, `/iterate_plan`
- Implementation: `/implement_plan`
- Debugging: `/debug`
- Code review: `/local_review`
- Handoff: `/create_handoff`, `/resume_handoff`

---

## 11. RELEASE & DEPLOYMENT

### macOS Release Process
1. **Create version tag**: `git tag v0.2.0`
2. **Push tag**: GitHub Actions workflow triggers
3. **Build artifacts**:
   - Daemon (Go): Cross-compiled for macOS ARM64
   - CLI (Node): Compiled via bun
   - WUI: Tauri DMG bundle
4. **Signing**: Ad-hoc signing to prevent "damaged app" errors
5. **Distribution**: GitHub release with DMG + binaries

### Installation Targets
- DMG: Direct installation to ~/Applications
- Homebrew: Package distribution
- npm: Global CLI package
- Binary distribution: Standalone executables

### Update Flow
- Nightly builds (experimental)
- Stable releases via tag
- Manual workflow dispatch for testing

---

## 12. KEY ARCHITECTURAL PATTERNS

### Pattern 1: Event Bus for Cross-Component Communication
```go
// Central coordination without tight coupling
EventBus → Subscribe/Publish pattern
- SessionCreated → approval system, rpc handlers
- ApprovalResponsed → session manager, event stream
```

### Pattern 2: Socket-Based IPC
- Unix domain sockets for local daemon communication
- JSON-RPC 2.0 protocol (line-delimited)
- Separate channels for requests vs. event subscriptions

### Pattern 3: Database-Centric State
- SQLite as single source of truth
- Event log for audit trail
- Schema versioning via migrations

### Pattern 4: Environment-Based Isolation
- Daemon socket path override via env var
- Database path isolation for dev/prod
- Version string injection via build flags
- Port allocation from ticket numbers

### Pattern 5: Parallel Development Environments
- Nightly (stable) for regular work
- Dev (testing) for breaking changes
- Ticket-isolated (per-feature) for parallel work
- Fresh DB copies for each isolation level

### Pattern 6: Turbo-Based Monorepo Caching
- Task graph with dependency tracking
- Distributed caching across workspace
- Per-component Makefiles for flexibility

---

## SUMMARY TABLE

| Aspect | Implementation | Notes |
|--------|----------------|-------|
| **Project Structure** | Turbo monorepo | 2 project groups: SDK (legacy) + Local Tools |
| **Language Mix** | Go (daemon) + TypeScript (CLI/UI) | Rust for Tauri desktop |
| **State Management** | SQLite + Event Bus | CRDT for collab notes |
| **Work Tracking** | Linear + GitHub | 30+ slash commands for AI workflows |
| **Development** | Dual environments (nightly/dev) | Parallel isolation via socket paths |
| **Testing** | Unit + Integration + E2E | Database isolation required |
| **Versioning** | Semantic (tag-based) | Build info via ldflags |
| **Documentation** | Component CLAUDE.md files | Root + per-component guidance |
| **Quality Gates** | format + lint + typecheck + test | CI/CD + pre-push hooks |
| **IPC** | JSON-RPC 2.0 over Unix sockets | Event streaming via SSE |
