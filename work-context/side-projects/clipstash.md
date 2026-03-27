# ClipStash — macOS Clipboard History Manager

> **Type**: Personal / Side Project
> **Stack**: Swift 6, SwiftUI + AppKit, GRDB (SQLite), SPM
> **Platform**: macOS 14+ (Sonoma)
> **Repo**: Private

---

## Overview

I built ClipStash as a native macOS clipboard history manager that preserves full-fidelity clipboard data. Unlike most clipboard managers that only store plain text, ClipStash captures and replays every pasteboard type — rich text, images, file references, custom UTIs — exactly as the source application wrote them.

The app runs as a menu bar utility with a floating panel for quick access. It's built entirely in Swift 6 with strict concurrency, using SwiftUI for the UI layer, AppKit for system integration (NSPanel, pasteboard, global hotkeys), and GRDB for SQLite persistence.

---

## Architecture

### Layer Diagram

```
┌─────────────────────────────────────┐
│           Menu Bar App              │  ← SwiftUI App lifecycle
├─────────────────────────────────────┤
│        Floating Panel (NSPanel)     │  ← AppKit window management
├─────────────────────────────────────┤
│     SwiftUI Views + ViewModels      │  ← Declarative UI
├─────────────────────────────────────┤
│        Clipboard Monitor            │  ← NSPasteboard polling
├─────────────────────────────────────┤
│         GRDB (SQLite)               │  ← Persistence layer
├─────────────────────────────────────┤
│      Paste Service + Hotkey         │  ← System integration
└─────────────────────────────────────┘
```

### Key Components

#### Clipboard Monitor
- Polls `NSPasteboard.general` on a timer (every 0.5s)
- Detects changes via `changeCount` property
- Captures all pasteboard types for the current item
- Serializes binary data (images, RTF, etc.) for storage
- Filters out internal pastes to avoid infinite loops

#### GRDB Persistence Layer
- SQLite via GRDB (not Core Data, not raw SQLite)
- Full clipboard item stored with all pasteboard types
- FTS5 full-text search on text content
- Automatic cleanup of old entries (configurable retention)

#### Floating Panel
- Custom `NSPanel` subclass for proper floating behavior
- Becomes key window without stealing focus from the frontmost app
- Dismisses on focus loss
- Positioned relative to menu bar icon

#### Paste Service
- Programmatic paste via `CGEvent` injection
- Writes selected clipboard item back to pasteboard
- Triggers Cmd+V in the frontmost application
- Handles timing edge cases (pasteboard write → keypress delay)

---

## GRDB Patterns

### Why GRDB Over Core Data?

I chose GRDB because:
1. **Direct SQL access**: I can write raw SQL when the ORM abstraction leaks
2. **Value types**: GRDB works with structs, not classes — better for Swift's value semantics
3. **Migration system**: Simple, numbered migrations without the complexity of Core Data model versioning
4. **Observation**: Built-in database observation for reactive UI updates
5. **Lightweight**: No heavyweight stack like Core Data's managed object context

### Record Types

```swift
struct ClipboardItem: Codable, FetchableRecord, PersistableRecord {
    var id: Int64?
    var textContent: String?
    var richTextData: Data?
    var imageData: Data?
    var sourceApp: String?
    var pasteboardTypes: [String]
    var createdAt: Date
    var isPinned: Bool
}
```

### Database Observation

I use GRDB's `ValueObservation` to drive reactive UI updates:

```swift
let observation = ValueObservation.tracking { db in
    try ClipboardItem
        .order(Column("createdAt").desc)
        .limit(100)
        .fetchAll(db)
}
```

This triggers SwiftUI view updates whenever the database changes — no manual refresh needed.

### FTS5 Search

Full-text search on clipboard text content:

```swift
// Migration
try db.create(virtualTable: "clipboardItemFts", using: FTS5()) { t in
    t.synchronize(withTable: "clipboardItem")
    t.column("textContent")
}

// Search
let pattern = FTS5Pattern(matchingAllTokensIn: query)
let items = try ClipboardItem
    .joining(required: ClipboardItem.fts.matching(pattern))
    .fetchAll(db)
```

