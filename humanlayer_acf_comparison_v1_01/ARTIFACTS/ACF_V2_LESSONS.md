# ACF v2: Strategic Lessons from HumanLayer

Based on the comprehensive comparison analysis between HumanLayer's production architecture and ACF v1, this document outlines what ACF would need to add to reach "v2" maturity by learning from HumanLayer's production-tested patterns.

## Executive Summary

**HumanLayer's Core Lesson:** *"Runtime coordination at scale requires moving beyond files."*

ACF v1 optimizes for simplicity (markdown + directories). ACF v2 should optimize for **real-time multi-agent orchestration** while keeping the checkpoint contract as the authoritative interface.

**Strategic Path:** Add runtime layers without breaking the file-based contract.

---

## 1. Runtime Coordination Layer ⭐⭐⭐

**Priority: CRITICAL - Foundation for all other improvements**

### What HumanLayer Has
- Real-time event bus (pub/sub pattern)
- In-memory state + SQLite persistence
- JSON-RPC over Unix sockets for IPC
- Immediate event propagation across components

### What ACF Currently Has
- Append-only event log (file-based)
- Pure markdown state
- CLI tools only
- Batch-processed automation

### ACF v2 Would Add

```bash
# Daemon mode
acft daemon start --port 7777
acft daemon stop
acft daemon status

# Event subscription (real-time)
acft events subscribe --types CHECKPOINT_CREATED,HARNESS_EXECUTED
acft events subscribe --follow --filter "checkpoint_path LIKE 'auth_%'"

# Query interface
acft query "show all active checkpoints with failed harness"
acft query --checkpoint ::WORK/auth_v2_03 --show-dependencies --depth 3
```

### Implementation Design

```
┌─────────────────────────────────────────────┐
│         ACF Daemon Architecture             │
├─────────────────────────────────────────────┤
│                                             │
│  CLI/TUI/Web → JSON-RPC → Daemon           │
│                              ↓              │
│                         Event Bus           │
│                       ↙    ↓    ↘          │
│                SQLite  File  Automation     │
│                                             │
└─────────────────────────────────────────────┘
```

### Why This Matters
- **Real-time coordination** between multiple agents without polling files
- **Queryable state** instead of grepping through markdown
- **Automation triggers** that react immediately, not batch-processed
- **Foundation** for all other v2 features

### Implementation Phases
1. **Phase 1**: Basic daemon with event bus
2. **Phase 2**: JSON-RPC interface
3. **Phase 3**: SQLite state backend
4. **Phase 4**: Automation hooks

---

## 2. Multi-Environment Isolation ⭐⭐⭐

**Priority: CRITICAL - Enables parallel work without conflicts**

### What HumanLayer Has

```bash
# Parallel isolated environments
make daemon-dev          # Dev: daemon-dev.sock, daemon-dev.db
make daemon-nightly      # Stable: daemon.sock, daemon.db
make daemon-ticket TICKET=ENG-2114  # Isolated per ticket

# Isolation mechanism
- Separate Unix sockets per environment
- Separate SQLite databases
- Separate log files
- Environment variable overrides
```

**Use Case:** Developer maintains stable nightly environment while testing breaking changes in isolated dev environment with fresh DB copies.

### What ACF Currently Has
- Directory-based isolation only
- One active checkpoint per branch/version
- No runtime isolation

### ACF v2 Would Add

```bash
# Environment profiles
acft env create dev --profile experimental
acft env create stable --profile production
acft env create ticket-2114 --profile isolated

# List environments
acft env list
# Output:
# dev        experimental  ~/.acft/work-dev       daemon:7777
# stable     production    ~/.acft/work-stable    daemon:7778
# ticket-2114 isolated     ~/.acft/work-2114      daemon:7779

# Switch contexts
export ACFT_ENV=dev
acft orient ::THIS  # Operates in dev profile

# Environment-specific config
ACFT_WORK_ROOT=~/.acft/work-dev
ACFT_EVENT_LOG=~/.acft/events-dev.log
ACFT_DATABASE=~/.acft/state-dev.db
ACFT_DAEMON_SOCKET=~/.acft/daemon-dev.sock
ACFT_DAEMON_PORT=7777
```

### Configuration Structure

