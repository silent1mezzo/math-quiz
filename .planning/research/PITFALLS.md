# Domain Pitfalls

**Domain:** iOS SwiftUI math drill game for kids (ages 6-12)
**Project:** Math Quiz
**Researched:** 2026-03-12
**Confidence:** HIGH (Apple official docs + Apple Developer Forums + verified community sources)

---

## Critical Pitfalls

Mistakes that cause rewrites, App Store rejection, or broken core features.

---

### Pitfall 1: Timer Runs (or Stops) in the Wrong App State

**What goes wrong:** A countdown timer launched with `Timer.publish` or `Timer.scheduledTimer` continues ticking when the app moves to the background, or alternatively it stops mid-session when the user briefly switches apps — both silently corrupt game state. A child tapping the Home button accidentally to check something should not lose their session or have the timer cheat.

**Why it happens:** Developers wire up the timer and forget that iOS fires `scenePhase` transitions. The timer keeps publishing events in the background but UI updates are suppressed, so the score/timer gets out of sync.

**Consequences:** Timer shows wrong remaining time when app is foregrounded; sessions end prematurely; streaks can be broken by accidental app-switch.

**Prevention:**
- Observe `@Environment(\.scenePhase)` and pause/resume the timer explicitly.
- Store `sessionStartDate: Date` instead of decrementing a counter each tick. On resume, compute `elapsed = Date.now - sessionStartDate` and derive remaining time from that — wall-clock arithmetic survives background pauses.
- Invalidate `Timer` on `.background`, restart on `.active`.

```swift
.onChange(of: scenePhase) { phase in
    if phase == .active {
        // recompute remaining from wall clock
    } else {
        timer.upstream.connect().cancel()
    }
}
```

**Detection:** Simulate during development: start a session, double-press Home, wait 30 seconds, return. Timer should reflect the real elapsed wall-clock time.

**Phase:** Core game loop (Phase 1/2)

---

### Pitfall 2: Streak Logic Breaks at Midnight / Timezone Changes

**What goes wrong:** The "practiced today" check compares stored `Date` objects using the wrong granularity or assumes UTC midnight. A user who practices at 11:58 PM and again at 12:02 AM (next calendar day) has their streak broken because date comparison is using `.timeIntervalSince1970` differences rather than calendar-day boundaries.

**Why it happens:** `Date` in Swift is a universal timestamp (seconds since epoch). Comparing two `Date` values directly gives elapsed seconds, not "same calendar day in the user's timezone." Developers forget to use `Calendar.current` with the device's local timezone.

**Consequences:** Streaks reset incorrectly; kids feel cheated and disengage; bugs are nearly impossible to reproduce in testing without timezone simulation.

**Prevention:**
- Always use `Calendar.current.isDateInToday(_:)` or `Calendar.current.isDate(_:inSameDayAs:)` for streak comparisons — never raw `Date` arithmetic.
- Store `lastPracticeDate: Date` (the `Date` value), not a string. Derive the calendar day at display/comparison time using the local calendar.
- For DST transitions: `Calendar.current` respects DST automatically; `TimeZone.current` is refreshed by the OS when the user travels. No manual adjustment needed if you use `Calendar.current` consistently.
- When user travels across timezones, `Calendar.current` reflects the new timezone — which is the right behavior (local day is what matters for a daily habit).

```swift
// Correct: Calendar-aware same-day check
func practicedToday(lastDate: Date?) -> Bool {
    guard let lastDate else { return false }
    return Calendar.current.isDateInToday(lastDate)
}

// Correct: Detect a broken streak (no practice yesterday)
func streakContinues(lastDate: Date?) -> Bool {
    guard let lastDate else { return false }
    return Calendar.current.isDateInYesterday(lastDate) || Calendar.current.isDateInToday(lastDate)
}
```

**Detection:** Unit test streak logic with an explicit `Calendar` and mock dates: midnight boundary yesterday/today, DST transition dates, a UTC date that falls on a different local day.

**Phase:** Statistics/persistence (Phase 2/3)

---

### Pitfall 3: SwiftData Schema Migration Breaks on First Update

**What goes wrong:** The app ships without a `VersionedSchema`. The first update adds a field (e.g., `operationType` to a `QuizSession` model). SwiftData cannot perform an automatic lightweight migration and crashes on launch for existing users.

**Why it happens:** Early SwiftData documentation (iOS 17) de-emphasized versioned schemas for simple models. Developers add models directly without `SchemaMigrationPlan`, then discover there is no escape hatch — you cannot retroactively assign a version to an unversioned schema.

**Consequences:** Existing users lose all saved statistics on app update. App Store reviews fill with "lost my data" complaints. Re-architecting the persistence layer post-ship is a significant rewrite.

