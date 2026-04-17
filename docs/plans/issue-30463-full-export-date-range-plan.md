# Issue #30463 Plan: Full Export Date Range

Issue: https://github.com/telegramdesktop/tdesktop/issues/30463

## Goal

Allow the "Export my data" flow to apply a date window across the full export, not only when exporting a single chat. This does not introduce a true stateful delta export. It exposes a full-export date filter that lets users emulate incremental backups by exporting only recent history.

## Current State

The export pipeline already has date-range support, but it is scoped to single-peer export.

- `Telegram/SourceFiles/export/export_settings.h`
  - `Settings` already stores `singlePeerFrom` and `singlePeerTill`.
- `Telegram/SourceFiles/export/view/export_view_settings.cpp`
  - `SettingsWidget::setupPathAndFormat()` only shows `addLimitsLabel()` when `_singlePeerId != 0`.
- `Telegram/SourceFiles/export/view/export_view_panel_controller.cpp`
  - `ResolveSettings()` clears both date fields when `!settings.onlySinglePeer()`.
- `Telegram/SourceFiles/export/data/export_data_types.cpp`
  - `SkipMessageByDate()` already filters exported messages by the stored range.
- `Telegram/SourceFiles/export/export_api_wrap.cpp`
  - split counting and range boundary checks already use the stored date fields.
- `Telegram/SourceFiles/export/export_controller.cpp`
  - `NormalizeSettings()` special-cases single-peer export but does not otherwise block the filter.

The key point is simple: the core filtering machinery already exists. Full export currently disables it at the settings/UI layer.

## Recommended Scope

Implement a full-export date window by generalizing the existing range fields instead of inventing a second filtering path.

The least risky MVP is:

- Keep the current persisted fields and validation rules.
- Show the existing date-window UI for full export as well.
- Stop zeroing the fields for non-single-peer exports.
- Let the existing filtering and split-counting code run unchanged for both single-peer and full export.

This avoids a broad rename or settings migration in the first pass.

## Non-Goals

- No stateful "export only what changed since last export" tracker.
- No deduplication across previous exports.
- No manifest reconciliation with old export folders.
- No output-format redesign.

The feature should be framed as "date-range export for full history," which users can use to emulate delta backups externally.

## Implementation Plan

### 1. Generalize the settings semantics

Treat `singlePeerFrom` and `singlePeerTill` as the export window for any export mode, not only single-peer export.

Work items:

- Keep `Settings::validate()` as-is unless a later change requires different bounds handling.
- Remove the full-export reset in `ResolveSettings()`.
- Audit any assumptions that a non-zero range implies `onlySinglePeer()`.

Files:

- `Telegram/SourceFiles/export/export_settings.h`
- `Telegram/SourceFiles/export/export_settings.cpp`
- `Telegram/SourceFiles/export/view/export_view_panel_controller.cpp`

### 2. Expose the date-range controls in the full-export UI

Reuse the existing limits editor instead of building a second widget.

Work items:

- Update `SettingsWidget::setupPathAndFormat()` so the full-export flow also shows `addLimitsLabel(container)`.
- Keep the single-chat UX intact.
- Confirm the full-export layout still fits after adding the extra row.
- Verify `requiredRows` in `export_view_content.cpp` if the settings/progress view assumes a fixed row count that was sized around the old layout.

Files:

- `Telegram/SourceFiles/export/view/export_view_settings.cpp`
- `Telegram/SourceFiles/export/view/export_view_settings.h`
- `Telegram/SourceFiles/export/view/export_view_content.cpp`

### 3. Preserve the existing data-plane filtering path

Do not fork the export logic if the current shared range path already works.

Work items:

- Verify that `SkipMessageByDate()` is applied in both JSON and HTML export paths for dialogs in full export.
- Verify that the message-count preflight in `export_api_wrap.cpp` still behaves correctly when the range is set during multi-chat export.
- Confirm that empty-in-range dialogs are either skipped cleanly or exported with the current intended minimal structure.

Files:

- `Telegram/SourceFiles/export/data/export_data_types.cpp`
- `Telegram/SourceFiles/export/export_api_wrap.cpp`
- `Telegram/SourceFiles/export/output/export_output_json.cpp`
- `Telegram/SourceFiles/export/output/export_output_html.cpp`

### 4. Decide whether to rename the settings fields

This should be treated as a follow-up decision, not required for the MVP.

Options:

- MVP-friendly: keep `singlePeerFrom` / `singlePeerTill` and only broaden semantics internally.
- Cleanup-heavy: rename them to generic names like `timeFrom` / `timeTill`.

Recommendation:

- Do not rename in the first implementation.
- If maintainers want cleanup later, do it as a separate patch with careful persistence compatibility review.

Reason:

- The current fields are already persisted and already threaded through the export stack.
- Renaming first increases churn without changing behavior.

## Validation Plan

### Manual checks

- Full export, no range set:
  - behavior must remain unchanged.
- Full export, `from` set:
  - only recent messages across all selected chats should be exported.
- Full export, `till` set:
  - only older history before the cutoff should be exported.
- Full export, both set:
  - only messages within the bounded window should be exported.
- Single-chat export with range:
  - behavior must remain unchanged.
- Empty result window:
  - export should complete cleanly without broken HTML/JSON output.

### Edge cases

- Boundary timestamps at midnight and with time edits.
- Chats with media but no messages inside the selected window.
- Very large multi-chat exports where the range excludes most splits.
- Topic export, because it already builds on the single-peer path.

## Risks

- There may be implicit assumptions in progress/state UI that a date range only appears for single-peer export.
- Some output code may expect "full export" to always include all messages for included chats.
- Split preflight logic could expose edge cases where a dialog has zero in-range messages but still creates partial scaffolding.

None of these look architectural. They look like integration cleanup.

## Suggested Delivery Order

1. UI exposure for the full-export limits.
2. Remove the full-export date reset.
3. Run manual exports against small and medium accounts.
4. Fix any empty-range or progress-state regressions.
5. Only then consider naming cleanup.

## Acceptance Criteria

- The full "Export my data" flow exposes the same date window controls already available for single-chat export.
- The selected date window affects exported messages across all selected chats.
- Existing single-chat export behavior is unchanged.
- Full export with no range remains behaviorally identical to current builds.
