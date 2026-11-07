---
name: rebase-on-upstream-zellij
description: Rebase this fork's tab-bar and compact-bar plugins onto a newer zellij upstream release. Use when the user asks to "rebase on upstream", "update to zellij vX.Y.Z", "bump zellij version", "pull in latest zellij", or any equivalent. The fork extracts default-plugins from zellij-org/zellij and adds two orthogonal customizations (show_tab_indices + permissions handshake); this skill walks through snapshot-and-reapply, verification, tagging, and release.
---

# Rebase on upstream zellij

## Strategy: snapshot + reapply, not bump-deps

Do not just bump `zellij-tile` versions. Upstream patch releases routinely change `default-plugins/*/src/*.rs` (new fields in `Action::*` variants needing `..` in `matches!`, new `EventType`/`KeybindsVec` items). A pure deps bump compile-fails or silently misses fixes.

1. Snapshot upstream `default-plugins/{tab-bar,compact-bar}` at the new tag, overwriting local sources and `Cargo.toml`s.
2. Reapply the fork's customizations on top as separate, named commits.
3. Verify final `diff -ruN <upstream-new>/.../src <plugin>/src` contains only the fork's customizations and nothing else.

This pulls in upstream fixes for free and keeps the squash commit minimal.

## The two fork customizations

Both apply to **both** plugins. The squash commit on `main` (e.g. `b69aaff`) is the source of truth — read its diff to see exactly what to reapply.

1. **`show_tab_indices`** — bool config option that renders `<1>`, `<2>` prefixes on tab labels. Touches `State` + parse in load/init + render loop in `main.rs`, and `render_tab`/`tab_style` signature + an `index_color` + `tab_parts: Vec<ANSIString>` builder in `tab.rs`.
2. **Permissions handshake** — `request_permission(&[ReadApplicationState, ChangeApplicationState])` instead of `set_selectable(false)` at startup, plus `EventType::PermissionRequestResult` subscription and `Granted`/`Denied` branches in `update()`. compact-bar splits `setup_subscriptions` by `is_tooltip` and adds a `resubscribe_events` helper.

**Critical merge point:** the post-`Granted` re-subscribe must include `EventType::InitialKeybinds`. This is the only place a fork patch and an upstream change overlap; everywhere else they're independent. Missing this silently breaks keybinds in Locked mode at cold start.

**Reapply in the upstream's current shape, not the old fork's shape.** The previous fork tip's diff is the *semantic* reference for what to reapply, not a *shape* reference. When upstream refactors a customization site (v0.45.1 rewrote compact-bar's `setup_subscriptions` into `let events = if …; subscribe(&events)`), express the customization on top of the new shape — the goal is the minimal diff against upstream, and reverting an upstream refactor to preserve the old fork layout inflates the diff for nothing. "Match the previous fork tip byte-for-byte" applies only to pure formatting in files where the fork patch and upstream changes don't overlap (operand order, one-line vs multi-line calls in `compact-bar/src/tab.rs`).

## Workflow

1. `git checkout -b rebase-<new_tag>` from clean main.
2. Run `scripts/three_way_diff.sh <old_tag> <new_tag>` to clone both upstream tags into `/tmp/zellij-rebase-<new_tag>/` and pre-compute fork/upstream patches. Read `upstream-*.patch` to see what's new upstream.
3. Verify `zellij-tile <new>` is on crates.io (`curl -s https://crates.io/api/v1/crates/zellij-tile | jq '.versions[0:5][].num'`). If not, fall back to a git dep on the tag.
4. Make one commit per logical change: snapshot upstream src+Cargo.toml; restore fork Cargo.toml metadata + crates.io deps; reapply show_tab_indices to tab-bar; reapply permissions handshake to tab-bar; same two for compact-bar. `cargo check --target wasm32-wasip1` after each.
5. After all reapplies, sanity-check:
   ```
   diff -ruN /tmp/zellij-rebase-<new_tag>/<new_tag>/default-plugins/tab-bar/src tab-bar/src
   diff -ruN /tmp/zellij-rebase-<new_tag>/<new_tag>/default-plugins/compact-bar/src compact-bar/src
   ```
   Output must equal exactly the fork's customizations. Stylistic drift in `compact-bar/src/tab.rs` (operand order, multi-line vs one-line call) is a common offender — match the previous fork tip byte-for-byte to keep the squash diff minimal.
   Then check for accidental upstream reverts: `diff -ruN <upstream-new>/…/<plugin>/src <plugin>/src | grep '^-[^-]'` lists deleted upstream lines — every one of them must be a customization's necessary replacement (e.g. `set_selectable(false)` → `request_permission`, single `tab_styled_text` → `tab_parts` array). A block of upstream lines deleted means you reverted an upstream refactor; redo that hunk in the upstream's new shape.
6. `cargo build --release` produces `target/wasm32-wasip1/release/{tab-bar,compact-bar}.wasm`. Zero warnings.

## Finalize

```bash
git checkout main
git merge --squash rebase-<new_tag>
git commit -m "rebase customizations on zellij <new_tag>"
git tag -a <new_tag> -m "Rebased on zellij <new_tag>"
git push origin main <new_tag>
```

Tag format matches existing pattern (`v0.44.0`, iterations as `v0.44.0-1`). CI (`.github/workflows/release.yml`) triggers on `v*.*.*[-*]` and publishes both `.wasm` + sha256sums.

## Environment notes

- `Cargo.lock` is `.gitignore`d here. `cargo update -p zellij-tile -p zellij-tile-utils` locally, do not commit.
- On macOS with rustup, cargo is not on `$PATH` by default. Prefix invocations with `export PATH="$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH"`.
- `[lib]` section in plugin `Cargo.toml`s is intentionally absent — plugins build as binary crates targeting wasm32-wasip1.
