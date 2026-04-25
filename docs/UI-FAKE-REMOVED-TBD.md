# UI Fake Removed TBD

This note captures the item `10` audit for `src/com/little_trader/ui`, the fake or legacy UI code removed in commit `#172.10`, and the pieces that remain real enough to drive the next UI adjustments.

## Audit Scope

- Workspace scanned: `src/com/little_trader/ui`
- Goal: separate live/usable UI from mock, fake, duplicate, orphan, or legacy UI
- Follow-up intent: remove non-authoritative UI surfaces before continuing the shell/header cleanup tickets

## Live UI Kept

These files remain because they are part of the current authenticated shell or active routed experience:

- `src/com/little_trader/ui/client.cljs`
- `src/com/little_trader/ui/header.cljs`
- `src/com/little_trader/ui/workspace.cljs`
- `src/com/little_trader/ui/strategy_editor.cljs`
- `src/com/little_trader/ui/meta_editor.cljs`
- `src/com/little_trader/ui/admin_panel.cljs`
- `src/com/little_trader/ui/trading_dashboard.cljs`
- `src/com/little_trader/ui/repl.cljs`
- `src/com/little_trader/ui/learn.cljs`
- `src/com/little_trader/ui/learn_studio.cljs`
- `src/com/little_trader/ui/learn_nnm.cljs`
- `src/com/little_trader/ui/fs_chart.cljs`
- `src/com/little_trader/ui/optimization_dashboard.cljs`
- `src/com/little_trader/ui/deribit_dashboard.cljs`
- Supporting shell/runtime files still referenced by the kept surfaces, including `mel_*`, `terminal_bus.cljs`, `param_box.cljs`, `payoff_chart.cljs`, `sim_transport.cljs`, `state.cljs`, `mutations.cljs`, and the `rf/` namespace files

## Removed UI Code

These files were removed because they were fake, mock-heavy, duplicated by a newer surface, orphaned from the live router, or clearly legacy:

- `src/com/little_trader/ui/app.cljs`
  - Old standalone UIx app
  - Not the active boot path; production boot goes through `com.little-trader.ui.client/init!`
- `src/com/little_trader/ui/admin_dashboard.cljs`
  - Legacy admin overlay
  - Duplicated by the routed `admin_panel.cljs`
- `src/com/little_trader/ui/chat.cljs`
  - Explicitly marked frozen in code
  - Removed from router, nav, and embedded learning panels
- `src/com/little_trader/ui/docs_drawer.cljs`
  - Orphaned docs UI
  - Not registered in the main router targets
- `src/com/little_trader/ui/lupii_junior.cljs`
  - Mock-first junior experience with frontend mock screener and mock assistant fallback
- `src/com/little_trader/ui/nnm_trading.cljs`
  - Placeholder-heavy NNM trading page with fake prediction and weights panels
- `src/com/little_trader/ui/opscan.cljs`
  - Standalone UI route not wired into the live shell
  - Server/domain payoff APIs still exist separately
- `src/com/little_trader/ui/trading_strategy_discrete.cljs`
  - Included mock/plain-text LLM fallback logic and was not part of the main user shell direction
- `src/com/little_trader/ui/workspace.cljs.bak`
  - Backup file, not source of truth

## Wiring Cleanup Performed

- Removed deleted routes and imports from `src/com/little_trader/ui/client.cljs`
- Removed the `AI Chat` nav entry from `src/com/little_trader/ui/client.cljs`
- Switched the header admin button in `src/com/little_trader/ui/header.cljs` away from the deleted overlay and onto `/#/admin-panel`
- Simplified `src/com/little_trader/ui/learn_studio.cljs` and `src/com/little_trader/ui/learn_nnm.cljs` to REPL-only tooling after removing the frozen chat UI

## Shell Findings From The Audit

These are not removed yet; they are real code and should guide the next ticket work:

- `workspace.cljs` still contains the floating `IDE Window` and `Panels` controls that overlap the header
- `workspace.cljs` still uses the `Panel Display` modal instead of a bar under the header
- `workspace.cljs` terminal still includes quick actions, MCP shortcuts, and top-bar controls slated for cleanup
- `strategy_editor.cljs` still contains tutorial/template buttons such as `Tutorial: Long Call 101` and `Hourly Long Call ATR`
- `header.cljs` still shows the `FS Cloud` state in the center header and still uses a UUID-style window token rather than the sequential user-facing numbering requested later in the ticket
- `meta_editor.cljs` is real, but the runtime/project-copy behavior still needs deeper alignment with the ticket language

## Docs Link Fix

The worktree `docs` symlink was broken before this cleanup.

- Broken target: `../little-trader-docs-and-learn/docs`
- Fixed target: `../../../../4coders/little-trader-docs-and-learn/docs`

This keeps the worktree pointed at the shared docs repo from the Codex worktree layout.

## Deferred / TBD

- Confirm whether any non-UI runtime or re-frame state left behind from the removed pages should also be pruned in a later cleanup pass
- Continue with the shell/header adjustments after this baseline removal commit lands
