# code-review-agents

A Claude Code plugin that orchestrates a 5-agent team to review pull requests. Each agent specializes in a different review domain, runs in its own tmux pane, and produces structured findings. An Oracle agent then validates, deduplicates, and scores the final review.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI installed
- `tmux` installed and running (agents spawn in separate panes)

## Installation

**Option 1: Test locally during development**
```bash
git clone https://github.com/galihcitta/code-review-agents.git
claude --plugin-dir ./code-review-agents
```

**Option 2: Install as a plugin**
```bash
# Add the marketplace (one-time)
/plugin marketplace add your-org/code-review-agents

# Install
/plugin install review-mr
```

## Quick Start

Run a review:
```
/review-mr:review /path/to/your-repo feature/my-branch
```

Zero configuration required — the plugin ships with default scoring criteria and reference standards for Node.js and Go.

## Usage

**Single repo:**
```
/review-mr:review /path/to/my-service feature/user-auth
```

**Single repo with spec:**
```
/review-mr:review /path/to/my-service feature/user-auth --spec user-auth
```

**Multi-repo (cross-repo review):**
```
/review-mr:review /path/to/api:feature/payments /path/to/gateway:feature/payments --spec payments
```

## Plugin Structure

```
code-review-agents/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest (name, version, description)
├── skills/
│   └── review/
│       ├── SKILL.md             # Orchestrator skill (invoked as /review-mr:review)
│       └── templates/
│           ├── security-memory.md
│           ├── architecture-lang.md
│           ├── database-observability.md
│           ├── testing-deployment.md
│           └── oracle.md
├── defaults/
│   ├── scoring.md               # 10-category scoring criteria + output template
│   └── references/
│       ├── nodejs.md            # Node.js standards checklist
│       └── golang.md            # Go standards checklist
├── config.example.yaml
├── README.md
└── LICENSE
```

## Configuration

Config resolution follows a layering model — each layer shallow-merges over the previous; arrays replace.

1. **Plugin defaults** — built-in values (works out of the box)
2. **Plugin config** — `config.yaml` in the plugin root
3. **Project config** — `.claude/pr-review.yaml` in your repo
4. **CLI flags** — highest priority

Copy `config.example.yaml` to `config.yaml` to customize:

```yaml
base_branches: [develop, main, master]
remote: origin
language: auto
references_dir: null
scoring_file: null
output_dir: ~/.claude/pr-review
reviewer_model: sonnet
oracle_model: opus
specs_dir: null
reviewer_timeout: 600
oracle_timeout: 900
```

### Project-Level Overrides

Add `.claude/pr-review.yaml` to any repo to override plugin settings for that project:

```yaml
base_branches: [main]
language: golang
```

## Customizing Standards

The plugin supports two reference layouts:

### Layout A: Single File (default)

One markdown file per language at `defaults/references/{language}.md`. All four reviewer agents receive the same file. Best for getting started or when your standards are concise.

### Layout B: Per-Category Files

A directory per language with individual files per domain:

```
references/nodejs/
├── security-standards.md
├── memory-safety-standards.md
├── http-standards.md
├── nodejs-standards.md
├── idempotency-patterns.md
├── database-standards.md
├── monitoring-standards.md
├── event-queue-standards.md
├── testing-standards.md
├── deployment-standards.md
├── documentation-standards.md
├── configuration-standards.md
└── cicd-standards.md
```

Each reviewer agent receives only the files relevant to its domain. Set `references_dir` in your config to point to your custom references directory.

## Architecture

```
User runs /review-mr:review
       │
       ▼
   Lead Agent (you)
       │
       ├── Phase 1: Parse args, load config
       ├── Phase 2: Gather context (diffs, commits, language detection)
       ├── Phase 3: Map reference files to teammates
       ├── Phase 4: Create team + tasks
       ├── Phase 5: Spawn teammates
       │     ├── T1: Security & Memory Safety    (reviewer model)
       │     ├── T2: Architecture & Language      (reviewer model)
       │     ├── T3: Database & Observability     (reviewer model)
       │     └── T4: Testing & Deployment         (reviewer model)
       │
       │   [T1-T4 write findings to JSON files]
       │
       ├── Phase 6: Wait for findings, spawn Oracle
       │     └── T5: Oracle                       (oracle model)
       │           ├── Validates each finding against actual code
       │           ├── Deduplicates cross-agent findings
       │           ├── Optionally challenges reviewers (1 round, max 3)
       │           ├── Scores 10 categories (1-5)
       │           └── Writes final review to markdown file
       │
       └── Present review, clean up team
```

