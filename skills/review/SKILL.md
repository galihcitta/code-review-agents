---
name: review
description: Orchestrate a 5-agent team to review PRs
version: "1.0.0"
allowed-tools: [Read, Write, Bash, Glob, Grep, TeamCreate, TaskCreate, TaskUpdate, TaskGet, TaskList, SendMessage, TeamDelete]
argument-hint: "<repo-path> <branch> [--spec <name>]"
---

You are the lead of a PR review agent team. You will create a team of 5 teammates, each running in its own tmux pane so the user can watch all reviewers work simultaneously.

**Model strategy:** Use the configured reviewer model for T1-T4 (focused review tasks with narrow context). Use the configured oracle model for T5 Oracle (synthesis, validation, scoring). You (the lead) run on whatever model the user started with.

## Phase 1: Parse Arguments and Load Config

### 1a. Parse Arguments

Parse: $ARGUMENTS

**Single-repo format:** `<repo-path> <branch-name> [--spec <epic-name>]`
**Multi-repo format:** `<repo1>:<branch1> [<repo2>:<branch2> ...] [--spec <epic-name>]`

Detection: if any argument contains `:` (and is not `--spec`), treat as multi-repo format.

Extract:
- REPOS[] — array of {path, branch} objects
  - Single-repo: one entry from positional args
  - Multi-repo: one entry per `path:branch` pair
- SPEC_NAME (optional, after --spec)

If no repos detected, ask the user.

Examples:
```
# Single repo
/code-review-agents:review /path/to/my-service feature/user-auth --spec user-auth

# Multi repo
/code-review-agents:review /path/to/api:feature/payments /path/to/gateway:feature/payments --spec payments
```

### 1b. Load Configuration

Derive PLUGIN_ROOT: resolve the absolute path two levels up from this SKILL.md file's location (i.e., the root of the code-review-agents plugin directory).

