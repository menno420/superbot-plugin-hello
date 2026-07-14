# Fleet cleanup audit — 2026-07-13

Scope: read-only audit of `menno420/superbot-plugin-hello` (local clone
`/workspace/superbot-plugin-hello`, `main` @ `bbaccec`), run as part of a
~20-repo fleet cleanup pass on the last night of the EAP (owner "ORDER 045").
This is a complementary audit, not a redispatch of work — no feature changes
were made to the plugin code itself.

## What this repo is

`superbot-plugin-hello` is the **game-plugin-contract hello-world** for the
host bot [`superbot-next`](https://github.com/menno420/superbot-next): one
command (`!hello` / `/hello`) that routes to one panel (`hello.home`),
declared as a `SubsystemManifest` and exported through the `sb.plugins`
`pyproject.toml` entry point. It exists to prove, end to end, that an
**out-of-tree** repo can declare a subsystem, get its manifest hash pinned
in the host's committed `plugins.lock.json`, and be joint-compiled and
dispatched by the host exactly like an in-tree `sb/manifest/<key>.py`
module. It is explicitly a minimal template other game repos (mining,
exploration/D&D, …) are meant to copy from — not a product surface with its
own roadmap.

## Structure

```
pyproject.toml                      # dist metadata + the sb.plugins entry point
substrate.config.json               # kit_version pin (1.13.0), mirrors host
superbot_plugin_hello/
  __init__.py                       # intentionally empty (docstring only)
  manifest.py                       # MANIFEST = SubsystemManifest(...) + @panel registration
tests/
  test_manifest.py                  # 4 authoring-time checks (needs the host `sb` dep)
```

Six tracked files total, all introduced in a single commit
(`bbaccec50aa21fc744b8d37ecded8666365a63a1`, "seed: game-plugin-contract
hello-world (ORDER 002 / ORDER 014)", 2026-07-12T13:29:35Z), moved verbatim
from `superbot-next`'s `examples/superbot-plugin-hello/`. There is no
`.github/` directory, no `LICENSE`, no `.gitignore`, and no control-bus
files (`control/status.md`, `control/inbox.md`) — this repo does not
participate in the fleet's per-seat heartbeat/coordinator convention that
other `menno420/*` repos use; it is a passive artifact repo, not a session
seat.

## CI setup and health

**There is no CI.** No `.github/workflows/` directory exists in the repo at
all. `tests/test_manifest.py` requires the host's `sb` package
(`pip install -e .[host]`, which pulls `superbot-next` from git) to even
import, so it cannot run standalone in this repo's own environment — by
design, per the README's "Developing" section. There is nothing for a
GitHub Actions workflow to gate here without also installing the host
package from git, which is presumably why none was set up. This is
consistent with the repo's stated purpose (a template/contract proof, not
a maintained service) but is worth naming explicitly: **a syntax error or
drift in `manifest.py` would not be caught by anything in this repo** — the
host's `tools/plugin_pin.py` / boot-time hash verification is the only
gate, and it lives in `superbot-next`, not here.

## Doc quality

The `README.md` is accurate against the current tree: the "Shape" section
lists exactly the six files present, the entry-point snippet matches
`pyproject.toml` verbatim, and the "Developing" / "Shipping into the host"
sections describe a workflow consistent with what `superbot-next`'s
`docs/game-plugin-contract.md` documents on the host side (verified via
`raw.githubusercontent.com` read of that doc, 2026-07-13). No stale links,
no broken cross-references, no TODOs left in place. For a six-file repo
this is essentially the doc quality ceiling — nothing to correct.

## Open PRs

**None.** `list_pull_requests` (state=open and state=closed) both return
zero results, and `search_issues` for the repo returns zero results. There
is no PR/issue history to disposition — no merges, no closures, nothing
left untouched. This matches the repo's single-commit history (`git log
--all` shows exactly one commit on `main`) and its single, protected
`main` branch (no stale feature branches).

## Inconsistency found (documentation, not code)

The seed commit's message (`bbaccec`) asserts:

> "the host's committed `plugins.lock.json` pin
> `sha256:06023075b8db1a16f4f3c1bb4a9400252e88931501de50deccdc095a825e93a0`
> stays valid unchanged"

Cross-checking the host repo (`menno420/superbot-next`) at the time of this
audit (2026-07-13 ~22:30Z):

- The **currently committed** `plugins.lock.json` on `superbot-next@main`
  pins `superbot-plugin-hello` at
  `sha256:ff75b9eba291ca659793b91545f71c1e5bb31120270d3d77f8d604fe1314bdf3`
  — a **different** hash than the one asserted in this repo's seed commit.
- Host PR **#311** ("Plugin-boot proof: boot a REAL external plugin
  headless against the committed pin (+ fix the stale hello lock)", merged
  `2026-07-12T23:06:54Z`) explains why: the `06023075…` pin was "born
  CORRECT in the contract-v1 seed (#75) but drifted at #232, which added
  the `CommandSpec` `modal` facet" — `serialize_manifest` is duck-typed
  over declared fields, so an unrelated host-side schema change moved the
  hash without a re-pin. #311 re-pinned the lock from `06023075…` to
  `ff75b9eb…`.
- This repo's seed commit landed at `2026-07-12T13:29:35Z` — **before**
  #311 (`23:06:54Z` the same day). At the moment this repo was created, the
  `06023075…` pin the commit message cites was *already* the stale one
  (per #311's own description of what it found broken); it was corrected
  in the host roughly 9.5 hours later.

Net effect: this repo's one commit message documents a plugin-lock hash
that was stale even at the time it was written, and is now further out of
date against the host's live pin. This is **not a functional problem** —
the host, not this repo, is the source of truth for the live pin, and
`plugins.lock.json` in `superbot-next` is correct today — but a future
reader of this repo's git history could reasonably (and incorrectly) trust
the commit message's hash as current. No code or doc *in this repo*
depends on that hash (the README doesn't quote it), so nothing here needed
correcting; this is flagged for the record rather than fixed, since editing
a historical commit message would rewrite history for no functional gain.

No other inconsistencies were found: `pyproject.toml`'s entry point,
`README.md`'s file listing, and `manifest.py`'s actual declared surface
(one command routing to one panel, `stores`/`data_invariants`/
`wizard_sections` all empty per the v1 contract) all agree with each other
and with the host's documented contract.

## Suggestions

1. **No CI is a reasonable default here, but a cheap guard is possible.** A
   minimal `pyproject.toml`-only workflow (`python -m py_compile
   superbot_plugin_hello/manifest.py`, or a `ruff`/`black --check` pass with
   no host dependency) would catch syntax errors and formatting drift
   without needing the host package installed. Not urgent given the repo's
   near-zero change rate, but cheap to add if this repo is meant to be a
   copy-from template for future game-plugin repos — a broken template is a
   worse failure mode than a broken product repo, since it gets propagated.
2. **Centralize the plugin-pin cross-check.** The inconsistency found above
   (stale hash in a commit message) exists because the pin lives in a
   *different* repo than the manifest it hashes, with no automated link
   between them. If the fleet ends up with more out-of-tree game plugins
   (mining, exploration/D&D per the README), consider a small script in
   `superbot-next` (or a shared fleet tool) that periodically diffs each
   plugin repo's live `manifest.py` hash against the committed
   `plugins.lock.json` entry and flags drift — rather than relying on each
   plugin repo's commit messages staying accurate by hand.
3. **This repo is a good candidate to exclude from any fleet-wide "every
   repo needs a control/status.md heartbeat" push**, if one exists — it has
   no coordinator seat, no recurring session work, and adding heartbeat
   machinery to a six-file contract-proof repo would be pure overhead. Worth
   an explicit allowlist/denylist entry in whatever tracks fleet repo
   conventions, so a future sweep doesn't "fix" this repo's missing
   control-bus files as if they were an oversight.
4. **`substrate.config.json`'s `kit_version` pin (currently `1.13.0`) should
   be checked against the host's own pin periodically** — the seed commit
   states it mirrors `superbot-next`'s `substrate.config.json` pin as of
   host PR #251; if the host bumps its kit version, this repo's copy will
   silently drift unless something reminds an agent to update it. Same
   category of risk as suggestion 2, smaller blast radius.

## Bottom line

This repo is exactly what it claims to be: a near-empty, six-file contract
demo with no open PRs, no CI, no doc errors, and no stale branches. The one
finding is a historical-commit-message hash mismatch against the host repo,
which is informational only (nothing in this repo or its consumers reads
that commit message programmatically). No merges, closures, or code fixes
were needed or performed during this audit.