**Prevention:**
- Start with `VersionedSchema` even if it is V1 with no changes. The cost is ~10 lines of boilerplate; the recovery cost of skipping this is enormous.
- Never add non-optional properties to a SwiftData model without a migration step — always provide a default value or use optional.
- Do not subclass SwiftData models (unlike Core Data, inheritance is a known source of bugs in SwiftData).
- If you anticipate adding iCloud sync later: all properties must be optional or have defaults; `@Attribute(.unique)` is incompatible with CloudKit sync.

```swift
// Phase 1: Start here even if no migration needed yet
enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] { [QuizSession.self] }

    @Model final class QuizSession { /* ... */ }
}
typealias QuizSession = SchemaV1.QuizSession
```

**Detection:** Write a migration smoke test: create a model store with V1, then instantiate the container with V2. Run this in CI on every schema change.

**Phase:** Data persistence setup (Phase 1) — must be done before first public build

---

### Pitfall 4: App Store Rejection for Kids Category Compliance

**What goes wrong:** App is submitted to the Kids Category (ages 6-8 or 9-11) but includes:
- A share button that opens `UIActivityViewController` (link outside app, no parental gate)
- Any third-party SDK (analytics, crash reporting, ads) that transmits device identifiers
- A link to a website in the "About" or "Privacy Policy" view without a parental gate

Apple guideline 1.3 is explicit and enforced: rejection is immediate with no grace period.

**Why it happens:** Developers add analytics for legitimate product reasons (Crashlytics, Firebase), not realizing the Kids Category has a near-total ban on third-party SDKs that transmit identifiable data. "Identifiable" in Apple's interpretation includes the IDFA, device model, and persistent identifiers.

**Consequences:** App rejection delays launch. Fixing post-rejection requires removing SDKs, resubmitting, re-review (typically 24-48 hours but can be longer). Repeated violations can trigger extended review periods.

**Prevention:**
- For a v1 local-only app with no accounts, no ads, no analytics: submit to Kids Category confidently. The app as described (no IAPs, no backend, no ads) is low-risk.
- Do NOT add any of the following without audit: Firebase, Crashlytics, Amplitude, Mixpanel, Branch, Adjust, or any third-party ad network.
- If crash reporting is needed, use Apple's own MetricKit or Xcode Organizer crash logs — both are first-party and Kids Category compliant.
- Any link to an external URL (privacy policy, support email, website) must be behind a parental gate — an arithmetic challenge or a multi-step confirmation an adult must complete.
- Age rating: set to "4+" in App Store Connect. Select the Kids Category age band appropriate for target users.
- Privacy Manifest (`PrivacyInfo.xcprivacy`): required as of May 2024 for all apps. Declare which APIs you use (e.g., `NSUserDefaults` counts as `UserDefaults`). Missing this causes rejection.

**Detection:** Before submission, audit every third-party dependency with `Instruments > Privacy`. Check the `PrivacyInfo.xcprivacy` lists every required reason API your code calls. Test on a physical device (simulators can mask privacy issues).

**Phase:** App Store submission prep (final phase) — but audit SDK choices at the start to avoid refactoring later

---

## Moderate Pitfalls

---

### Pitfall 5: Number Pad Input Accepts Pasted Text and External Keyboard Input

**What goes wrong:** Using `.keyboardType(.numberPad)` on a `TextField` only changes the software keyboard shown. A child (or parent testing) can long-press and paste "abc" or "12.5" into the field, bypassing numeric validation. Users with an external keyboard (iPad + Magic Keyboard) can type any character.

**Why it happens:** `keyboardType` is a presentation hint, not an input filter.

**Consequences:** App crashes or displays wrong feedback when non-integer input is submitted. Division answers like "3.5" may be correct but break an integer-only check.

**Prevention:**
- Use a fully custom number pad (a `LazyVGrid` of `Button`s in SwiftUI) rather than a system `TextField`. This is actually simpler for kids (large buttons, no keyboard animation, no cursor management) and eliminates all paste/external keyboard issues.
- If using `TextField`: filter input with `.onChange(of:)` using Combine's character filter or a regex. Validate on submission, not display.
- For this specific app: a custom number pad with a "delete" and "submit" button is the recommended architecture. It also gives complete control over button size (critical for ages 6-8).

**Detection:** On iPad with external keyboard attached, test typing letters. Long-press the input field and test Paste.

**Phase:** Core input UI (Phase 1)

---

### Pitfall 6: SwiftUI View Body Thrashing Causes Animation Jank

**What goes wrong:** Every correct-answer animation (scale bounce, color flash, confetti) triggers a full view re-render because state is held at too high a level. A `@State var score: Int` change at the root view causes every child view — including the number pad — to re-render during the animation.

