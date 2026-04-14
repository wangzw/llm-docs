---
name: docs-update
description: Update an existing raw/ document (or a batch under a raw/ directory) in-place with audit logging. Use when a spec, plan, or other raw document needs correction or evolution during implementation. Propagates changes to wiki and logs the mutation for traceability.
allowed-tools: Read Write Edit Bash Glob Grep
user-invocable: true
argument-hint: [raw-file-path | raw-dir-path] [reason | --from-commits [range]]
---

# docs-update: Update Raw Document

Update an existing `docs/raw/` document in-place. While raw documents are designed to be stable records, real-world workflows require controlled mutations — specs evolve during implementation, plans need adjustment when issues are discovered. This skill provides a disciplined way to make those changes while maintaining full traceability via git history and log.md.

## Input Parsing

Parse `$ARGUMENTS`:

1. **No argument (default)** — auto-detect mode. Analyze recent git commits to find which raw documents need updating. This is the most common usage.
   - Example: `/docs-update`
2. **`--from-commits` [range] only** — same as above but with an explicit commit range
   - Example: `/docs-update --from-commits HEAD~10..HEAD`
3. **Path + `--from-commits` [range]** — update a specific raw file OR all raw files under a raw directory based on git commit history
   - File example: `/docs-update docs/raw/specs/2026-04-08-design.md --from-commits`
   - Directory example: `/docs-update docs/raw/specs/ --from-commits abc123..def456`
4. **Path + reason** — update a specific raw file or directory with a manually described reason
   - File example: `/docs-update docs/raw/specs/2026-04-08-design.md "add error handling section"`
   - Directory example: `/docs-update docs/raw/guides/gpt-oss/ "align with new naming convention"`
5. **Path only** — path to a `docs/raw/` file or directory, no reason provided
   - Ask the user for the reason before proceeding

**Path validation**: if a path is provided, use `test -d` / `test -f` to distinguish file vs. directory. Validate it exists and is under `docs/raw/`. If not under `docs/raw/`, stop with an error — this skill only updates raw records in place (to ingest new content, use `/docs-ingest` instead).

**Directory input expansion**: when the path is a directory, enumerate all raw document files under it (recursively) via Glob (`<dir>/**/*.md`, etc.). The resulting file set is the update target group — all subsequent steps operate on this group. If the directory contains no raw files, stop with an error. Show the user the enumerated list and let them exclude specific files before proceeding.

**Batch coherence**: when multiple files are targeted (via directory input or auto-detect selecting several), treat them as a batch — one reason typically applies to all of them, wiki re-synthesis happens once after all raw edits, and the log records a single batch update entry (see Step 6).

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

> "Based on commit history analysis, the following documents need updating:
> 1. `docs/raw/specs/2026-04-08-design.md` — <reason summary>
> 2. `docs/raw/plans/2026-04-08-impl.md` — <reason summary>
> Select documents to update (all / confirm each / skip):"

If a specific file path was provided, only show the analysis for that file:

> "Based on commit history analysis, the following changes are needed:
> - <change 1>
> - <change 2>
> Proceed?"

Proceed to Step 3 with user-confirmed documents and their synthesized changes. For each confirmed document, read it in full and read its associated wiki pages before updating.

### Step 2-alt: Manual Mode

If a path was provided without `--from-commits` and without a reason:

1. Read the target raw file(s) in full. For directory input, read every enumerated file.
2. Read `docs/README.md` to find wiki pages that reference any of the target raw files (union set)
3. Read those wiki pages to understand the current synthesized state
4. Ask the user:
   - What changes are needed? (If directory input and changes differ per file, ask per file; if changes are uniform across the batch, ask once)
   - Why? (bug found, scope change, implementation feedback, etc.)

### Step 3: Update the Raw File(s)

Apply the requested changes to each target raw document. Preserve existing frontmatter, but:

- Update `tags` if the changes affect the document's topics
- Do NOT change `ingested_at` — it records when the document was first created
- Do NOT change `source_type` — it records how the document originally entered the system
- Do NOT change `batch_id` if present — it preserves the grouping from the original directory ingest

The `immutable: true` field may still be present in legacy files — leave it as-is (it is no longer enforced for updates via this skill).

For batch updates (directory input or multiple files selected), apply edits to all files before moving on to wiki propagation — wiki re-synthesis should see the final state of every changed raw file.

### Step 4: Propagate to Wiki

Collect the set of wiki pages whose `sources` frontmatter references ANY of the updated raw files (union across the batch).

For each affected wiki page:

1. Read the wiki page
2. Re-read ALL raw files listed in that wiki page's `sources` (not just the updated ones — neighbors may still contribute)
3. Re-synthesize the wiki page to reflect the updated information
4. Set `last_updated` to today's date
5. Update `tags` if topics changed

If a single wiki page is affected by multiple updated raw files in the batch, re-synthesize it once after all raw edits, not once per file.

If the update introduces a topic not covered by any existing wiki page, create a new one and add it to `docs/README.md`.

### Step 5: Update README.md

If any wiki page descriptions changed or new wiki pages were created, update `docs/README.md` accordingly.

### Step 6: Append to Log

For a single-file update, append:

```markdown
## [YYYY-MM-DD] update | <document title>
- raw: raw/<category>/<filename>.md
- source: <manual | commits (range)>
- reason: <why the update was needed>
- changes: <brief summary of what changed>
- wiki updated: wiki/<page>.md (updated) [, wiki/<page2>.md (updated)]
```

For a batch update (directory input or multiple files), append a single entry that summarizes the batch:

```markdown
## [YYYY-MM-DD] update | <batch title> (N files)
- raw: raw/<category>/ (N files: <list file basenames, or "see scope" if many>)
- scope: <path/to/dir-or-glob-used>
- source: <manual | commits (range)>
- reason: <why the update was needed>
- changes: <brief summary; note per-file variations if significant>
- wiki updated: wiki/<page>.md (updated) [, wiki/<page2>.md (updated)]
```

When `--from-commits` was used, `source` should record the commit range, e.g. `commits (abc123..def456)` or `commits (since 2026-04-01)`.

### Step 7: Update Search Index

If qmd is available (`command -v qmd`), refresh the search index so updated pages are immediately searchable:

```bash
qmd update && qmd embed
```

This step is silent — no output needed unless it fails.

### Step 8: Present Summary

For single-file:

> **Update complete**
> - Updated document: `docs/raw/<category>/<filename>.md`
> - Reason: <reason>
> - Changes summary: <brief changes>
> - Wiki synced: `docs/wiki/<page>.md` (updated)

For batch:

> **Batch update complete**
> - Update scope: `<dir path>` (N files)
> - Reason: <reason>
> - Changes summary: <brief summary>
> - Wiki synced: <list of updated wiki pages>

### Step 9: Commit

Stage all changed files under `docs/`. Create a git commit:

```
docs: update <document title> — <reason>
```

For batch updates:

```
docs: update <batch title> (N files) — <reason>
```
