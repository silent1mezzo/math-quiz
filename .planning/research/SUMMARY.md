# Project Research Summary

**Project:** Math Quiz (iOS math drill game for kids, ages 6-12)
**Domain:** Native iOS educational game / kids drill app
**Researched:** 2026-03-12
**Confidence:** HIGH

## Executive Summary

Math Quiz is a native iOS SwiftUI drill app where children practice arithmetic facts through timed sessions with immediate feedback, streaks, and session statistics. The research is highly convergent: build natively with iOS 17 as the minimum target, MVVM with `@Observable`, SwiftData for session history, and a custom in-app number pad. There is exactly one meaningful third-party dependency (Pow for animation effects). The entire technology stack is first-party Apple frameworks, which is both the correct call for this scope and a prerequisite for Apps Category compliance.

The recommended approach delivers a polished core loop in an early phase (custom number pad, timer, answer feedback, in-session scoring), then layers in persistence and statistics, then polishes with celebration moments and sound. Feature research is clear that the 8 MVP features are all achievable with low-to-medium complexity, and the most compelling differentiators (smart problem weighting, personal bests) have natural dependencies on session history — making deferral to a later phase both technically correct and appropriate for scope management.

The key risks are operational, not technical. Three must be handled from day one: (1) SwiftData requires a `VersionedSchema` from the initial commit or existing users lose data on first update; (2) the Privacy Manifest (`PrivacyInfo.xcprivacy`) must be added at project creation or the app fails automated App Store binary review; (3) touch targets must be designed for children's motor precision (70pt minimum on number pad) or the app earns 1-star reviews from parents. Timer background-handling and streak midnight-boundary logic are the two areas most likely to produce subtle bugs if not addressed directly during core game loop development.

## Key Findings

### Recommended Stack

The stack is iOS 17+, Swift 6, Xcode 16, with SwiftUI for all UI, `@Observable` for ViewModels, SwiftData for session records, and `@AppStorage` for scalar preferences. Sound uses `AVAudioPlayer` for custom clips and `.sensoryFeedback()` for haptics. Notifications use `UNUserNotificationCenter`. The number pad is a custom SwiftUI `LazyVGrid` — not the system keyboard. Testing uses Swift Testing for unit tests and XCTest for UI flows. The entire dependency list is one package: Pow (EmergeTools), installed via SPM for answer-feedback animation effects.

**Core technologies:**
- SwiftUI (iOS 17 SDK): All UI — native, declarative, first-class Apple support
- `@Observable` macro (iOS 17+): ViewModel state management — replaces `ObservableObject`, more performant
- SwiftData: Session history persistence — type-safe, `@Observable`-compatible, iOS 17+
- `@AppStorage`: Scalar preferences (mode selection, notification prefs, streak data) — zero boilerplate
- `.sensoryFeedback()`: Haptic feedback on correct/incorrect — SwiftUI-native, iOS 17+
- `AVAudioPlayer`: Short sound effects — battle-tested, minimal overhead
- `UNUserNotificationCenter`: Daily streak reminders — only Apple API for local notifications
- Pow (EmergeTools/Pow, SPM): Particle bursts and bounce effects for answer feedback — open-source, actively maintained
- Swift Testing: Unit tests for game logic and ViewModels — Apple's current standard (Xcode 16)
- XCTest: UI tests for critical flows — required since Swift Testing does not yet support UI testing

### Expected Features

**Must have (table stakes):**
- Operation selection (+ - × ÷) — every competitor offers this; defines practice scope
- Custom number pad with large touch targets (70pt+ buttons) — system keyboard is wrong UX for this context
- Immediate correct/incorrect feedback (visual + sound + haptic) — core learning loop; kids disengage without it
- Per-session countdown timer — creates urgency and pacing
- Score display during session — motivational engine; kids watch the number go up
- End-of-session summary (score, accuracy, time) — closes the session loop
- Daily streak tracking with persistence — streak psychology drives return visits (Duolingo data: 7-day streak = 2.4x retention)
- Session history statistics — parents and kids want to see improvement over time

**Should have (high value, low cost):**
- Sound toggle — kids use apps in school, bedtime; parents expect it, instantly accessible
- Session length choice (10/25/50 problems) — accommodates different attention spans
- Post-session problem review (what did I miss?) — closes the learning loop
- Difficulty levels (operand range control) — aligns with grade-level curriculum without adaptive engine
- Speed-based scoring bonus — makes sessions more game-like without over-rewarding rushing
- Celebratory milestone moments (streak milestones, first perfect session) — brief, earned celebrations

