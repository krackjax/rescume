# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Rescume is a **Claude Code plugin** (not a standalone application). It is distributed via the plugin marketplace system (`.claude-plugin/marketplace.json` + `.claude-plugin/plugin.json`) and, once installed, extends Claude Code with a skill, six subagents, and three slash commands that together tailor a user's resume to a job description and render it to a single-page PDF via Typst.

There is no build step, package.json, or test suite — the "code" is markdown agent/skill definitions plus a handful of standalone Python scripts invoked as CLI tools.

## Commands

```bash
# Install Python dependencies used by the scripts
pip install python-docx pdfplumber --break-system-packages

# Typst CLI is a separate system dependency
brew install typst        # macOS
typst --version            # verify

# List available Typst templates
python skills/typst-renderer/scripts/list_templates.py

# Compile a JSON resume to PDF (positional args, not flags)
python skills/typst-renderer/scripts/compile.py <content.json> <template-name> <output.pdf>

# JSON resume database (data/comprehensive_db/) CLI tools
python skills/json-database/scripts/db_init.py --output data/comprehensive_db/
python skills/json-database/scripts/db_load.py --db-path data/comprehensive_db/ [--file experiences]
python skills/json-database/scripts/db_save.py --db-path data/comprehensive_db/ --data '<json>'
python skills/json-database/scripts/db_add.py --db-path data/comprehensive_db/ --type experience --data '<json>'
python skills/json-database/scripts/db_validate.py --db-path data/comprehensive_db/

# Coverage checking (skill-gap analysis)
python skills/coverage-tracker/scripts/check_coverage.py --resume <docx> --requirements <jd.json>
```

There are no lint/test commands defined in this repo — validate changes by running the relevant script directly and inspecting its JSON output/exit code (see "Error Handling" sections in each `SKILL.md`).

## Architecture

### Plugin anatomy

- `agents/*.md` — six subagent definitions (YAML frontmatter + system prompt). Frontmatter declares `tools`, `model: inherit`, and which custom `skills:` the agent may invoke.
- `skills/*/SKILL.md` — four tool skills the agents/coordinator call into; each skill's Python scripts live in its `scripts/` subdirectory.
- `commands/*.md` — slash commands (`/rescume parse`, `/rescume analyze`, `/rescume start`) that are thin wrappers describing when/how to trigger the workflow; the `rescume` skill itself also auto-triggers on natural-language requests.
- `templates/*/` — Typst template packages (`template.typ`, `metadata.json`, example PDF) consumed by the `typst-renderer` skill.
- `hooks/hooks.json` — install/update/uninstall lifecycle hooks (installs Typst/pip deps, initializes `data/comprehensive_db/`, offers to preserve/delete data on uninstall).

### Two-phase workflow

**Phase 1 — Database building (one-time):** `resume-parser` agent ingests uploaded resumes (DOCX/PDF/TXT/MD/JSON) into a structured JSON database via the `json-database` skill, then `interview-conductor` (mode `initial_setup`) asks follow-up questions to enrich it.

**Phase 2 — Per-job tailoring:** `ats-analyzer` extracts required/preferred skills from a job description → `coverage-mapper` (using the `coverage-tracker` skill) maps the database against those requirements and finds gaps → gaps trigger `interview-conductor` (mode `gap_filling`) in a loop until must-have coverage is 100% → `content-generator` emits structured JSON content (see `skills/rescume/content_schema.json`) — it never makes formatting decisions → `typst-renderer` skill compiles that JSON to a single-page PDF via a chosen template → `hr-critic` evaluates content quality (`comprehensive` mode, then `final_validation` mode for an APPROVED/NEEDS_REVISION gate).

The full orchestration contract (which subagent to call when, what each expects/returns, file paths for intermediate artifacts) lives in `skills/rescume/SKILL.md` — read it before modifying agent handoffs.

### v2.0 design principle: LLM writes content, Typst does layout

This is the core invariant introduced in the v2.0 rewrite (see `CHANGELOG.md`): agents (`content-generator`, `hr-critic`) must never reason about fonts, word counts, or page limits — that's entirely the `typst-renderer` skill's job via its auto-fit loop (`skills/typst-renderer/scripts/compile.py`): start at 11pt, recompile at 0.5pt steps down to a 9pt floor, checking PDF page count with `pdfplumber` after each attempt; if still >1 page at the floor, return a structured "trim N bullets" recommendation instead of guessing. When touching `content-generator.md`, `hr-critic.md`, or the templates, preserve this separation — don't reintroduce word-count/compression logic that v2.0 deliberately removed.

### Data flow / persistence

All state is flat JSON files under `data/` (created by the install hook), not a database server:
```
data/comprehensive_db/{experiences,skills,projects,education,metadata}.json   # Phase 1 output, persists across jobs
data/job_applications/[job_id]/{jd_analyzed,coverage_matrix,content}.json     # Phase 2, per-job
```
Final PDFs are written under `data/job_applications/[job_id]/` (README) — some older docs reference `outputs/[job_id]/`; treat `skills/rescume/SKILL.md` and the hooks as authoritative over README where they disagree.

### Known inconsistency to watch for: hardcoded template/binary paths

`skills/typst-renderer/scripts/compile.py` and `list_templates.py` hardcode `TEMPLATES_DIR = ~/.claude/skills/rescume/templates` (a plugin-install location), **not** this repo's `templates/` directory, and `compile.py` hardcodes `TYPST_CLI = ~/.local/bin/typst` rather than resolving `typst` from `PATH`. If you're testing template compilation from a repo checkout rather than an installed plugin, these paths won't resolve as-is — either symlink/copy `templates/` into the expected location or adjust the constants locally (don't casually "fix" this in a way that breaks the installed-plugin case without checking how the plugin installer lays out `~/.claude/skills/`).

### Documentation staleness across the repo

This repo went through a significant v1→v2 architecture change (DOCX+word-count-compression → JSON+Typst-auto-fit) and not everything was updated consistently — e.g. `commands/start.md` and `skills/coverage-tracker/SKILL.md` still describe DOCX output and word-count/compression steps that no longer apply. `agents/AGENT_UPDATE_ACTION_LIST.md` and `skills/rescume/COMPRESSION_STRATEGIST_REMOVAL.md` track this cleanup. When editing docs, prefer `skills/rescume/SKILL.md` and `CHANGELOG.md` as the source of truth for current (v2.0) behavior, and flag/fix other stale references you encounter rather than propagating them.