### Agent Team

| Agent | Role | Scores |
|-------|------|--------|
| T1: security-memory | Security vulnerabilities, resource/memory leaks | security_standards, memory_safety |
| T2: architecture-lang | Architecture, HTTP standards, language patterns | architecture_design, http_standards, language_standards |
| T3: database-observability | Queries, monitoring, event/queue patterns | database_standards, monitoring_standards |
| T4: testing-deployment | Tests, deployment, docs, config, CI/CD | testing_standards, documentation_standards, deployment_standards |
| T5: oracle | Validate, deduplicate, score, produce final review | All 10 categories (final) |

## Scoring

Reviews produce scores across 10 categories on a 1-5 scale:

| Score | Label | Decision |
|-------|-------|----------|
| 4.5 - 5.0 | EXCELLENT | Approve immediately |
| 3.5 - 4.4 | GOOD | Approve with minor suggestions |
| 2.5 - 3.4 | ACCEPTABLE | Request changes before merge |
| 1.5 - 2.4 | NEEDS_WORK | Major improvements required |
| 1.0 - 1.4 | REJECT | Reject MR |

Any single category score of 1 triggers an automatic REJECT regardless of average.

The full scoring criteria and output template are in `defaults/scoring.md`. You can customize by pointing `scoring_file` to your own file.

## Sample Output

The final review is saved as a markdown file and printed to terminal. Here's what it looks like:

```markdown
# MR Review Summary

**MR Title:** feature/user-auth
**Author:** Jane Doe
**Reviewed by:** Claude Code Agent Team
**Date:** 2026-03-14

## Approval Status
Approved with comments (GOOD)

## Key Findings

### Finding 1: Unbounded query in user listing
**File:** `app/repository/user_repository.js` (Lines 45-48)
**Severity:** critical
**Issue:** findAll() called without limit — will load entire table into memory.
**Suggested fix:**
  const users = await User.findAll({ where: filters, limit: 100, offset });

### Finding 2: Missing Idempotency-Key on POST
**File:** `app/controllers/v1/auth.js` (Lines 12-15)
**Severity:** warning
...

## Category Scores

| Category | Score | Label |
|----------|-------|-------|
| Architecture & Design | 4 | GOOD |
| HTTP Standards | 3 | ACCEPTABLE |
| Language Standards | 4 | GOOD |
| ...
| **Average** | 3.8 | **GOOD** |
```

## Adding Language Support

The plugin ships with Node.js and Go standards. To add your own language (Python, Rust, Java, etc.):

1. Create a reference file at `defaults/references/{language}.md` (Layout A) or a directory at `defaults/references/{language}/` (Layout B)
2. Follow the same section structure as `nodejs.md` or `golang.md` — each section maps to a scoring category
3. Mark critical items with **REJECT** and non-blocking items with **WARNING**
4. Set `language: {language}` in your config (auto-detection only supports Node.js and Go)

The agents will use your reference file the same way they use the built-in ones.

## Known Limitations

- Built-in reference standards cover **Node.js and Go** only. Other languages work but rely on the agents' built-in knowledge (no custom checklist). Add your own via `references_dir`.
- Task data does not survive TeamDelete — the plugin writes all output to files before cleanup, but if the process is killed mid-review, check `{output_dir}/findings/` for partial results.
- The `--spec` flag requires `specs_dir` to be configured. Without it, requirements alignment is scored as -1 (excluded from average).
- Reviews run 5 concurrent agents — expect ~3-5 minutes for a typical PR depending on diff size and model speed.

## Contributing

Contributions are welcome. Some areas that would be particularly useful:

- Reference standards for additional languages (Python, Rust, Java, TypeScript-specific, etc.)
- Improvements to scoring criteria and decision matrix
- Better auto-detection for more languages and frameworks

Please open an issue first to discuss significant changes.

## Author

**Galih Citta** — [GitHub](https://github.com/galihcitta)

## License

MIT — see [LICENSE](LICENSE).