**Defer to v2+:**
- Smart problem weighting (re-surface weak facts) — requires per-problem tracking infrastructure to be solid first
- Personal bests / records — depends on reliable session history
- Configurable number ranges — useful but difficulty levels cover most needs initially
- Multiple user profiles — adds friction; validate family use case before building

**Hard anti-features (never build in v1):**
- Ads, in-app purchases, subscriptions — top parent complaint category in App Store reviews for this genre
- Third-party analytics SDKs — Kids Category compliance prohibits them
- Multiplayer / leaderboards — out of scope, adds backend complexity
- Parent/teacher dashboards — require accounts, backend, COPPA compliance

### Architecture Approach

The recommended architecture is MVVM with a thin service layer, splitting concerns between transient session state (timer, current question, score in progress — lives in `GameViewModel`, never persisted) and persistent history state (sessions, stats, streaks — flushed to SwiftData atomically only when a session completes). Navigation uses a root `NavigationStack` with programmatic path management so the back button never appears during active gameplay.

**Major components:**
1. `GameViewModel` (`@Observable`) — owns timer, current question, session state, answer validation, and session finalization; the heart of the app; fully unit-testable without any views
2. `QuestionGenerator` service — produces `MathQuestion` value types for given operation and difficulty; pure function, trivially testable
3. `StreakManager` service — reads/writes `StreakData` via `@AppStorage`; always uses `Calendar.current` for date comparisons, never raw `TimeInterval`
4. `GameSession` + `QuestionResult` (`@Model`, SwiftData) — persisted session and per-question records; written atomically on session completion; wrapped in `VersionedSchema` from day one
5. `NumberPadView` — custom SwiftUI `LazyVGrid` of `Button` views; stateless, callback-based; 70pt minimum button size; eliminates paste/external keyboard vulnerabilities
6. `ContentView` (NavigationStack root) — programmatic navigation path; prevents back-navigation during game; owns `ModelContainer`

The suggested build order from ARCHITECTURE.md is correct and must be followed: models and enums first, then `GameViewModel` (testable before any views), then stateless leaf views, then screen-level views, then navigation shell last.

### Critical Pitfalls

1. **Timer state corruption on background** — Use `@Environment(\.scenePhase)` to pause/resume; anchor to wall-clock `Date` at session start and compute remaining time from elapsed wall time on resume, not from tick counts. Test by double-pressing Home mid-session and waiting 30 seconds before returning.

2. **Streak logic breaks at midnight / timezone change** — Always use `Calendar.current.isDateInToday()` and `Calendar.current.isDateInYesterday()` for streak comparisons. Never use raw `TimeInterval` differences. Unit-test with mock dates at midnight boundary and DST transition dates.

3. **SwiftData schema migration crashes on first update** — Wrap models in `VersionedSchema` (V1) from the initial commit. Cost is ~10 lines of boilerplate; recovery cost of skipping it is a full persistence rewrite. All new properties must have default values or be optional.

4. **App Store rejection for Kids Category** — Never add Firebase, Crashlytics, or any third-party SDK that transmits device identifiers. Add `PrivacyInfo.xcprivacy` at project creation (mandatory since May 2024). Any link to external URLs must be behind a parental gate.

5. **Touch targets too small for children** — Enforce 70pt minimum (90pt on iPad) on all number pad buttons. Use multi-modal feedback (full-screen color flash + large icon + haptic) — never rely on color change alone. Real-child testing (age 7-8) is more informative than any simulator session.

## Implications for Roadmap

Based on combined research, the architecture's explicit build-order dependency graph and the feature dependency tree from FEATURES.md point to a clean 4-phase structure.

### Phase 1: Foundation and Core Game Loop

**Rationale:** Everything downstream depends on a working game session. The architecture research is explicit: build models and `GameViewModel` before any views. Three pitfalls (VersionedSchema, PrivacyInfo, touch targets) must be addressed in this phase or create expensive rework.

**Delivers:** A fully playable timed drill session — operation selection, countdown timer, custom number pad, correct/incorrect feedback with haptics and sound, end-of-session score summary. No persistence yet; data is in-memory only.

**Addresses features:** Operation selection, custom number pad, immediate feedback, per-session timer, score display during session, end-of-session summary.

**Must avoid:** Timer background corruption (scenePhase handling from day one), touch targets below 70pt, system keyboard `.numberPad` on `TextField` (paste vulnerabilities).

**Must do at project setup:** Add `PrivacyInfo.xcprivacy`, set up `VersionedSchema` V1, delete SpriteKit boilerplate, set deployment target iOS 17, add Pow via SPM.