[KEY_POINTS]
- GRDB gives SQL power with Swift type safety — best of both worlds
- ValueObservation drives reactive UI without Combine plumbing
- FTS5 enables instant search across clipboard history
- Struct-based records align with Swift's value type philosophy
[/KEY_POINTS]

---

## SwiftUI + AppKit Integration

### The Hybrid Challenge

ClipStash requires both SwiftUI (for the UI) and AppKit (for system-level features). This is the classic macOS app challenge: SwiftUI can't do everything AppKit can, but AppKit doesn't have SwiftUI's declarative model.

### NSPanel Management

The floating panel is an `NSPanel` — not an `NSWindow` — because:
- `NSPanel` can float above other windows without activating the app
- It supports the `.nonactivatingPanel` style mask
- It can become key (for keyboard input) without becoming main

```swift
class FloatingPanel: NSPanel {
    init() {
        super.init(
            contentRect: .zero,
            styleMask: [.nonactivatingPanel, .titled, .fullSizeContentView],
            backing: .buffered,
            defer: false
        )
        self.isFloatingPanel = true
        self.level = .floating
        self.hidesOnDeactivate = false
    }
}
```

### Panel Refresh Pattern

I developed a panel refresh pattern to handle a SwiftUI limitation: when the panel's content needs to update but the SwiftUI view hierarchy doesn't know the panel was dismissed and re-shown:

1. Panel dismissed → view state preserved
2. Panel re-shown → force refresh by toggling a state key
3. SwiftUI re-evaluates the view hierarchy

This avoids stale data when the user opens the panel after new clipboard items arrive.

### Form Padding Pattern

SwiftUI's default form padding is inconsistent on macOS. I created a `.formPadding()` view modifier that normalizes padding across different macOS versions:

```swift
extension View {
    func formPadding() -> some View {
        self.padding(.horizontal, 16)
            .padding(.vertical, 8)
    }
}
```

### renderInWindow Pattern

For snapshot testing, I created a `renderInWindow` helper that hosts a SwiftUI view in an offscreen `NSWindow` for pixel-accurate rendering:

```swift
func renderInWindow<V: View>(_ view: V, size: CGSize) -> NSImage {
    let window = NSWindow(contentRect: NSRect(origin: .zero, size: size), ...)
    window.contentView = NSHostingView(rootView: view)
    // ... capture window contents as NSImage
}
```

This was essential because SwiftUI previews and snapshot testing have different rendering pipelines.

[COMMON_MISTAKES]
- Using NSWindow instead of NSPanel for floating UI: NSWindow steals focus from the frontmost app
- Not handling `hidesOnDeactivate`: Panel disappears when any other window gets focus
- Forgetting to set `level = .floating`: Panel goes behind other windows
- SwiftUI view state persists across panel show/hide cycles — must force refresh
[/COMMON_MISTAKES]

---

## Snapshot Testing

### Approach

I use `swift-snapshot-testing` (Point-Free) for visual regression testing of the UI. Every significant view has snapshot tests that capture pixel-accurate renders.

### Test Infrastructure

```swift
import SnapshotTesting

final class ClipboardListTests: XCTestCase {
    func testClipboardListWithItems() {
        let view = ClipboardListView(items: .mock)
        assertSnapshot(
            matching: renderInWindow(view, size: CGSize(width: 320, height: 480)),
            as: .image
        )
    }
}
```

### Key Patterns

- **renderInWindow**: Required for accurate macOS rendering (see above)
- **Mock data**: All views have `.mock` static properties for consistent test data
- **Size variants**: Test at multiple panel sizes (compact, expanded, wide)
- **Dark mode**: Every snapshot has a dark mode variant

### Why Snapshot Testing?

1. Catches unintended visual regressions
2. Documents what the UI should look like
3. Fast feedback — no manual visual inspection
4. Especially valuable for NSPanel-hosted SwiftUI views where preview fidelity is unreliable

[FOLLOW_UP]
- How do you handle snapshot test failures across macOS versions?
- What's the tradeoff between snapshot testing and XCTest UI testing?
- How do you prevent snapshot bloat in the repo?
[/FOLLOW_UP]

