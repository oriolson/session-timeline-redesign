# Issue 1: Flat Timeline — Remove Session Header and Expansion Gate

**Labels**: `enhancement`, `copilot-agents`, `ui`
**Priority**: P1 — Foundation for all subsequent changes
**Depends on**: Nothing

---

## Summary

Remove the session header cell and the `isExpanded` gate from `AgentSessionSection`. All session items should render chronologically in a flat timeline — no expand/collapse at the session level.

## Context

Currently, each session renders a header (`AgentSessionHeaderCell`) that the user taps to expand/collapse. When collapsed, ALL content (messages, tool calls, permissions) is hidden. This creates an all-or-nothing view that hides important context.

The new approach renders everything flat. Noise reduction is handled separately by work burst grouping (Issue 3).

## What to Change

### `AgentTaskSessionSection.swift`

In the `viewModels` lazy var (~line 63):

1. **Remove** the `AgentSessionHeaderCell` rendering (the session header row)
2. **Remove** the `guard viewModel.isExpanded else { return models }` gate
3. Keep the `userMessage` rendering at the top (this already renders outside the gate)
4. Keep all item rendering logic (the `for item in sessionViewModel.items` loop) — just remove the expansion guard so items always render

**Before** (simplified):
```swift
models.append(sessionHeader)     // ← remove
guard isExpanded else { return } // ← remove
for item in items { ... }        // ← keep (always renders now)
```

**After**:
```swift
if let userMessage = sessionViewModel.userMessage {
    models.append(userMessageCell)
}
// All items render unconditionally
for item in sessionViewModel.items { ... }
```

### `AgentTaskViewController.swift`

In `sections` (~line 322), the `onSelectHeader` closure can be left in place for now (it becomes a no-op since there's no header to tap). Clean removal is in Issue 7.

## How to Test

1. Open a task with a completed session
2. ✅ All messages and tool calls are visible immediately — no header to tap
3. ✅ User message still renders at the top
4. ✅ Working indicator still shows for in-progress sessions
5. Open a task with multiple sessions
6. ✅ All sessions render their full content inline (may be long — that's expected, Issue 3 will compress tool calls)

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskSessionSection.swift`

## Estimated Scope

~20 lines removed, ~0 lines added
