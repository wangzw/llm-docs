---
name: docs-lint
description: Check documentation consistency and audit docs against code. Validates wiki links, source references, detects contradictions between wiki pages, and performs deep comparison of documentation against actual code (API endpoints, data models, configs, function signatures). Use when you suspect docs are out of date or before major releases.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: [--full]
---

# docs-lint: Documentation Consistency Check & Code Audit

Check the documentation system for internal consistency and audit documentation against the actual codebase.

## Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

## Guiding Principle: Wiki Organization is Lazy, Not Upfront

Karpathy's LLM Wiki Pattern derives structure from **cross-references** (`[[wiki-links]]`), not from directory hierarchy. This has direct consequences for what this skill should and should not flag:

- **Flat `docs/wiki/` is the default healthy state**, regardless of page count. Do NOT flag a flat wiki as an organizational issue just because `docs/raw/` has subdirectories. `raw/` is classified by document *type* (spec/plan/adr/...) — a software-engineering axis; `wiki/` is organized by *topic*, and topics emerge from the corpus rather than being imposed upfront.
- **Subdirectories in `wiki/` are a lazy decision** — introduced only when the flat structure genuinely impairs navigation AND clear topical clusters have emerged. Never mirror `raw/`'s type-based categorization in `wiki/`.
- **Mimicry is the anti-pattern**, not flatness. A wiki with 50 flat pages cross-linked via `[[wiki-links]]` is healthier than a wiki with 10 pages split across 5 thinly-populated subdirectories.

This principle governs check 1e below and should inform any auto-fix suggestions about wiki organization.

## Scope Detection

Determine the lint scope based on the current branch:

### Branch Mode (current branch != main)

When on a feature/development branch, lint only the documentation changes introduced on this branch:

1. Run `git diff --name-only main...HEAD` to find files changed on the branch
2. Filter to files under `docs/` — these are the documentation changes to lint
3. Also include any code files changed on the branch — these are the code changes to cross-reference against documentation
4. The lint scope is narrowed to:
   - **Phase 1**: Only check consistency of changed/added wiki pages and their links (not the entire docs/ tree)
   - **Phase 2**: Only audit changed wiki pages against changed code. Also check: does the changed code introduce new behaviors that should be documented but aren't?

### Main Mode (current branch == main)

When on main, lint incrementally since the last lint:

1. Read `docs/log.md` and find the most recent `lint` entry. Extract its `commit` field (the commit ID recorded at last lint).
2. If a previous lint commit exists:
   - Run `git diff --name-only <last-lint-commit>..HEAD` to find files changed since last lint
   - Narrow scope to those changes (same logic as Branch Mode)
3. If no previous lint entry exists, perform a **full lint** of all docs and code (no scope narrowing)

### Full Mode Override

If `$ARGUMENTS` contains `--full`, always perform a full lint regardless of branch or prior lint state.

## Lint Workflow

### Phase 1: Document Internal Consistency

#### 1a. Link Validation

1. Read `docs/README.md`. Extract all `[[wiki-links]]`.
2. For each link, verify the target file exists using Glob.
3. Read all wiki pages in `docs/wiki/`. Extract all `[[wiki-links]]` and `sources` references.
4. For each link, verify the target exists.
5. Report broken links as 🔴 errors.

#### 1b. Orphan Detection

1. Collect all wiki pages from `docs/wiki/`.
2. Collect all pages referenced from `docs/README.md`.
3. Pages that exist in wiki/ but are not referenced from README.md are orphans.
4. Report orphans as 🟡 warnings.

#### 1c. Unprocessed Raw Files

1. Collect all files in `docs/raw/` (recursively).
2. Read `docs/log.md` and extract all raw file paths mentioned in ingest entries.
3. Files in raw/ with no corresponding log entry are unprocessed.
4. Report unprocessed files as 🟡 warnings.

#### 1d. Wiki Contradiction Detection

1. Read all wiki pages.
2. Identify pages that cover overlapping topics (based on tags and content).
3. For overlapping pages, check whether they make contradictory claims about the same concept.
4. Report contradictions as 🔴 errors with specific quotes from both pages.

#### 1e. Wiki Organization Check

Applies the guiding principle above. Flag two anti-patterns:

