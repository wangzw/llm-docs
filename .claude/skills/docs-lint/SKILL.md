---
name: docs-lint
description: Check documentation consistency and audit docs against code. Validates wiki links, source references, detects contradictions between wiki pages, and performs deep comparison of documentation against actual code (API endpoints, data models, configs, function signatures). Use when you suspect docs are out of date or before major releases.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: ""
---

# docs-lint: Documentation Consistency Check & Code Audit

Check the documentation system for internal consistency and audit documentation against the actual codebase.

## Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

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
- checked: N wiki pages, M source files
- issues: N (summary)
- fixed: description of fixes applied
```

5. Commit:

```
docs: lint fix — update wiki to match codebase
```

If the user chooses to skip, no files are modified and no log entry is written.
