---
type: Source
title: "templates/hub/scripts/okf.py key regions (excerpt)"
description: "Excerpts of the shared toolchain that ships with the hub: its subcommands, the status/lifecycle mapping, and the lint rules that give the bundle its conventions."
status: draft
lifecycle: draft
created: 2026-11-20
updated: 2026-11-20
source_url: https://github.com/yishao1224-design/dsh-okf-plugin/blob/95b0615aefbbd4d1e16040efdd37d1e1884e5ed0/templates/hub/scripts/okf.py
tags: []
generated:
  by: dsh/agent
  at: 2026-11-20
---

# Capture: okf.py excerpts

`templates/hub/scripts/okf.py` is 1006 lines at commit `95b0615aefbbd4d1e16040efdd37d1e1884e5ed0`. These are the regions that define the bundle conventions; line numbers refer to that commit.

## Module docstring and subcommands (lines 2-27)

```text
okf.py — mechanical toolchain for an OKF bundle (default: okf/).

Subcommands:
  init             bootstrap the bundle skeleton + sync skills/ to per-tool
                   copies (.claude/skills/, .github/prompts/ for Copilot)
  lint             conformance + lifecycle audit (exit 1 on errors)
  lint --inbound P list files linking to bundle path P (safe-delete check)
  index            regenerate the bundle's index.md from tree + frontmatter
  links            expand [[slug]] shorthand into canonical bundle links
  log "MESSAGE"    prepend a log entry under today's date
  stats            counts by type/lifecycle, review queue
  drift            recheck pinned sources against their current git blob
  drift --hash F   print the blob sha to pin for file F

Every subcommand takes --bundle NAME to point at a differently-named
bundle directory. Without it the bundle is auto-detected (any top-level
directory holding a SCHEMA.md), falling back to the okf/ default.

The bundle targets OKF v0.2: `status` carries a standard v0.2 value and
this bundle's richer five-state lifecycle lives in the `lifecycle`
extension key, derived from it by LIFECYCLE_STATUS below.

Stdlib only. Uses PyYAML if installed, else a minimal frontmatter parser
sufficient for the SCHEMA.md templates (scalars, inline lists and maps,
block lists of scalars or mappings, one level of nested mapping).
```

## Constants (lines 35-55)

```python
ROOT = Path(__file__).resolve().parent.parent
DEFAULT_BUNDLE = "okf"
KB = ROOT / DEFAULT_BUNDLE
NON_BUNDLE_DIRS = {".git", ".github", ".claude", ".agents", ".cursor",
                   "node_modules", "scripts", "skills", "dist", "__pycache__"}
RESERVED = {"index.md", "log.md"}
OKF_VERSION = "0.2"
OKF_STATUSES = {"draft", "stable", "deprecated"}
LIFECYCLE = {"draft", "active", "needs_review", "deprecated", "archived"}
LIFECYCLE_STATUS = {"draft": "draft", "active": "stable",
                    "needs_review": "stable", "deprecated": "deprecated",
                    "archived": "deprecated"}
LINK_RE = re.compile(r"\[[^\]]*\]\(([^)\s]+)\)")
WIKILINK_RE = re.compile(r"\[\[([^\]|\n]+?)(?:\|([^\]\n]+?))?\]\]")
TAG_DEF_RE = re.compile(r"^\s*[-*]\s+`([A-Za-z0-9_-]+)`", re.M)
```

## Lint: type/lifecycle conformance (lines 446-501, abridged)

```python
        # OKF conformance
        typ = str(meta.get("type", "")).strip()
        if not typ:
            errors.append(f"{r}: frontmatter has no non-empty 'type' (OKF §9)")
        elif types_known is not None:
            parts = p.relative_to(KB).parts
            top = parts[0] if len(parts) > 1 else ""
            if typ not in types_known:
                warns.append(f"{r}: type '{typ}' not in SCHEMA.md type ...")
            elif top != "archive" and top != types_known[typ]:
                errors.append(f"{r}: type '{typ}' belongs in ... per SCHEMA.md type registry")

        status = str(meta.get("status", "")).strip()
        life = str(meta.get("lifecycle", "")).strip()
        if status and status not in OKF_STATUSES:
            errors.append(f"{r}: status '{status}' is not an OKF v0.2 value ...")
        if life and life not in LIFECYCLE:
            errors.append(f"{r}: unknown lifecycle '{life}'")
        if life in LIFECYCLE and status and LIFECYCLE_STATUS[life] != status:
            errors.append(f"{r}: lifecycle '{life}' implies status "
                          f"'{LIFECYCLE_STATUS[life]}', found '{status}'")
        ...
        if life == "active" and not meta.get("stale_after"):
            warns.append(f"{r}: active but no stale_after")
        if life == "deprecated" and not meta.get("superseded_by") \
                and "## Deprecation" not in body:
            errors.append(f"{r}: deprecated without superseded_by or '## Deprecation' section")
```

## Lint: tags, links, orphans (lines 540-579, abridged)

```python
        # tags
        if tags_known is not None:
            ...
            for t in tags:
                if t not in tags_known:
                    warns.append(f"{r}: tag '{t}' not in SCHEMA.md taxonomy")

        # links
        n_out = 0
        for t in LINK_RE.findall(strip_code(body)):
            tgt = resolve_link(t, p)
            ...
        if life in ("active", "needs_review") and n_out < 2:
            warns.append(f"{r}: only {n_out} outbound bundle link(s); minimum is 2")

        # wiki-links — authoring sugar that must be expanded before commit
        for m in WIKILINK_RE.finditer(strip_code(body)):
            ...
        # orphan check (not linked from anywhere, incl. index)
        if p.resolve() not in inbound and p.name != "SCHEMA.md":
            warns.append(f"{r}: orphan — no inbound links and not in index.md ...")
```

## index.md output shape (lines 609-637)

```python
def cmd_index(args):
    entries = {}
    for p in concept_files():
        meta, _, _ = load(p)
        meta = meta or {}
        top = p.relative_to(KB).parts[0]
        section = top if (KB / top).is_dir() else "_root"
        title = meta.get("title") or p.stem
        desc = str(meta.get("description") or "").strip()
        life = str(meta.get("lifecycle") or "").strip()
        badge = f" *({life})*" if life in ("needs_review", "deprecated", "archived") else ""
        entries.setdefault(section, []).append(
            f"* [{title}]({p.relative_to(KB).as_posix()}) - {desc}{badge}")

    lines = ['---', f'okf_version: "{OKF_VERSION}"', '---', '',
             '# Bundle Index', '']
    ...
```

## SCHEMA.md seed and bundle skeleton (lines 744-775, 868-906, abridged)

`cmd_init` creates `concepts/ entities/ comparisons/ queries/ archive/ sources/`, seeds `SCHEMA.md` and `log.md` when missing, syncs `skills/` into `.claude/skills/` and `.github/prompts/`, then regenerates `index.md` and runs lint. The seeded `SCHEMA.md` carries the type registry (`Concept`, `Entity`, `Comparison`, `Query`, `Source`, `Schema`), the shared frontmatter template, and the tag taxonomy (`unverified`, `contested`).
