# Issue 2: Remove Card Styling and Set Consistent Margins

**Labels**: `enhancement`, `copilot-agents`, `ui`
**Priority**: P1 — Visual foundation
**Depends on**: Issue 1

---

## Summary

Remove card backgrounds from all agent session cells and set consistent 16pt horizontal margins. The timeline should feel clean and edge-to-edge, not like stacked cards.

## What to Change

### `BaseCardCell.swift`

1. Set `cardView.backgroundColor = .clear` (line ~23)
2. Change card constraints from inset to edge-to-edge:
   ```swift
   // Before:
   make.leading.trailing.equalToSuperview().inset(grid(4))
   // After:
   make.leading.trailing.equalToSuperview()
   ```

### All cell files — set horizontal margins to `grid(4)` (16pt)

Each cell's `bind()` method sets margins via `UIHostingConfiguration.margins()`. Update the horizontal value to `grid(4)` in:

| File | Current | New |
|------|---------|-----|
| `MessageEventCell.swift` | `grid(4)` | `grid(4)` ✅ already correct |
| `ToolCallEventCell.swift` | varies | `.margins(.horizontal, grid(4))` |
| `ToolCallEventGroupItemCell.swift` | varies | `.margins(.horizontal, grid(4))` |
| `PermissionRequestCell.swift` | varies | `.margins(.horizontal, grid(4))` |
| `PromptEventCell.swift` | varies | `.margins(.horizontal, grid(4))` |
| `WorkingIndicatorView.swift` | varies | `.margins(.horizontal, grid(4))` |
| `SessionUserMessageCell.swift` | check | `.margins(.horizontal, grid(4))` |
| `InlineUserMessageCell.swift` | check | `.margins(.horizontal, grid(4))` |

### Message cell top margin

Set `MessageEventCell` top margin to `0` (was `grid(2)`). The `MessageEventView` already has internal `.padding(.vertical, grid(2))` which provides sufficient spacing.

```swift
// Before:
.margins(.top, grid(2))
// After:
.margins(.top, 0)
```

## How to Test

1. Open any agent session
2. ✅ No card backgrounds visible (cells should be transparent)
3. ✅ Content has 16pt margins on both sides
4. ✅ Text and icons are properly aligned across all cell types
5. ✅ Content doesn't touch screen edges

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/BaseCardCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/MessageEventCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/ToolCallEventCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/ToolCallEventGroupItemCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/PermissionRequestCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/PromptEventCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/WorkingIndicatorView.swift`
- `Modules/GitHub/Sources/Copilot/Agents/SessionUserMessageCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/InlineUserMessageCell.swift`

## Estimated Scope

~1–2 lines changed per file, ~15 lines total