**Why it happens:** SwiftUI re-evaluates `body` for any view that reads changed state, including descendants. Putting all game state in one `@Observable` or `@StateObject` at the top level means every update re-renders everything.

**Consequences:** Animations stutter, especially on older devices (iPhone SE, iPad 6th gen) that kids are likely to use. The feedback animation feels broken right at the moment of reward.

**Prevention:**
- Split state: keep `score`, `streak`, `sessionStats` in a session model; keep `currentAnswer`, `inputBuffer`, `feedbackState` in the immediate question view as local `@State`.
- Use `.animation(.spring, value: feedbackState)` scoped to just the feedback view, not wrapping the entire screen.
- Profile with Instruments (SwiftUI template) before shipping. Look for unexpected body re-evaluations.
- Use `Equatable` conformance on views that should not re-render when irrelevant state changes.

**Detection:** In Xcode 15+, enable "Strict Concurrency" warnings and SwiftUI view update logging. Run on an iPhone SE (first or second generation) in the simulator to stress-test.

**Phase:** Animation/feedback polish (Phase 2/3)

---

### Pitfall 7: SwiftData @Query Blocks Main Thread on Session History Views

**What goes wrong:** A statistics screen uses `@Query` to fetch all `QuizSession` records. After 6 months of daily use (180+ sessions), the query loads all records into memory on the main actor, causing a visible stutter when navigating to the stats screen.

**Why it happens:** `@Query` runs on `MainActor` by default. SwiftData loads full model objects (not just selected fields) into memory for every result.

**Consequences:** Stats screen takes 0.5-2 seconds to appear after a year of use. This is especially noticeable on devices with slow flash storage (older iPads).

**Prevention:**
- Add a sort descriptor and fetch limit to `@Query`: `@Query(sort: \QuizSession.date, order: .reverse, limit: 30)` for a "recent sessions" view.
- For aggregate stats (total correct, average accuracy), use `FetchDescriptor` with `fetchCount` or computed aggregates, not `count` on a `@Query` result array (which loads all objects).
- Design stats model to pre-aggregate: store `totalCorrectAllTime` and `totalSessionsAllTime` as incrementing counters on a `UserStats` model, updated at session end. Avoid scanning all sessions to show a number.

**Detection:** Seed the SwiftData store with 200+ synthetic sessions in a test and profile the stats screen navigation with Instruments > Time Profiler.

**Phase:** Statistics screen (Phase 2/3)

---

### Pitfall 8: Kids UX — Touch Targets Too Small, Feedback Too Subtle

**What goes wrong:** Number pad buttons are sized at the default SwiftUI button frame (~44pt minimum per HIG) but a 6-year-old's finger covers 10-12mm (~38-45pt). Buttons that are exactly 44pt minimum feel unreliable for young children. Similarly, a green checkmark or a red X as the only feedback after an answer is too subtle — children expect immediate, unambiguous, multi-modal feedback.

**Why it happens:** Developers test on their own devices with adult fingers. Default HIG minimums are designed for adult accessibility, not children's motor precision.

**Consequences:** Children repeatedly miss buttons, become frustrated, and associate math practice with failure rather than the actual math problem. App gets 1-star reviews from parents ("the buttons don't work").

**Prevention:**
- Number pad buttons: minimum 70x70pt on iPhone, 90x90pt on iPad. Use `.frame(width:height:)` explicitly on each button.
- Correct/incorrect feedback must be: (a) full-screen color flash (green/red background), (b) large icon (SF Symbol at 64pt+), AND (c) haptic feedback (`UIImpactFeedbackGenerator` with `.medium` style for correct, `.rigid` for incorrect). Never rely on color alone (colorblindness).
- Text size: minimum 24pt for question display. Use `font(.system(size: 48, weight: .bold))` for the math problem itself — it should be the biggest thing on screen.
- Inter-button spacing: minimum 12pt gap. Fat-finger taps on adjacent buttons are a real failure mode.
- No complex navigation: the game loop should be a single screen. No tab bars, no navigation stacks during active play.

