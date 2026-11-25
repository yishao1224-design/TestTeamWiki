---
type: Source
title: "templates/hub/scripts/okf.py key regions at da10648 (excerpt)"
description: "Excerpts of the shared toolchain after the lifecycle key was retired: the status-only model, TOOLCHAIN_VERSION, review_due, the lint rules, the new migrate subcommand and the seeded SCHEMA.md text."
status: draft
created: 2026-11-25
updated: 2026-11-25
source_url: https://github.com/yishao1224-design/dsh-okf-plugin/blob/da10648a182b38b410d22f952d8e1d973d8825f0/templates/hub/scripts/okf.py
tags: []
generated:
  by: dsh/agent
  at: 2026-11-25
---

# Capture: okf.py excerpts at da10648

`templates/hub/scripts/okf.py` is 1154 lines at commit `da10648a182b38b410d22f952d8e1d973d8825f0`, up from 1006 at `95b0615`. These are the regions that define the bundle's review conventions after the `lifecycle` frontmatter key was retired; line numbers refer to `da10648`.

## Module docstring and subcommands (lines 2-30)

```text
okf.py — mechanical toolchain for an OKF bundle (default: okf/).

Subcommands:
  init             bootstrap the bundle skeleton + sync skills/ to per-tool
                   copies (.claude/skills/, .github/prompts/ for Copilot)
  lint             conformance + review-state audit (exit 1 on errors)
  lint --inbound P list files linking to bundle path P (safe-delete check)
  index            regenerate the bundle's index.md from tree + frontmatter
  links            expand [[slug]] shorthand into canonical bundle links
  log "MESSAGE"    prepend a log entry under today's date
  stats            counts by type/status, review queue
  migrate          drop the legacy `lifecycle` key (needs_review -> tag
                   needs-review); --dry-run lists the changes only
  drift            recheck pinned sources against their current git blob
  drift --hash F   print the blob sha to pin for file F

Every subcommand takes --bundle NAME to point at a differently-named
bundle directory. Without it the bundle is auto-detected (any top-level
directory holding a SCHEMA.md), falling back to the okf/ default.

The bundle targets OKF v0.2 and uses its standard `status` alone
(draft | stable | deprecated). The finer review states are derived, not
stored: a stable page *needs review* when it carries the `needs-review`
tag or its `stale_after` date has passed; a deprecated page is
*archived* when it lives under archive/. Bundles written before this
carried a five-state `lifecycle` extension key; `okf.py migrate` folds
it into `status` + tag, and lint warns until that is done.
```

## Constants: the toolchain version and the retired key (lines 46-66)

```python
OKF_VERSION = "0.2"
# The toolchain's own version, as a date. Bump it on every change to this
# file: tools that ship a copy (e.g. a team-wiki plugin seeding hub repos)
# compare it to decide whether a repository's copy is out of date.
TOOLCHAIN_VERSION = "2026.11.23"
# OKF v0.2 defines `status` with exactly these three values; it is the
# only state field a page carries.
OKF_STATUSES = {"draft", "stable", "deprecated"}
# The persistent "re-check this" flag on a stable page (a contradiction
# found, a source changed). An overdue `stale_after` needs no flag.
REVIEW_TAG = "needs-review"
# The retired `lifecycle` extension key and the status each value maps to;
# read only to lint and migrate bundles written before it was dropped.
LEGACY_LIFECYCLE_STATUS = {"draft": "draft", "active": "stable",
                           "needs_review": "stable", "deprecated": "deprecated",
                           "archived": "deprecated"}
```

## `tags_of` and `review_due` (lines 214-236)

```python
def tags_of(meta):
    """A page's tags as a list of strings."""
    tags = meta.get("tags") or []
    if isinstance(tags, str):
        tags = [tags]
    return [str(t) for t in tags]


def review_due(meta, today):
    """Why a stable page is in the review queue, or None.

    The two signals that need no git: the `needs-review` tag (a person or
    agent flagged it) and an overdue `stale_after`. Pinned sources that
    moved are the third, reported by drift and lint separately.
    """
    if str(meta.get("status", "")).strip() != "stable":
        return None
    if REVIEW_TAG in tags_of(meta):
        return "tagged needs-review"
    sa = parse_date(meta.get("stale_after"))
    if sa and sa < today:
        return f"due {sa}"
    return None
```

## The lint review-state block (lines 499-541)

