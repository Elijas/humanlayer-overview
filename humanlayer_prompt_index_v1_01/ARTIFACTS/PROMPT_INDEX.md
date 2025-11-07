# HumanLayer Repository - Complete Prompt Index

## Executive Summary

**Total Prompt Files: 41**
- 27 Slash Commands (`.claude/commands/*.md`)
- 6 Agent Prompts (`.claude/agents/*.md`)
- 8 Component Guidance Files (`**/CLAUDE.md`)
- 0 Other prompt-related files

**Total Lines: ~6,150** across all prompt files

**Purpose**: This index catalogs every prompt, agent configuration, and guidance document in the HumanLayer repository. Each entry includes location, line count, purpose, and key characteristics.

---

## Table of Contents

1. [Slash Commands](#1-slash-commands) (27 files)
2. [Agent Prompts](#2-agent-prompts) (6 files)
3. [Component Guidance](#3-component-guidance) (8 files)
4. [Summary Statistics](#4-summary-statistics)
5. [Cross-Reference Index](#5-cross-reference-index)

---

## 1. Slash Commands

All slash commands live in `::PROJECT/.claude/commands/` and are invoked with `/command_name` in Claude Code sessions.

### 1.1 Commit & PR Management (5 files)

#### `ci_commit.md`
- **Path**: `::PROJECT/.claude/commands/ci_commit.md`
- **Lines**: 34
- **Purpose**: Create git commits for session changes with clear, atomic messages (CI mode - no user confirmation)
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Fully automated commit workflow without user approval
  - Reviews git status and diffs to create atomic commits
  - Never commits `thoughts/` directory or dummy files
  - Uses `git commit -m` directly after adding specific files
- **When to Use**: CI/CD pipelines, automated workflows
- **Related Commands**: `commit.md` (interactive version)

#### `commit.md`
- **Path**: `::PROJECT/.claude/commands/commit.md`
- **Lines**: 44
- **Purpose**: Create git commits with user approval and no Claude attribution
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Interactive commit workflow with user confirmation
  - Presents commit plan before execution
  - Never adds co-author information or Claude attribution
  - Groups related changes into atomic commits
  - Uses specific files with `git add`, never `-A` or `.`
- **When to Use**: Standard development workflow when user wants to review commits
- **Related Commands**: `ci_commit.md` (automated version)

#### `ci_describe_pr.md`
- **Path**: `::PROJECT/.claude/commands/ci_describe_pr.md`
- **Lines**: 76
- **Purpose**: Generate comprehensive PR descriptions following repository templates (CI mode)
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Reads PR description template from `thoughts/shared/pr_description.md`
  - Gathers full PR diff and commit history via `gh pr diff`
  - Runs verification commands and marks checklist items
  - Updates PR description directly with `gh pr edit`
  - Syncs to thoughts directory via `humanlayer thoughts sync`
- **When to Use**: CI/CD pipelines for automated PR descriptions
- **Related Commands**: `describe_pr.md` (interactive version)

#### `describe_pr.md`
- **Path**: `::PROJECT/.claude/commands/describe_pr.md`
- **Lines**: 77
- **Purpose**: Generate comprehensive PR descriptions following repository templates
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Interactive version with user confirmation
  - Reads template from `thoughts/shared/pr_description.md`
  - Analyzes changes thoroughly with "ultrathink"
  - Runs verification commands when possible
  - Saves to `thoughts/shared/prs/{number}_description.md`
  - Syncs with `humanlayer thoughts sync`
- **When to Use**: Standard workflow for creating PR descriptions
- **Related Commands**: `describe_pr_nt.md` (no thoughts variant), `ci_describe_pr.md` (CI version)

#### `describe_pr_nt.md`
- **Path**: `::PROJECT/.claude/commands/describe_pr_nt.md`
- **Lines**: 90
- **Purpose**: Generate comprehensive PR descriptions following repository templates (no thoughts directory)
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Uses inline template instead of reading from thoughts/
  - Saves to `/tmp/{repo_name}/prs/{number}_description.md`
  - No sync step
  - Otherwise follows same process as describe_pr.md
- **When to Use**: Projects without thoughts directory integration
- **Related Commands**: `describe_pr.md` (thoughts variant)

---

### 1.2 Planning & Research (9 files)

#### `create_plan.md`
- **Path**: `::PROJECT/.claude/commands/create_plan.md`
- **Lines**: 450
- **Purpose**: Create detailed implementation plans through interactive research and iteration
- **Model**: opus
- **Key Characteristics**:
  - Interactive planning process with skeptical, thorough approach
  - Spawns parallel research agents: codebase-locator, codebase-analyzer, thoughts-locator
  - Reads all mentioned files FULLY before spawning sub-tasks (critical ordering)
  - Creates phased implementation plans with automated and manual success criteria
  - No open questions in final plan - all decisions made before finalization
  - Uses TodoWrite to track planning tasks
  - Syncs via `humanlayer thoughts sync`
- **When to Use**: Creating new implementation plans with full research
- **Related Commands**: `create_plan_generic.md`, `create_plan_nt.md`, `iterate_plan.md`
- **Output**: `thoughts/shared/plans/{TICKET}-{description}.md`

#### `create_plan_generic.md`
- **Path**: `::PROJECT/.claude/commands/create_plan_generic.md`
- **Lines**: 443
- **Purpose**: Create detailed implementation plans with thorough research and iteration (generic version)
- **Model**: opus
- **Key Characteristics**:
  - Similar to create_plan.md but without thoughts-specific agents
  - Uses specialized agents for different research types
  - Maintains separation between automated and manual verification criteria
  - Focuses on incremental, testable changes
  - Includes common patterns for database changes, new features, refactoring
- **When to Use**: Alternative planning workflow, possibly for external projects
- **Related Commands**: `create_plan.md`, `create_plan_nt.md`

#### `create_plan_nt.md`
- **Path**: `::PROJECT/.claude/commands/create_plan_nt.md`
- **Lines**: 440
- **Purpose**: Create implementation plans with thorough research (no thoughts directory)
- **Model**: opus
- **Key Characteristics**:
  - Variant for projects without thoughts directory
  - Saves plans to `thoughts/shared/plans/` but no sync step
  - Otherwise follows same interactive planning approach
  - Uses specialized research agents
  - Maintains quality standards for success criteria
- **When to Use**: Projects without thoughts directory integration
- **Related Commands**: `create_plan.md` (thoughts variant)

#### `iterate_plan.md`
- **Path**: `::PROJECT/.claude/commands/iterate_plan.md`
- **Lines**: 250
- **Purpose**: Iterate on existing implementation plans with thorough research and updates
- **Model**: opus
- **Key Characteristics**:
  - Updates existing plans based on user feedback
  - Only spawns research if changes require new technical understanding
  - Makes surgical edits, not wholesale rewrites
  - Confirms understanding before making changes
  - Syncs with `humanlayer thoughts sync`
  - Maintains quality standards for success criteria
- **When to Use**: Refining existing plans, adapting to feedback
- **Related Commands**: `iterate_plan_nt.md`, `create_plan.md`

#### `iterate_plan_nt.md`
- **Path**: `::PROJECT/.claude/commands/iterate_plan_nt.md`
- **Lines**: 239
- **Purpose**: Iterate on existing implementation plans with thorough research and updates (no thoughts sync)
- **Model**: opus
- **Key Characteristics**:
  - Same as iterate_plan.md but without sync step
  - Makes focused, precise edits to existing plans
  - Be skeptical and surgical with changes
  - Preserves good content that doesn't need changing
- **When to Use**: Projects without thoughts directory, plan refinement
- **Related Commands**: `iterate_plan.md` (thoughts variant)

#### `research_codebase.md`
- **Path**: `::PROJECT/.claude/commands/research_codebase.md`
- **Lines**: 214
- **Purpose**: Document codebase as-is with thoughts directory for historical context
- **Model**: opus
- **Key Characteristics**:
  - **CRITICAL**: Only document and explain, never suggest improvements
  - Spawns parallel sub-agents: codebase-locator, codebase-analyzer, thoughts-locator
  - Reads mentioned files FULLY before spawning sub-tasks
  - Runs `hack/spec_metadata.sh` for metadata
  - Creates research docs with YAML frontmatter
  - Handles thoughts/searchable/ paths correctly
  - Syncs with `humanlayer thoughts sync`
- **When to Use**: Documenting how existing code works
- **Related Commands**: `research_codebase_generic.md`, `research_codebase_nt.md`
- **Philosophy**: Documentarian, not critic - explains HOW without WHY or SHOULD

#### `research_codebase_generic.md`
- **Path**: `::PROJECT/.claude/commands/research_codebase_generic.md`
- **Lines**: 180
- **Purpose**: Research codebase comprehensively using parallel sub-agents
- **Model**: opus
- **Key Characteristics**:
  - Generic version without thoughts-specific functionality
  - Uses specialized research agents
  - Gathers metadata for research documents
  - Creates structured research with frontmatter
  - Handles follow-up questions
- **When to Use**: Alternative research workflow
- **Related Commands**: `research_codebase.md`, `research_codebase_nt.md`

#### `research_codebase_nt.md`
- **Path**: `::PROJECT/.claude/commands/research_codebase_nt.md`
- **Lines**: 191
- **Purpose**: Document codebase as-is without evaluation or recommendations
- **Model**: opus
- **Key Characteristics**:
  - **CRITICAL**: Document only, never evaluate or suggest improvements
  - No thoughts directory sync
  - Uses Bash tools to generate metadata
  - Creates research docs at `thoughts/shared/research/`
  - Focuses on technical mapping/documentation
- **When to Use**: Projects without thoughts directory
- **Related Commands**: `research_codebase.md` (thoughts variant)

#### `validate_plan.md`
- **Path**: `::PROJECT/.claude/commands/validate_plan.md`
- **Lines**: 167
- **Purpose**: Validate implementation against plan, verify success criteria, identify issues
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Validates that implementation plan was correctly executed
  - Runs comprehensive checks: `make check test`
  - Spawns parallel research tasks for discovery
  - Checks completion status of each phase
  - Generates validation report with pass/fail status
  - Assesses manual testing requirements
  - Identifies deviations and potential issues
- **When to Use**: After implementation, before PR/handoff
- **Related Commands**: `implement_plan.md`

---

### 1.3 Implementation & Workflow (3 files)

#### `implement_plan.md`
- **Path**: `::PROJECT/.claude/commands/implement_plan.md`
- **Lines**: 85
- **Purpose**: Implement technical plans from thoughts/shared/plans with verification
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Reads plan completely and checks for existing checkmarks
  - Implements each phase fully before moving to next
  - Runs success criteria checks (usually `make check test`)
  - Updates checkboxes in plan file itself using Edit
  - Pauses for human verification after automated checks pass
  - Adapts when plan doesn't match reality (asks for guidance)
- **When to Use**: Executing approved implementation plans
- **Related Commands**: `validate_plan.md`, `create_plan.md`

#### `create_worktree.md`
- **Path**: `::PROJECT/.claude/commands/create_worktree.md`
- **Lines**: 42
- **Purpose**: Create worktree and launch implementation session for a plan
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Reads `hack/create_worktree.sh` script
  - Creates worktree with Linear branch name
  - Uses relative paths for plan files (thoughts/ synced between worktrees)
  - Confirms details with user before launching
  - Launches with `humanlayer launch --model opus`
- **When to Use**: Starting implementation in isolated worktree
- **Related Commands**: `local_review.md`, `implement_plan.md`

#### `debug.md`
- **Path**: `::PROJECT/.claude/commands/debug.md`
- **Lines**: 201
- **Purpose**: Debug issues by investigating logs, database state, and git history
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Pure investigation command, no file editing
  - Checks MCP logs: `~/.humanlayer/logs/mcp-claude-approvals-*.log`
  - Checks combined logs: `~/.humanlayer/logs/wui-${BRANCH_NAME}/codelayer.log`
  - Queries SQLite database: `~/.humanlayer/daemon-{BRANCH_NAME}.db`
  - Spawns parallel Task agents for efficient investigation
  - Generates structured debug report with evidence and next steps
- **When to Use**: Investigating bugs, understanding system state
- **Related Commands**: None (investigation-focused)

---

### 1.4 Session Management (2 files)

#### `create_handoff.md`
- **Path**: `::PROJECT/.claude/commands/create_handoff.md`
- **Lines**: 96
- **Purpose**: Create handoff document for transferring work to another session
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Creates thorough but concise context compaction documents
  - Uses specific naming convention: `YYYY-MM-DD_HH-MM-SS_ENG-XXXX_description.md`
  - Runs `scripts/spec_metadata.sh` for metadata generation
  - Includes sections: Tasks, Recent changes, Learnings, Artifacts, Action Items
  - Syncs with `humanlayer thoughts sync`
  - Avoids excessive code snippets, prefers file:line references
- **When to Use**: Ending work session, transferring to another agent/human
- **Related Commands**: `resume_handoff.md`
- **Output**: `thoughts/shared/handoffs/ENG-XXXX/{timestamp}_{description}.md`

#### `resume_handoff.md`
- **Path**: `::PROJECT/.claude/commands/resume_handoff.md`
- **Lines**: 218
- **Purpose**: Resume work from handoff document with context analysis and validation
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Can resume from path or ticket number (ENG-XXXX)
  - Locates most recent handoff in `thoughts/shared/handoffs/ENG-XXXX/`
  - Reads handoff and all referenced artifacts FULLY (no sub-agent)
  - Spawns research tasks to verify current state
  - Validates all mentioned changes still exist
  - Creates action plan with TodoWrite
  - Handles diverged codebase scenarios
- **When to Use**: Starting work from previous session's handoff
- **Related Commands**: `create_handoff.md`

---

### 1.5 Linear Integration (5 files)

#### `linear.md`
- **Path**: `::PROJECT/.claude/commands/linear.md`
- **Lines**: 389
- **Purpose**: Manage Linear tickets - create, update, comment, and follow workflow patterns
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Creates tickets from thoughts documents
  - Follows specific workflow: Triage → Spec Needed → Research → Plan → Dev → Done
  - Maps thoughts/ paths to GitHub URLs in links parameter
  - Applies automatic labels: hld, wui, meta
  - Default project: "M U L T I C L A U D E"
  - All tickets must have "problem to solve" section
  - Includes hardcoded IDs for teams, labels, states, users
- **When to Use**: Creating/managing Linear tickets
- **Related Commands**: `ralph_*` automation commands
- **Workflow States**: Triage, Research Needed, Research In Progress, Research In Review, Spec Needed, Plan In Progress, Plan In Review, Ready for Dev, In Dev, In Review, Done

#### `ralph_research.md`
- **Path**: `::PROJECT/.claude/commands/ralph_research.md`
- **Lines**: 82
- **Purpose**: Research highest priority Linear ticket needing investigation
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Fetches top 10 priority items in "research needed"
  - Selects highest priority issue
  - Moves to "research in progress"
  - Conducts unbiased research (documents how things work today)
  - Creates research doc: `thoughts/shared/research/YYYY-MM-DD-ENG-XXXX-description.md`
  - Moves to "research in review" when complete
- **When to Use**: Automated research workflow (LinearLayer bot)
- **Related Commands**: `ralph_plan.md`, `ralph_impl.md`
- **Workflow**: Research Needed → Research In Progress → Research In Review

#### `ralph_plan.md`
- **Path**: `::PROJECT/.claude/commands/ralph_plan.md`
- **Lines**: 60
- **Purpose**: Create implementation plan for highest priority Linear ticket ready for spec
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Fetches top 10 priority items in "ready for spec"
  - Selects highest priority SMALL or XS issue
  - Moves to "plan in progress" using MCP tools
  - Creates plan following create_plan.md instructions
  - Attaches plan to ticket using MCP tools
  - Moves to "plan in review" when complete
- **When to Use**: Automated planning workflow (LinearLayer bot)
- **Related Commands**: `ralph_research.md`, `ralph_impl.md`, `create_plan.md`
- **Workflow**: Ready for Spec → Plan In Progress → Plan In Review

#### `ralph_impl.md`
- **Path**: `::PROJECT/.claude/commands/ralph_impl.md`
- **Lines**: 34
- **Purpose**: Implement highest priority small Linear ticket with worktree setup
- **Model**: sonnet
- **Key Characteristics**:
  - Fetches top 10 priority items in "ready for dev" status
  - Selects highest priority SMALL or XS issue
  - Moves to "in dev" using MCP tools
  - Creates worktree with Linear branch name
  - Launches implementation session with opus model
  - Uses `humanlayer-nightly launch` command
- **When to Use**: Automated implementation workflow (LinearLayer bot)
- **Related Commands**: `ralph_plan.md`, `implement_plan.md`
- **Workflow**: Ready for Dev → In Dev

#### `founder_mode.md`
- **Path**: `::PROJECT/.claude/commands/founder_mode.md`
- **Lines**: 20
- **Purpose**: Create Linear ticket and PR for experimental features after implementation
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - For features that didn't get proper ticketing upfront
  - Creates Linear ticket after commit is made
  - Cherry-picks commit to new branch
  - Creates PR with `gh pr create --fill`
  - Follows up with describe_pr.md
- **When to Use**: Experimental features, quick prototypes that need proper ticketing
- **Related Commands**: `linear.md`, `describe_pr.md`

---

### 1.6 Automation & Orchestration (3 files)

#### `oneshot.md`
- **Path**: `::PROJECT/.claude/commands/oneshot.md`
- **Lines**: 7
- **Purpose**: Research ticket and launch planning session
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Calls `/ralph_research` with ticket number
  - Launches new session with opus model
  - Skips permissions for 14m
  - Calls `/oneshot_plan ENG-XXXX`
- **When to Use**: End-to-end automation: research → plan
- **Related Commands**: `oneshot_plan.md`, `ralph_research.md`

#### `oneshot_plan.md`
- **Path**: `::PROJECT/.claude/commands/oneshot_plan.md`
- **Lines**: 7
- **Purpose**: Execute ralph plan and implementation for a ticket
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Calls `/ralph_plan` with ticket number
  - Calls `/ralph_impl` with ticket number
  - End-to-end automation
- **When to Use**: End-to-end automation: plan → implement
- **Related Commands**: `oneshot.md`, `ralph_plan.md`, `ralph_impl.md`

#### `local_review.md`
- **Path**: `::PROJECT/.claude/commands/local_review.md`
- **Lines**: 49
- **Purpose**: Set up worktree for reviewing colleague's branch
- **Model**: Default (sonnet)
- **Key Characteristics**:
  - Parses format: `gh_username:branchName`
  - Adds remote and creates worktree
  - Copies Claude settings: `cp .claude/settings.local.json`
  - Runs `make setup` in worktree
  - Initializes thoughts: `humanlayer thoughts init --directory humanlayer`
- **When to Use**: Code review setup, testing colleague's branch
- **Related Commands**: `create_worktree.md`

---

## 2. Agent Prompts

All agent prompts live in `::PROJECT/.claude/agents/` and are invoked via the Task tool with `subagent_type` parameter.

### 2.1 Codebase Research Agents (3 files)

#### `codebase-locator.md`
- **Path**: `::PROJECT/.claude/agents/codebase-locator.md`
- **Lines**: 123
- **Purpose**: Locates files, directories, and components relevant to a feature or task
- **Description**: "Locates files, directories, and components relevant to a feature or task. Call `codebase-locator` with human language prompt."
- **Tools**: Grep, Glob, LS
- **Model**: sonnet
- **Agent Type**: File finder - finds WHERE code lives
- **Key Characteristics**:
  - **CRITICAL**: Only document locations, never critique structure
  - Searches by topic/feature
  - Categorizes: Implementation, Tests, Config, Docs, Types
  - Returns structured results grouped by purpose
  - Checks multiple naming patterns and extensions
  - Never reads file contents, just reports locations
- **When to Use**: Finding relevant files for a feature/task
- **Output Format**: Categorized file lists with paths
- **Related Agents**: `codebase-analyzer.md`, `codebase-pattern-finder.md`

#### `codebase-analyzer.md`
- **Path**: `::PROJECT/.claude/agents/codebase-analyzer.md`
- **Lines**: 144
- **Purpose**: Analyzes codebase implementation details
- **Description**: "Analyzes codebase implementation details. Call the codebase-analyzer agent when you need to find detailed information about specific components."
- **Tools**: Read, Grep, Glob, LS
- **Model**: sonnet
- **Agent Type**: Documentarian - explains HOW code works without critique
- **Key Characteristics**:
  - **CRITICAL**: Only document, never suggest improvements
  - Analyzes implementation details, traces data flow
  - Identifies architectural patterns as they exist
  - Provides file:line references for all claims
  - Structured output: Overview, Entry Points, Core Implementation, Data Flow
  - Never evaluates code quality or identifies bugs
- **When to Use**: Understanding how existing code works
- **Output Format**: Structured analysis with file:line references
- **Related Agents**: `codebase-locator.md`, `codebase-pattern-finder.md`
- **Philosophy**: Documentarian, not critic

#### `codebase-pattern-finder.md`
- **Path**: `::PROJECT/.claude/agents/codebase-pattern-finder.md`
- **Lines**: 228
- **Purpose**: Finds similar implementations, usage examples, or existing patterns
- **Description**: "Finds similar implementations, usage examples, or existing patterns that can be modeled after. It will give you concrete code examples!"
- **Tools**: Grep, Glob, Read, LS
- **Model**: sonnet
- **Agent Type**: Pattern cataloger - shows existing patterns as they are
- **Key Characteristics**:
  - **CRITICAL**: Show patterns without evaluation or preference
  - Finds similar implementations to use as templates
  - Extracts reusable patterns with code snippets
  - Shows multiple variations that exist
  - Includes test patterns
  - Never recommends which pattern is "better"
- **When to Use**: Finding examples to model new code after
- **Output Format**: Code snippets with context and file:line references
- **Related Agents**: `codebase-locator.md`, `codebase-analyzer.md`
- **Philosophy**: Catalog patterns, don't judge them

---

### 2.2 Thoughts Research Agents (2 files)

#### `thoughts-locator.md`
- **Path**: `::PROJECT/.claude/agents/thoughts-locator.md`
- **Lines**: 128
- **Purpose**: Discovers relevant documents in thoughts/ directory
- **Description**: "The `thoughts` equivalent of `codebase-locator`. Used for figuring out if we have random thoughts written down."
- **Tools**: Grep, Glob, LS
- **Model**: sonnet
- **Agent Type**: Document finder for thoughts/ directory
- **Key Characteristics**:
  - Searches thoughts/shared/, thoughts/allison/, thoughts/global/
  - Handles thoughts/searchable/ (read-only search directory)
  - Categorizes: Tickets, Research, Plans, PRs, Notes
  - **CRITICAL**: Corrects searchable/ paths to actual editable paths
  - Only removes "searchable/" from path, preserves all other structure
  - Never analyzes document contents deeply
- **When to Use**: Finding existing thoughts documents
- **Output Format**: Categorized document lists with corrected paths
- **Related Agents**: `thoughts-analyzer.md`
- **Special Handling**: Automatically converts `thoughts/searchable/X/Y` → `thoughts/X/Y`

#### `thoughts-analyzer.md`
- **Path**: `::PROJECT/.claude/agents/thoughts-analyzer.md`
- **Lines**: 146
- **Purpose**: Extracts high-value insights from thoughts documents
- **Description**: "The research equivalent of codebase-analyzer. Use this when wanting to deep dive on a research topic."
- **Tools**: Read, Grep, Glob, LS
- **Model**: sonnet
- **Agent Type**: Insight curator - extracts actionable information
- **Key Characteristics**:
  - Deeply analyzes documents and filters noise
  - Extracts: Key decisions, constraints, technical specs, actionable insights
  - Filters aggressively: removes exploration, rejected options, superseded info
  - Validates relevance and applicability
  - Structured output: Context, Decisions, Constraints, Specifications
  - Questions everything: "Why should user care about this?"
- **When to Use**: Deep analysis of thoughts documents
- **Output Format**: Curated insights with context
- **Related Agents**: `thoughts-locator.md`
- **Philosophy**: Aggressive filtering for high signal-to-noise ratio

---

### 2.3 Web Research Agent (1 file)

#### `web-search-researcher.md`
- **Path**: `::PROJECT/.claude/agents/web-search-researcher.md`
- **Lines**: 110
- **Purpose**: Researches information from web sources
- **Description**: "Use when you need information that you're not well-trained on. Modern information only discoverable on the web."
- **Tools**: WebSearch, WebFetch, TodoWrite, Read, Grep, Glob, LS
- **Model**: sonnet
- **Color**: yellow
- **Agent Type**: Web research specialist
- **Key Characteristics**:
  - Focuses on accurate, relevant web information
  - Uses WebSearch and WebFetch
  - Executes strategic searches from multiple angles
  - Prioritizes official docs, reputable technical blogs
  - Synthesizes findings with exact quotes and attribution
  - Structured output: Summary, Detailed Findings, Resources, Gaps
  - Always cites sources with direct links
- **When to Use**: Need modern/external information not in training data
- **Output Format**: Synthesized research with citations
- **Related Agents**: None (specialized external research)

---

## 3. Component Guidance

CLAUDE.md files provide component-specific guidance to Claude Code when working in those directories.

### 3.1 Repository Root

#### `CLAUDE.md` (root)
- **Path**: `::PROJECT/CLAUDE.md`
- **Lines**: 89
- **Component**: Root - entire monorepo
- **Key Topics**:
  - Repository overview: HumanLayer SDK & Platform + Local Tools Suite
  - Project 1: humanlayer-ts/, humanlayer-go/, humanlayer-ts-vercel-ai-sdk/, docs/
  - Project 2: hld/, hlyr/, humanlayer-wui/, claudecode-go/
  - Architecture flow diagram
  - Development commands: make setup, check-test, check, test
  - TODO annotation system: TODO(0-4) and PERF
  - TypeScript: ES6+, strict config, CommonJS/ESM compatibility
  - Go: Context-first API design, generate mocks
- **Purpose**: High-level guidance for entire repository
- **When to Use**: Always visible when working in repo

---

### 3.2 Go Component (1 file)

#### `hld/CLAUDE.md`
- **Path**: `::PROJECT/hld/CLAUDE.md`
- **Lines**: 56
- **Component**: hld/ (HumanLayer Daemon - Go)
- **Key Topics**:
  - Daemon that powers the WUI
  - Cannot run or restart process - must ask user to rebuild
  - Logs: `~/.humanlayer/logs/daemon-*.log`
  - WUI logs: `~/.humanlayer/logs/wui-{branch}/codelayer.log`
  - Database: `~/.humanlayer/*.db` (production and dev variants)
  - Unix sockets: `~/.humanlayer/daemon.sock` and `daemon-dev.sock`
  - Can test RPC calls with nc: `echo '{"jsonrpc":"2.0",...}' | nc -U SOCKET_PATH`
  - Go style: async goroutines accept context.Context, never store contexts on structs
  - See TESTING.md for testing guidelines
- **Purpose**: Guide Claude when working on Go daemon
- **Special Constraints**: Cannot restart daemon (breaks user's session)

---

### 3.3 Tauri/React Component (1 file)

#### `humanlayer-wui/CLAUDE.md`
- **Path**: `::PROJECT/humanlayer-wui/CLAUDE.md`
- **Lines**: 62
- **Component**: humanlayer-wui/ (Desktop UI - Tauri + React)
- **Key Topics**:
  - Desktop application for managing AI agent approvals
  - Logs: `~/.humanlayer/logs/wui-{branch-id}/codelayer.log`
  - Platform-specific production logs (macOS: ~/Library/Logs/...)
  - Communicates with daemon via JSON-RPC over Unix socket
  - Hot-reloads automatically on code changes
  - Prefer ShadCN components, Tailwind styling, Zustand for state
  - React 19: ref is standard prop, forwardRef deprecated
  - Vim-style keyboard navigation: j/k, shift+j/shift+k, x, e
  - Stateless anchor management for selections
- **Purpose**: Guide Claude when working on desktop UI
- **Special Features**: Vim-style navigation, real-time daemon communication

---

### 3.4 Bun/TypeScript Components (6 files - identical content)

All six files below have identical content (107 lines) focusing on Bun-first development:

#### `apps/daemon/CLAUDE.md`
- **Path**: `::PROJECT/apps/daemon/CLAUDE.md`
- **Lines**: 107
- **Component**: apps/daemon (TypeScript daemon)

#### `apps/react/CLAUDE.md`
- **Path**: `::PROJECT/apps/react/CLAUDE.md`
- **Lines**: 107
- **Component**: apps/react (React application)

#### `packages/contracts/CLAUDE.md`
- **Path**: `::PROJECT/packages/contracts/CLAUDE.md`
- **Lines**: 107
- **Component**: packages/contracts (TypeScript contracts)

#### `packages/database/CLAUDE.md`
- **Path**: `::PROJECT/packages/database/CLAUDE.md`
- **Lines**: 107
- **Component**: packages/database (TypeScript database)

#### `scripts/CLAUDE.md`
- **Path**: `::PROJECT/scripts/CLAUDE.md`
- **Lines**: 107
- **Component**: scripts/ (utility scripts)

**Shared Key Topics** (all 6 files):
- Default to Bun instead of Node.js
- Use `bun <file>`, `bun test`, `bun install`, `bun run`
- Bun APIs: Bun.serve(), bun:sqlite, Bun.redis, Bun.sql
- HTML imports with Bun.serve() instead of vite
- WebSocket support built-in
- React with Bun's bundler
- Bun.$ for shell commands instead of execa

**Purpose**: Guide Claude to use Bun-specific APIs and patterns
**Note**: These files could potentially be consolidated since they're identical

---

## 4. Summary Statistics

### 4.1 By Category

| Category | Files | Total Lines | Avg Lines/File |
|----------|-------|-------------|----------------|
| Slash Commands | 27 | ~4,500 | ~167 |
| Agent Prompts | 6 | ~900 | ~150 |
| Component Guidance | 8 | ~750 | ~94 |
| **TOTAL** | **41** | **~6,150** | **~150** |

### 4.2 By Purpose

| Purpose | Files | Description |
|---------|-------|-------------|
| **Planning & Research** | 9 | create_plan variants (3), research_codebase variants (3), validate_plan, iterate_plan (2) |
| **Implementation & Workflow** | 3 | implement_plan, create_worktree, debug |
| **Commit & PR Management** | 5 | commit variants (2), describe_pr variants (3) |
| **Linear Integration** | 5 | linear.md, ralph_* (3), founder_mode |
| **Session Management** | 2 | create_handoff, resume_handoff |
| **Automation** | 3 | oneshot variants (2), local_review |
| **Codebase Research Agents** | 3 | codebase-locator, codebase-analyzer, codebase-pattern-finder |
| **Thoughts Research Agents** | 2 | thoughts-locator, thoughts-analyzer |
| **Web Research Agent** | 1 | web-search-researcher |
| **Component Guidance** | 8 | CLAUDE.md files for components |

### 4.3 Model Distribution

| Model | Files | Usage Pattern |
|-------|-------|---------------|
| **sonnet** (default) | 30+ | Most commands, all agents, fast iteration |
| **opus** | 6 | Planning (create_plan variants 3x, iterate_plan 2x), research_codebase |
| **Not specified** | ~5 | Use session default |

**Pattern**: Opus reserved for complex planning/research requiring deep thinking; Sonnet for execution/automation.

### 4.4 Key Patterns

#### Variant System
- **`_nt` suffix**: "No Thoughts" - variants for projects without thoughts directory integration
  - `create_plan_nt.md`, `iterate_plan_nt.md`, `describe_pr_nt.md`, `research_codebase_nt.md`
- **`_generic` suffix**: Alternative implementation without project-specific assumptions
  - `create_plan_generic.md`, `research_codebase_generic.md`
- **`ci_` prefix**: CI/CD mode - automated, no user confirmation
  - `ci_commit.md`, `ci_describe_pr.md`

#### Agent Philosophy: Documentarian Over Critic
All research agents (codebase-*, thoughts-*) explicitly instructed to:
- ✅ Document HOW things work
- ✅ Show WHAT exists
- ❌ NEVER critique or suggest improvements
- ❌ NEVER evaluate quality
- ❌ NEVER recommend "better" approaches

This is a **critical pattern** - research agents are pure documentarians.

#### Parallel Research Pattern
Many commands spawn multiple Task agents concurrently:
- `create_plan.md`: codebase-locator + codebase-analyzer + thoughts-locator
- `debug.md`: Multiple parallel investigation tasks
- `validate_plan.md`: Parallel discovery tasks

**Critical Ordering**: Commands emphasize "Read files FULLY before spawning sub-tasks" to avoid agents working with stale assumptions.

#### Thoughts Integration
Commands with thoughts directory integration:
- Create/update: `create_plan`, `research_codebase`, `create_handoff`
- Read: `implement_plan`, `resume_handoff`, `describe_pr`
- Sync command: `humanlayer thoughts sync` (appears in 10+ commands)

#### TodoWrite Usage
Commands that track progress with TodoWrite:
- `create_plan.md` (extensive todo management)
- `resume_handoff.md` (creates action plan)
- `web-search-researcher.md` (tracks research tasks)

---

## 5. Cross-Reference Index

### 5.1 Workflow Chains

#### Research → Plan → Implement → Validate
```
/research_codebase ENG-XXXX
    ↓
/create_plan ENG-XXXX
    ↓
/implement_plan ENG-XXXX
    ↓
/validate_plan ENG-XXXX
    ↓
/describe_pr XXXX
    ↓
/commit
```

#### Automated Linear Workflow (Ralph)
```
/ralph_research          # Research Needed → Research In Progress
    ↓
/ralph_plan             # Ready for Spec → Plan In Progress
    ↓
/ralph_impl             # Ready for Dev → In Dev
```

#### Session Handoff
```
[Work in progress...]
    ↓
/create_handoff ENG-XXXX
    ↓
[New session starts]
    ↓
/resume_handoff ENG-XXXX
```

### 5.2 Agent Dependencies

**Commands → Agents**:
- `create_plan.md` → `codebase-locator`, `codebase-analyzer`, `thoughts-locator`
- `research_codebase.md` → `codebase-locator`, `codebase-analyzer`, `thoughts-locator`
- `validate_plan.md` → `codebase-analyzer`, `codebase-pattern-finder`
- `resume_handoff.md` → `codebase-locator`, `thoughts-locator`

**Agent Relationships**:
- `codebase-locator` (WHERE) → `codebase-analyzer` (HOW) → `codebase-pattern-finder` (EXAMPLES)
- `thoughts-locator` (WHICH DOCS) → `thoughts-analyzer` (WHAT MATTERS)

### 5.3 File Type Cross-Reference

#### Plans
- **Created by**: `create_plan.md`, `ralph_plan.md`
- **Updated by**: `iterate_plan.md`, `implement_plan.md` (checkboxes)
- **Consumed by**: `implement_plan.md`, `validate_plan.md`
- **Location**: `thoughts/shared/plans/{TICKET}-{description}.md`

#### Research Documents
- **Created by**: `research_codebase.md`, `ralph_research.md`
- **Consumed by**: `create_plan.md`, `thoughts-analyzer` agent
- **Location**: `thoughts/shared/research/YYYY-MM-DD-ENG-XXXX-{description}.md`

#### Handoffs
- **Created by**: `create_handoff.md`
- **Consumed by**: `resume_handoff.md`
- **Location**: `thoughts/shared/handoffs/ENG-XXXX/YYYY-MM-DD_HH-MM-SS_{description}.md`

#### PR Descriptions
- **Created by**: `describe_pr.md`, `ci_describe_pr.md`
- **Location**: `thoughts/shared/prs/{number}_description.md`

---

## 6. Usage Guidelines

### 6.1 When to Use Which Command

**Starting new work**:
- Unknown codebase → `/research_codebase` first
- Have requirements → `/create_plan` directly
- Resume previous work → `/resume_handoff ENG-XXXX`

**During implementation**:
- Executing plan → `/implement_plan`
- Need examples → Use `codebase-pattern-finder` agent
- Debugging → `/debug`
- Need to hand off → `/create_handoff`

**Finishing work**:
- Verify implementation → `/validate_plan`
- Create commits → `/commit`
- Create PR → `gh pr create --fill`
- Describe PR → `/describe_pr`

**Linear automation**:
- One-shot research+plan → `/oneshot ENG-XXXX`
- One-shot plan+implement → `/oneshot_plan ENG-XXXX`
- Fully automated → `/ralph_research`, `/ralph_plan`, `/ralph_impl`

### 6.2 Model Selection Guidance

**Use Opus** (slower, deeper thinking):
- Creating implementation plans (`create_plan`, `iterate_plan`)
- Researching unfamiliar codebase (`research_codebase`)
- Complex architectural decisions

**Use Sonnet** (faster, sufficient for most tasks):
- Executing known plans (`implement_plan`)
- All automation workflows (`ralph_*`, `oneshot_*`)
- Debugging, validation, PR descriptions
- All agent tasks (locator, analyzer, pattern-finder)

### 6.3 Thoughts Directory Integration

**Commands that require thoughts directory**:
- `create_plan.md`, `iterate_plan.md`
- `research_codebase.md`
- `create_handoff.md`, `resume_handoff.md`
- `describe_pr.md`

**Use `_nt` variants if no thoughts directory**:
- `create_plan_nt.md`
- `iterate_plan_nt.md`
- `describe_pr_nt.md`
- `research_codebase_nt.md`

### 6.4 Critical Patterns to Follow

1. **Read files FULLY before spawning agents** - prevents stale assumptions
2. **Use TodoWrite for complex multi-step work** - maintains progress visibility
3. **Prefer documentarian agents over direct file reading** - they're optimized for discovery
4. **Never suggest improvements in research** - document only, evaluate separately
5. **Sync thoughts after major updates** - `humanlayer thoughts sync`

---

## 7. Maintenance Notes

### 7.1 Duplicate Content Alert

**6 identical CLAUDE.md files** (Bun guidance):
- `apps/daemon/CLAUDE.md`
- `apps/react/CLAUDE.md`
- `packages/contracts/CLAUDE.md`
- `packages/database/CLAUDE.md`
- `scripts/CLAUDE.md`

**Recommendation**: Consider consolidating into a single `BUNTYPESCRIPT.md` file and having component files reference it, or use Claude Code's inheritance mechanism if available.

### 7.2 Naming Conventions

- Slash commands: lowercase with underscores (`create_plan.md`)
- Agents: lowercase with hyphens (`codebase-analyzer.md`)
- Component guidance: UPPERCASE (`CLAUDE.md`)
- Variants: `_nt` (no thoughts), `_generic` (generic), `ci_` (CI mode)

### 7.3 Frontmatter Standards

Most command files use YAML frontmatter for metadata:
```yaml
---
model: opus  # or sonnet, or omit for default
---
```

Agent files include:
```yaml
---
tools: [Read, Grep, Glob, LS]
model: sonnet
color: yellow  # optional
---
```

---

## 8. Changelog

### v1.01 - 2025-11-06
- Initial comprehensive index created
- 41 files cataloged across 3 categories
- ~6,150 total lines documented
- Cross-reference index added
- Usage guidelines documented