### Phase 2: Persistence, Streaks, and Statistics

**Rationale:** Stats and streaks depend on completed session records existing in SwiftData. `StreakManager` and `StatsView` depend on `GameSession` being written atomically. This phase wires the persistence layer that Phase 1 left in-memory.

**Delivers:** Session history stored in SwiftData, daily streak tracking, `StatsView` showing history and aggregate numbers, daily reminder notifications.

**Addresses features:** Daily streak tracking, session history statistics, sound toggle (persist preference via `@AppStorage`).

**Must avoid:** Raw `Date` comparison for streak logic (use `Calendar.current`), `@Query` without fetch limit on StatsView (pre-aggregate totals as incrementing counters), saving partial sessions mid-game.

### Phase 3: Polish, Gamification, and Session Options

**Rationale:** Celebratory moments and difficulty options have no technical dependencies — they can be added once the core loop and persistence are solid. Session length choice and difficulty levels are configuration changes to `GameConfig`. Milestone celebrations are animation additions to existing state transitions.

**Delivers:** Session length choice, difficulty levels, celebratory milestone screens (streak milestones, first perfect session), speed-based scoring bonus, post-session problem review (missed questions summary), sound toggle made easily accessible.

**Addresses features:** Session length choice, difficulty levels, celebratory milestones, speed-based scoring, post-session problem review.

**Must avoid:** View body thrashing during animations — scope `feedbackState` to question view as local `@State`, not root-level state; profile on iPhone SE before shipping.

### Phase 4: App Store Submission Hardening

**Rationale:** Kids Category compliance, Privacy Manifest completeness, and physical-device testing are final gates. Separating this as an explicit phase prevents compliance issues from being discovered during review.

**Delivers:** Privacy Manifest fully declared, Kids Category age rating configured, parental gate on any external links, physical-device testing completed (timer drift, touch targets with real children), TestFlight beta, App Store submission.

**Must avoid:** Any third-party SDK that wasn't present in Phase 1 (audit all dependencies), missing `PrivacyInfo.xcprivacy` reason declarations for `UserDefaults` usage.

### Phase Ordering Rationale

- `GameViewModel` must be built and unit-tested before views exist — architecture research is explicit on this dependency chain
- SwiftData persistence must follow a working in-memory session, not precede it — this prevents partial-session persistence bugs
- Gamification layers (milestone celebrations, speed bonuses) have no technical dependencies and are additive — correctly deferred to Phase 3
- App Store compliance is last but the groundwork (VersionedSchema, PrivacyInfo, no third-party SDKs) is laid in Phase 1 — Phase 4 is verification and submission, not remediation

### Research Flags

Phases likely needing deeper `/gsd:research-phase` during planning:
- **Phase 2 (Persistence):** SwiftData `VersionedSchema` setup patterns and in-memory container testing for CI warrant a closer look at the exact boilerplate required — the research identified the pattern but not every line of code.
- **Phase 4 (App Store):** Kids Category parental gate implementation details (what constitutes an approved gate challenge) and the current `PrivacyInfo.xcprivacy` required-reason list should be verified against Apple's current documentation at submission time, as these requirements have changed before.

Phases with standard patterns (can skip research-phase):
- **Phase 1 (Core Game Loop):** MVVM + SwiftUI + custom number pad + `@Observable` are extremely well-documented patterns. Build order is clear from architecture research.
- **Phase 3 (Polish):** SwiftUI animation APIs and Pow integration are well-documented. Gamification patterns (milestone screens, streak celebrations) are straightforward additions to existing views.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | iOS 17 minimum, SwiftData, `@Observable`, and Pow all verified against Apple official docs and community consensus. One uncertainty: Pow's exact minimum iOS version was rate-limited during research — verify Package.swift at project setup. |
| Features | HIGH for table stakes / MEDIUM for differentiators | Table stakes cross-referenced against multiple competitor analyses and App Store review research. Differentiator value propositions (smart weighting, speed bonus) are well-reasoned but not independently validated against user research. |
| Architecture | HIGH | Component boundaries, data flow, and build order are backed by AzamSharp (2025), Hacking with Swift (authoritative), and Luke Roberts streak system (code-level verified). `@Observable` pattern confirmed against Apple migration guide. |
| Pitfalls | HIGH | All critical pitfalls sourced from Apple official guidelines, Apple Developer Forums, and verified community sources. Kids Category rejection risk confirmed against App Store guideline 1.3 directly. |

