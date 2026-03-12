# Technology Stack

**Project:** Math Quiz (iOS math drill game for kids)
**Researched:** 2026-03-12

---

## Recommended Stack

### Platform Target

| Decision | Value | Why |
|----------|-------|-----|
| Minimum iOS | **iOS 17** | Unlocks SwiftData, @Observable macro, and `.sensoryFeedback()` — all three are load-bearing for this project. As of 2026, iOS 17+ covers nearly all active devices. Setting iOS 16 only saves compatibility for a tiny fraction of users at the cost of significant architectural compromise. |
| Swift | **Swift 6** | Ships with Xcode 16+. Strict concurrency by default. No reason to target earlier. |
| Xcode | **16.x** | Required for Swift 6 and Swift Testing framework. |

**Confidence:** HIGH — iOS 17 requirement is confirmed by SwiftData docs and well-established in the 2025/2026 community.

---

### Core Framework

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| SwiftUI | iOS 17 SDK | All UI | Native, declarative, first-class Apple support. Handles forms, buttons, custom layouts, and animations well. The PROJECT.md explicitly rejects SpriteKit — SwiftUI is the correct choice for a stats/score/input-heavy app with no physics simulation. |
| @Observable macro | iOS 17+ | State management in ViewModels | Replaces `ObservableObject`/`@Published`. More performant (view only re-renders when specific observed property changes, not any property). Less boilerplate. Available iOS 17+, which is already the minimum target. Use `@Observable` classes as ViewModels, held as `@State` at the root and passed as environment objects or constructor dependencies. |

**Do NOT use:** `ObservableObject` / `@Published` / `@StateObject` — these are the iOS 16-era patterns superseded by `@Observable`. Starting a greenfield iOS 17+ app with the old pattern is unnecessary technical debt.

**Confidence:** HIGH — Apple's own migration guide and community consensus (2024–2025) confirm `@Observable` is the preferred pattern.

---

### Architecture Pattern

**Use: MVVM (Model-View-ViewModel)**

- Views are pure SwiftUI structs — no business logic
- ViewModels are `@Observable` classes — hold state and game logic (timer, score, streak)
- Models are plain Swift structs/enums — `Question`, `SessionResult`, `OperationType`

**Why MVVM over alternatives:**
- **vs. TCA (The Composable Architecture):** TCA has a steep learning curve and is designed for large, complex state machines with many side effects. A math drill app has simple, linear state — overkill.
- **vs. Clean Architecture:** Adds multiple indirection layers (use cases, repositories, interface adapters). Appropriate for team-scale apps; adds friction for a solo project of this scope.
- **vs. no architecture (views with @State only):** Fine for toy apps; breaks down when you need to unit-test timer logic, scoring, or streak calculations.

MVVM is the sweet spot: testable ViewModels, clean Views, no unnecessary ceremony.

**Confidence:** HIGH — Supported by multiple 2025/2026 architecture guides and fits the project scope.

---

### Data Persistence

**Use a split strategy — not one tool for everything:**

| Layer | Technology | What Goes Here | Why |
|-------|------------|---------------|-----|
| App preferences | `AppStorage` (UserDefaults wrapper) | Operation mode selection, notification preference toggle | Zero boilerplate, `@AppStorage` works directly in SwiftUI views, appropriate for scalar values |
| Session statistics & streaks | **SwiftData** | `SessionResult` records, daily streak counter, accuracy history | Structured data with relationships; SwiftData provides type-safe, `@Observable`-compatible model layer with no boilerplate |

**SwiftData specifics:**
- Use `@Model` macro on model classes
- Use `@Query` in views for automatic fetch + reactive updates
- Use an in-memory `ModelContainer` in tests (confirmed supported pattern via Hacking with Swift)
- SwiftData requires iOS 17 — already the target minimum

**Do NOT use CoreData:** CoreData is the correct choice for apps requiring iOS 16 support, advanced migration choreography, or NSFetchedResultsController patterns. For a greenfield iOS 17+ SwiftUI app of this scope, CoreData imposes enormous boilerplate with no benefit. Apple is clearly investing in SwiftData as the forward path.

**Data volume reality check:** This app will store hundreds of `SessionResult` records at most. SwiftData's performance relative to CoreData is only a concern at extreme scale — not relevant here.

**Confidence:** HIGH — SwiftData for iOS 17+ greenfield apps is the current community and Apple recommendation. WWDC 2025 continued expanding SwiftData (model inheritance, Codable filters, persistent history).

---

### Animations

