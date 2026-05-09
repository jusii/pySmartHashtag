# CLAUDE.md — pySmartHashtag (Jusii fork)

This file is fork-only metadata. It lives on `integration/kvlt` only — `main` stays clean for fast-forward syncs from upstream, and short-lived `feat/*` branches don't carry fork-only files that would dirty their upstream PR diffs.

## What This Is

Personal fork of [`DasBasti/pySmartHashtag`](https://github.com/DasBasti/pySmartHashtag) — a Python wrapper for the Smart cloud API (#1 / #3 / #5).

Two roles:

1. **Upstream contribution path.** PRs go from `jusii:feat/*` to `DasBasti:main`. Each `feat/*` branch is a single focused, PR-able feature.
2. **Runtime dependency for HashtagKvlt** (`~/Devel/Omat/hashtagkvlt/`). HashtagKvlt's `pyproject.toml` pins to **immutable `kvlt-deps-YYYY.MM.DD` tags** on `integration/kvlt` via `[tool.uv.sources]`. Don't break tagged commits — HashtagKvlt is consuming them.

## Branch model — the three layers

```
upstream/main  (DasBasti/pySmartHashtag — read-only)
     │
     ├── feat/journal-page-loop       ←  one PR-able feature per branch.
     ├── feat/trip-trackpoints        ←  Stay focused & clean. Don't pile new
     ├── feat/vehicle-state-endpoint  ←  work onto these — they're the unit
     ├── feat/<future-feat-X>         ←  of upstream contribution.
     │       │
     │       └── (PR upstream when stable)
     │
     └── integration/kvlt   ←  octopus / sequential merges of every still-
                               pending feat/* + upstream/main. NO force-push.
                               Tagged at meaningful checkpoints.
                                 │
                                 ▼
           kvlt-deps-2026.05.09   ←  immutable tag — what HashtagKvlt's
                                     pyproject.toml pins to.
```

**Discipline:**

- `feat/*` branches: one feature each. Avoid rebasing once a branch has an open PR (use fixup commits, squash on merge). Rebasing changes hashes, which makes `integration/kvlt` re-merges tangle.
- `integration/kvlt`: NEVER force-push. Re-merge `upstream/main` when it advances. Re-merge each `feat/*` when it advances. If conflicts get bad, the recovery is to rebuild integration from `upstream/main` + re-merge all current feat tips (lose merge history, get clean state, re-tag).
- `kvlt-deps-*` tags: immutable. Cut a fresh date-stamped tag when integration advances meaningfully; don't move existing tags. Old tags stick around so any historical HashtagKvlt deploy can be reproduced.
- `feat/trip-journal` is **chained ancestry** of `feat/journal-page-loop` (page-loop is stacked on it). Don't double-merge — merging journal-page-loop pulls trip-journal in.

## Working pattern when contributing back

```
new feature idea
    ↓
draft on integration/kvlt + run in HashtagKvlt
    ↓ (when stable)
extract clean feat/X branch on the fork
    ↓
push to fork; smoke-test integration/kvlt is rebuildable around it
    ↓
open PR upstream against DasBasti/pySmartHashtag
    ↓ ↑ (review cycle — fixup commits on feat/X, re-merge into integration/kvlt)
merge upstream
    ↓
upstream/main caught up; drop feat/X from integration/kvlt's merge set
    ↓
re-tag kvlt-deps-YYYY.MM.DD; bump HashtagKvlt pin
```

When the diff between `integration/kvlt` and `upstream/main` shrinks to zero: retire `integration/kvlt` and HashtagKvlt pins to upstream PyPI.

## Failure modes to know

- **Orphan SHA pins.** Don't pin HashtagKvlt to a raw fork SHA — branches get rebased to address PR review and the SHA becomes orphan. Always pin to a `kvlt-deps-*` tag. (Hit on 2026-05-09 with `f32f71d`.)
- **Upstream re-cuts your PR.** Detect with `git log upstream/main..your-branch` AND `git log your-branch..upstream/main` — if both sides have content and the upstream side looks thematically similar, suspect re-cut absorption. Recovery: close the original PR with a "superseded by #NNN" comment, delete the now-regressive fork branch, do NOT merge it into integration. (Hit on 2026-05-09 with PR #190 → upstream #193.)
- **`feat/X` rebase tangles `integration/kvlt`.** If a fork branch is force-pushed (e.g. interactive rebase to address PR review), the cheapest recovery is to re-create `integration/kvlt` from `upstream/main` + re-merge all current `feat/*` tips. Re-tag with a new date-stamp.

## Knowledge Base (Obsidian Vault)

Project context lives in the vault at `~/Devel/Obsidian/ObsidianAiVault/`, accessed via the **obsidian-vault MCP server** (globally configured at user scope).

**Read at session start:**

Vault essentials:
- `CLAUDE.md` (vault root) — vault structure and information placement rules
- `Machine/Coding Standards.md` — global rules (commit discipline, no AI attribution)

This project:
- `Machine/Personal/HashtagKvlt/Fork-Strategy.md` — load-bearing strategy doc; the *how* of the three-layer model
- `Machine/Personal/Smart/pySmartHashtag.md` — fork relationship, branch table, upstream-PR status
- `Machine/Personal/Smart/Overview.md` — ecosystem umbrella
- `Machine/Personal/Smart/Cloud-API-Findings.md` — vault index to `~/Devel/Omat/Smart/FINDINGS.md`
- `Machine/Personal/HashtagKvlt/Upstream-Port.md` — porting plan for moving HashtagKvlt's local pysmart_client extensions back here

**Read `~/Devel/Omat/Smart/FINDINGS.md` (in the multirepo, NOT the vault) before adding any new endpoint** — non-obvious quirks: auth-grant ceremony for some endpoints but not others, response-envelope shape, page-index semantics, HMAC signing's query-param gotcha.

## HMAC signing pattern

When adding an endpoint, follow `account.py::get_vehicle_information` exactly:

1. Build a `params` dict.
2. Pass to `generate_default_header(..., params=params)` — included in HMAC payload.
3. Pass the *same* `params` to httpx as `client.get(url, params=params)` — emitted on the wire.

Mixing these halves yields `code:1445 "Check signature error"`. See `Cloud-API-Findings.md` §"HMAC signing — the gotcha".

## Code conventions

- Python 3, packaged via `pyproject.toml` + `setup.py`
- Source under `pysmarthashtag/`
- Capture/probe scripts at the repo root (`capture_journal.py`, `probe_*.py`) — RE tools, not part of the package. Don't ship them in releases.
- Captures stored in `captures/` (gitignored — never commit captures, they contain VINs and tokens)

## Where new information goes

| Kind | Goes to |
|------|---------|
| Code changes | The repo (commit + PR upstream when feature is stable) |
| Fork-strategy refinement | vault: `Machine/Personal/HashtagKvlt/Fork-Strategy.md` |
| RE findings (new endpoint, status code, signing quirk) | `~/Devel/Omat/Smart/FINDINGS.md` (long form) — and update `Machine/Personal/Smart/Cloud-API-Findings.md` for the headline |
| Cross-fork / ecosystem context | `Machine/Personal/Smart/<note>.md` via obsidian-vault MCP |
| Upstream-PR status changes | `Machine/Personal/Smart/pySmartHashtag.md` PR table + `Machine/Personal/Smart/Changelog.md` |
| Integration tag cut, branch lifecycle, fork-side rebase | `Machine/Personal/Smart/Changelog.md` (dated entry) |
| Anything that affects HashtagKvlt's pin | `Machine/Personal/HashtagKvlt/Upstream-Port.md` + `Machine/Personal/Smart/Changelog.md` |

**Do not write project facts into auto-memory.** Memory is for user preferences and conversation-specific context.

## Commit discipline

Follow `Machine/Coding Standards.md`. Load-bearing: **never mention AI tools in commit messages** — no `Co-Authored-By`, no "generated with", no Claude references. No exceptions.

Git committer identity: **personal** (`Jusii <jussi@alanara.fi>`).

When opening a PR against `DasBasti/pySmartHashtag`:
- Tight, focused PRs (PR #194 sets the pattern)
- Match upstream's existing commit-message style (`git log upstream/main`)
- Tests passing locally before opening the PR

## Cloud-API politeness — load-bearing

This is the lib that talks to `apiv2.ecloudeu.com`. **Be gentle.**

- **Single TLS session** for any probe sequence (don't re-open per request)
- **2-second gap** between probes
- **Bail on 429** immediately (don't retry)
- Never burst. Default poll interval ≥ 15 min for journal endpoints

Capture script `capture_journal.py` already follows this discipline; mirror it in any new probe scripts.

## Reference: cloud-API quick facts

Pulled from `Cloud-API-Findings.md` for in-context recall — full detail in `~/Devel/Omat/Smart/FINDINGS.md`.

- **EU host:** `apiv2.ecloudeu.com`
- **Auth (EU):** 4.5-step OIDC + Gigya — context redirects → Gigya `socialize.getIDs` bootstrap → Gigya login → `authorize/continue` redirects. The Gigya bootstrap (Step 1.5) is required because Smart's Gigya tenant rejects fresh `accounts.login`.
- **Response envelope:** `{code, data, success, hint, httpStatus, sessionId, message}`. HTTP status is *always 200* — `code` (string!) carries the real status.
- **Status codes:** `1000` success / `1402` token expired / `1405` endpoint not on this host / `1445` HMAC signature error / `8006` re-`select_active_vehicle` / `8153` telematics datum not yet available (NOT "TBox offline")
- **Auth-grant ceremony** is required for `journalLogV4` but NOT for the `historyV2` per-trip polyline endpoint. Don't blindly call `grant_journal_authorization` before every endpoint — it changes signing in ways that break the calls that don't need it.
