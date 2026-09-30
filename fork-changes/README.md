# Fork Changes

Custom features added on top of upstream `zed-industries/zed`. Kept here so we
can re-apply or re-verify them when merging upstream `main`.

## Current changes (as of 2026-09-30)

The fork retains these custom features:

1. **Compare with Branch…** — right-click a file in the project panel (or run
   `git: compare file with branch` from the command palette) → branch picker →
   opens a `FileDiffView` of that file against the chosen branch's merge-base
   version.
2. **Show Diff** — right-click a file → `git: show file diff` → opens the
   `SoloDiffView` (same as Git Panel "File Diff" click behavior) showing the
   whole file diffed against HEAD.
3. **Project panel type-ahead** — focus the file tree and type; printable
   keystrokes accumulate into a query (resets after 1s idle). Case-insensitive
   fuzzy subsequence matching selects visible entries and highlights matched
   characters, with a query pill overlay. Backspace edits the query, Up/Down
   cycle through matches, and Escape cancels it. The `type_ahead` key context
   activates these bindings while a query is active.
4. **Git Graph branch filtering** — merged from upstream PR #57266 (branch
   `pr-57266` / tip `2c27164c97`, "show only current branch" toggle, branch
   chips, `+` add-branch picker). Merge required small fixes: unified `ui::`
   import list, restored a closing brace after `deploy_header_context_menu`,
   and added the now-required `PickerDelegate::name()` (`"branch-add"`) for
   `BranchAddPickerDelegate`. The PR was never merged upstream; if it lands,
   prefer dropping our copy.

Upstream references: `git: compare with branch` (repo-wide) landed via PR
#57886; File History menu item pattern from PR #42441.

## Files touched

| File | Change |
|---|---|
| `crates/git/src/git.rs` | New actions `git::CompareFileWithBranch`, `git::ShowFileDiff` (defined in `git` crate so `project_panel` can reference them without a dependency cycle) |
| `crates/project/src/git_store.rs` | Made `Repository::load_blob_content` `pub` (was private) |
| `crates/git_ui_core/src/file_diff_view.rs` | New `FileDiffView::open_with_base_text()` — old side is an in-memory buffer with the branch's blob content; new `base_label` field overrides the tab title (`{branch} ↔ {file}`) |
| `crates/git_ui_core/src/git_ui_core.rs` | Three global-opener indirections following the existing `set_file_history_opener` pattern: `set_compare_file_with_branch_opener`, `set_show_file_diff_opener` (+ their `open_*` fns) |
| `crates/git_ui/src/branch_diff.rs` | `BranchDiff::compare_file_with_branch()` — resolves repo path, opens branch picker modal (`branch_picker::select_modal`), on select: merge-base `diff_tree` → `load_blob_content` → `FileDiffView::open_with_base_text` |
| `crates/git_ui/src/git_ui.rs` | Installs the two openers in `init()`; show-file-diff opener defers via `window.spawn` (see Gotchas) |
| `crates/project_panel/src/project_panel.rs` | Context menu items ("Show Diff", "Compare with Branch…"), three `capture_action` renderers forwarding `git::FileHistory`-style actions to `git_ui_core` openers, plus a workspace-level `register_action` for `git::CompareFileWithBranch` (palette-invocable: uses panel selection if panel focused, else active editor's file at cursor) |

## Key architectural pattern

`project_panel` cannot depend on `git_ui` (cycle), so cross-crate actions are
handled via:

```
action (defined in `git` crate)
  → project_panel capture_action / register_action
    → git_ui_core global opener (installed by git_ui at init)
      → git_ui implementation
```

This mirrors how upstream implements `git::FileHistory`. Any new per-file git
action should follow the same shape.

## Gotchas learned the hard way

- **GPUI double-lease panic**: do not call `workspace_entity.read()` /
  functions that read the workspace while already inside a
  `workspace.update(...)`. `SoloDiffView::open_or_focus` reads the workspace,
  so the show-file-diff opener must defer via `window.spawn(cx, async move |cx|
  cx.update(...))` and scope its reads before calling it.
- **`unbind` in keymap.json is a map** of `keystroke → action` (string or
  `[name, input]`); `null` values are rejected. Zed's keymap editor can emit
  broken `unbind` blocks.
- `RepoPath` has no `Display`; use `file_name()` / `as_unix_str()`.
- `Entity::update` on a **strong** entity in an async context returns `R`
  directly; on `WeakEntity` it returns `Result`. `Repository::diff_tree`
  returns `oneshot::Receiver<Result<TreeDiff>>` → `.await??`.
- Closures stored in `Arc<dyn Fn>` must be `Fn`: clone captured
  `WeakEntity<Workspace>` handles at the top of the closure body before moving
  them into inner `async move` blocks.

## Not upstreamed / local-only

- `git::ShowFileDiff` and `git::CompareFileWithBranch` actions do not exist in
  upstream — user keymap entries referencing them will log warnings in stock
  Zed.
- User keymap (`~/.config/zed/keymap.json`) has local bindings:
  `cmd-d` → `git::ShowFileDiff` in `ProjectPanel && not_editing`.

## Discarded approach (kept for history)

An earlier iteration filtered the whole-repo `DiffBufferList` /
`BranchDiff` (project diff multibuffer) by path (`with_path_filter`). It
worked but showed only changed hunks — user wanted the full-file view — and
added plumbing through `BranchDiff` serialization. Fully reverted; see git
history if needed.

## Merge-with-upstream checklist

1. `git fetch upstream && git merge upstream/main` (or rebase).
2. Conflicts likely in `crates/git/src/git.rs` (actions list),
   `crates/project_panel/src/project_panel.rs` (context menu + action
   renderers — upstream actively edits these), `crates/git_ui/src/git_ui.rs`
   (`init()`), `crates/git_ui_core/src/git_ui_core.rs` (opener globals).
3. After resolving, verify the gotchas above still hold (especially whether
   `open_or_focus` / `FileDiffView::open*` signatures changed).
4. `./script/clippy -p git -p project -p project_panel -p git_ui_core -p git_ui`
   then `cargo test -p git_ui --lib && cargo test -p git_ui_core --lib`.
5. Smoke test in a real repo: context menu items, `cmd-d`, palette action,
   panel-selection vs editor-cursor behavior.

## Upstream sync on 2026-09-30

Merged upstream `main` at `39b5329322` into `fork-features`, retaining the
existing fork commits. Resolved conflicts by keeping both Git Graph branch
filter state and upstream navigation history, both type-ahead and upstream
drag tests, and the combined project panel imports. `Repository::load_blob_content`
remains public with upstream's new shared-reference implementation and
background object-read limiter.

Validation: `./script/clippy -p git -p project -p project_panel -p git_ui_core
-p git_ui` passed, as did the type-ahead regression test and the complete
`git_ui` (162 tests) and `git_ui_core` (28 tests) library suites, including
`test_diff_view_with_base_text`. Clippy required minor cleanup of the retained
fork code: mark an unused context parameter, copy an `Oid` directly, and
construct unit actions without `default()`. The complete `project_panel`
library suite also passed (137 tests).