**Use: Native SwiftUI animations + Pow for "juice"**

| Technology | Purpose | Why |
|------------|---------|-----|
| SwiftUI built-in animations | Screen transitions, button feedback, timer bar | `withAnimation`, `.animation()`, spring physics, `.transition()` — zero dependencies for basic motion |
| **Pow** (EmergeTools/Pow) | Correct/incorrect answer feedback effects, streak celebration | Open-source SwiftUI effects library providing particle bursts, bounces, shakes, and custom transitions. Integrates via SwiftUI's `Transition` API and change effects. Last updated Feb 2025, actively maintained. Install via Swift Package Manager. |

**Pow installation:**
```
https://github.com/EmergeTools/Pow
```
Add via Xcode → File → Add Package Dependencies.

**Do NOT use:** Lottie (JSON-based animation). Lottie requires external animation assets, a design workflow, and is better suited for onboarding/loading indicators. For in-game feedback (correct answer flash, wrong answer shake), native SwiftUI + Pow is faster to build and stays entirely in code.

**Do NOT use:** SpriteKit particle systems for UI effects — that's mixing rendering engines without benefit.

**Confidence:** MEDIUM — Pow's iOS 17 minimum version needs confirmation from Package.swift (rate-limited during research), but the library's use of `ControlSize.extraLarge` confirms iOS 17 feature usage. Verify minimum version at project setup.

---

### Sound and Haptics

**Use: SwiftUI `.sensoryFeedback()` modifier + `AVAudioPlayer` for custom sounds**

| Technology | Purpose | Why |
|------------|---------|-----|
| `.sensoryFeedback(.success, trigger:)` | Correct answer haptic | Declarative, SwiftUI-native, iOS 17+. Types: `.success`, `.error`, `.impact()`, `.selection` |
| `.sensoryFeedback(.error, trigger:)` | Wrong answer haptic | Same API, different feedback type |
| `AVAudioPlayer` | Short sound effects (ding, buzz) | Simple, battle-tested, handles short audio clips without overhead of AVAudioEngine |

**Do NOT use:** `AudioServicesPlaySystemSound()` — works but is non-declarative, harder to control timing, and the legacy API. `.sensoryFeedback()` is the correct SwiftUI-era approach for haptics.

**Sound assets:** Keep as `.mp3` or `.caf` bundled in the app target. Keep files under 30 seconds to avoid `AVAudioPlayer` streaming overhead.

**Confidence:** HIGH — `.sensoryFeedback()` is documented in Apple's SwiftUI framework since iOS 17.

---

### Local Notifications (Daily Streaks)

**Use: `UserNotifications` framework (UNUserNotificationCenter)**

| Component | Usage |
|-----------|-------|
| `UNUserNotificationCenter` | Request permission, schedule, cancel |
| `UNCalendarNotificationTrigger` | Daily repeat at user-configured time |
| `UNMutableNotificationContent` | Title, body, sound |

**Pattern:**
1. Request `.alert`, `.sound`, `.badge` permissions on first launch (or deferred to settings screen)
2. On streak update: cancel existing daily reminder, re-schedule with updated streak count in body
3. Store user's notification preference and scheduled time in `@AppStorage`

**Streak data persistence:** Store streak counter and last-practice date as two `@AppStorage` values (integers and ISO date string). No need to put streak metadata in SwiftData — it's scalar state, not a record collection.

**Confidence:** HIGH — `UNUserNotificationCenter` is the only Apple-provided API for local notifications. No third-party library needed or advisable.

---

### Number Pad Input

