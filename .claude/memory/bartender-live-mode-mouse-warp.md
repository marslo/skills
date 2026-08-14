---
name: bartender-live-mode-mouse-warp
description: Bartender 6 Live layout mode warps the mouse and breaks Chrome hover; fix is On-Demand mode
metadata: 
  node_type: memory
  type: reference
  originSessionId: f778c50e-adf9-4e9d-877e-e054fb4d7f47
  modified: 2026-08-14T00:10:18.011Z
---

Bartender 6 (v6.6.2, macOS Tahoe 26.x) in **Live** layout mode auto-arranges menu bar items whenever a fast-updating item changes (clock-with-seconds, network up/down, memory, CPU temp). Each arrange warps the cursor to the top-right menu bar (visible as a tiny mouse jitter), which desyncs mouse tracking in other apps — Chrome hover effects freeze until Bartender is quit.

**Symptom:** after a webpage is open seconds-to-minutes, a tiny mouse jitter, then Chrome hover stops working entirely.

**Root cause (confirmed via debug log):** `MenuItemPosition / Arranging profile` entries with `cause: liveUpdate`. On-Demand mode → zero such entries.

**Fix:** Bartender → General → Layout Mode → switch **Live → On-Demand**. Keeps notch/hidden-icon management; hidden items just don't auto-reveal on update (click/swipe empty menu bar to show).

Not related: AccessibilityMatcher runMatchingLoop (the screen-recording indicator) and Advanced toggles (Suppress Clicks During Moves, Disable Pausing, Item Indexing) — none warp the cursor.