1. **Premature categorization**: `docs/wiki/` contains subdirectories where any subdirectory has fewer than 3 pages. The hierarchy was imposed upfront rather than emerging from a growing corpus.
   - Report as 🟡 warning: "wiki/ subdirectory `<dir>` contains only N page(s) — flat wiki with `[[wiki-links]]` is preferred until topical clusters clearly emerge"
   - Auto-fix suggestion: flatten (move pages to wiki/ root, remove empty dirs, update links)

2. **Overgrown flat wiki without cross-references**: `docs/wiki/` has many pages (roughly 20+) at the top level AND the pages lack `[[wiki-links]]` between each other. The knowledge is flat but also disconnected — no compounding synthesis.
   - Report as 🟡 warning: "wiki/ has N pages with sparse cross-references — consider adding `[[wiki-links]]` between related pages, or grouping into topical subdirectories if clear clusters exist"
   - Note: a flat wiki with rich cross-references is NOT a warning, regardless of page count.

A flat `docs/wiki/` that does not trigger either condition above is healthy — do not suggest reorganization.

### Phase 2: Documentation-Code Deep Audit

This phase reads the actual codebase and compares it against documentation claims.

#### 2a. Discover Code Structure

Use Glob and Grep to identify:

1. **API endpoints**: Search for route definitions, handler registrations, controller annotations
2. **Data models**: Search for schema definitions, ORM models, struct/class definitions, migration files
3. **Configuration**: Search for config files, environment variable references, feature flags
4. **Key functions/interfaces**: Search for exported functions, public interfaces, service boundaries

Build a mental model of what the code actually does.

#### 2b. Cross-reference with Wiki

For each wiki page:

1. Read the page content
2. Extract claims about code behavior (e.g., "the auth service uses JWT tokens", "the payment API has 3 endpoints", "retry policy is exponential backoff with max 5 retries")
3. Verify each claim against the code:
   - **Documented but not in code**: The doc describes something that doesn't exist in the codebase → 🔴 error
   - **In code but not documented**: A significant code structure (API endpoint, data model, service) has no documentation → 🟡 warning
   - **Doc contradicts code**: The doc describes behavior differently from what the code implements → 🔴 error
   - **Consistent**: The doc accurately reflects the code → 🟢 pass

### Phase 3: Report

Present a structured lint report:

```
# Documentation Lint Report

## Summary
- 🔴 Errors: N
- 🟡 Warnings: N
- 🟢 Passed: N checks

## 🔴 Errors

### [E001] Broken link in README.md
- Location: docs/README.md line 15
- Link: [[wiki/nonexistent-page]]
- Action: Remove link or create the wiki page

### [E002] Wiki contradicts code
- Wiki: docs/wiki/auth.md claims "uses session-based auth"
- Code: src/auth/handler.go implements JWT-based auth
- Action: Update wiki page to reflect JWT implementation

## 🟡 Warnings

### [W001] Orphan wiki page
- Page: docs/wiki/old-design.md
- Not referenced from README.md
- Action: Add to README.md or archive if obsolete

### [W002] Undocumented API endpoint
- Endpoint: POST /api/v2/webhooks (src/api/webhooks.go:45)
- No wiki page covers this endpoint
- Action: Document in relevant wiki page

## 🟢 Passed Checks
- All source references in wiki pages point to existing raw files
- Data model documentation matches current schema
- ...
```

### Phase 4: Auto-fix (Optional)

After presenting the report, ask the user:

> "是否自动修复？我可以：
> - 更新 wiki 页面以匹配代码实际行为
> - 移除断裂的链接
> - 将孤立页面添加到 README.md
>
> 选择：全部修复 / 逐项确认 / 跳过"

If the user chooses to fix:
1. Apply fixes to wiki pages (update content to match code, fix broken links, add orphans to README.md)
2. Do NOT modify raw/ files (they are immutable)
3. Update `last_updated` in modified wiki pages
4. Append to `docs/log.md`:

```markdown
## [YYYY-MM-DD] lint
- scope: <branch (branch-name vs main) | incremental (last-commit..current-commit) | full>
- commit: <current HEAD commit ID (short hash)>
- checked: N wiki pages, M source files
- issues: N (summary)
- fixed: description of fixes applied
```

The `commit` field is always written — it serves as the checkpoint for the next incremental lint on main.

5. Commit:

```
docs: lint fix — update wiki to match codebase
```

If the user chooses to skip, no files are modified and no log entry is written.