**Use: Custom SwiftUI number pad (build it, don't import it)**

A custom number pad for this app is 10 digit buttons + backspace + submit, arranged in a `LazyVGrid`. This is 30–50 lines of SwiftUI. No library is warranted.

Benefits of building custom:
- Full control over button size (critical for kids — large touch targets)
- Match game visual style
- Haptic feedback on each key press via `.sensoryFeedback(.impact)`
- No dependency surface

**Do NOT use:** System `.numberPad` keyboard — it appears at the bottom of the screen, can't be styled, and behaves poorly for a game-feel input experience. A visible, in-app number pad is a better UX for kids.

**Confidence:** HIGH — Standard pattern in iOS game-style apps. The library `SwiftNumberPad` exists but is unnecessary overhead for this scope.

---

### Testing

**Use: Swift Testing (primary) + XCTest (UI tests only)**

| Framework | Purpose | Why |
|-----------|---------|-----|
| **Swift Testing** (`import Testing`) | Unit tests for ViewModels, game logic, scoring, streak calculation | Apple's new framework (WWDC24), ships with Xcode 16. Cleaner `#expect()` assertions, `@Test` macro, parallel execution by default. Recommended by Apple for new unit test development. |
| **XCTest** | UI tests only | Swift Testing does not yet support UI testing (as of early 2026). Use `XCUIApplication`-based UI tests for critical flows (e.g., answer submission, score display). |

**Do NOT use:** Third-party test frameworks (Quick/Nimble). Swift Testing provides expressive enough syntax natively. Adding dependencies for test infrastructure on a project of this scope adds friction without payoff.

**Test targets to create:**
- `MathQuizTests` — Swift Testing unit tests for `GameViewModel`, scoring logic, streak logic
- `MathQuizUITests` — XCTest UI tests for critical user flows

**Confidence:** MEDIUM — Swift Testing is confirmed as Apple's recommendation, but it is still under active development. Monitor for any limitations affecting the specific test patterns needed (async timer testing, SwiftData in-memory container testing).

---

## Full Dependency List

| Package | Source | Install via |
|---------|--------|-------------|
| Pow | github.com/EmergeTools/Pow | Swift Package Manager |

Everything else is first-party Apple frameworks:
- SwiftUI
- SwiftData
- UserNotifications
- AVFoundation
- Swift Testing (Xcode built-in)
- XCTest (Xcode built-in)

**Total third-party dependencies: 1.** This is intentional. The iOS SDK in 2026 covers everything this app needs except animation "juice." Minimize dependency surface for a single-developer project.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| State management | `@Observable` + MVVM | TCA | Steep learning curve, wrong scale |
| State management | `@Observable` + MVVM | `ObservableObject` | Superseded in iOS 17+, unnecessary re-render overhead |
| Persistence | SwiftData | CoreData | Massive boilerplate, no benefit at this scale for iOS 17+ target |
| Persistence | SwiftData | Realm | Third-party dependency, no SwiftUI-native `@Query` integration |
| Animations | Pow + native SwiftUI | Lottie | Requires external assets, design pipeline dependency |
| Number pad | Custom SwiftUI | System keyboard `.numberPad` | Can't style, wrong UX for game context |
| Testing | Swift Testing + XCTest | Quick/Nimble | Unnecessary dependency; Swift Testing is expressive enough |

---

## Xcode Project Setup Notes

Starting from the existing SpriteKit template described in PROJECT.md:

1. Delete SpriteKit boilerplate (`GameScene.swift`, `GameScene.sks`, `Actions.sks`)
2. Set deployment target to iOS 17.0
3. Replace `AppDelegate`/`SceneDelegate` pattern with `@main` `App` struct if not already present
4. Add Pow via Swift Package Manager
5. Create `MathQuizTests` and `MathQuizUITests` targets

---

## Sources

- [SwiftData updates — Apple Developer Documentation](https://developer.apple.com/documentation/updates/swiftdata)
- [Migrating from ObservableObject to @Observable — Apple Developer Documentation](https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro)
- [Swift Testing — Apple Developer](https://developer.apple.com/xcode/swift-testing/)
- [SwiftUI Data Persistence in 2025 — DEV Community](https://dev.to/swift_pal/swiftui-data-persistence-in-2025-swiftdata-core-data-appstorage-scenestorage-explained-with-5g2c)
- [Core Data vs SwiftData: Which Should You Use in 2025? — DistantJob](https://distantjob.com/blog/core-data-vs-swiftdata/)
- [Modern iOS App Architecture in 2026: MVVM vs Clean Architecture vs TCA — 7Span](https://7span.com/blog/mvvm-vs-clean-architecture-vs-tca)
- [Pow — EmergeTools/Pow on GitHub](https://github.com/EmergeTools/Pow)
- [SwiftUI Sensory Feedback — Use Your Loaf](https://useyourloaf.com/blog/swiftui-sensory-feedback/)
- [How to write unit tests for SwiftData code — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftdata/how-to-write-unit-tests-for-your-swiftdata-code)
- [Scheduling local notifications (SwiftUI) — Hacking with Swift](https://www.hackingwithswift.com/books/ios-swiftui/scheduling-local-notifications)
- [@Observable macro performance — Antoine van der Lee / SwiftLee](https://www.avanderlee.com/swiftui/observable-macro-performance-increase-observableobject/)
- [Storage options on iOS compared — Donny Wals](https://www.donnywals.com/storage-options-on-ios-compared/)