```toml
# ~/.acft/environments.toml
[env.dev]
work_root = "~/.acft/work-dev"
database = "~/.acft/state-dev.db"
socket = "~/.acft/daemon-dev.sock"
port = 7777
profile = "experimental"

[env.stable]
work_root = "~/.acft/work-stable"
database = "~/.acft/state-stable.db"
socket = "~/.acft/daemon-stable.sock"
port = 7778
profile = "production"

[env.ticket-2114]
work_root = "~/.acft/work-2114"
database = ":memory:"  # Ephemeral
socket = "~/.acft/daemon-2114.sock"
port = 7779
profile = "isolated"
```

### Why This Matters
- Prevents daemon restarts from breaking active work
- **Parallel experimentation** without pollution
- **Safe testing** of breaking changes
- **Isolated per-ticket work** (similar to HumanLayer's ticket isolation)

### Implementation Phases
1. **Phase 1**: Environment config system
2. **Phase 2**: Multi-daemon support
3. **Phase 3**: Auto-port allocation from environment name
4. **Phase 4**: Environment lifecycle management (copy, archive, delete)

---

## 3. Database-Backed State ⭐⭐

**Priority: HIGH - Enables fast queries and relational data**

### What HumanLayer Has

```sql
-- Core schema
sessions (id, run_id, status, parent_id, metadata)
conversations (session_id, message_index, role, content)
approvals (id, session_id, status, response_time)

-- Query capabilities
SELECT * FROM sessions WHERE status = 'blocked_on_approval';
SELECT COUNT(*) FROM sessions WHERE parent_id IS NOT NULL;
```

### What ACF Currently Has
- Markdown frontmatter (YAML)
- Event log (newline-delimited JSON)
- No query interface (must parse all files)

### ACF v2 Would Add

```sql
-- SQLite schema
CREATE TABLE checkpoints (
    path TEXT PRIMARY KEY,
    branch TEXT NOT NULL,
    version INTEGER NOT NULL,
    step INTEGER NOT NULL,
    valid BOOLEAN NOT NULL DEFAULT false,
    lifecycle TEXT NOT NULL CHECK(lifecycle IN ('active', 'superseded', 'archived')),
    signal TEXT CHECK(signal IN ('pass', 'fail', 'blocked', 'pending')),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    last_harness_run TIMESTAMP,
    parent_checkpoint TEXT,
    FOREIGN KEY (parent_checkpoint) REFERENCES checkpoints(path)
);

CREATE TABLE deliverables (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    checkpoint_path TEXT NOT NULL,
    artifact_path TEXT NOT NULL,
    purpose TEXT NOT NULL,
    verified_at TIMESTAMP,
    FOREIGN KEY (checkpoint_path) REFERENCES checkpoints(path)
);

CREATE TABLE dependencies (
    checkpoint_path TEXT NOT NULL,
    depends_on TEXT NOT NULL,
    dependency_type TEXT NOT NULL CHECK(dependency_type IN ('checkpoint', 'system')),
    status TEXT NOT NULL CHECK(status IN ('ready', 'pending', 'blocked')),
    notes TEXT,
    PRIMARY KEY (checkpoint_path, depends_on),
    FOREIGN KEY (checkpoint_path) REFERENCES checkpoints(path)
);

CREATE TABLE events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT NOT NULL,
    checkpoint_path TEXT NOT NULL,
    timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actor TEXT,
    payload JSON,
    FOREIGN KEY (checkpoint_path) REFERENCES checkpoints(path)
);

CREATE INDEX idx_checkpoints_lifecycle ON checkpoints(lifecycle);
CREATE INDEX idx_checkpoints_valid ON checkpoints(valid);
CREATE INDEX idx_events_type ON events(type);
CREATE INDEX idx_events_timestamp ON events(timestamp);
```

### Query Interface

```bash
# Query checkpoints
acft query --sql "SELECT * FROM checkpoints WHERE valid = false AND lifecycle = 'active'"

# Dependency analysis
acft query --checkpoint auth_v2_03 --show-dependencies --depth 3

# Event history
acft query --events --checkpoint auth_v2_03 --since "2025-11-01"

# Aggregate statistics
acft stats --branch auth --show-success-rate
# Output:
# auth_v1: 3/5 checkpoints passed (60%)
# auth_v2: 5/8 checkpoints passed (62.5%)
```

### Dual-Source-of-Truth Strategy

```
CHECKPOINT.md (authoritative for humans/agents)
    ↕ (bidirectional sync)
SQLite (authoritative for queries/automation)

Write operations:
1. Update CHECKPOINT.md
2. Emit event
3. Daemon updates SQLite

Read operations:
- Humans/agents read CHECKPOINT.md
- Automation queries SQLite
```

### Why This Matters
- **Fast queries** without parsing every `CHECKPOINT.md`
- **Relational data** (dependencies, hierarchies)
- **Audit trail** with structured queries
- **Analytics** (success rates, time to completion)

### Implementation Phases
1. **Phase 1**: Schema design + migration system
2. **Phase 2**: Sync daemon (CHECKPOINT.md ↔ SQLite)
3. **Phase 3**: Query CLI interface
4. **Phase 4**: Analytics and reporting

---

## 4. Automated Verification & CI/CD ⭐⭐⭐

**Priority: HIGH - Reduces manual burden and ensures consistency**

### What HumanLayer Has

```yaml
# .github/workflows/main.yml
name: CI
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - Format check (prettier, go fmt)
      - Lint (biome, golangci-lint)
      - Type check (tsc, cargo check)
      - Test (vitest, go test -race)
      - E2E (real daemon validation)
```

**Key insight:** Every PR runs full validation before merge. No manual "remember to run tests" burden.

### What ACF Currently Has
- Manual `acft verify --record`
- Manual `acft validate`
- Agent must remember to run harness

### ACF v2 Would Add

```yaml
# .acft/ci.yml
name: Checkpoint CI
on:
  checkpoint:
    - CHECKPOINT_CREATED
    - MANIFEST_UPDATED
    - CHECKPOINT_CLOSED

jobs:
  validate:
    name: Validate structure
    run: |
      acft validate ::THIS --strict
      acft manifest --mode full --json
    on_failure:
      action: set_valid_false
      notify: creator

  verify:
    name: Execute harness
    run: |
      acft verify --record --auto-fix
    on_failure:
      action: keep_valid_false
      notify: creator
      create_issue: true

  dependency_check:
    name: Check dependencies
    run: |
      acft query --checkpoint ::THIS --check-dependencies
    on_failure:
      action: set_signal_blocked
      notify: creator

  notify:
    name: Send notifications
    if: always()
    run: |
      acft notify slack #eng-checkpoints
      acft notify email creator@example.com
```

### Automation Triggers

```bash
# Sentinel (reacts to creation)
acft sentinel start --listen CHECKPOINT_CREATED
# → Runs acft validate --strict
# → Checks rooted paths
# → Verifies required sections

# Verifier (reacts to closure)
acft verifier start --listen CHECKPOINT_CLOSED
# → Confirms harness ran
# → Checks MANIFEST LEDGER populated
# → Validates VALID=true only if harness passed

# Auditor (periodic scans)
acft auditor start --interval 1h
# → Finds quiescent checkpoints
# → Detects validation theater
# → Checks for timeline gaps
```

### Pre-commit Hooks

```bash
# .git/hooks/pre-commit
#!/bin/bash
# Auto-generated by: acft install-hooks

# Find all modified checkpoints
for checkpoint in $(acft find-modified-checkpoints); do
  echo "Validating $checkpoint..."
  acft validate "$checkpoint" --strict || exit 1
done

echo "All checkpoints valid ✓"
```

### Why This Matters
- **Catch failures early** (before hand-off)
- **Consistent quality** across all checkpoints
- **Reduced manual burden** on agents
- **Prevents validation theater** (VALID=true without harness)

### Implementation Phases
1. **Phase 1**: Event-driven sentinel/verifier/auditor
2. **Phase 2**: Pre-commit hooks
3. **Phase 3**: CI/CD workflow integration
4. **Phase 4**: Auto-remediation (e.g., fix relative paths)

---

## 5. Configuration Precedence System ⭐

**Priority: MEDIUM - Improves flexibility**

### What HumanLayer Has

```
Configuration Precedence (highest to lowest):
1. CLI flags
2. Environment variables
3. Configuration files (XDG spec)
4. Default values

Example:
HUMANLAYER_DAEMON_SOCKET=~/.humanlayer/daemon-dev.sock  # Env var
hlyr --daemon-socket /tmp/daemon.sock                     # CLI flag wins
```

### What ACF Currently Has
- Hardcoded paths (`::PROJECT/`, `::WORK/`, `::THIS/`)
- No override mechanism
- Marker files (`checkpoints_project.toml`, `checkpoints_work.toml`)

### ACF v2 Would Add

```bash
# CLI flags (highest precedence)
acft orient --work-root /custom/path --project-root /repo

# Environment variables
export ACFT_WORK_ROOT=/custom/path
export ACFT_PROJECT_ROOT=/repo
acft orient ::THIS

# Config file (~/.acft/config.toml)
[paths]
work_root = "/custom/path"
project_root = "/repo"

[paths.overrides]
"::PROJECT/" = "/Users/user/Development/humanlayer"
"::WORK/" = "/Users/user/Development/humanlayer/work"

# Defaults (lowest precedence)
::WORK/ → $(find_work_root)  # Walk up to checkpoints_work.toml
::PROJECT/ → $(find_project_root)  # Walk up to checkpoints_project.toml
```

### Configuration File Format

```toml
# ~/.acft/config.toml
version = "2.0"

[paths]
work_root = "~/.acft/work"
project_root = "~/Development/humanlayer"

[daemon]
socket = "~/.acft/daemon.sock"
port = 7777
auto_start = true

[events]
log_path = "~/.acft/events.log"
retention_days = 90

[validation]
strict = true
auto_fix_paths = true

[notifications]
slack_webhook = "https://hooks.slack.com/..."
email = "alerts@example.com"

[environments.default]
profile = "production"

[environments.dev]
profile = "experimental"
work_root = "~/.acft/work-dev"
```

### Why This Matters
- **Flexibility** for different workflows
- **Environment-specific** overrides
- **Testing isolation** without changing files
- **Cross-platform** configuration (XDG compliance)

### Implementation Phases
1. **Phase 1**: Config file parsing + precedence logic
2. **Phase 2**: CLI flag overrides
3. **Phase 3**: Environment variable support
4. **Phase 4**: Config validation and migration

---

## 6. Rich UI Layer ⭐

**Priority: MEDIUM - Improves usability and onboarding**

### What HumanLayer Has
- **WUI (Tauri desktop app)**: Real-time session visualization, approval management
- **TUI (terminal interface)**: Interactive command-line interface
- **Real-time event streaming**: SSE for live updates

### What ACF Currently Has
- CLI only
- Text output
- No interactive mode

### ACF v2 Would Add

```bash
# TUI mode
acft tui
# Interactive dashboard:
# - Checkpoint tree (navigable with arrow keys)
# - Real-time harness logs
# - Dependency graph
# - Quick actions (validate, verify, close)

# Web UI
acft dashboard --port 8080
# Features:
# - Visual checkpoint hierarchy
# - Drag-and-drop dependency management
# - One-click harness execution
# - Real-time event stream viewer
# - Markdown editor with live preview

# Visualization
acft visualize --checkpoint ::WORK/auth_v2_03 --format svg
# Outputs dependency graph as SVG

acft visualize --work-root ::WORK --format html
# Outputs interactive HTML tree
```

### TUI Design (ASCII mockup)

```
┌─ ACF Dashboard ────────────────────────────────────────────┐
│                                                            │
│ Checkpoints                    Status                     │
│ ├─ auth_v1_01      [✓ VALID]   ▓▓▓▓▓░░░░░ 50% harness     │
│ ├─ auth_v1_02      [✓ VALID]   ▓▓▓▓▓▓▓▓▓▓ 100% passed     │
│ ├─ auth_v2_01      [✗ INVALID] ░░░░░░░░░░ 0% (stale)      │
│ └─ auth_v2_02      [⟳ ACTIVE]  ▓▓▓░░░░░░░ 30% building    │
│                                                            │
│ Recent Events                                              │
│ 2025-11-07 10:32 CHECKPOINT_CREATED auth_v2_02            │
│ 2025-11-07 10:35 HARNESS_EXECUTED auth_v2_01 (FAIL)       │
│ 2025-11-07 10:40 MANIFEST_UPDATED auth_v2_02              │
│                                                            │
│ [V]alidate [H]arness [Q]uit                               │
└────────────────────────────────────────────────────────────┘
```

### Why This Matters
- **Visual context** for complex hierarchies
- **Faster navigation** than CLI
- **Better onboarding** for humans
- **Real-time feedback** during harness execution

### Implementation Phases
1. **Phase 1**: TUI with checkpoint tree navigation
2. **Phase 2**: Real-time event viewer in TUI
3. **Phase 3**: Web dashboard (React + Tailwind)
4. **Phase 4**: Desktop app (Tauri wrapper)

---

## 7. Session Hierarchy & Lifecycle ⭐⭐

**Priority: HIGH - Improves tracking and debugging**

### What HumanLayer Has

```go
// Rich session lifecycle
type SessionStatus string
const (
    StatusLaunching         SessionStatus = "launching"
    StatusRunning          SessionStatus = "running"
    StatusBlockedOnApproval SessionStatus = "blocked_on_approval"
    StatusCompleted        SessionStatus = "completed"
    StatusFailed           SessionStatus = "failed"
)

// Parent/child hierarchy
type Session struct {
    ID       string
    ParentID *string  // Nullable for hierarchy
    Status   SessionStatus
    CreatedAt time.Time
    BlockedAt *time.Time
}
```

### What ACF Currently Has

```yaml
# Simple lifecycle
LIFECYCLE: active  # active|superseded|archived
SIGNAL: pass       # pass|fail|blocked|pending
```

### ACF v2 Would Add

```yaml
# Richer lifecycle with temporal tracking
LIFECYCLE:
  state: running
  started_at: 2025-11-07T10:00:00Z
  blocked_at: 2025-11-07T12:30:00Z
  blocked_on:
    - ::WORK/dep_v1_03
  unblocked_at: 2025-11-07T14:15:00Z
  completed_at: null

# Parent/child tracking
PARENT_CHECKPOINT: ::WORK/auth_v2_03
CHILD_CHECKPOINTS:
  - path: ::WORK/auth_v2_03/fix_bug_v1_01
    status: completed
    completed_at: 2025-11-06T18:00:00Z
  - path: ::WORK/auth_v2_03/add_tests_v1_01
    status: running
    started_at: 2025-11-07T09:00:00Z

COMPLETION_STATUS:
  children_completed: 1
  children_total: 2
  percent_complete: 50

# Time tracking
METRICS:
  time_to_first_harness: 2h 30m
  time_blocked: 1h 45m
  total_duration: 4h 15m
  harness_runs: 3
  harness_success_rate: 66.7%
```

### Query Interface

```bash
# Find blocked checkpoints
acft query --status blocked --show-blockers

# Output:
# auth_v2_03: blocked on [dep_v1_03, system:credentials]
# data_v1_02: blocked on [auth_v2_03]

# Timeline analysis
acft timeline --checkpoint auth_v2_03
# Output:
# 2025-11-07 10:00:00 CREATED
# 2025-11-07 10:30:00 HARNESS_EXECUTED (fail)
# 2025-11-07 12:30:00 BLOCKED (dependency: dep_v1_03)
# 2025-11-07 14:15:00 UNBLOCKED
# 2025-11-07 15:00:00 HARNESS_EXECUTED (pass)
# 2025-11-07 15:30:00 CLOSED (valid=true)
```

### Why This Matters
- **Track blocking dependencies** explicitly
- **Time-based metrics** (how long blocked? how long to completion?)
- **Progress visibility** across hierarchy
- **Bottleneck detection** (which checkpoints block the most?)

### Implementation Phases
1. **Phase 1**: Enhanced lifecycle states
2. **Phase 2**: Temporal tracking (timestamps for state transitions)
3. **Phase 3**: Parent/child relationship tracking
4. **Phase 4**: Metrics and analytics

---

## 8. Smart Automation & Work Tracking ⭐⭐

**Priority: MEDIUM - Enables multi-agent coordination**

### What HumanLayer Has

```bash
# Linear integration
- Auto-assigns tickets to agents
- WIP limits (5 research tasks max)
- Status transitions (research needed → research in review)
- Auto-triggered workflows (/create_plan → /implement_plan)

# Workflow automation
.github/workflows/linear-research-tickets.yml
.github/workflows/linear-create-plan.yml
.github/workflows/linear-implement-plan.yml
```

### What ACF Currently Has
- Manual checkpoint creation
- No work queue management
- No agent coordination

### ACF v2 Would Add

```bash
# Work queue management
acft queue add "Implement auth feature" --priority high --tags auth,security
acft queue add "Fix bug in data layer" --priority critical --tags bug,data
acft queue list --status ready
acft queue list --priority critical

# Agent claims work
acft queue claim auth_v2_04
# Creates checkpoint + assigns to agent

# Multi-agent coordination
acft agent-pool start --agents 3 --claim-from-queue --tags auth
# Starts 3 agents, each claims auth-related work

# Work distribution policies
acft config set queue.policy round-robin
acft config set queue.wip-limit 5
acft config set queue.auto-assign true

# Integration hooks
acft integrate linear --project ENG --sync-bidirectional
acft integrate github --repo humanlayer/humanlayer --link-prs
acft integrate slack --channel #eng-checkpoints --notify-on-block
```

### Queue Schema

```sql
CREATE TABLE work_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    description TEXT,
    priority TEXT NOT NULL CHECK(priority IN ('low', 'medium', 'high', 'critical')),
    status TEXT NOT NULL CHECK(status IN ('backlog', 'ready', 'claimed', 'completed', 'blocked')),
    tags JSON,
    claimed_by TEXT,
    claimed_at TIMESTAMP,
    checkpoint_path TEXT,
    external_id TEXT,  -- Linear/Jira ticket ID
    external_system TEXT,  -- linear, jira, github
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (checkpoint_path) REFERENCES checkpoints(path)
);
```

### Automation Examples

```yaml
# .acft/automation.yml
rules:
  - name: Auto-assign high priority
    trigger: work_queue.created
    condition: priority == 'critical'
    action:
      - notify: slack #eng-urgent
      - claim: auto

  - name: WIP limit enforcement
    trigger: work_queue.claimed
    condition: claimed_count > 5
    action:
      - reject: "WIP limit reached (5 max)"
      - notify: agent

  - name: Dependency blocking
    trigger: checkpoint.blocked
    condition: blocked_on != []
    action:
      - update_queue: status=blocked
      - notify: blocked_by
```

### Why This Matters
- **Work distribution** across multiple agents
- **Priority management** (critical work gets attention)
- **External tool integration** (Linear, Jira, GitHub)
- **WIP limits** prevent overload
- **Automatic coordination** reduces manual overhead

### Implementation Phases
1. **Phase 1**: Work queue schema + basic CLI
2. **Phase 2**: Agent claiming and assignment
3. **Phase 3**: External integration (Linear, GitHub)
4. **Phase 4**: Automation rules engine

---

## 9. Testing Infrastructure ⭐⭐

**Priority: MEDIUM - Ensures framework stability**

### What HumanLayer Has

```go
// Mandatory database isolation
func TestSession(t *testing.T) {
    // In-memory database for test isolation
    t.Setenv("HUMANLAYER_DATABASE_PATH", ":memory:")

    // Test code...
}

// Integration tests
go test -tags=integration -race ./...

// E2E tests
func TestE2E(t *testing.T) {
    daemon := StartDaemon(t)
    defer daemon.Stop()

    // Validate full API flow
}
```

### What ACF Currently Has
- Manual harness execution
- No framework-level testing
- No isolation for testing

### ACF v2 Would Add

```bash
# Test mode with isolated environment
acft test --env isolated create-checkpoint auth_test_v1_01

# Test harness execution (dry-run)
acft test verify-harness --checkpoint auth_v2_03 --mock-dependencies

# Isolated test environment
ACFT_ENV=test acft new test_checkpoint_v1_01
# Uses:
# - :memory: SQLite database
# - Temp directories (auto-cleanup)
# - Isolated event log
# - Mock daemon

# Integration test framework
acft test suite run --pattern "harness_*_test.sh"
acft test suite run --checkpoint auth_v2_03

# Checkpoint-level testing
acft test checkpoint auth_v2_03 \
  --mock-dependencies \
  --assert valid=true \
  --assert signal=pass

# Framework testing
acft test framework \
  --test-cli-commands \
  --test-event-emission \
  --test-database-sync
```

### Test Suite Structure

```
tests/
├── unit/
│   ├── test_path_expansion.sh
│   ├── test_manifest_ledger.sh
│   └── test_dependency_resolution.sh
├── integration/
│   ├── test_checkpoint_lifecycle.sh
│   ├── test_harness_execution.sh
│   └── test_event_propagation.sh
├── e2e/
│   ├── test_multi_agent_handoff.sh
│   ├── test_dependency_blocking.sh
│   └── test_work_queue.sh
└── fixtures/
    ├── valid_checkpoint/
    ├── invalid_checkpoint/
    └── mock_dependencies/
```

### Test Helpers

```bash
# test_helpers.sh
create_test_checkpoint() {
    local name=$1
    ACFT_ENV=test acft new "$name" --no-emit
}

assert_valid() {
    local checkpoint=$1
    local valid=$(acft query --checkpoint "$checkpoint" --field valid)
    [[ "$valid" == "true" ]] || fail "Expected valid=true"
}

mock_dependency() {
    local checkpoint=$1
    local dependency=$2
    acft test mock --checkpoint "$checkpoint" --dependency "$dependency" --status ready
}

cleanup_test_env() {
    rm -rf ~/.acft/test-*
}
```

### Why This Matters
- **Validate framework changes** before production
- **Regression testing** for ACF CLI tools
- **Checkpoint-level testing** without side effects
- **Confidence in refactoring**

### Implementation Phases
1. **Phase 1**: Test environment isolation
2. **Phase 2**: Basic test suite + fixtures
3. **Phase 3**: Integration tests for CLI commands
4. **Phase 4**: E2E tests for multi-agent workflows

---

## Strategic Priorities: Build Order

### Tier 1 (MVP for ACF v2) 🔥

**Build these first - they're foundational**

1. **Runtime Daemon** (4-6 weeks)
   - Event bus architecture
   - JSON-RPC interface
   - Basic state management
   - Socket-based IPC

2. **Database State** (3-4 weeks)
   - SQLite schema design
   - Bidirectional sync (CHECKPOINT.md ↔ DB)
   - Query interface
   - Migration system

3. **Multi-environment Isolation** (2-3 weeks)
   - Environment config system
   - Multi-daemon support
   - Environment switching
   - Isolated databases/logs

**Total: 9-13 weeks (~3 months)**

### Tier 2 (Production Hardening) 💪

**Build these next - they ensure quality**

4. **Automated CI/CD** (3-4 weeks)
   - Sentinel/Verifier/Auditor automation
   - Event-driven validation
   - Pre-commit hooks
   - Notification system

5. **Session Lifecycle** (2-3 weeks)
   - Enhanced state machine
   - Temporal tracking
   - Parent/child relationships
   - Metrics collection

6. **Configuration System** (1-2 weeks)
   - Precedence hierarchy
   - Config file format
   - Environment overrides
   - Validation

**Total: 6-9 weeks (~2 months)**

### Tier 3 (Enhanced UX) ✨

**Build these last - they improve usability**

7. **TUI/Web Dashboard** (4-6 weeks)
   - Terminal UI with navigation
   - Web dashboard (React)
   - Real-time event viewer
   - Visualization tools

8. **Work Queue** (3-4 weeks)
   - Queue management
   - Agent claiming
   - External integration (Linear, GitHub)
   - Automation rules

9. **Testing Framework** (2-3 weeks)
   - Test environment
   - Test suite + fixtures
   - Integration tests
   - E2E tests

**Total: 9-13 weeks (~3 months)**

---

## Architecture Evolution

### ACF v1 (Current)

```
┌─────────────────────────────────────────────┐
│         ACF v1 Architecture                 │
├─────────────────────────────────────────────┤
│                                             │
│  CHECKPOINT.md ──┐                          │
│  Event log ──────┼──> File system           │
│  CLI tools ──────┘                          │
│                                             │
│  Characteristics:                           │
│  - File-based state                         │
│  - Batch processing                         │
│  - Manual validation                        │
│  - Single agent focus                       │
│                                             │
└─────────────────────────────────────────────┘
```

### ACF v2 (HumanLayer-inspired)

```
┌─────────────────────────────────────────────┐
│         ACF v2 Architecture                 │
├─────────────────────────────────────────────┤
│                                             │
│  CHECKPOINT.md ──┐                          │
│  Event bus ──────┼──> SQLite ──> Daemon    │
│  CLI/TUI/Web ────┤                 ↓        │
│  Config system ──┘             JSON-RPC     │
│                                     ↓        │
│                           ┌─────────────────┤
│                           │ Agent Pool      │
│                           ├─────────────────┤
│                           │ - Work queue    │
│                           │ - Auto-claim    │
│                           │ - Coordination  │
│                           └─────────────────┘
│                                             │
│  Characteristics:                           │
│  - Dual state (file + database)            │
│  - Real-time events                         │
│  - Automated validation                     │
│  - Multi-agent coordination                 │
│  - Rich UI options                          │
│                                             │
└─────────────────────────────────────────────┘
```

---

## Key Design Principles

### 1. Backward Compatibility

**Principle:** ACF v2 must not break ACF v1 checkpoints.

**Implementation:**
- `CHECKPOINT.md` remains authoritative
- Database is derived state (can be rebuilt)
- CLI commands maintain same interface
- Daemon is optional (CLI works standalone)

### 2. Progressive Enhancement

**Principle:** Features should be additive, not replacements.

**Implementation:**
- Start with just CLI (like v1)
- Add daemon for real-time features
- Add database for queries
- Add UI for visualization

### 3. Polyglot Support

**Principle:** Don't force a single tech stack.

**Implementation:**
- Core daemon in Go (fast, single binary)
- CLI in Go (portability)
- TUI in Go (bubbletea)
- Web UI in TypeScript/React (rich ecosystem)

### 4. Local-First

**Principle:** No cloud dependencies required.

**Implementation:**
- SQLite (not Postgres/MySQL)
- Unix sockets (not HTTP APIs)
- Local file system (not S3)
- Optional cloud sync (not required)

---

## Migration Path: v1 → v2

### Phase 1: Side-by-side (No breaking changes)

```bash
# v1 continues to work
acft orient ::THIS
acft validate ::THIS

# v2 features are opt-in
acft daemon start  # Optional
acft tui           # Optional
```

### Phase 2: Gradual adoption

```bash
# Migrate existing checkpoints to database
acft migrate import-checkpoints ::WORK

# Start using v2 features
acft query --status blocked
acft env create dev
```

### Phase 3: Full v2

```bash
# v2 becomes default
acft daemon start --on-boot

# But v1 compatibility maintained
acft orient ::THIS  # Still works without daemon
```

---

## Success Metrics

### Technical Metrics

- **Query performance**: < 100ms for checkpoint queries
- **Event latency**: < 10ms from emit to daemon processing
- **Sync lag**: < 1s between CHECKPOINT.md update and database sync
- **Test coverage**: > 80% for core functionality

### User Experience Metrics

- **Time to first harness**: Reduced by 50% (with automation)
- **Manual validation burden**: Reduced by 80% (with CI/CD)
- **Context switching**: Reduced by 60% (with TUI/dashboard)
- **Onboarding time**: Reduced by 40% (with better UX)

### Operational Metrics

- **Checkpoint throughput**: 10x increase (multi-agent coordination)
- **Failure detection time**: 90% reduction (real-time vs batch)
- **Time blocked on dependencies**: 50% reduction (better visibility)

---

## Risks & Mitigation

### Risk 1: Complexity Explosion

**Risk:** Adding too many features makes ACF harder to understand.

**Mitigation:**
- Keep CHECKPOINT.md as simple source of truth
- Make advanced features opt-in
- Maintain excellent documentation
- Provide migration guides

### Risk 2: Database Sync Divergence

**Risk:** CHECKPOINT.md and SQLite get out of sync.

**Mitigation:**
- CHECKPOINT.md is always authoritative
- Database can be rebuilt from files
- Sync validation in CI/CD
- Conflict detection and auto-resolution

### Risk 3: Performance Degradation

**Risk:** Database queries slow down with many checkpoints.

**Mitigation:**
- Proper indexing on critical fields
- Query optimization
- Pagination for large result sets
- Archive old checkpoints

### Risk 4: Multi-daemon Conflicts

**Risk:** Multiple daemons interfere with each other.

**Mitigation:**
- Separate sockets per environment
- File locking on shared resources
- Environment isolation guarantees
- Clear error messages

---

## Conclusion

**ACF v2 = ACF v1 + Runtime Coordination**

The path forward is clear:

1. **Keep** the checkpoint contract (CHECKPOINT.md)
2. **Add** runtime coordination (daemon + event bus + database)
3. **Enhance** automation (CI/CD + work queue + multi-agent)
4. **Improve** UX (TUI + web dashboard + better CLI)

**Core Insight from HumanLayer:**

> "At scale, file-based coordination hits limits. Add runtime layers for real-time multi-agent orchestration, but keep files as the authoritative contract."

ACF v2 makes this possible without breaking ACF v1's simplicity.

---

## Next Steps

1. **Validate** this design with stakeholders
2. **Prototype** Tier 1 features (daemon + database + environments)
3. **Test** with real multi-agent workflows
4. **Iterate** based on feedback
5. **Document** migration path
6. **Release** ACF v2.0

**Estimated timeline:** 8-10 months for full v2 implementation
**Recommended approach:** Ship Tier 1 as v2.0-alpha, gather feedback, iterate