**Detection:** Hand the device to an actual 7-8 year old (neighbor's kid, family). Watch where they tap and whether they miss. 5 minutes of real child testing reveals more than any amount of simulator testing.

**Phase:** Core UI design (Phase 1) — establish these as constraints before building, not retrofitting

---

## Minor Pitfalls

---

### Pitfall 9: Simulator Gives False Confidence for Timer Tests

**What goes wrong:** Timer accuracy tests pass on the simulator but fail on-device because the simulator runs on Mac hardware with near-perfect timing. On-device, timer drift under CPU load (e.g., during animation) is measurable.

**Prevention:** Run all timer-related tests on a physical device. Use `Date()` wall-clock snapshots at session start/end to validate elapsed time, not tick counts.

**Phase:** Testing (all phases)

---

### Pitfall 10: Forgetting scenePhase for "Session in Progress" State

**What goes wrong:** User is mid-session, receives a phone call (iOS moves to `.inactive` then `.background`), answers the call, comes back. The timer has run out but the session result was never saved because the save happens only at intentional session-end.

**Prevention:** Save in-progress session state to SwiftData or `UserDefaults` on every `scenePhase` transition to `.inactive`. On app launch, check for an interrupted session and offer "Resume" or "Discard".

**Phase:** Core game loop (Phase 1/2)

---

### Pitfall 11: Privacy Manifest Missing Required Reason API Declarations

**What goes wrong:** App uses `UserDefaults` (for settings) or `Date()` APIs. Since May 2024, Apple requires a `PrivacyInfo.xcprivacy` file declaring the "required reason" for using certain APIs. Missing this causes App Store rejection at submission time, not review time — it fails the automated binary check before a human reviewer even sees it.

**Prevention:** Add `PrivacyInfo.xcprivacy` to the Xcode project at project creation. Declare:
- `NSPrivacyAccessedAPICategoryUserDefaults` with reason `CA92.1` (accessing your own app's defaults)
- `NSPrivacyAccessedAPICategorySystemBootTime` if using `ProcessInfo.systemUptime` (avoid this; use `Date()` instead)

Reference: [Apple Required Reason API documentation](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api)

**Detection:** Run `Instruments > App Store Privacy Report` before submission. Xcode 15.3+ will warn about missing required reason declarations during build.

**Phase:** Project setup (Phase 1) — add the file immediately at project creation

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Project setup | Missing `PrivacyInfo.xcprivacy` → automatic rejection | Add file at day 1 with UserDefaults declaration |
| Project setup | SwiftData without `VersionedSchema` → data loss on first update | Start with V1 VersionedSchema from day 1 |
| Core game loop | Timer not paused on background → corrupted session state | Use `scenePhase` + wall-clock time anchor |
| Core game loop | `TextField` with `.numberPad` → paste exploits | Use custom SwiftUI number pad grid instead |
| Feedback/animation | Top-level state mutation → animation jank on old devices | Scope `@State` to question view, not root |
| Statistics | `@Query` with no limit → main thread stall after months of use | Pre-aggregate stats; limit history queries |
| Statistics | Raw `Date` comparison → streak breaks at midnight | Use `Calendar.current.isDateInToday` exclusively |
| Kids UX | 44pt touch targets → children miss buttons | Enforce 70pt+ on all number pad buttons |
| App Store submission | Third-party SDKs → Kids Category rejection | Audit all dependencies; use only first-party APIs |
| App Store submission | External links without parental gate → rejection | Gate any off-app navigation behind parental challenge |

---

## Sources

- [Apple Kids Category Guidelines (Guideline 1.3)](https://developer.apple.com/app-store/review/guidelines/#kids-category)
- [Apple Kids Developer Resources](https://developer.apple.com/kids/)
- [Apple Required Reason API — Privacy Manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api)
- [Apple HIG — Accessibility: Touch Targets](https://developer.apple.com/design/human-interface-guidelines/accessibility)
- [SwiftData Key Considerations — fatbobman.com](https://fatbobman.com/en/posts/key-considerations-before-using-swiftdata/)
- [SwiftData Migration Guide — Atomic Robot](https://atomicrobot.com/blog/an-unauthorized-guide-to-swiftdata-migrations/)
- [SwiftData Common Errors — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftdata/common-swiftdata-errors-and-their-solutions)
- [High Performance SwiftData — Jacob Bartlett](https://blog.jacobstechtavern.com/p/high-performance-swiftdata)
- [SwiftUI Performance — Wesley Matlock / Medium](https://medium.com/@wesleymatlock/why-your-swiftui-app-is-slower-than-you-think-c3e9bb46174b)
- [SwiftUI Timer + scenePhase — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-a-timer-with-swiftui)
- [iOS Background Timer Limits — Medium / Deuk](https://medium.com/deuk/overcoming-ios-background-limits-a-time-tracker-app-in-swift-ui-5d157a58df68)
- [COPPA Compliance 2025 — promise.legal](https://blog.promise.legal/startup-central/coppa-compliance-in-2025-a-practical-guide-for-tech-edtech-and-kids-apps/)
- [SwiftData iOS 18 Memory Issues — Apple Developer Forums](https://developer.apple.com/forums/thread/761522)
- [SwiftData Custom Migration Crash — Apple Developer Forums](https://developer.apple.com/forums/thread/758874)
- [Testing in Simulator versus Device — Apple Developer Documentation](https://developer.apple.com/documentation/xcode/testing-in-simulator-versus-testing-on-hardware-devices)