```python
        # review state — OKF v0.2 `status` alone; see review_due()
        status = str(meta.get("status", "")).strip()
        if status and status not in OKF_STATUSES:
            errors.append(f"{r}: status '{status}' is not an OKF v0.2 value "
                          f"({'|'.join(sorted(OKF_STATUSES))}) — flag a page "
                          f"for review with the '{REVIEW_TAG}' tag instead")
        if "lifecycle" in meta:
            life = str(meta.get("lifecycle") or "").strip()
            if life in LEGACY_LIFECYCLE_STATUS and status \
                    and LEGACY_LIFECYCLE_STATUS[life] != status:
                errors.append(f"{r}: legacy lifecycle '{life}' implies status "
                              f"'{LEGACY_LIFECYCLE_STATUS[life]}', found '{status}'")
            warns.append(f"{r}: legacy 'lifecycle' key — run 'okf.py migrate' "
                         f"to fold it into status (+ tag {REVIEW_TAG})")
        ...
        for field in ("title", "description", "status", "updated"):
            if not meta.get(field):
                warns.append(f"{r}: missing '{field}'")
        archived = p.is_relative_to(KB / "archive")
        if status == "stable" and not meta.get("stale_after"):
            warns.append(f"{r}: stable but no stale_after")
        if status == "deprecated" and not archived and not meta.get("superseded_by") \
                and "## Deprecation" not in body:
            errors.append(f"{r}: deprecated without superseded_by or '## Deprecation' section")
        if archived and status and status != "deprecated":
            warns.append(f"{r}: under /archive/ but status is '{status}' — "
                         f"archived pages are 'deprecated'")

        sa = parse_date(meta.get("stale_after"))
        if status == "stable" and sa and sa < today and REVIEW_TAG not in tags_of(meta):
            warns.append(f"{r}: review overdue since {sa} — re-verify, or "
                         f"tag it {REVIEW_TAG} if it cannot be done now")
```

## The `index` badge (lines 659-668)

```python
        status = str(meta.get("status") or "").strip()
        if top == "archive":
            badge = " *(archived)*"
        elif status == "deprecated":
            badge = " *(deprecated)*"
        elif status == "stable" and REVIEW_TAG in tags_of(meta):
            badge = " *(needs review)*"
        else:
            badge = ""
        entries.setdefault(section, []).append(
            f"* [{title}]({p.relative_to(KB).as_posix()}) - {desc}{badge}")
```

## `stats`: by status, and the review queue with a reason (lines 761-790)

```python
def cmd_stats(args):
    by_type, by_status, queue = {}, {}, []
    today = date.today()
    files = concept_files()
    for p in files:
        meta, _, _ = load(p)
        meta = meta or {}
        by_type[str(meta.get("type", "?"))] = by_type.get(str(meta.get("type", "?")), 0) + 1
        status = str(meta.get("status", "?"))
        if p.is_relative_to(KB / "archive") and status == "deprecated":
            status = "deprecated (archived)"
        by_status[status] = by_status.get(status, 0) + 1
        why = review_due(meta, today)
        if why is not None:
            sa = parse_date(meta.get("stale_after"))
            queue.append((sa or date.min, rel(p), why))
    ...
    print("\nby status:")
    ...
    print(f"\nreview queue ({len(queue)}):")
    for _, r, why in sorted(queue):
        print(f"  {r} ({why})")
```

## `cmd_migrate` (lines 862-882)

```python
def cmd_migrate(args):
    changed = []
    for p in concept_files():
        if p.is_relative_to(KB / "sources"):
            continue  # captures are immutable
        raw = p.read_bytes().decode("utf-8")
        new, note = migrate_text(raw)
        if note is None:
            continue
        changed.append((rel(p), note))
        if not args.dry_run:
            p.write_bytes(new.encode("utf-8"))
    for r, note in changed:
        print(f"{'would migrate' if args.dry_run else 'migrated'} {r}: {note}")
    print(f"\nmigrate: {len(changed)} page(s) "
          f"{'to change' if args.dry_run else 'changed'}")
    if changed and not args.dry_run:
        print("next: add 'needs-review' to SCHEMA.md's tag taxonomy if it is "
              "missing, drop lifecycle from its templates, then run index, "
              "log and lint")
    return 0
```

`migrate_text(text)` does the per-page edit as text, keeping key order and
comments: it drops the `lifecycle:` line, writes the mapped `status`
(`draft|active|needs_review|deprecated|archived` → `draft|stable|stable|deprecated|deprecated`),
and for `needs_review` adds `needs-review` to `tags:` — inline (`[a, b]`), block
(`- a`) or scalar (lines 795-859).

## `--version`, and migrate in the dispatch table (lines 1124-1150)

```python
    ap.add_argument("--version", action="version",
                    version=f"okf.py {TOOLCHAIN_VERSION} (OKF {OKF_VERSION})")
    sub = ap.add_subparsers(dest="cmd", required=True)
    ...
    p = sub.add_parser("migrate", parents=[common])
    p.add_argument("--dry-run", action="store_true",
                   help="list the pages that would change, write nothing")
    ...
    return {"init": cmd_init, "lint": cmd_lint, "index": cmd_index,
            "links": cmd_links, "log": cmd_log, "stats": cmd_stats,
            "drift": cmd_drift, "migrate": cmd_migrate}[args.cmd](args)
```

## The state paragraph `init` seeds into SCHEMA.md (lines 958-962)

```text
`status` (OKF v0.2) is the only state field: `draft` until verified,
`stable` once verified, `deprecated` when superseded (archived pages
are `deprecated` pages under `archive/`). A stable page needs review
when its `stale_after` has passed or it carries the `needs-review` tag
(a contradiction found, a source changed); `okf.py stats` lists them.
```

The seeded frontmatter template drops `lifecycle` and reads
`stale_after: {today}     # required once status: stable` (lines 930-955), and the
seeded tag taxonomy gains a third entry (lines 980-984):

```text
- `unverified` — claims not yet checked against a second source
- `contested` — sources actively disagree
- `needs-review` — a stable page to re-check (contradiction found, source changed)
```
