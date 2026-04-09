# Issue 3: Work Burst Grouping — Collapse Consecutive Tool Calls

**Labels**: `enhancement`, `copilot-agents`, `ui`
**Priority**: P1 — Core noise reduction feature
**Depends on**: Issue 1, Issue 2

---

## Summary

Group 3 or more consecutive tool calls into a single collapsible "Used N tools" row. This is the main noise reduction mechanism — it turns a 30-tool-call session into a handful of burst rows interspersed with messages.

## Mental Model

An agent works in **bursts**: it reasons briefly, calls several tools in quick succession, then writes a response. Our grouping reflects this rhythm.

```
👤 "Fix the auth bug"
🔧 Search files                     ← only 2 tools, stays flat
🔧 View auth.swift
🤖 "I found the issue. Let me fix it."
   ▸ Used 7 tools · 12s             ← 7 consecutive tools → collapsed burst
🤖 "Done! I've updated auth.swift to..."
```

## What to Change

### `AgentTaskViewModelBuilder.swift`

Add a `groupToolCallBursts(_:)` function as a **post-processing step** that runs on the final `[AgentSessionCellViewModel]` array, AFTER all items are built.

**Why post-processing?** The event loop's `flushTools()` creates small groups (1–2 tools) because assistant messages between tool calls trigger flushes. Grouping must happen on the final array where we can see the full sequence.

#### Algorithm

```
Input:  [item1, item2, item3, ...]
Output: [item1, burstGroup, item5, ...]

Scan left to right:
- .toolCall → add to currentBurst
- .message(assistant) while burst is open → hold as pendingMessage
  - If next item is a tool call → convert pending to thinking item, add to burst
  - If next item is something else → flush burst, emit pending as regular messages
- .message(user) → flush burst, emit message
- .permissionRequest / .prompt / other → flush burst, emit item
- "task" tool calls (background agents) → flush burst, emit standalone

When flushing:
- ≤2 tool calls → emit individually (not worth grouping)
- 3+ tool calls → create burst header with children

Absorb intermediate assistant messages into bursts:
- All pending assistant messages except the last one with content → convert to thinking items inside the burst
- The last assistant message with content → keep as regular message (it's the response)
```

#### Burst Header VM

Add to `ToolCallEventViewModel`:

```swift
init(workBurstFrom children: [ToolCallEventViewModel]) {
    self.id = "burst-\(children.first?.id ?? "")"
    self.toolName = "burst"
    self.icon = Asset.tools16.swiftUIImage
    self.iconColor = Asset.iconSecondary.swiftUIColor
    self.title = "Used \(children.count) tools"
    self.subtitle = formatDuration(from: children)  // "12s", "2m", "1m 30s"
    self.status = .succeeded
    self.showChevron = true
    self.isExpanded = false
    self.groupChildren = children
}
```

#### Thinking Item VM

For intermediate assistant messages absorbed into bursts:

```swift
static func thinkingItem(from message: MessageEventViewModel) -> ToolCallEventViewModel {
    let text = message.reasoning.isEmpty ? message.content : message.reasoning
    let preview = String(text.prefix(60)).trimmingCharacters(in: .whitespacesAndNewlines)
    return ToolCallEventViewModel(
        id: "thinking-\(message.id)",
        toolName: "thinking",
        icon: Asset.copilot16.swiftUIImage,       // Copilot icon
        iconColor: Asset.iconPrimary.swiftUIColor,
        title: preview.isEmpty ? "Thinking…" : preview + "…",
        status: .succeeded,
        showChevron: !text.isEmpty,
        outputContent: text.isEmpty ? nil : text   // tap to see full reasoning
    )
}
```

### `AgentTaskSessionSection.swift`

In `makeToolCallViewModels(for:)`, render expanded burst children using `ToolCallEventCell` (not `ToolCallEventGroupItemCell`). This shows each child with its proper icon and title.

### Call Site

In `makeSessionItemsAndMetadata()`, after `flushTools()` and before returning:

```swift
items = groupToolCallBursts(items)
```

## How to Test

1. Open a session with many tool calls (10+)
2. ✅ Consecutive tool calls are grouped into "Used N tools" rows
3. ✅ Bursts of ≤2 tools stay flat (not grouped)
4. ✅ Tapping a burst row expands it showing all child tool calls with icons
5. ✅ Tapping again collapses it
6. ✅ User messages break bursts (not absorbed)
7. ✅ Background agent ("task") tool calls are NOT grouped into bursts
8. ✅ Permission and prompt cards are NOT grouped into bursts
9. ✅ Tapping a child tool call inside an expanded burst shows its output

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskViewModelBuilder.swift`
- `Modules/GitHub/Sources/Copilot/Agents/ToolCallEventViewModel.swift`
- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskSessionSection.swift`

## Estimated Scope

~100 lines added (grouping function + VM initializers)
