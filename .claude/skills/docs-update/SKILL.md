---
name: docs-update
description: Update an existing raw/ document in-place with audit logging. Use when a spec, plan, or other raw document needs correction or evolution during implementation. Propagates changes to wiki and logs the mutation for traceability.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: [raw-file-path] [reason | --from-commits [range]]
---

# docs-update: Update Raw Document

Update an existing `docs/raw/` document in-place. While raw documents are designed to be stable records, real-world workflows require controlled mutations — specs evolve during implementation, plans need adjustment when issues are discovered. This skill provides a disciplined way to make those changes while maintaining full traceability via git history and log.md.

## Input Parsing

Parse `$ARGUMENTS`:

1. **No argument (default)** — auto-detect mode. Analyze recent git commits to find which raw documents need updating. This is the most common usage.
   - Example: `/docs-update`
2. **`--from-commits` [range] only** — same as above but with an explicit commit range
   - Example: `/docs-update --from-commits HEAD~10..HEAD`
3. **File path + `--from-commits` [range]** — update a specific raw file based on git commit history
   - Example: `/docs-update docs/raw/specs/2026-04-08-design.md --from-commits`
   - Example: `/docs-update docs/raw/specs/2026-04-08-design.md --from-commits abc123..def456`
4. **File path + reason** — update a specific raw file with a manually described reason
   - Example: `/docs-update docs/raw/specs/2026-04-08-design.md "add error handling section"`
5. **File path only** — path to a `docs/raw/` file, no reason provided
   - Ask the user for the reason before proceeding

If a file path is provided, validate that it exists and is under `docs/raw/`. If not, stop with an error.

## Workflow

### Step 1: Pre-flight Check

Read `docs/schema.md`. If it does not exist, tell the user to run `/docs-init` first and stop.

### Step 2: Gather Commit History

This step applies when no file path is provided, or when `--from-commits` is used (with or without a file path). If a file path + manual reason was provided, skip to **Step 3**.

#### 2.1 Determine Commit Range

- If an explicit range was provided (e.g. `HEAD~10..HEAD`), use it
- Otherwise, find the last boundary from `docs/log.md`: scan for the most recent `ingest` or `update` entry, use that date as `--since`. If no log entry exists, use the earliest `ingested_at` date across all raw files

#### 2.2 Analyze Commits

1. Run `git log --oneline <range>` (or `git log --since=<date> --oneline`). Exclude commits that only touch files under `docs/`
2. For relevant commits, run `git show --stat <hash>` and read full commit messages

#### 2.3 Match Commits to Raw Documents

1. Read all raw files under `docs/raw/` and build a topic map (file → title, tags, key concepts)
2. For each non-docs commit, determine which raw documents it relates to by matching:
   - Changed file paths against topics described in raw documents
   - Commit message keywords against raw document titles and tags
   - Module/component names against concepts in raw documents
3. Group commits by matched raw document

#### 2.4 Identify Documents Needing Update

For each raw document with matched commits, assess whether the commits introduce changes that make the document outdated or incomplete. Produce a list of `(raw file, reason, changes needed)`.

If no file path was provided, present the full list to the user:

> "根据提交记录分析，以下文档需要更新：
> 1. `docs/raw/specs/2026-04-08-design.md` — <reason summary>
> 2. `docs/raw/plans/2026-04-08-impl.md` — <reason summary>
> 选择要更新的文档编号（全部/逐项确认/跳过）："

If a specific file path was provided, only show the analysis for that file:

> "根据提交记录分析，以下内容需要更新：
> - <change 1>
> - <change 2>
> 是否继续？"

Proceed to Step 3 with user-confirmed documents and their synthesized changes. For each confirmed document, read it in full and read its associated wiki pages before updating.

### Step 2-alt: Manual Mode

If a file path was provided without `--from-commits` and without a reason:

1. Read the target raw file in full
2. Read `docs/README.md` to find wiki pages that reference this raw file
3. Read those wiki pages to understand the current synthesized state
4. Ask the user:
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
- source: <manual | commits (range)>
- reason: <why the update was needed>
- changes: <brief summary of what changed>
- wiki updated: wiki/<page>.md (updated) [, wiki/<page2>.md (updated)]
```

When `--from-commits` was used, `source` should record the commit range, e.g. `commits (abc123..def456)` or `commits (since 2026-04-01)`.

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