**Config resolution order** (each layer shallow-merges; arrays replace):
1. **DEFAULTS** — built-in values:
   ```
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
2. **PLUGIN_CONFIG** — read `{PLUGIN_ROOT}/config.yaml` if it exists
3. **PROJECT_CONFIG** — read `{REPOS[0].path}/.claude/pr-review.yaml` if it exists (first repo)
4. **CLI_FLAGS** — any explicit flags from $ARGUMENTS override everything

Shallow-merge each layer: `CONFIG = DEFAULTS ← PLUGIN_CONFIG ← PROJECT_CONFIG ← CLI_FLAGS`

## Phase 2: Context Gathering

**For each repo in REPOS[]:**

1. **Detect base branch:**
   Iterate `CONFIG.base_branches` array. For each candidate, check:
   ```
   cd {repo.path} && git rev-parse --verify {CONFIG.remote}/{candidate} 2>/dev/null
   ```
   Use the first one that exists. Store as repo.base_branch.

   Use `{CONFIG.remote}/{repo.base_branch}` for all diff commands.

2. **Get diff stat:**
   ```
   cd {repo.path} && git diff {CONFIG.remote}/{repo.base_branch}...{repo.branch} --stat
   ```
   If diff is empty for ALL repos, report "No changes detected" and stop.
   If diff is empty for one repo, warn and exclude it from the review.

3. **Get commit log:**
   ```
   cd {repo.path} && git log {CONFIG.remote}/{repo.base_branch}...{repo.branch} --oneline
   ```

4. **Extract repo name** from path (last directory component, e.g., `/path/to/my-service` → `my-service`)

5. **Read REPO_CONTEXT:**
   Read `{repo.path}/CLAUDE.md` if it exists → store as repo.context
   If not found: repo.context = "No CLAUDE.md found."

**Aggregate across all repos:**

6. **Detect language** from file extensions across ALL diff stats:
   - If `CONFIG.language` is not "auto", use that value directly
   - Otherwise detect from file extensions:
     - `.js`, `.ts`, `.mjs`, `.cjs`, `package.json` → `nodejs`
     - `.go`, `go.mod` → `golang`
     - `.tsx`, `.jsx`, `.vue` → `nodejs` (frontend)
     - Both detected → LANG = "mixed", LANGS = ["nodejs", "golang"]
     - None matched → LANG = "other" (no dedicated refs, general checklist only)

7. **Calculate total diff size** — sum of changed lines across all repos for Large Diff Guidance.

8. **Requirements summary:**
   If --spec provided:
   - If `CONFIG.specs_dir` is null, warn: "specs_dir not configured — ignoring --spec flag" and set REQUIREMENTS_SUMMARY = "No spec linked. Set requirements_alignment to -1."
   - Otherwise:
     - Try `{CONFIG.specs_dir}/{SPEC_NAME}/INDEX.md` first
     - Fall back to `{CONFIG.specs_dir}/{SPEC_NAME}.md`
     - If found, read and distill into a REQUIREMENTS_SUMMARY (key requirements, acceptance criteria)
     - If not found, warn and set REQUIREMENTS_SUMMARY = "Spec not found at configured path. Set requirements_alignment to -1."
   If no --spec: REQUIREMENTS_SUMMARY = "No spec linked. Set requirements_alignment to -1."

9. **Build REPOS_CONTEXT** — a text block listing all repos for injection into templates:
   ```
   ### Repo: {repo.name}
   - Path: {repo.path}
   - Branch: {repo.branch}
   - Base: {CONFIG.remote}/{repo.base_branch}
   - Diff command: cd {repo.path} && git diff {CONFIG.remote}/{repo.base_branch}...{repo.branch}
   - Diff stat command: cd {repo.path} && git diff {CONFIG.remote}/{repo.base_branch}...{repo.branch} --stat
   ```
   Repeat for each repo.

10. **Build REPO_CONTEXTS** — a text block with all repo CLAUDE.md contents:
    ```
    ### {repo.name}
    {repo.context}
    ```
    Repeat for each repo.

## Phase 3: Reference File Mapping

Resolve REFERENCES_DIR:
- If `CONFIG.references_dir` is set → use that path
- Otherwise → `{PLUGIN_ROOT}/defaults/references`

Map each teammate to its reference file paths based on detected LANG.

**Layout detection:**
- If `{REFERENCES_DIR}/{LANG}` is a directory → **Layout B** (per-file references)
- If `{REFERENCES_DIR}/{LANG}.md` is a file → **Layout A** (single-file reference)
- If LANG = "other" → no references, teammates use built-in checklist
- If LANG = "mixed" → load both languages' refs (try both layouts independently)

### Layout A: Single-file reference

All 4 reviewers get the same file: `{REFERENCES_DIR}/{LANG}.md`

Since templates have varying numbers of `{REF_N}` placeholders (T1 has 2, T2/T3 have 3, T4 has 5), inject the single file path into `{REF_1}` and **remove** the remaining `- {REF_N}` lines from the template before injection. Do not leave unresolved placeholders.

### Layout A with mixed language (LANG = "mixed")

Load both `{REFERENCES_DIR}/nodejs.md` and `{REFERENCES_DIR}/golang.md`. Inject both into each template: `{REF_1}` = nodejs.md, `{REF_2}` = golang.md. Remove any remaining `- {REF_N}` lines beyond `{REF_2}`.

### LANG = "other" (no references)

Strip the entire "## Reference Standards" section (the header and all `- {REF_N}` lines) from each template before injection. Teammates will review using their built-in checklist only.

### Layout B: Per-file references

Map files to teammates by naming convention:

| Teammate | Files |
|----------|-------|
| T1: security-memory | `{LANG}/security-standards.md`, `{LANG}/memory-safety-standards.md` |
| T2: architecture-lang | `{LANG}/http-standards.md`, `{LANG}/{LANG_FILE}-standards.md`, `{LANG}/idempotency-patterns.md` |
| T3: database-observability | `{LANG}/database-standards.md`, `{LANG}/monitoring-standards.md`, `{LANG}/event-queue-standards.md` |
| T4: testing-deployment | `{LANG}/testing-standards.md`, `{LANG}/deployment-standards.md`, `{LANG}/documentation-standards.md`, `{LANG}/configuration-standards.md`, `{LANG}/cicd-standards.md` |

For `{LANG_FILE}-standards.md`: use `nodejs-standards.md` for Node.js or `golang-standards.md` for Go.

**Mixed language (LANG = "mixed"):** Load references from **both** languages for each teammate. Each teammate gets all files listed above for both languages. Set `{LANG}` to `"mixed"` in the teammate template so it knows to apply both language checklists.

## Phase 4: Create Team and Tasks

1. **Generate file paths for durable output.** All agent output goes to files — the task system is used only for status signaling.
   - `FINDINGS_PATH_1`: `{CONFIG.output_dir}/findings/t1-security-memory.json`
   - `FINDINGS_PATH_2`: `{CONFIG.output_dir}/findings/t2-architecture-lang.json`
   - `FINDINGS_PATH_3`: `{CONFIG.output_dir}/findings/t3-database-observability.json`
   - `FINDINGS_PATH_4`: `{CONFIG.output_dir}/findings/t4-testing-deployment.json`
   - `OUTPUT_PATH`: `{CONFIG.output_dir}/review-{BRANCH_SLUG}.md` where BRANCH_SLUG is the branch name with `/` replaced by `-` (e.g., `feature/user-auth` → `review-feature-user-auth.md`). For multi-repo, use the first repo's branch.

   Delete all of these files if they exist from a previous run (prevents stale reads).
   Create the `{CONFIG.output_dir}/findings/` directory if it doesn't exist.

2. **Create the agent team** using TeamCreate with the name "pr-review".

3. **Create 5 shared tasks** using TaskCreate:
   - "Security & Memory Safety Findings" — status: pending
   - "Architecture & Language Standards Findings" — status: pending
   - "Database & Observability Findings" — status: pending
   - "Testing & Deployment Findings" — status: pending
   - "Final Scored Review" — status: pending (depends on tasks 1-4)

   Store all 5 task IDs.

## Phase 5: Spawn Teammates

Resolve template and scoring paths:
- `TEMPLATE_DIR = {PLUGIN_ROOT}/skills/review/templates`
- `SCORING_PATH = CONFIG.scoring_file or {PLUGIN_ROOT}/defaults/scoring.md`

For each reviewer teammate (T1-T4):
1. Read its agent template from `{TEMPLATE_DIR}/{name}.md`
2. In the template content, replace these placeholders with runtime values:
   - `{REPOS_CONTEXT}` → the built REPOS_CONTEXT block from Phase 2
   - `{LANG}` → detected language
   - `{REF_1}`, `{REF_2}`, etc. → full paths to reference files from the mapping
   - `{TASK_ID}` → the corresponding task ID
   - `{FINDINGS_PATH}` → the corresponding FINDINGS_PATH for this reviewer
3. Spawn the teammate into the team with the injected instructions. **Use `CONFIG.reviewer_model`** for all reviewers.

**Spawn all 4 reviewer teammates.** Each appears in its own tmux pane.

For teammate 5 (Oracle), after T1-T4 complete their tasks:
1. Read `{TEMPLATE_DIR}/oracle.md`
2. Replace placeholders:
   - `{REPOS_CONTEXT}` → the built REPOS_CONTEXT block
   - `{LANG}` → detected language
   - `{TASK_ID_1}` through `{TASK_ID_4}` → reviewer task IDs
   - `{TASK_ID_5}` → Oracle's own task ID
   - `{FINDINGS_PATH_1}` through `{FINDINGS_PATH_4}` → reviewer findings file paths from Phase 4
   - `{REQUIREMENTS_SUMMARY}` → requirements summary text
   - `{REPO_CONTEXTS}` → all repo CLAUDE.md contents block
   - `{OUTPUT_PATH}` → the OUTPUT_PATH generated in Phase 4
   - `{SCORING_PATH}` → resolved scoring file path
3. Spawn the Oracle into the team. **Use `CONFIG.oracle_model`** for the Oracle — it handles validation, dedup, false positive filtering, cross-repo analysis, and final scoring.

## Phase 6: Wait for Reviewers, Spawn Oracle, Read Output File

After spawning T1-T4, wait for all 4 reviewer findings files to appear. Poll by checking if FINDINGS_PATH_1 through FINDINGS_PATH_4 exist (attempt to Read each file every 15 seconds). A reviewer is done when its findings file exists on disk. You can also check task status via TaskGet as a secondary signal.

If any reviewer's findings file hasn't appeared after `CONFIG.reviewer_timeout` seconds, treat it as timed out and proceed with whatever findings files are available.

Once all 4 findings files exist (or timeout):

**Step 1: Spawn Oracle.**
Spawn the Oracle (T5) as described in Phase 5.

**Step 2: Wait for the output file.**
The Oracle writes its final review to OUTPUT_PATH. Poll for this file using the Read tool every 15 seconds. The file exists when the Oracle is done. Do NOT rely on TaskGet for the review content — task data may not survive cleanup.

As a secondary signal, you can also check the Oracle's task status via TaskGet. But the file is the authoritative output.

Timeout: if the file has not appeared after `CONFIG.oracle_timeout` seconds, check the Oracle's task status. If the task is "completed" but no file exists, fall back to reading inbox messages.

**Step 3: Present the review.**
Use `Bash(cat OUTPUT_PATH)` to print the full review to the terminal so it appears in the recording/scrollback. Then tell the user where the file is saved.

**Step 4: Clean up.**
Only after you have presented the output, clean up the team using TeamDelete. The output file persists on disk regardless of team cleanup.

**Failure handling:**
- If any reviewer teammate (T1-T4) fails or times out:
  - Note which review area is missing
  - Spawn the Oracle with whatever findings are available
  - Oracle sets missing category scores to -1
  - Include warning: "WARNING: {teammate name} did not complete. {categories} scores are provisional."
- If the Oracle fails or times out without producing the output file:
  - Check inbox messages for any partial Oracle output
  - If no Oracle output at all, manually aggregate available reviewer findings ordered by severity
  - Present without dedup/validation
  - Note: "Scoring was not completed — findings are unvalidated."

## Large Diff Guidance

If the total diff size across all repos shows >2000 lines changed, **append** the following text to the end of each reviewer template's content (after the Follow-Up section) before spawning:

```
## Large Diff Priority

LARGE DIFF: This PR has >2000 lines changed. Prioritize reviewing (1) new files, (2) function signatures, (3) database queries/migrations, (4) route definitions, (5) error handling blocks. Skip pure whitespace/formatting changes and auto-generated files.
```

## Teammate Communication

Teammates can message each other for cross-cutting concerns using SendMessage. For example:
- T1 (Security) finds raw SQL → messages T3 (Database): "Found raw SQL at my-service:file.js:45"
- T3 (Database) finds unbounded query → messages T1 (Memory Safety): "findAll without limit at gateway:model.js:30"

The `cross_reference` field in findings JSON enables the Oracle to deduplicate these cross-reported issues.
