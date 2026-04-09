# Issue 5: Working Indicator — Pulsing Dot + Shimmer Text

**Labels**: `enhancement`, `copilot-agents`, `ui`
**Priority**: P2 — Visual polish
**Depends on**: Issue 2 (for consistent margins/spacing)

---

## Summary

Replace the current "Working…" indicator (rotating green commit square + static text) with a pulsing dot and shimmering text. The text style and spacing should match tool call rows for visual consistency.

## Current State

```swift
// WorkingIndicatorView.swift
HStack(spacing: grid(2)) {
    CommitSquare()                              // rotating green square
        .frame(width: grid(3), height: grid(3))
    Text("Working…")
        .font(.footnote.weight(.medium))        // footnote = small
        .foregroundStyle(Asset.textSecondary)
}
.padding(.horizontal, grid(3))
.padding(.vertical, grid(3))
```

## New Design

```swift
HStack(spacing: grid(2)) {
    PulsingDot()                                // 8pt pulsing circle
    Text("Working…")
        .font(.subheadline)                     // match tool call rows
        .shimmerEffect()                        // animated highlight sweep
        .foregroundStyle(Asset.textSecondary)
}
.padding(.leading, halfGrid)                    // match ToolCallEventView
```

### Pulsing Dot

A small circle (8pt / `grid(2)`) with a repeating scale+fade animation. Matches the desktop app's `PulseIndicator` component.

```swift
struct PulsingDot: View {
    @State private var isPulsing = false

    var body: some View {
        ZStack {
            Circle()
                .fill(Color.accentColor.opacity(0.3))
                .scaleEffect(isPulsing ? 2.0 : 1.0)
                .opacity(isPulsing ? 0 : 0.75)
            Circle()
                .fill(Color.accentColor)
        }
        .frame(width: grid(2), height: grid(2))
        .onAppear {
            withAnimation(.easeOut(duration: 1.0).repeatForever(autoreverses: false)) {
                isPulsing = true
            }
        }
    }
}
```

### Text Shimmer

A highlight that sweeps left-to-right across the text. Implement as a gradient mask animation.

```swift
struct ShimmerModifier: ViewModifier {
    @State private var phase: CGFloat = -1

    func body(content: Content) -> some View {
        content
            .overlay(
                LinearGradient(
                    stops: [
                        .init(color: .clear, location: max(0, phase - 0.15)),
                        .init(color: .white.opacity(0.5), location: phase),
                        .init(color: .clear, location: min(1, phase + 0.15))
                    ],
                    startPoint: .leading,
                    endPoint: .trailing
                )
                .blendMode(.sourceAtop)
            )
            .onAppear {
                withAnimation(.linear(duration: 1.5).repeatForever(autoreverses: false)) {
                    phase = 2
                }
            }
    }
}
```

### Cell Margins

Match `ToolCallEventCell`:
```swift
.margins(.horizontal, grid(4))
.margins(.top, 0)
.margins(.bottom, grid(6))  // bottom padding since it's always the last item
```

## Reference

Desktop app (`github-app`) uses:
- `PulseIndicator` — 8px circle with `animate-ping` CSS (scale out + fade)
- `TextShimmer` — gradient overlay that sweeps across text
- `ConversationLoadingIndicator` — combines loader + elapsed timer

We're taking the simpler pulsing dot + shimmer approach, which is more appropriate for mobile.

## How to Test

1. Open an in-progress session
2. ✅ Working indicator shows pulsing dot (not rotating square)
3. ✅ "Working…" text has shimmer animation
4. ✅ Text size matches tool call rows (`.subheadline`)
5. ✅ Horizontal alignment matches tool call rows
6. ✅ Animation is smooth and not distracting

## Files Changed

- `Modules/GitHub/Sources/Copilot/Agents/WorkingIndicatorView.swift`

## Estimated Scope

~60 lines changed (replace `CommitSquare` with `PulsingDot` + add `ShimmerModifier`)
