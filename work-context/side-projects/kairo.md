# Kairo — macOS Menu Bar Calendar

> **Type**: Personal / Side Project
> **Stack**: Swift/SwiftUI (macOS), Tauri/Rust (cross-platform port)
> **Platform**: macOS (native), Windows/Linux (Tauri)
> **Repo**: Private

---

## Overview

I built Kairo as a lightweight menu bar calendar for macOS. The idea came from frustration with clicking through to the full Calendar app just to see my schedule. Kairo lives in the menu bar and shows today's events at a glance — one click to see your day, no app switching needed.

After building the native macOS version, I ported the concept to Tauri (Rust + web frontend) for cross-platform support on Windows and Linux.

---

## macOS Native Architecture

### Stack
- **Swift 6 + SwiftUI**: Declarative UI with strict concurrency
- **EventKit**: System calendar integration (read-only access to Calendar.app events)
- **MenuBarExtra**: macOS 13+ menu bar API

### Architecture

```
┌──────────────────────────┐
│    MenuBarExtra (SwiftUI)│  ← App entry point
├──────────────────────────┤
│    Calendar Views        │  ← Day/Week/Month views
├──────────────────────────┤
│    EventKit Service      │  ← Calendar data access
├──────────────────────────┤
│    macOS Calendar Store  │  ← System calendars
└──────────────────────────┘
```

### Key Design Decisions

**MenuBarExtra over NSStatusItem**: I used `MenuBarExtra` with `.menuBarExtraStyle(.window)` for a proper windowed popover. This gives full SwiftUI layout capabilities inside the menu bar panel, unlike the older `NSStatusItem` + `NSPopover` approach.

**EventKit Integration**: Kairo reads from the system calendar store, so it works with any calendar provider (iCloud, Google, Exchange) that the user has configured in macOS Calendar preferences. No separate account setup needed.

**SwiftUI-Only**: Unlike ClipStash which needs AppKit for system integration, Kairo is pure SwiftUI. The calendar display doesn't require NSPanel tricks or pasteboard access.

### Features
- Today's agenda view in menu bar popover
- Mini month calendar with event indicators
- Week view for planning
- Click-to-open events in Calendar.app
- Multiple calendar support with color coding
- Real-time updates when calendar changes

[KEY_POINTS]
- Pure SwiftUI menu bar app — demonstrates MenuBarExtra mastery
- EventKit for zero-config calendar integration
- Lightweight design: shows information without replacing Calendar.app
- Shared architecture patterns with ClipStash (menu bar, SwiftUI)
[/KEY_POINTS]

---

## Tauri Cross-Platform Port

### Motivation

After building the macOS native version, I wanted to make Kairo available on Windows and Linux. Rather than building native apps for each platform, I chose Tauri — a Rust-based framework that wraps a web frontend in a native window.

### Stack
- **Rust**: Backend/system integration via Tauri
- **TypeScript/React**: Frontend UI
- **Tauri v2**: Framework for native desktop apps with web frontends

### Architecture

```
┌─────────────────────────────────┐
│     React Frontend (TypeScript) │  ← Calendar UI
├─────────────────────────────────┤
│     Tauri IPC Bridge            │  ← Rust ↔ JS communication
├─────────────────────────────────┤
│     Rust Backend                │  ← System tray, calendar access
├─────────────────────────────────┤
│     Platform Calendar APIs      │  ← OS-specific calendar stores
└─────────────────────────────────┘
```

### Cross-Platform Calendar Access

The biggest challenge in the Tauri port: calendar access differs across platforms.

| Platform | Calendar API | Notes |
|----------|-------------|-------|
| macOS | EventKit (via Rust FFI) | Same as native version |
| Windows | Windows Calendar API | COM-based, requires unsafe Rust |
| Linux | Evolution Data Server | D-Bus interface |

I abstracted calendar access behind a trait:

```rust
trait CalendarProvider: Send + Sync {
    async fn fetch_events(&self, range: DateRange) -> Result<Vec<Event>>;
    async fn calendars(&self) -> Result<Vec<Calendar>>;
}
```

Each platform implements this trait with its native calendar API.

### System Tray Integration

Tauri v2 has built-in system tray support. I used it to replicate the menu bar experience from the macOS native version:

- Tray icon shows the current date
- Click opens the calendar popover
- Right-click shows a context menu (settings, quit)

### Tradeoffs: Native vs. Tauri

| Aspect | Native (Swift) | Tauri (Rust + React) |
|--------|---------------|---------------------|
| Performance | Native rendering, instant | WebView rendering, slight lag |
| Bundle size | ~5 MB | ~15 MB |
| Calendar access | First-class EventKit | Platform-specific FFI |
| UI fidelity | Pixel-perfect macOS | Web-styled, less native |
| Cross-platform | macOS only | macOS + Windows + Linux |
| Development speed | Moderate | Fast (React frontend) |

[COMMON_MISTAKES]
- Assuming Tauri apps feel "native" — they don't match platform UI conventions without significant styling work
- EventKit permission prompts differ between standalone apps and system tray apps
- Windows calendar COM API requires careful memory management in Rust
[/COMMON_MISTAKES]

---

## Interview Talking Points

**"Tell me about a cross-platform project."**
I built Kairo, a menu bar calendar, first as a native macOS app in Swift/SwiftUI, then ported it to Tauri (Rust + React) for Windows and Linux. The interesting challenge was abstracting calendar access — EventKit on macOS, COM APIs on Windows, D-Bus on Linux — behind a common Rust trait.

**"When would you choose native vs. cross-platform?"**
For Kairo, I built both. Native SwiftUI gives pixel-perfect macOS integration. Tauri gives cross-platform reach with a 10 MB cost in bundle size and some UI fidelity loss. The right choice depends on whether platform-perfect UX or broad reach matters more.

**"How do you approach learning new technologies?"**
Kairo's Tauri port was my introduction to Rust for desktop apps. I learned by building — starting with the tray icon, then IPC bridge, then platform-specific calendar providers. Having the Swift version as a reference architecture made the port straightforward.

[FOLLOW_UP]
- How does Tauri compare to Electron for this use case?
- What was the hardest part of the Rust FFI for platform calendar APIs?
- How do you handle calendar permission requests across platforms?
[/FOLLOW_UP]

---

## Cross-References

- See [Side Projects → ClipStash](clipstash.md) for another macOS menu bar app with deeper AppKit integration
- See [Side Projects → Surmai](surmai.md) for another project with TypeScript/React frontend
- See [Side Projects → Other Projects](other-projects.md) for ColimaUI (another macOS native app)

---

*Last updated: Auto-generated from work-context-refresh skill*
