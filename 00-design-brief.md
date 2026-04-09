# Copilot Session Timeline Redesign — Design Brief

**Platform**: iOS
**Branch**: `ui-iterations`
**Scope**: `Modules/GitHub/Sources/Copilot/Agents/`

---

## TL;DR

On iOS, Copilot agent sessions live inside expandable cards — collapse it and you lose everything, expand it and you get a wall of tool calls with the actual answer buried at the bottom. We asked ourselves: if we're going to show everything anyway, why have a card at all? So we removed it. Sessions now render as a flat conversation timeline — user message, agent response, done. The dozens of tool calls in between? They collapse into compact "Used N tools" rows you can expand if you're curious. The result is a view that reads like a conversation instead of a debug log.

---

## Why We Made These Changes

### The Problem

When a user opens a Copilot agent session on iOS, they're presented with a flat list of *every* event — every tool call, every intermediate reasoning message, every file read — interleaved with the content they actually care about: what they asked and what the agent said back.

On a long session with 30+ tool calls, this means:
- The final answer is buried under dozens of tool call rows
- There's no visual distinction between "what happened" and "what matters"
- Scanning a completed session is slow and noisy

Meanwhile, Web (github-app) groups tool calls into collapsible accordions by intent. Android groups same-name tool calls into timeline cards. iOS was the only platform rendering everything completely flat.

### The Mental Model

We landed on a simple concept: **work bursts**.

An agent works in bursts — it thinks, calls several tools in quick succession, then responds. Our UI should reflect that rhythm:

```
👤 User message
   ▸ Used 7 tools · 12s        ← collapsed burst (tap to expand)
🤖 Agent response
   ▸ Used 4 tools · 8s         ← another burst
🤖 Final answer
```

A **work burst** is any run of 3+ consecutive tool calls. Shorter runs (1–2 tools) stay visible individually since they don't create noise. The burst collapses behind a single "Used N tools" row with a duration badge and a chevron to expand.

### What We Changed (Summary)

| Before | After |
|--------|-------|
| Session header + expand/collapse gate | Flat timeline — no gate, all items visible |
| Card backgrounds on every cell | No card backgrounds — clean, edge-to-edge |
| Every tool call rendered individually | 3+ consecutive tool calls grouped into "Used N tools" row |
| Intermediate assistant reasoning shown as full messages | Absorbed into bursts as compact "thinking" rows |
| Static "Working…" with rotating commit square | *(Planned)* Pulsing dot + shimmer text |

---

## Design Decisions

### 1. Flat Timeline (No Session Header)

**Decision**: Remove the session header and expansion gate entirely. All items render chronologically.

**Why**: The expand/collapse gate created an all-or-nothing view — either you see everything or nothing. With work bursts handling the noise reduction, there's no need for a second layer of collapse at the session level.

**Pros**:
- Simpler mental model — what you see is what happened
- Matches Web's flat conversation approach
- No "black box" sessions where outcomes are hidden

**Cons**:
- Multi-session tasks show ALL sessions inline (no quick collapse)
- Longer scroll for tasks with many sessions

**Mitigated by**: Work burst grouping compresses the longest sections (tool calls) so overall timeline length is manageable.

### 2. Work Burst Grouping (Not Tool-Name Grouping)

**Decision**: Group ALL consecutive tool calls together regardless of type, rather than grouping by tool name (like Android) or by file path (like Web).

**Why**: We tried tool-name grouping first. It broke the reading flow because the same tool (e.g., "View") appears at different points in the agent's reasoning and serves different purposes each time. Grouping by name felt arbitrary. Grouping by *time* (consecutive runs) matches how the agent actually works.

**Pros**:
- Reflects the agent's actual work rhythm
- Simpler implementation — no need to categorize tools
- Reduces a 30-tool session to ~3-4 burst rows

**Cons**:
- Loses per-tool-type summary (can't see "5 edits, 3 searches" at a glance)
- All tool calls treated equally inside the burst

**Mitigated by**: Expanded burst shows each tool call with its icon and name, so the detail is one tap away.

### 3. No Card Styling

**Decision**: Remove card backgrounds from all cells. Content renders edge-to-edge with 16pt side margins.

**Why**: Cards added visual weight without adding information. In a flat timeline where content flows chronologically, card boundaries were noise — they didn't map to meaningful groups.

**Pros**:
- Cleaner, lighter visual weight
- More horizontal space for content
- Consistent with modern mobile chat UIs

**Cons**:
- Less visual separation between items
- Harder to distinguish item boundaries at a glance

**Mitigated by**: Icon + text style differences between cell types provide sufficient visual distinction.

### 4. Thinking Messages Absorbed Into Bursts

**Decision**: When the agent sends a reasoning/thinking message between tool calls, it's shown as a compact row *inside* the burst (with a Copilot icon) rather than as a full message bubble.

**Why**: These intermediate messages ("Let me check the test file…") are context for the tool calls, not content the user needs to read at the top level. Showing them as full messages breaks the signal-to-noise ratio.

**Pros**:
- Keeps the top-level timeline focused on user ↔ agent conversation
- Thinking context is preserved and accessible inside the expanded burst

**Cons**:
- Users might miss important reasoning if they don't expand bursts
- Relies on correct classification of "thinking" vs "response"

**Mitigated by**: The last assistant message with content always stays outside the burst as the response. Only intermediate messages get absorbed.

### 5. Auto-Expand Last Burst

**Decision**: The last work burst in a session is always expanded.

**Why**: If a session ends with a burst (no trailing message), collapsing it would hide the last thing the agent did. For in-progress sessions, the expanded burst shows live progress.

**Pros**:
- Users always see the most recent work
- Natural "what just happened?" entry point

**Cons**:
- Can be visually heavy if the last burst has many items

---

## What's Not Changing

- **Task header** — still shows at the top
- **User message bubble** — still renders above session content
- **Permission cards** — pending permissions still render as interactive cards; resolved ones are absorbed into bursts
- **Prompt cards** — question/plan-approval prompts still render as interactive cards
- **Chat input bar** — unchanged
- **Auto-scroll behavior** — still scrolls to bottom on initial load and during streaming

---

## Alignment With Other Platforms

| Concept | iOS (New) | Android | Web |
|---------|-----------|---------|-----|
| Tool grouping | Work bursts (consecutive runs) | Same-name tool groups | File-path accordions |
| Collapse behavior | Bursts collapse; timeline is flat | Tool groups collapse | Intent groups collapse |
| Thinking/reasoning | Inside bursts as compact rows | Not shown separately | Not shown separately |
| Working indicator | *(Planned)* Pulse dot + shimmer | Spinner | ASCII dot animation + timer |

iOS is now closer to Web's flat timeline approach while using a mobile-appropriate grouping strategy. Android alignment is a separate effort.
