# Issue 4: Burst Polish — Duration, Auto-Expand, No Subtitles, Queued Label

**Labels**: `enhancement`, `copilot-agents`, `ui`
**Priority**: P2 — Refinements that improve burst UX
**Depends on**: Issue 3

---

## Summary

Three small refinements to the work burst feature:

1. **Duration display** on burst headers ("12s", "2m 30s")
2. **Auto-expand the last burst** in a session
3. **Remove subtitles from individual tool calls** (grep pattern, search query, etc.)

---

## 4a: Duration Display

Show the elapsed time of a burst as a subtitle on the header row, next to the chevron.

### What to Change

In `ToolCallEventViewModel.workBurstFrom`:

```swift
private static func formatBurstDuration(from children: [ToolCallEventViewModel]) -> String? {
    guard let first = children.first?.timestamp,
          let last = children.last?.timestamp else { return nil }
    let seconds = Int(last.timeIntervalSince(first))
    guard seconds > 0 else { return nil }
    if seconds < 60 { return "\(seconds)s" }
    let minutes = seconds / 60
    let remainingSeconds = seconds % 60
    if remainingSeconds == 0 { return "\(minutes)m" }
    return "\(minutes)m \(remainingSeconds)s"
}
```

Set `self.subtitle = formatBurstDuration(from: children)` in the burst header initializer.

### How to Test

1. Open a session with work bursts
2. ✅ Burst headers show duration (e.g., "Used 7 tools · 12s")
3. ✅ Very fast bursts (<1s) show no duration

---

## 4b: Auto-Expand Last Burst

The last work burst in a session should always be expanded. This ensures:
- **In-progress sessions**: users see what the agent is currently doing
- **Completed sessions ending with a burst**: users see the final work (not hidden behind a collapsed row)

### What to Change

In `makeSessionItemsAndMetadata()`, after `groupToolCallBursts()`:

```swift
if let lastIndex = items.indices.last,
   case .toolCall(var vm) = items[lastIndex],
   vm.groupChildren != nil {
    vm.isExpanded = true
    items[lastIndex] = .toolCall(vm)
}
```

### How to Test

1. Open a session where the last items are tool calls (no trailing message)
2. ✅ The final burst is expanded, showing its children
3. ✅ Earlier bursts are collapsed
4. ✅ User can still manually collapse the last burst

---

## 4c: Remove Subtitles From Individual Tool Calls

Individual tool calls were showing contextual subtitles (grep patterns, search queries, URLs) next to the chevron. In the flat timeline these are visual noise — the subtitle space should be reserved for burst headers (duration).

### What to Change

In `ToolCallEventViewModel.init(from toolCall:)`, set `self.subtitle = nil` for these tool names:
- `grep` (was showing `pattern`)
- `glob` (was showing `pattern`)
- `web_fetch` (was showing `url`)
- `web_search` (was showing `query`)
- `ask_user` (was showing `question`)
- `create_pull_request` / `submit-pr` (was showing `title`)

**Do not change**: Background agent tool calls keep their subtitles (tool call count, status).

### How to Test

1. Open a session with grep/search tool calls
2. ✅ Tool call rows show icon + title + chevron only — no subtitle text
3. ✅ Burst headers still show duration subtitle
4. ✅ Background agent rows still show their status subtitles

---

## 4d: Move "Queued" Label Below User Bubble

When a user sends a message while the agent is busy, it shows as a bubble with a "Queued" label. Currently the label sits above the bubble — move it below so the bubble leads visually and the status feels like a footnote.

### What to Change

In `InlineUserMessageCell.swift`, in `AgentSessionInlineUserMessageView.body`, move the `if viewModel.isQueued` block from before the `Text(viewModel.content)` bubble to after it:

```swift
VStack(alignment: .trailing, spacing: grid(1)) {
    Text(viewModel.content)           // bubble first
        .font(.body)
        .padding(...)
        .background(bubbleShape)
    if viewModel.isQueued {            // label underneath
        Text(L10n.labelQueued)
            .font(.caption)
            .foregroundStyle(Asset.textSecondary.swiftUIColor)
    }
}
```

### How to Test

1. Send a message while an agent session is in progress
2. ✅ "Queued" label appears below the message bubble, right-aligned
3. ✅ Label disappears once the server confirms the message

---

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/ToolCallEventViewModel.swift`
- `Modules/GitHub/Sources/Copilot/Agents/AgentTaskViewModelBuilder.swift`
- `Modules/GitHub/Sources/Copilot/Agents/InlineUserMessageCell.swift`

## Estimated Scope

~30 lines changed
