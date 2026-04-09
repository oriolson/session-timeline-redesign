# Session Timeline Redesign — Implementation Sequence

## Overview

6 issues, ordered by dependency. Each is independently shippable and testable.

```
Issue 1: Flat Timeline                    ← FOUNDATION
    │
    ▼
Issue 2: Remove Card Styling             ← VISUAL FOUNDATION
    │
    ├──────────────────┐
    ▼                  ▼
Issue 3: Work Burst    Issue 5: Working
Grouping               Indicator
    │
    ▼
Issue 4: Burst Polish
(duration, auto-expand,
 no subtitles)
    │
    ▼
Issue 6: Cleanup                          ← LAST
```

## Issue Summary

| # | Title | Priority | Depends On | Lines Changed | Files |
|---|-------|----------|------------|---------------|-------|
| 1 | Flat Timeline — Remove session header & expansion gate | P1 | — | ~20 removed | 1 |
| 2 | Remove Card Styling & Set Consistent Margins | P1 | #1 | ~15 changed | 9 |
| 3 | Work Burst Grouping — Collapse consecutive tool calls | P1 | #1, #2 | ~100 added | 3 |
| 4 | Burst Polish — Duration, auto-expand, no subtitles | P2 | #3 | ~25 changed | 2 |
| 5 | Working Indicator — Pulsing dot + shimmer text | P2 | #2 | ~60 changed | 1 |
| 6 | Cleanup — Remove dead code & debug logging | P3 | #1–#5 | ~80 removed | 5 |

**Total**: ~300 lines across 11 files in `Modules/GitHub/Sources/Copilot/Agents/`

## Merge Strategy

**Recommended**: Ship issues 1–4 together as one PR (they're tightly coupled and the flat timeline without burst grouping is noisy). Issue 5 can be a separate PR. Issue 6 is a final cleanup PR.

**Alternative**: Ship each issue as its own PR for smaller reviews. Issues 1 and 2 should merge together at minimum.

## Branch

All work is prototyped on `ui-iterations`. Engineers should branch from `main` and re-implement using these issues as specs — the prototype branch has debug logging and dead code that shouldn't carry over.

## Key Architecture Notes

- `grid(n)` = `n × 4pt`, `halfGrid` = 2pt (from `GitHubUI/Grid.swift`)
- `BaseCardCell` is the parent class for all content cells
- `AgentSessionSection` builds `[ViewModelBinder]` which the `Adapter` renders into `UITableView` rows
- The event processing pipeline is: raw events → `makeSessionItemsAndMetadata()` → `[AgentSessionCellViewModel]` → `groupToolCallBursts()` (post-processing) → section builds `ViewModelBinder` array
- `UIHostingConfiguration` wraps SwiftUI views in UIKit cells — `.margins()` controls cell-level spacing
