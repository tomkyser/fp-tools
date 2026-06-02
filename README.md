> **Important!** This repo is an in progress development for an internal distribution method of tools built for an internal team. The current state of this is experimental and not guarenteed to work without fine tuning or tweaking on a case by case basis per user OR may not be up to date with underlying packages this is meant to provide and support. The underlying tool packages are also in active development. Consider this as a preview of aspirational functionality for now.

# fp-docs

Documentation management system for the Foreign Policy WordPress codebase. fp-docs is a Claude Code plugin that automates the creation, revision, validation, and maintenance of technical documentation by reading your source code directly and keeping docs in sync with every change.

fp-docs enforces zero-tolerance verbosity (every source item must be documented), cross-references every claim against actual code, manages citations with provenance tracking, and maintains a separate docs git repo that branch-mirrors your codebase. It ships 19 commands, 9 specialized engines (including a universal orchestration engine that coordinates multi-agent execution), and an automated 8-stage post-modification pipeline that runs after every documentation change.

### What Problems It Solves

- **Documentation drift**: Code changes but docs don't. fp-docs detects git diffs and auto-updates affected documentation.
- **Inaccurate documentation**: Docs claim things that aren't true. fp-docs sanity-checks every factual claim against actual source code with zero-tolerance verification.
- **Incomplete documentation**: LLMs naturally summarize and truncate. fp-docs enforces anti-compression rules that ban summarization phrases ("etc.", "and more", "various") and require exhaustive enumeration of every function, parameter, hook, and constant.
- **Missing citations**: Docs make claims without evidence. fp-docs generates verifiable code citations with file paths, symbol names, line ranges, and verbatim code excerpts.
- **Stale API references**: Function signatures change but reference tables don't. fp-docs extracts actual signatures from source and tracks provenance (PHPDoc, Verified, Authored).
- **Undocumented data contracts**: WordPress template components pass `$locals` arrays between files without formal contracts. fp-docs annotates source code with `@locals` PHPDoc blocks and generates contract tables.
- **Branch mismatch**: The docs repo and codebase repo can drift to different branches. fp-docs detects this on session start and synchronizes branches.
- **No documentation lifecycle**: Most codebases lack a systematic process from creation through maintenance to deprecation. fp-docs provides the full lifecycle: create, maintain, validate, deprecate.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Command Reference](#command-reference)
- [Common Workflows](#common-workflows)
- [Architecture](#architecture)
- [Development Guide](#development-guide)
- [Configuration Reference](#configuration-reference)
- [Troubleshooting](#troubleshooting)

---

## Installation

### Marketplace Install (Production)

```
/plugin marketplace add tomkyser/fp-docs
/plugin install fp-docs@fp-tools
```

### Local Development

```bash
claude --plugin-dir ~/cc-plugins/fp-docs/plugins/fp-docs
```

**Important**: Point `--plugin-dir` at `plugins/fp-docs/`, not the repo root. The repo root is the marketplace wrapper (`fp-tools`). The installable plugin lives at `plugins/fp-docs/`.

### Default Permissions

The plugin auto-allows Read, Grep, and Glob. Write, Edit, and Bash operations will prompt for user approval per-session. Validation commands run without permission prompts; modification commands will ask.

---

## Quick Start

1. **Install the plugin** using one of the methods above.

2. **Run setup** to verify your environment:

   ```
   /fp-docs:setup
   ```

   Setup checks plugin structure, detects the docs repo, verifies the codebase `.gitignore` includes the docs directory, and confirms branch sync status. It reports overall health as HEALTHY, DEGRADED, or BROKEN.

3. **Sync branches** if prompted:

   ```
   /fp-docs:sync
   ```

4. **Try your first command** -- run a quick audit to see the state of your docs:

   ```
   /fp-docs:audit --depth quick
   ```

5. **Update docs after code changes**:

   ```
   /fp-docs:auto-update
   ```

From here, the SessionStart hooks handle branch detection and plugin context injection automatically on every new session.

---

## Command Reference

### Summary Table

| Command | Specialist Engine | Description |
|---------|------------------|-------------|
| `/fp-docs:revise` | modify | Fix specific documentation you know is wrong |
| `/fp-docs:add` | modify | Create docs for new code |
| `/fp-docs:auto-update` | modify | Auto-detect code changes and update affected docs |
| `/fp-docs:auto-revise` | modify | Batch-process the needs-revision tracker |
| `/fp-docs:deprecate` | modify | Mark docs as deprecated or removed |
| `/fp-docs:audit` | validate | Compare docs against source code |
| `/fp-docs:verify` | validate | Run 10-point verification checklist |
| `/fp-docs:sanity-check` | validate | Zero-tolerance claim validation |
| `/fp-docs:test` | validate | Runtime tests against local dev environment |
| `/fp-docs:citations` | citations | Manage code citations (generate/update/verify/audit) |
| `/fp-docs:api-ref` | api-refs | Generate or audit API Reference sections |
| `/fp-docs:locals` | locals | Manage locals contract documentation |
| `/fp-docs:verbosity-audit` | verbosity | Scan for verbosity gaps and summarization |
| `/fp-docs:update-index` | index | Refresh PROJECT-INDEX.md |
| `/fp-docs:update-claude` | index | Regenerate CLAUDE.md template |
| `/fp-docs:update-skills` | system | Regenerate skill files from definitions |
| `/fp-docs:setup` | system | Initialize or verify installation |
| `/fp-docs:sync` | system | Synchronize docs branch with codebase branch |
| `/fp-docs:parallel` | orchestrate | Run operations in parallel across files |

All commands route through the **orchestrate** engine, which delegates to the specialist engine listed above. Write operations use 3+ agents (orchestrator + specialist + validator). Read-only operations use 2 agents (orchestrator + specialist).

### Documentation Lifecycle Commands

#### `/fp-docs:revise`

Fix documentation you know is wrong or outdated. Reads the doc and its corresponding source code, identifies discrepancies, and makes targeted corrections.

```
/fp-docs:revise "fix the posts helper documentation"
/fp-docs:revise "the ad insertion shortcode docs say it defaults to 'sidebar' but it actually defaults to 'inline'"
/fp-docs:revise "update the newsletter taxonomy docs to reflect the new 'frequency' term meta field"
```

#### `/fp-docs:add`

Create documentation for new code that has no docs yet. Finds a sibling doc in the same section as a format template, reads the new source code, and generates complete documentation.

```
/fp-docs:add "document the new meilisearch helper"
/fp-docs:add "new podcast post type at inc/post-types/podcast.php"
/fp-docs:add "the new paywall feature at features/paywall/"
```

#### `/fp-docs:auto-update`

Auto-detect recent code changes (via `git diff`) and update all affected docs. Maps changed source files to documentation targets using the source-to-docs mapping table, then updates each affected doc.

```
/fp-docs:auto-update
/fp-docs:auto-update "only helpers/"
/fp-docs:auto-update "just the posts helper"
```

#### `/fp-docs:auto-revise`

Batch-process items from the `needs-revision-tracker.md` file. Moves completed items to the Completed section with timestamps.

```
/fp-docs:auto-revise                    # Process all pending items
/fp-docs:auto-revise --item 3           # Process only item #3
/fp-docs:auto-revise --item "posts"     # Process item matching "posts"
/fp-docs:auto-revise --range 1-5        # Process items 1 through 5
/fp-docs:auto-revise --dry-run          # Preview without making changes
```

#### `/fp-docs:deprecate`

Handle deprecated or removed code. For deprecated code still in the codebase, adds `[LEGACY]` markers and deprecation notices. For removed code, adds REMOVED notices and cleans up cross-references.

```
/fp-docs:deprecate "the AMP integration was removed"
/fp-docs:deprecate "the legacy gallery shortcode is deprecated, replaced by the new media-grid component"
```

### Validation Commands

These commands are read-only -- they never modify files.

#### `/fp-docs:audit`

Compare docs against source code at three depth levels:

- **quick**: File existence, source-to-doc cross-references, markdown link validation
- **standard** (default): Quick checks plus git change detection from the last 30 days
- **deep**: Standard checks plus full content comparison of every doc against its source

```
/fp-docs:audit --depth quick
/fp-docs:audit --depth deep docs/06-helpers/
/fp-docs:audit --depth standard --section 02
```

#### `/fp-docs:verify`

Run the 10-point verification checklist: file existence, orphan check, index completeness, appendix spot-check, link validation, changelog check, citation format, API reference provenance, locals contracts, verbosity compliance.

```
/fp-docs:verify
/fp-docs:verify docs/06-helpers/
/fp-docs:verify docs/02-post-types/article.md
```

#### `/fp-docs:sanity-check`

Cross-reference every factual claim in a doc against source code. Classifies each claim as VERIFIED, MISMATCH, HALLUCINATION, or UNVERIFIABLE. Returns an overall confidence level (HIGH or LOW).

```
/fp-docs:sanity-check docs/06-helpers/posts.md
/fp-docs:sanity-check docs/09-api/
```

#### `/fp-docs:test`

Validate documentation against a running local dev environment. Requires `https://foreignpolicy.local/` to be accessible and `ddev wp` for WP-CLI.

```
/fp-docs:test rest-api     # Test REST endpoint docs against live API
/fp-docs:test cli          # Test CLI docs against actual WP-CLI output
/fp-docs:test templates    # Verify template files exist at documented paths
```

### Citations and References

#### `/fp-docs:citations`

Manage code citations embedded in documentation. Subcommands:

| Subcommand | Description |
|------------|-------------|
| `generate` | Create new citation blocks for docs that lack them |
| `update` | Refresh stale citations after source code changed |
| `verify` | Check citation format compliance |
| `audit` | Deep accuracy check of existing citations |

```
/fp-docs:citations generate docs/06-helpers/posts.md
/fp-docs:citations update docs/06-helpers/
/fp-docs:citations verify
/fp-docs:citations audit docs/06-helpers/
```

#### `/fp-docs:api-ref`

Generate or audit API Reference table sections in documentation.

| Subcommand | Description |
|------------|-------------|
| `generate` | Extract function signatures from source and create reference tables |
| `audit` | Check existing API reference sections for accuracy |

```
/fp-docs:api-ref generate docs/06-helpers/posts.md
/fp-docs:api-ref audit docs/06-helpers/
```

### Component Contracts

#### `/fp-docs:locals`

Manage `$locals` contract documentation for WordPress template components.

| Subcommand | Description |
|------------|-------------|
| `annotate` | Add `@locals` PHPDoc annotations to template source files |
| `contracts` | Generate locals contract tables in documentation |
| `cross-ref` | Cross-reference locals across templates and consumers |
| `validate` | Validate contracts against actual code usage |
| `shapes` | Document shared shape definitions |
| `coverage` | Report locals documentation coverage |

```
/fp-docs:locals annotate components/
/fp-docs:locals contracts docs/05-components/
/fp-docs:locals validate
/fp-docs:locals coverage
```

#### `/fp-docs:verbosity-audit`

Scan documentation for verbosity gaps: missing items, summarization language, unexpanded enumerables.

```
/fp-docs:verbosity-audit --depth quick
/fp-docs:verbosity-audit --depth deep docs/06-helpers/
```

### Index and System Commands

#### `/fp-docs:update-index`

Refresh the PROJECT-INDEX.md codebase reference. Modes: `update` (incremental) or `full` (rebuild).

```
/fp-docs:update-index update
/fp-docs:update-index full
```

#### `/fp-docs:update-claude`

Regenerate the CLAUDE.md template with the current skill inventory.

#### `/fp-docs:update-skills`

Regenerate all plugin skill files from their prompt definitions.

#### `/fp-docs:setup`

Initialize or verify the plugin installation. Runs a 4-phase check: plugin structure, docs repo, codebase gitignore, and branch sync.

#### `/fp-docs:sync`

Synchronize the docs repo branch with the codebase branch.

```
/fp-docs:sync                 # Create or switch to matching docs branch
/fp-docs:sync merge           # Merge docs feature branch into docs master
/fp-docs:sync --force         # Force switch even with uncommitted changes
```

#### `/fp-docs:parallel`

Run any docs operation in parallel across multiple files using Agent Teams. Requires the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable. Falls back to sequential execution for scopes under 3 files.

```
/fp-docs:parallel auto-update docs/06-helpers/
/fp-docs:parallel audit --depth deep docs/
```

### The Documentation Lifecycle

The commands above map to a full documentation lifecycle:

1. **Create**: `/fp-docs:add` generates docs for new code; `/fp-docs:setup` initializes the environment
2. **Maintain**: `/fp-docs:revise`, `/fp-docs:auto-update`, `/fp-docs:auto-revise` keep docs current; citations, api-ref, locals, and index commands maintain specific subsystems
3. **Validate**: `/fp-docs:audit`, `/fp-docs:verify`, `/fp-docs:sanity-check`, `/fp-docs:test`, and `/fp-docs:verbosity-audit` check accuracy without modifying anything
4. **Deprecate**: `/fp-docs:deprecate` handles removed or replaced code with proper notices and tracker updates

### Common Flags

These flags work with modification commands (revise, add, auto-update, auto-revise, deprecate):

| Flag | Effect |
|------|--------|
| `--no-citations` | Skip citation generation/update |
| `--no-sanity-check` | Skip sanity-check stage |
| `--no-verbosity` | Skip verbosity enforcement |
| `--no-api-ref` | Skip API reference sync |
| `--no-index` | Skip PROJECT-INDEX update |
| `--mode plan` | Show what would change without executing |
| `--mode audit+plan` | Audit first, then show planned changes |

---

## Common Workflows

### Code Changed, Docs Need Updating

The most common scenario. You have made code changes and need docs to reflect them.

1. Start a session. The SessionStart hooks automatically check branch sync.
2. Run auto-update to detect and fix all affected docs:

   ```
   /fp-docs:auto-update
   ```

3. The command diffs recent commits, maps changed source files to their docs, updates each affected doc, and runs the full pipeline (verbosity, citations, sanity-check, verification, changelog, commit).

For a targeted fix where you know exactly what is wrong:

```
/fp-docs:revise "the posts helper now returns WP_Post objects instead of arrays"
```

### Writing Docs for New Code

You wrote new code and need documentation from scratch.

1. Make sure your code is committed (or at least saved).
2. Run add with a description:

   ```
   /fp-docs:add "new podcast post type at inc/post-types/podcast.php"
   ```

3. The engine finds a sibling doc in the same section for format guidance, reads your source code, generates complete documentation, and runs the full pipeline.
4. Review the output. If anything is marked `[NEEDS INVESTIGATION]`, resolve it with:

   ```
   /fp-docs:revise "clarify the podcast post type thumbnail sizes"
   ```

### Auditing Documentation Accuracy

You want to check how accurate your docs are without changing anything.

**Quick health check:**
```
/fp-docs:audit --depth quick
```

**Standard check** (includes recent git changes):
```
/fp-docs:audit --depth standard
```

**Deep check** (reads every doc and source file):
```
/fp-docs:audit --depth deep docs/06-helpers/
```

**Zero-tolerance claim verification** on a specific file:
```
/fp-docs:sanity-check docs/06-helpers/posts.md
```

**Full 10-point checklist:**
```
/fp-docs:verify
```

### Preparing Docs Before Merging a PR

Before merging a feature branch, make sure docs are complete and accurate.

1. Deep audit to find all discrepancies:

   ```
   /fp-docs:audit --depth deep
   ```

2. Fix issues found:

   ```
   /fp-docs:auto-update
   /fp-docs:revise "fix specific issue from audit"
   ```

3. Verify the 10-point checklist passes:

   ```
   /fp-docs:verify
   ```

4. Merge the docs branch into docs master:

   ```
   /fp-docs:sync merge
   ```

### Batch Processing Multiple Files

For large-scope operations across many files:

**Process all items in the revision tracker:**
```
/fp-docs:auto-revise
```

**Parallel operations** (requires Agent Teams):
```
/fp-docs:parallel auto-update docs/06-helpers/
/fp-docs:parallel audit --depth deep docs/
```

The chunk delegation system auto-triggers when scope exceeds 8 docs or 50 functions per agent.

---

## Architecture

### Repository Structure

```
fp-docs/                              # Git root (marketplace container)
├── .claude-plugin/
│   └── marketplace.json              # fp-tools marketplace definition
└── plugins/
    └── fp-docs/                      # THE ACTUAL PLUGIN (install target)
        ├── .claude-plugin/
        │   └── plugin.json           # Plugin manifest (v2.8.0)
        ├── settings.json             # Default permissions (Read, Grep, Glob)
        ├── agents/                   # 9 engine agent definitions
        ├── modules/                  # 11 shared modules (preloaded by engines)
        ├── skills/                   # 19 user-facing commands
        ├── hooks/
        │   └── hooks.json            # 4 hook event definitions
        ├── scripts/                  # 6 bash scripts (hooks + utility)
        └── framework/
            ├── manifest.md           # System manifest
            ├── config/               # system-config.md, project-config.md
            ├── algorithms/           # 6 on-demand algorithm files
            └── instructions/         # Per-engine instruction files
```

### Engine-Skill Routing Pattern

Every user command follows the same routing flow:

```
User: /fp-docs:revise "fix the posts helper"
  |
  v
Skill (skills/revise/SKILL.md)
  - Declares: agent: orchestrate, context: fork
  - Body: Engine: modify, Operation: revise, Instruction: ..., User request: $ARGUMENTS
  |
  v
Orchestrator (agents/orchestrate.md)
  - Parses routing metadata (Engine, Operation, Instruction)
  - Classifies command type (write/read-only/admin/batch)
  - Analyzes scope for delegation strategy
  |
  v
Write Phase: Specialist Engine (agents/modify.md — Mode: DELEGATED)
  - Executes primary operation + enforcement stages 1-3
  - Returns Delegation Result
  |
  v
Review Phase: Validate Engine (agents/validate.md — Mode: PIPELINE-VALIDATION)
  - Runs sanity-check (stage 4) + 10-point verification (stage 5)
  - Returns Pipeline Validation Report
  |
  v
Finalize Phase: Orchestrator
  - Stage 6: Changelog update
  - Stage 7: Index update (conditional)
  - Stage 8: Docs commit & push
  |
  v
Orchestration Report (aggregated from all phases)
```

**Skills** are thin routing files. They declare `agent: orchestrate` and `context: fork`, providing routing metadata (Engine, Operation, Instruction path) and the user's input (`$ARGUMENTS`). All logic lives in the engines and instruction files.

**The Orchestrator** is the universal entry point. It parses routing metadata, classifies the command, delegates to specialist engines, coordinates pipeline phases, and handles finalization. Multi-agent execution is the default.

**Specialist Engines** are domain-specific subagent definitions. When invoked in Delegated Mode, they execute only their primary operation and enforcement stages (1-3), returning a structured Delegation Result. In Standalone Mode (for backward compatibility), they execute the full pipeline.

**Instruction files** are the source of truth for operation behavior. They contain the step-by-step procedure for each specific operation.

### The 9 Engines

| Engine | Agent File | Role | Modules Preloaded |
|--------|-----------|------|-------------------|
| orchestrate | agents/orchestrate.md | Universal routing & delegation | mod-standards, mod-project, mod-pipeline, mod-changelog, mod-orchestration |
| modify | agents/modify.md | Write (docs) | mod-standards, mod-project, mod-pipeline, mod-changelog, mod-index |
| validate | agents/validate.md | Read-only | mod-standards, mod-project, mod-validation |
| citations | agents/citations.md | Write (citations) | mod-standards, mod-project, mod-citations |
| api-refs | agents/api-refs.md | Write (API refs) | mod-standards, mod-project, mod-api-refs |
| locals | agents/locals.md | Write (locals) | mod-standards, mod-project, mod-locals |
| verbosity | agents/verbosity.md | Read-only | mod-standards, mod-project, mod-verbosity |
| index | agents/index.md | Write (index) | mod-standards, mod-project, mod-index |
| system | agents/system.md | Admin | mod-standards, mod-project |

Read-only engines (`validate`, `verbosity`) enforce this with `disallowedTools: [Write, Edit]` in their frontmatter. All 8 specialist engines support Delegation Mode (invoked by the orchestrator) and Standalone Mode (backward-compatible direct invocation).

### The 8-Stage Post-Modification Pipeline

After every doc-modifying operation, the pipeline runs automatically:

| Stage | Name | Description | Skippable? |
|-------|------|-------------|------------|
| 1 | Verbosity Enforcement | Count source items, verify 100% coverage, ban summarization | Yes (config flag) |
| 2 | Citation Generation/Update | Generate new or refresh stale code citations | Yes (config flag) |
| 3 | API Reference Sync | Verify/update API reference tables | Yes (config flag) |
| 4 | Sanity Check | Cross-reference every claim against source code | Yes (`--no-sanity-check`) |
| 5 | Verification | Run the 10-point checklist | Never |
| 6 | Changelog Update | Append entry to `docs/changelog.md` | Never |
| 7 | Index Update | Update PROJECT-INDEX.md | Auto (only on structural changes) |
| 8 | Docs Repo Commit | `git -C {docs-root} add -A && commit` | Never (skips if no repo) |

On completion, the engine outputs a pipeline marker:
```
Pipeline complete: [verbosity: PASS] [citations: PASS] [sanity: HIGH] [verify: PASS] [changelog: updated] [docs-commit: committed]
```

The `post-modify-check.sh` SubagentStop hook validates this marker.

### Module System

The 11 modules in `modules/` are shared rule files preloaded into engines via the engine's `skills:` frontmatter. They are NOT user-invocable commands.

| Module | Purpose |
|--------|---------|
| mod-standards | Universal formatting, naming, structural, and depth rules |
| mod-project | FP-specific paths, source-to-doc mappings, environment config |
| mod-pipeline | 8-stage pipeline definition, trigger matrix, skip conditions, delegation protocol |
| mod-changelog | Changelog entry format and update procedure |
| mod-index | PROJECT-INDEX.md update rules and modes |
| mod-citations | Citation format, tiers, staleness model, provenance |
| mod-api-refs | API Reference table format, columns, scope rules |
| mod-locals | `$locals` contract format, shapes, validation rules |
| mod-validation | 10-point checklist, sanity-check algorithm, claim classification |
| mod-verbosity | Anti-compression rules, banned phrases, scope manifests |
| mod-orchestration | Delegation thresholds, batching strategy, report formats, team protocol |

**Key distinction**: Modules define WHAT (rules, formats, classification systems). On-demand algorithm files in `framework/algorithms/` define HOW (step-by-step procedures). Modules are always in the engine's context. Algorithm files are loaded only when needed during pipeline execution, then discarded.

### On-Demand Algorithm Files

| Algorithm File | Loaded During | Purpose |
|----------------|---------------|---------|
| verbosity-algorithm.md | Pipeline stage 1 | Item counting and coverage verification |
| citation-algorithm.md | Pipeline stage 2 | Citation generation and staleness detection |
| api-ref-algorithm.md | Pipeline stage 3 | API reference table construction |
| validation-algorithm.md | Pipeline stages 4-5 | Claim verification and 10-point checklist |
| codebase-analysis-guide.md | Source scanning | Guide for reading PHP/JS source files |
| git-sync-rules.md | Sync operations | Branch mirroring and diff report rules |

### Hook System

Four event types with six hook scripts:

| Event | Script | Purpose |
|-------|--------|---------|
| SessionStart | `inject-manifest.sh` | Inject plugin root path and manifest into context |
| SessionStart | `branch-sync-check.sh` | Detect branch mismatch, warn if out of sync |
| SubagentStop (modify) | `post-modify-check.sh` | Validate pipeline completion (checks changelog) |
| SubagentStop (orchestrate) | `post-orchestrate-check.sh` | Validate orchestration completion and delegation |
| TeammateIdle | `teammate-idle-check.sh` | Validate teammate delegation results and enforcement stages |
| TaskCompleted | `task-completed-check.sh` | Validate task outputs and check for HALLUCINATION markers |

A seventh script, `docs-commit.sh`, is a utility called by engines (not a hook) to commit docs changes.

### Design Philosophy

Several design choices are worth understanding as they inform how the system behaves:

**Engine-Skill Separation**: Skills are thin routers with no logic. All behavior lives in engine agents and instruction files. This makes the system composable — new commands only need a SKILL.md file and an instruction file.

**Read-Only Validation**: The validate and verbosity engines explicitly disallow Write and Edit tools. Validation operations cannot accidentally modify documentation.

**Anti-Compression Philosophy**: The verbosity system is philosophically opposed to LLM summarization tendencies. It maintains banned phrase lists and regex patterns, scope manifests that count every enumerable item, and zero-tolerance gap checking. "Length is not a concern. Completeness is the only concern."

**Citation-as-Evidence**: Every documentable code claim requires a citation block linking to the exact source file, symbol, and line range. Citations have a freshness model (Fresh, Stale, Drifted, Broken, Missing) with staleness detection. Three tiers (Full, Signature, Reference) scale verbatim code inclusion by function length.

**Pipeline-as-Quality-Gate**: The 8-stage pipeline is not optional. Verification and changelog stages never skip. The SubagentStop hook validates completion. Every doc modification goes through the full quality process.

**On-Demand Algorithm Loading**: Algorithm files are loaded during pipeline stages and then discarded, keeping engine context smaller until the algorithm is actually needed. This is distinct from modules which are always preloaded.

**`[NEEDS INVESTIGATION]` Over Guessing**: When engines cannot verify something from source code, they insert `[NEEDS INVESTIGATION]` markers instead of guessing. This makes gaps visible rather than hiding them behind plausible-sounding but potentially incorrect content.

### Three-Repo Git Model

fp-docs operates across three independent git repositories:

| Repo | Git Root | Purpose |
|------|----------|---------|
| Codebase | `wp-content/` | FP WordPress source code. Gitignores `docs/`. |
| Docs | `themes/foreign-policy-2017/docs/` | Nested inside codebase. Independently tracked. Branch-mirrors codebase. |
| Plugin | This repo (standalone) | Distributed via fp-tools marketplace. |

**Branch mirroring**: The docs repo's `master` branch is canonical for the codebase's `origin/master`. Feature branches in the docs repo match codebase feature branches by exact name.

**Path resolution**:
- Codebase root: `git rev-parse --show-toplevel` from working directory
- Docs root: `{codebase-root}/themes/foreign-policy-2017/docs/`
- Plugin root: `$CLAUDE_PLUGIN_ROOT` (injected by SessionStart hook)

---

## Development Guide

### Adding a New Command

1. Create the skill file at `skills/{name}/SKILL.md`:

   ```yaml
   ---
   name: fp-docs:{name}
   description: "What the command does"
   argument-hint: "expected arguments"
   context: fork
   agent: {engine-name}
   ---

   Operation: {operation-name}

   Read the instruction file at `framework/instructions/{engine}/{operation}.md`
   and follow it exactly.

   User request: $ARGUMENTS
   ```

2. Create the instruction file at `framework/instructions/{engine}/{operation}.md` with step-by-step procedures.

3. If the command needs a new engine, see "Adding a New Engine" below. Otherwise, use an existing engine that matches the command's domain.

### Adding a New Module

1. Create the module file at `modules/mod-{name}/SKILL.md`:

   ```yaml
   ---
   name: mod-{name}
   description: "Shared module providing {what it provides}"
   user-invocable: false
   disable-model-invocation: true
   ---

   # {Module Name} Module

   [Rule content here]
   ```

2. Add the module name to the `skills:` list in any engine agent frontmatter that needs it.

3. Follow the deduplication rules: each rule lives in exactly one module; FP-specific values go in `mod-project`; domain rules go in domain-specific modules; universal rules go in `mod-standards`.

### Adding a New Engine

1. Create the engine agent file at `agents/{name}.md`:

   ```yaml
   ---
   name: {name}
   description: |
     Description of what this engine does.
   tools:
     - Read
     - Grep
     - Glob
     - Bash
     # Add Write, Edit only if the engine modifies files
   skills:
     - mod-standards
     - mod-project
     # Add domain-specific modules as needed
   model: opus
   color: green
   maxTurns: 75
   ---

   # {Engine Name} Engine

   [System prompt: identity, parsing, instruction loading, execution, reporting]
   ```

2. For read-only engines, add `disallowedTools: [Write, Edit]` to the frontmatter.

3. Create instruction files at `framework/instructions/{engine}/` for each operation the engine supports.

### Configuration Files

| File | Scope | What to Change |
|------|-------|----------------|
| `framework/config/system-config.md` | System-wide | Feature flags, thresholds, tier boundaries, banned phrases |
| `framework/config/project-config.md` | FP-specific | Source-to-doc mappings, paths, repo URLs, feature enables |

Use per-command flags (`--no-citations`, `--no-sanity-check`, etc.) for one-off overrides instead of changing config files.

---

## Configuration Reference

### Feature Flags (system-config.md)

| Setting | Default | Description |
|---------|---------|-------------|
| `citations.enabled` | `true` | Whether citations are required in docs |
| `citations.full_body_max_lines` | `15` | Line threshold for Full citation tier |
| `citations.signature_max_lines` | `100` | Line threshold for Signature citation tier |
| `api_ref.enabled` | `true` | Whether API Reference sections are required |
| `api_ref.provenance_values` | `PHPDoc, Verified, Authored` | Valid source column values |
| `verbosity.enabled` | `true` | Master switch for verbosity enforcement |
| `verbosity.gap_tolerance` | `0` | Zero tolerance for gaps between source and docs |
| `sanity_check.default_enabled` | `true` | Whether sanity-check runs by default |
| `sanity_check.multi_agent_threshold_docs` | `5` | Doc count that triggers multi-agent sanity review |
| `chunk_delegation.max_docs_per_agent` | `8` | Max docs a single agent processes |
| `chunk_delegation.max_functions_per_agent` | `50` | Max functions a single agent processes |
| `orchestration.enabled` | `true` | Master switch for multi-agent orchestration |
| `orchestration.parallel_threshold_files` | `3` | Fan-out threshold for parallel Agent spawns |
| `orchestration.team_threshold_files` | `8` | Team creation threshold for batched teammates |
| `orchestration.max_teammates` | `5` | Max concurrent teammates |
| `orchestration.validation_retry_limit` | `1` | Max retries on LOW validation confidence |
| `orchestration.single_commit` | `true` | Aggregate changes into one git commit |

### Source-to-Documentation Mapping (project-config.md)

| Source Path | Documentation Target |
|-------------|---------------------|
| `functions.php` | `docs/01-architecture/bootstrap-sequence.md` |
| `inc/post-types/` | `docs/02-post-types/` |
| `inc/taxonomies/` | `docs/03-taxonomies/` |
| `inc/custom-fields/` | `docs/04-custom-fields/` |
| `components/` | `docs/05-components/` |
| `helpers/` | `docs/06-helpers/` |
| `inc/shortcodes/` | `docs/07-shortcodes/` |
| `inc/hooks/` | `docs/08-hooks/` |
| `inc/rest-api/` | `docs/09-api/rest-api/` |
| `inc/endpoints/` | `docs/09-api/custom-endpoints/` |
| `layouts/` | `docs/10-layouts/` |
| `features/` | `docs/11-features/` |
| `lib/autoloaded/` | `docs/12-integrations/` |
| `inc/cli/` | `docs/16-cli/` |
| `inc/admin-settings/` | `docs/17-admin/` |
| `assets/src/scripts/` | `docs/18-frontend-assets/js/` |
| `assets/src/styles/` | `docs/18-frontend-assets/css/` |
| `build/` | `docs/00-getting-started/build-system.md` |
| `inc/roles/` | `docs/20-exports-notifications/` |

---

## Troubleshooting

### Three-Repo Git Confusion

The most common pitfall. The docs directory is a separate git repo nested inside the codebase workspace.

- **Codebase git**: `git -C {wp-content-root}` -- tracks source code
- **Docs git**: `git -C {docs-root}` -- tracks documentation
- The codebase repo MUST gitignore `themes/foreign-policy-2017/docs/`
- Never run `git add` in the codebase repo expecting to capture doc changes
- Never run docs git commands expecting to see codebase files

If `/fp-docs:setup` reports issues with the gitignore, let it fix the configuration.

### Branch Sync Warnings on Session Start

If you see a branch mismatch warning when starting a session, it means your docs repo is on a different branch than your codebase. Run:

```
/fp-docs:sync
```

This creates or switches to the matching docs branch. If you recently switched codebase branches, always sync before doing any doc work.

### Permission Prompts

Modification commands (revise, add, auto-update, etc.) will prompt for Write/Edit/Bash permissions because the plugin only auto-allows Read, Grep, and Glob. This is by design -- validation commands run without prompts.

Approve these permissions when prompted. They apply to the current session only.

### Pipeline Changelog Warning

If you see a warning from `post-modify-check.sh` about a missing changelog update, it means the modify engine did not complete its pipeline. This can happen if the engine ran out of turns or encountered an error. Re-run the command.

### `[NEEDS INVESTIGATION]` Markers

Engines use `[NEEDS INVESTIGATION]` for anything they cannot verify from source code. This is intentional -- it is better than guessing. Search for these markers in generated docs and resolve them with:

```
/fp-docs:revise "clarify the [NEEDS INVESTIGATION] item in posts.md"
```

### Verbosity Failures

The verbosity engine has zero tolerance by default. If a source file has 12 functions, the docs must document all 12. Summarization language like "and more", "etc.", "various", or "several" is actively banned. If verbosity enforcement fails, the pipeline blocks until all items are documented.

### Parallel Operations Not Working

`/fp-docs:parallel` requires the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable to be set. Without it, the command falls back to sequential execution. It also falls back for scopes under 3 files.

### Plugin Not Updating to Latest Version

Claude Code's plugin update mechanism may not pull the latest version due to stale local cache or marketplace clones. The `plugin update` command sometimes does not fetch the latest commit from the remote repository before checking for updates.

**Manual git pull in the marketplace directory** (recommended workaround):

1. Navigate to the local marketplace clone in your terminal:

   ```bash
   cd ~/.claude/plugins/marketplaces/<marketplace-name>
   ```

   Replace `<marketplace-name>` with the marketplace that hosts the plugin. For fp-docs, this is the `fp-tools` marketplace directory (check `~/.claude/plugins/marketplaces/` for the exact folder name).

2. Pull the latest changes:

   ```bash
   git pull origin master
   ```

   Use `main` instead of `master` if the repository's default branch is `main`.

3. After pulling, run the plugin update command again or reinstall the plugin to apply the changes:

   ```
   /plugin update fp-docs
   ```

   If that still does not pick up the new version, reinstall:

   ```
   /plugin install fp-docs@fp-tools
   ```

This is a known Claude Code issue — the marketplace clone is a local git repository, and the update command does not always run `git fetch` before comparing versions.

### Docs Repo Not Found

If engines report they cannot find the docs repo, run:

```
/fp-docs:setup
```

Setup will detect whether the docs repo exists at `{codebase-root}/themes/foreign-policy-2017/docs/.git` and offer to clone it if missing.

### Key Files in the Docs Repo

| File | Purpose |
|------|---------|
| `docs/changelog.md` | Written to by every modification operation |
| `docs/needs-revision-tracker.md` | Queue consumed by `/fp-docs:auto-revise` |
| `docs/About.md` | Documentation hub and table of contents |
| `docs/claude-code-docs-system/PROJECT-INDEX.md` | Master codebase reference index |
| `docs/diffs/` | Accumulated branch diff reports (do not clean up) |
