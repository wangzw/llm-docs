---
name: docs-update
description: Update an existing raw/ document in-place with audit logging. Use when a spec, plan, or other raw document needs correction or evolution during implementation. Propagates changes to wiki and logs the mutation for traceability.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: <raw-file-path> [reason]
---

# docs-update: Update Raw Document

Update an existing `docs/raw/` document in-place. While raw documents are designed to be stable records, real-world workflows require controlled mutations — specs evolve during implementation, plans need adjustment when issues are discovered. This skill provides a disciplined way to make those changes while maintaining full traceability via git history and log.md.

## Input Parsing

Parse `$ARGUMENTS`:

1. **File path + reason** — first argument is a path to a file under `docs/raw/`, remaining text is the reason for the update
   - Example: `docs/raw/specs/2026-04-08-design.md "add error handling section"`
2. **File path only** — path to a `docs/raw/` file, no reason provided
   - Ask the user for the reason before proceeding
3. **No argument** — ask the user which raw file to update and why

Validate that the target file exists and is under `docs/raw/`. If not, stop with an error.

## Workflow

### Step 1: Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

### Step 2: Understand Context

1. Read the target raw file in full
2. Read `docs/README.md` to find wiki pages that reference this raw file
3. Read those wiki pages to understand the current synthesized state
4. Ask the user (if not already provided via argument or conversation context):
   - What changes are needed?
   - Why? (bug found, scope change, implementation feedback, etc.)

### Step 3: Update the Raw File

Apply the requested changes to the raw document. Preserve the existing frontmatter, but:

- Update `tags` if the changes affect the document's topics
- Do NOT change `ingested_at` — it records when the document was first created
- Do NOT change `source_type` — it records how the document originally entered the system

The `immutable: true` field may still be present in legacy files — leave it as-is (it is no longer enforced for updates via this skill).

### Step 4: Propagate to Wiki

Find all wiki pages whose `sources` frontmatter references the updated raw file.

For each affected wiki page:

1. Read the wiki page
2. Re-read ALL raw files listed in that wiki page's `sources` (not just the updated one)
3. Re-synthesize the wiki page to reflect the updated information
4. Set `last_updated` to today's date
5. Update `tags` if topics changed

If the update introduces a topic not covered by any existing wiki page, create a new one and add it to `docs/README.md`.

### Step 5: Update README.md

If any wiki page descriptions changed or new wiki pages were created, update `docs/README.md` accordingly.

### Step 6: Append to Log

Append an entry to `docs/log.md`:

```markdown
## [YYYY-MM-DD] update | <document title>
- raw: raw/<category>/<filename>.md
- reason: <why the update was needed>
- changes: <brief summary of what changed>
- wiki updated: wiki/<page>.md (updated) [, wiki/<page2>.md (updated)]
```

### Step 7: Present Summary

Show the user:

> **Update 完成**
> - 更新文档：`docs/raw/<category>/<filename>.md`
> - 原因：<reason>
> - 变更摘要：<brief changes>
> - Wiki 同步：`docs/wiki/<page>.md` (updated)

### Step 8: Commit

Stage all changed files under `docs/`. Create a git commit:

```
docs: update <document title> — <reason>
```