---

## Global Hotkey

I register a global keyboard shortcut (Cmd+Shift+V by default) using Carbon's `RegisterEventHotKey` API:

- Carbon API because AppKit/SwiftUI don't have global hotkey support
- Hotkey triggers panel toggle (show/hide)
- Works even when ClipStash is not the frontmost app
- User-configurable key combination stored in UserDefaults

---

## Menu Bar Integration

The app lifecycle is `MenuBarExtra`-based (macOS 13+ API):

```swift
@main
struct ClipStashApp: App {
    var body: some Scene {
        MenuBarExtra("ClipStash", systemImage: "clipboard") {
            MenuBarView()
        }
        .menuBarExtraStyle(.window)
    }
}
```

The menu bar icon shows recent clips and provides access to the full panel, settings, and search.

---

## Swift 6 Strict Concurrency

ClipStash uses Swift 6's strict concurrency model:
- All mutable state is either `@MainActor` or protected by actors
- GRDB operations run on a background `DatabaseQueue`
- Clipboard monitoring runs on a dedicated actor
- No data races — the compiler enforces this

### Actor-Based Clipboard Monitor

```swift
actor ClipboardMonitor {
    private var lastChangeCount: Int = 0
    private let database: DatabaseQueue

    func checkForChanges() async {
        let currentCount = await MainActor.run {
            NSPasteboard.general.changeCount
        }
        guard currentCount != lastChangeCount else { return }
        lastChangeCount = currentCount
        // ... capture and store
    }
}
```

[KEY_POINTS]
- Full-fidelity clipboard capture: stores all pasteboard types, not just text
- GRDB + FTS5 for instant search across clipboard history
- NSPanel for proper floating window behavior without stealing focus
- Swift 6 strict concurrency — compiler-verified thread safety
- Snapshot testing with renderInWindow for pixel-accurate macOS UI testing
[/KEY_POINTS]

---

## Interview Talking Points

**"Tell me about a personal project you're proud of."**
ClipStash is a macOS clipboard history manager I built from scratch in Swift 6. The interesting challenge was preserving full clipboard fidelity — images, rich text, file references — not just plain text. I used GRDB for SQLite persistence with FTS5 full-text search, and built a hybrid SwiftUI/AppKit architecture with a floating NSPanel that doesn't steal focus from the active app.

**"How do you handle concurrency?"**
Swift 6 strict concurrency. The clipboard monitor is an actor, all UI state is @MainActor, and GRDB operations run on a background database queue. The compiler catches data races at build time.

**"How do you test UI?"**
Snapshot testing with swift-snapshot-testing. I built a renderInWindow helper that hosts SwiftUI views in an offscreen NSWindow for pixel-accurate macOS rendering. Every view has mock data and dark mode variants.

---

### Key Development Patterns (from MEMORY.md)

- **Panel content refresh**: Content view MUST be recreated on each `show()` call, not cached. `.onAppear` only fires once if the panel is reused — this was a subtle bug that took significant debugging.
- **Settings alignment**: Don't add `.padding()` on top of `.formStyle(.grouped)` — the grouped form already provides internal spacing. Double padding clips content.
- **Snapshot testing**: Use `renderInWindow()` helper — host SwiftUI views in a real NSWindow with RunLoop pumping for `.onAppear`/state updates to process. Set `window.appearance` for consistent dark/light mode. Use `bitmapImageRepForCachingDisplay` + `cacheDisplay` for accurate captures.
- **Test target → executable target**: In Swift 5.9+/SPM, the test target can depend on the executable target — `@main` is handled correctly.
- **Workflow**: After any code edits, run `./scripts/install-local.sh` to build and install to `/Applications/ClipStash.app`.

---

## Cross-References

- See [Nimble Download → Global Trade Compliance](../nimble-download/global-trade-compliance.md) for my SQLite expertise in a different context
- See [Side Projects → Kairo](kairo.md) for another macOS menu bar app
- See [Side Projects → Other Projects](other-projects.md) for ColimaUI (another macOS native app)

---

*Last updated: Auto-generated from work-context-refresh skill*
