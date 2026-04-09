# Issue 6: Cleanup — Remove Dead Code and Debug Logging

**Labels**: `chore`, `copilot-agents`
**Priority**: P3 — Housekeeping before merge
**Depends on**: Issues 1–5 (do this last)

---

## Summary

Remove dead code and debug logging introduced during prototyping. This is a pure cleanup — no behavior changes.

## What to Remove

### Debug Logging

**`AgentTaskViewModelBuilder.swift`** — Remove all `#if DEBUG` print statements in `groupToolCallBursts()`:
- `[WorkBurst] Input items (N): ...`
- `[WorkBurst] Flushing burst of N tool calls, M pending msgs`

These were useful during development but shouldn't ship.

### Dead Code

| File | What to Remove | Why It's Dead |
|------|---------------|---------------|
| `MessageEventCell.swift` | `OutcomeMessageCell` class | Was created for promoting outcome outside cards — approach was abandoned in favor of flat timeline |
| `AgentTaskViewModel.swift` | `finalOutcome` computed property | Same reason — no longer extracting final outcome separately |
| `AgentTaskViewModel.swift` | `itemsWithoutFinalOutcome` computed property | Same reason |
| `AgentTaskSessionSection.swift` | `AgentSessionHeaderCell` class (~line 261+) | Session header was removed in Issue 1 |
| `ToolCallEventViewModel.swift` | `init(fromGroup:)` initializer | Replaced by `workBurstFrom` initializer |
| `ToolCallEventViewModel.swift` | `groupHeaderFor` static method (if present) | Same reason |

### Verify Before Removing

Before deleting each item, do a quick search to confirm it's not referenced:

```bash
grep -r "OutcomeMessageCell" Modules/GitHub/Sources/
grep -r "finalOutcome" Modules/GitHub/Sources/
grep -r "itemsWithoutFinalOutcome" Modules/GitHub/Sources/
grep -r "AgentSessionHeaderCell" Modules/GitHub/Sources/
grep -r "fromGroup" Modules/GitHub/Sources/Copilot/Agents/ToolCallEventViewModel.swift
```

If any are still referenced from Xcode project files (`.pbxproj`), update those too.

## How to Test

1. Build the project — `xcodebuild -workspace GitHub.xcworkspace -scheme GitHub -destination 'platform=iOS Simulator,name=iPhone 17 Pro' build`
2. ✅ No build errors
3. ✅ No warnings about unused code
4. ✅ Run the app — session timeline works exactly as before cleanup

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskViewModelBuilder.swift`
- `Modules/GitHub/Sources/Copilot/Agents/MessageEventCell.swift`
- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskViewModel.swift`
- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskSessionSection.swift`
- `Modules/GitHub/Sources/Copilot/Agents/ToolCallEventViewModel.swift`

## Estimated Scope

~80 lines removed, 0 lines added