**Overall confidence:** HIGH

### Gaps to Address

- **Pow minimum iOS version:** The Pow Package.swift was rate-limited during research. Confirm `platforms: [.iOS(.v17)]` when adding the package in Phase 1. If minimum is iOS 16, it's still compatible with our target — but confirm the specific animation effects needed are available.
- **Swift Testing async timer patterns:** Swift Testing is still under active development. If async timer testing in GameViewModel requires patterns not supported yet, fall back to XCTest for those specific tests. Monitor Swift Testing release notes during Phase 1.
- **Parental gate implementation:** The specific challenge format Apple accepts for Kids Category parental gates (math puzzle vs. multi-step confirmation) should be checked against current App Store review guidelines before implementing any external link in Phase 4.
- **`@AppStorage` with `Codable` encoding for StreakData:** The architecture proposes encoding `StreakData` as a JSON string stored in UserDefaults. Confirm this pattern works correctly with SwiftUI's `@AppStorage` wrapper, which expects `RawRepresentable` or scalar types — may require a custom `RawRepresentable` conformance.

## Sources

### Primary (HIGH confidence)
- [SwiftData — Apple Developer Documentation](https://developer.apple.com/documentation/updates/swiftdata) — SwiftData model macros, `VersionedSchema`, `@Query`
- [Migrating from ObservableObject to @Observable — Apple Developer](https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro) — `@Observable` pattern confirmation
- [Swift Testing — Apple Developer](https://developer.apple.com/xcode/swift-testing/) — framework recommendation
- [Apple Kids Category Guidelines (1.3)](https://developer.apple.com/app-store/review/guidelines/#kids-category) — compliance requirements
- [Apple Required Reason API — Privacy Manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api) — `PrivacyInfo.xcprivacy` requirements
- [Apple HIG — Accessibility: Touch Targets](https://developer.apple.com/design/human-interface-guidelines/accessibility) — minimum touch target guidance
- [SwiftUI Timer + scenePhase — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-a-timer-with-swiftui) — timer background handling
- [How to write unit tests for SwiftData code — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftdata/how-to-write-unit-tests-for-your-swiftdata-code) — in-memory container testing
- [SwiftData Architecture Patterns — AzamSharp (2025)](https://azamsharp.com/2025/03/28/swiftdata-architecture-patterns-and-practices.html) — persistence architecture
- [Designing a Daily Streak System in Swift — Luke Roberts](https://blog.lukeroberts.co/posts/streak-system/) — Calendar-aware streak logic
- [Pow — EmergeTools/Pow on GitHub](https://github.com/EmergeTools/Pow) — animation effects library

### Secondary (MEDIUM confidence)
- [Modern iOS App Architecture in 2026: MVVM vs TCA — 7Span](https://7span.com/blog/mvvm-vs-clean-architecture-vs-tca) — MVVM recommendation for this scope
- [High Performance SwiftData — Jacob Bartlett](https://blog.jacobstechtavern.com/p/high-performance-swiftdata) — `@Query` performance, pre-aggregation pattern
- [SwiftData Key Considerations — fatbobman.com](https://fatbobman.com/en/posts/key-considerations-before-using-swiftdata/) — migration pitfalls
- [SwiftUI Sensory Feedback — Use Your Loaf](https://useyourloaf.com/blog/swiftui-sensory-feedback/) — `.sensoryFeedback()` API usage
- [Duolingo Streak Psychology — JustAnotherPM](https://www.justanotherpm.com/blog/the-psychology-behind-duolingos-streak-feature) — streak retention data
- [Best Math Apps for Kids in 2026 — Kidslox](https://kidslox.com/guide-to/math-apps-for-kids/) — competitor feature analysis
- [@Observable macro performance — Antoine van der Lee](https://www.avanderlee.com/swiftui/observable-macro-performance-increase-observableobject/) — `@Observable` vs `ObservableObject`

### Tertiary (context/supporting)
- [Dark Patterns in Kids Apps — Fairplay for Kids](https://fairplayforkids.org/wp-content/uploads/2021/05/darkpatterns.pdf) — anti-features rationale
- [COPPA Compliance 2025 — promise.legal](https://blog.promise.legal/startup-central/coppa-compliance-in-2025-a-practical-guide-for-tech-edtech-and-kids-apps/) — compliance context
- [SwiftData iOS 18 Memory Issues — Apple Developer Forums](https://developer.apple.com/forums/thread/761522) — awareness of known SwiftData issues

---
*Research completed: 2026-03-12*
*Ready for roadmap: yes*
