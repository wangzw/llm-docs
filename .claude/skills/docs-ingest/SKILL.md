---
name: docs-ingest
description: Add a new document to the LLM-Docs system. Supports file paths, directory paths, free text, and URLs. Classifies the document, archives it to docs/raw/, updates the wiki and index. Use when recording design decisions, importing existing docs, or archiving meeting notes.
allowed-tools: Read Write Edit Bash Glob Grep WebFetch
user-invocable: true
argument-hint: [file-path | dir-path | "free text" | https://url]
---

# docs-ingest: Add Document to Documentation System

Add a new document to the LLM-Docs documentation system. The document is classified, archived as an immutable raw record, and the wiki is updated to reflect the new knowledge.

## Input Parsing

The argument `$ARGUMENTS` determines the input type:

1. **No argument (default)** — auto-discover mode. Compare the current branch against `main` to find new or changed files that should be ingested (specs, plans, design docs, ADRs, etc.). See **Auto-discover Mode** below.
2. **File path** — argument is a path to an existing file (check with Glob/Read)
   - Read the file content
   - `source_type: file`
3. **Directory path** — argument is a path to an existing directory (check with `ls -la` or Glob)
   - Enumerate documents within the directory and process as a batch
   - `source_type: dir`
   - See **Directory Mode** below
4. **URL** — argument starts with `http://` or `https://`
   - Fetch content via WebFetch
   - `source_type: url`
   - Record the URL in `source_url` frontmatter field
5. **Free text** — argument is quoted text or anything that is not a file/dir path or URL
   - Use the text directly as content
   - `source_type: text`

To distinguish file vs. directory, run `test -d <path>` or check with `ls`. Do NOT assume — the user may pass either.

**In-place paths**: if the resolved file or directory path is already under `docs/raw/`, those files are treated as already-archived raw records — do not copy them, rewrite them, or add frontmatter. See **Step 3.0** for how this affects the workflow.

### Auto-discover Mode

When no argument is provided:

1. Run `git diff --name-only --diff-filter=A main...HEAD` to find files added on the current branch (compared to `main`)
2. Also run `git diff --name-only --diff-filter=M main...HEAD` to find modified files
3. Filter candidates to documentation-relevant files:
   - Markdown files (`.md`) outside of `docs/` (files inside `docs/` are managed by the system itself)
   - Files matching common design doc patterns: `*spec*`, `*design*`, `*adr*`, `*rfc*`, `*plan*`, `*architecture*`, `*prd*`
   - Other files the LLM judges to be documentation based on content (e.g. a `.md` file describing an API)
4. Cross-reference with `docs/log.md` — exclude files that have already been ingested (matching by file path in `source` field)
5. If candidates are found, present them to the user:

> "发现以下文件可能需要归档：
> 1. `path/to/new-spec.md` — (new) 看起来是设计规格
> 2. `path/to/updated-plan.md` — (modified) 看起来是实现计划
> 选择要 ingest 的文件（全部/逐项确认/跳过）："

6. For each confirmed file, proceed with the normal ingest workflow (Step 2 onwards)
7. If no candidates are found, tell the user and ask if they want to provide input manually

### Directory Mode

When the argument is a directory path, treat it as a **batch ingest** — multiple related documents belonging to one coherent topic.

#### D1. Enumerate documents

Use Glob to list document files in the directory (recursively):

- Primary patterns: `**/*.md`, `**/*.markdown`, `**/*.txt`, `**/*.rst`
- Exclude: hidden files (`.*`), binaries, lock files, anything under `node_modules/`, `.git/`, `dist/`, `build/`
- If the directory is empty of documents, tell the user and stop

Show the user the discovered file list and ask for confirmation before proceeding. Offer to exclude specific files.

#### D2. Detect the index file

Look for an index/entry file that describes the directory as a whole. Check in this priority order:

1. `README.md` / `README` / `readme.md`
2. `index.md` / `INDEX.md`
3. `SUMMARY.md` (common in mdBook/GitBook structures)
4. `TOC.md` / `_toc.md`
5. `<dirname>.md` (e.g. `architecture.md` inside a directory named `architecture/`)

If found, the index file serves two purposes:
- **Classification hint**: its title and content strongly influence the category choice for the whole batch
- **Wiki scaffolding**: its structure (headings, links) informs how the resulting wiki page(s) are organized

If no index file is detected, synthesize one from the content of all documents to structure the wiki page.

#### D3. Classify as a batch

The entire directory typically maps to **one category** under `docs/raw/`. Use the index file (if present) plus a sampling of the other files to classify.

If documents in the directory clearly span multiple categories (rare — usually signals the user passed the wrong root directory), ask the user whether to:
- (a) ingest everything under one category anyway, or
- (b) split into multiple categories per file, or
- (c) abort and have the user point to a more specific subdirectory

#### D4. Archive each file individually

Each source file becomes its own raw record under `docs/raw/<category>/`. This preserves provenance — consumers can still trace any claim to a specific source document.

For each file:
- Filename: `YYYY-MM-DD-<dir-slug>--<file-slug>.md` (double dash separates directory context from file identity)
- Frontmatter `source_type: dir`
- Add `source_path: <original/relative/path>` field to record the file's location within the input directory
- Add `batch_id: <dir-slug>-YYYY-MM-DD` so all files from one ingest are linkable
- If the file is the detected index, add `index: true`
- Body = original content, unmodified

#### D5. Wiki strategy for directory input

Unlike single-file ingest, a directory typically produces **one cohesive wiki page** (or a small cluster) that synthesizes across all raw files:

- If an index file exists, use its structure as the skeleton for the wiki page
- Treat other files as sections, subsystems, or supporting details
- The wiki page's `sources` frontmatter lists ALL raw files from the batch
- Cross-reference raw files with `[[raw/<category>/<filename>]]` so readers can drill into any specific source

For large directories (10+ files spanning distinct sub-topics), consider creating an index wiki page plus one sub-page per major topic. Ask the user before fanning out this way.

#### D6. Then continue with the normal workflow

After the batch-specific steps above, resume at Step 4 (Check for Unprocessed Raw Files), then Step 5 onwards. The log entry (Step 7) should summarize the batch rather than list every file:

```markdown
## [YYYY-MM-DD] ingest | <batch title> (directory, N files)
- source: dir (<original/path>)
- raw: raw/<category>/YYYY-MM-DD-<dir-slug>--*.md (N files, batch_id: <...>)
- wiki updated: wiki/<page>.md (<created | updated>)
- index updated: <description of README.md changes>
```

## Workflow

### Step 1: Load Schema

Read `docs/schema.md` to understand the current directory structure, classification table, and format conventions.

If `docs/schema.md` does not exist, tell the user to run `/docs-init` first and stop.

### Step 2: Analyze Content

Analyze the input content:

1. **Classify**: Determine which `docs/raw/` subdirectory it belongs to based on the classification table in schema.md. If no existing category fits, propose a new category to the user and await confirmation. If confirmed, create the subdirectory and update schema.md.
2. **Title**: Extract or generate a concise title
3. **Tags**: Generate relevant tags based on content semantics
4. **Key entities**: Identify important concepts, decisions, components, or systems mentioned

### Step 3: Archive to Raw

#### Step 3.0: Decide whether to archive

Before writing anything, check where the input lives:

1. **Input path is already under `docs/raw/`** (file or directory) — DO NOT create new archive files and DO NOT modify the originals. They are already raw records. Skip the rest of Step 3 entirely. In subsequent steps (wiki update, log), reference the existing files by their current path. Rationale: `docs/raw/` is immutable and already the canonical archive location; re-archiving would duplicate and break provenance.

2. **Input path is OUTSIDE `docs/raw/`** (or input is text/URL/dir that isn't under `docs/raw/`) — ask the user before archiving:

   > "即将把以下内容归档到 `docs/raw/<category>/` 并添加 frontmatter：
   > - <file 1 / batch summary>
   > - ...
   > 是否继续？(archive / wiki-only / cancel)"

   - `archive` (default): proceed with Step 3.1 below — copy content into `docs/raw/` and add frontmatter
   - `wiki-only`: skip archiving; reference the original path in wiki `sources` instead of a `raw/` path, and skip the log's `raw:` field
   - `cancel`: abort the whole workflow

3. **Input is text / URL / skill output** — these have no on-disk source under `docs/raw/`, so archiving is always required. Still confirm with the user if it's not obvious from context (e.g. free text paste).

Only proceed to Step 3.1 if the user chose `archive`, or if this is a text/URL/skill input.

#### Step 3.1: Write the raw file(s)

Write the content to `docs/raw/<category>/YYYY-MM-DD-<topic>.md` where:
- `YYYY-MM-DD` is today's date
- `<topic>` is a kebab-case slug derived from the title
- If a file with the same name already exists, append a numeric suffix (e.g., `-2`)

Frontmatter:

```yaml
---
title: <extracted title>
source_type: file | dir | text | url | skill
ingested_at: YYYY-MM-DD
source_url: <if URL input>
source_path: <if dir input — original relative path within input directory>
batch_id: <if dir input — shared id for all files in the batch>
index: <if dir input and this file is the detected index — true>
skill_name: <if skill output>
tags: [tag1, tag2]
immutable: true
---
```

The body is the original content, unmodified.

For directory input, see **Directory Mode → D4** above for filename convention and batch frontmatter fields.

### Step 4: Check for Unprocessed Raw Files

Scan `docs/raw/` for files that exist but have no corresponding entry in `docs/log.md`. These are files written directly by other skills (e.g., brainstorming specs, implementation plans).

If found, ask the user:

> "发现 N 个未处理的原始文档：
> - docs/raw/specs/2026-04-08-example.md
> - ...
> 是否一并更新 wiki？"

If yes, include them in the wiki update step below.

### Step 5: Update Wiki

Read `docs/README.md` to understand the current wiki structure. Then decide:

1. **Create new wiki page**: If the ingested content introduces a new topic not covered by any existing wiki page
2. **Update existing wiki page**: If the content updates, extends, or supersedes information in an existing wiki page
3. **Both**: If it's a new aspect of an existing topic

For each wiki page created or updated:
- Set `sources` in frontmatter to include all raw files that contributed
- Set `last_updated` to today's date
- Set `tags` based on the content's topics and the tags from contributing raw files
- Use `[[wiki-links]]` for cross-references to other wiki pages
- Use `[[raw/category/filename]]` for source references
- Synthesize a coherent current-state view — don't just append. The wiki page should read as a standalone document reflecting the latest understanding.

### Step 6: Update README.md

Read the current `docs/README.md`. Add or update entries for any new or changed wiki pages. Each entry is one line:

```markdown
- [[wiki/page-name]] — One-line description of what this page covers
```

Group entries by topic. If a new topic group is needed, add a new `## Section` header.

### Step 7: Append to Log

Append an entry to `docs/log.md`:

```markdown
## [YYYY-MM-DD] ingest | <document title>
- source: <source_type> (<source description>)
- raw: raw/<category>/<filename>.md
- wiki updated: wiki/<page>.md (<created | updated>)
- index updated: <description of README.md changes>
```

### Step 8: Present Summary

Show the user a concise summary:

> **Ingest 完成**
> - 原始文档：`docs/raw/<category>/<filename>.md`
> - 分类：<category>
> - Wiki 更新：`docs/wiki/<page>.md` (created/updated)
> - 索引更新：已添加/更新 README.md 条目

### Step 9: Commit

Stage all changed files under `docs/`. Create a git commit:

```
docs: ingest <document title>
```
