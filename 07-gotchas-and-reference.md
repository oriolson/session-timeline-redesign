# Gotchas & Lessons From the Prototype

**Branch**: `ui-iterations`
**Purpose**: Save engineers time by documenting every non-obvious bug and dead end hit during prototyping.

---

## Bug 1: Grouping in `flushTools()` doesn't work — use post-processing

**What we tried**: Grouping tool calls inside `flushTools()` in the event loop.

**What went wrong**: Assistant messages between tool calls trigger `flushTools()`, so each flush only contains 1–2 tool calls. You never get groups of 3+.

**Fix**: `groupToolCallBursts()` runs as a **post-processing step** on the final `[AgentSessionCellViewModel]` array, after all items are built. This is the only place where you can see the full sequence of consecutive tool calls.

**Key insight**: The event loop's `flushTools()` should stay simple — just pair start/complete events into `ToolCall` objects. All grouping logic belongs in the post-processing step.

---

## Bug 2: All bursts auto-expand (not just the last one)

**What we tried**: `isExpanded = status == .running` on burst headers.

**What went wrong**: Some tool calls in completed sessions never received completion events, so their status stayed `.running`. This caused ALL bursts to expand, defeating the purpose.

**Fix**: Default all bursts to `isExpanded = false`. Then, as a separate step after grouping, set `isExpanded = true` only on the very last item if it's a burst:

```swift
if let lastIndex = items.indices.last,
   case .toolCall(var vm) = items[lastIndex],
   vm.groupChildren != nil {
    vm.isExpanded = true
    items[lastIndex] = .toolCall(vm)
}
```

---

## Bug 3: Final assistant message gets swallowed by the burst

**What we tried**: `flushBurst()` that creates the burst group and emits pending messages.

**What went wrong**: When the burst was the last thing in the array, `pendingMessages` (which contained the final assistant response) were silently dropped in an early version.

**Fix**: `flushBurst()` must ALWAYS emit `pendingMessages` after the burst group — they're the trailing response, not part of the group. Also, intermediate assistant messages (thinking/reasoning) need to be absorbed INTO the burst, while the last assistant message with content stays outside. See the `lastAssistantIndex` logic in the final implementation.

---

## Bug 4: Expanded children render as mono text, not icon rows

**What we tried**: Used `ToolCallEventGroupItemCell` for expanded burst children.

**What went wrong**: That cell renders tool calls as plain monospace text (the old grouped-by-name style). Users expected to see the same icon + title format as standalone tool calls.

**Fix**: Use `ToolCallEventCell` for expanded children — same cell type as standalone tool calls. This gives each child its proper icon, title, and tap-to-view-output behavior.

---

## Bug 5: Thinking messages leak outside bursts

**What we tried**: Only absorbing messages with `reasoning` field and empty `content`.

**What went wrong**: Some assistant "thinking" messages put their text in `content` rather than `reasoning`. The narrow check missed them.

**Fix**: In `flushBurst()`, absorb ALL pending assistant messages into the burst as thinking items, EXCEPT the last one with content (which is the actual response). Use index-based detection:

```swift
let lastAssistantIndex = pendingMessages.lastIndex { item in
    if case .message(let msg) = item, msg.role == .assistant, !msg.content.isEmpty { return true }
    return false
}
// Everything before lastAssistantIndex → thinking item inside burst
// lastAssistantIndex → regular message outside burst
```

---

## Architecture Cheat Sheet

```
Raw events
    │
    ▼
makeSessionItemsAndMetadata()     ← event loop, pairs tool start/complete
    │
    ▼
[AgentSessionCellViewModel]       ← flat array: messages, tool calls, permissions
    │
    ▼
groupToolCallBursts()             ← POST-PROCESSING: groups 3+ consecutive tools
    │
    ▼
auto-expand last burst            ← one-liner after grouping
    │
    ▼
AgentSessionSection.viewModels    ← builds ViewModelBinder array for UITableView
    │
    ▼
makeToolCallViewModels(for:)      ← handles burst headers + expanded children
    │
    ▼
UITableView via Adapter           ← renders cells
```

### Key Types

| Type | Role |
|------|------|
| `AgentSessionCellViewModel` | Enum: `.toolCall`, `.message`, `.permissionRequest`, `.prompt`, `.workingIndicator` |
| `ToolCallEventViewModel` | Covers both individual tools AND burst headers (check `groupChildren != nil`) |
| `BaseCardCell.Position` | `.top`, `.middle`, `.bottom`, `.standalone` — controls corner radius and bottom spacing |
| `grid(n)` | `n × 4pt`. `halfGrid` = 2pt. From `GitHubUI/Grid.swift` |

### Spacing Reference (final values)

| Element | Horizontal | Top | Bottom |
|---------|-----------|-----|--------|
| Tool call rows | `grid(4)` = 16pt | 0 | 0 |
| Message cells | `grid(4)` = 16pt | 0 | 0 (or `grid(6)` if last) |
| Inline user messages | `grid(4)` = 16pt | `grid(6)` = 24pt | `grid(4)` = 16pt |
| Session user messages | `grid(4)` = 16pt | `grid(4)` = 16pt | `grid(4)` = 16pt |
| Working indicator | `grid(4)` = 16pt | 0 | `grid(6)` = 24pt |
| HStack internal (tool rows) | — | — | spacing: `grid(2)` = 8pt |
| Tool call icon container | 16×16pt | — | — |
| Pulsing dot | 8pt in 16×16 container | — | — |
