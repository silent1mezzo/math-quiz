# Phase 1: Foundation and Core Game Loop - Research

**Researched:** 2026-03-12
**Domain:** SwiftUI iOS app — MVVM game loop, custom number pad, haptics, sound, speed scoring
**Confidence:** HIGH

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| TECH-01 | App targets iOS 17 minimum | Deployment target set in project settings; delete SpriteKit template, create `@main` App struct targeting iOS 17 |
| TECH-02 | SwiftData persistence wrapped in VersionedSchema from initial implementation | VersionedSchema V1 boilerplate (~15 lines) required at `ModelContainer` creation; prevents migration crashes on first app update |
| TECH-03 | App includes PrivacyInfo.xcprivacy manifest declaring UserDefaults usage | File must be added as a target resource; `NSPrivacyAccessedAPICategoryUserDefaults` with reason `CA92.1` is required |
| TECH-04 | No third-party SDKs that transmit device identifiers (Kids Category compliance) | Only dependency is Pow (EmergeTools); Pow is MIT licensed with no data transmission; no Firebase, Crashlytics, or analytics SDKs |
| GAME-01 | User can select which operations to practice (any combination) | `OperationPicker` view binding to `[MathOperation]` on `GameConfig`; simple multi-select toggle UI |
| GAME-02 | App presents one math problem at a time with operands visible | `QuestionView` driven by `GameViewModel.currentQuestion: MathQuestion`; question is a value type produced by `QuestionGenerator` |
| GAME-03 | User enters answer using a custom in-app number pad (not system keyboard) | `NumberPadView` as `LazyVGrid` of `Button` views; no `TextField` with `.numberPad` keyboard type |
| GAME-04 | Number pad buttons are minimum 70pt for reliable kid touch targets | `.frame(minWidth: 70, minHeight: 70)` on each pad button; verified via HIG accessibility guidance |
| GAME-05 | App shows correct/incorrect feedback immediately (visual + haptic) | Full-screen color flash via `feedbackState` + `.sensoryFeedback(.success/.error, trigger:)` modifier on iOS 17+ |
| GAME-06 | App plays a sound on correct and a different sound on incorrect | Two preloaded `AVAudioPlayer` instances in `SoundPlayer` service; called synchronously from `GameViewModel.submitAnswer()` |
| GAME-07 | Elapsed time is displayed during session | `Timer.publish(every: 1)` + session `startDate: Date` anchored at session start; elapsed = `Date.now.timeIntervalSince(startDate)` |
| GAME-08 | Current score is displayed prominently during session | `score: Int` on `GameViewModel`; prominently displayed in `GameView` header |
| GAME-09 | Speed-based bonus points for fast correct answers | `questionStartDate: Date` set when each question appears; bonus decreases as `Date.now.timeIntervalSince(questionStartDate)` grows |
</phase_requirements>

---

## Summary

Phase 1 builds the complete playable game session from the ground up on a SwiftUI foundation, replacing all SpriteKit boilerplate. The technology choices are settled: iOS 17 minimum, `@Observable` MVVM, a custom `LazyVGrid` number pad, `AVAudioPlayer` for sounds, `.sensoryFeedback()` for haptics, `Date`-anchored elapsed timers with `scenePhase` handling, and Pow for answer animations. SwiftData with `VersionedSchema` and `PrivacyInfo.xcprivacy` are required infrastructure items that must be planted in Phase 1 even though they are not exercised until later phases.

The existing Xcode project is a SpriteKit game template. Phase 1 starts with a clean-slate task: delete `GameScene.swift`, `GameScene.sks`, `Actions.sks`, `GameViewController.swift`, and `AppDelegate.swift`, then create a SwiftUI `@main App` struct with a `NavigationStack` root. `Main.storyboard` and `LaunchScreen.storyboard` in `Base.lproj` must also be removed and replaced with a SwiftUI launch setup (delete `UIMainStoryboardFile` from Info.plist).

The build order within the phase is strictly: models/enums first, then `GameViewModel` (unit-testable before any views exist), then leaf views (`NumberPadView`, `QuestionView`, `ScoreHeaderView`), then the full `GameView` screen, then the `OperationPickerView` start screen, then the navigation shell wiring them together. `GameViewModel` must not depend on any View type; views observe it. This allows Swift Testing unit tests to run against the full scoring and answer-validation logic before the first Simulator boot.

**Primary recommendation:** Follow the strict bottom-up build order — models → ViewModel → leaf views → screens → navigation. Do not deviate into building screens before `GameViewModel` has passing unit tests.

---

## Existing Project State (What Must Be Deleted)

The Xcode project at `Math Quiz.xcodeproj` was created with the SpriteKit game template. The following files exist and must be deleted before any SwiftUI work begins:

| File | Why Delete |
|------|-----------|
| `Math Quiz/GameScene.swift` | SpriteKit scene; entirely replaced |
| `Math Quiz/GameScene.sks` | SpriteKit scene resource |
| `Math Quiz/Actions.sks` | SpriteKit actions resource |
| `Math Quiz/GameViewController.swift` | UIKit/SpriteKit view controller; replaced by SwiftUI App |
| `Math Quiz/AppDelegate.swift` | UIKit `@main` entry point; replaced by `@main` SwiftUI App struct |
| `Math Quiz/Base.lproj/Main.storyboard` | UIKit storyboard; replaced by SwiftUI |
| `Math Quiz/Base.lproj/LaunchScreen.storyboard` | Old launch screen; replaced by SwiftUI launch |

After deletion, `Info.plist` must have `UIMainStoryboardFile` key removed. The `Assets.xcassets` (containing `AppIcon` and `AccentColor`) should be kept.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| SwiftUI | iOS 17 SDK | All UI — views, layout, state binding | First-party, declarative, required for `@Observable` integration |
| `@Observable` macro | iOS 17+ (Swift 5.9) | ViewModel state management | Replaces `ObservableObject`; property-level change tracking, no `@Published` needed |
| SwiftData | iOS 17 SDK | `VersionedSchema` setup (data written in Phase 2) | Type-safe persistence; `@Observable`-compatible; required from initial commit for migration safety |
| `AVFoundation` / `AVAudioPlayer` | iOS 17 SDK | Sound effects for correct/incorrect answers | Battle-tested, minimal overhead, no external dependency |
| `.sensoryFeedback()` | iOS 17 SDK | Haptic feedback on answer submission | SwiftUI-native haptics; `.success` and `.error` built in |
| Swift Testing | Xcode 16 | Unit tests for game logic and `GameViewModel` | Apple's current standard testing framework; `@Test` macro |
| XCTest | Xcode 16 | UI tests for critical user flows | Required; Swift Testing does not yet support UI testing |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Pow (EmergeTools/Pow) | 1.0.5 | Answer feedback particle animations (`.spray`, `.jump`, `.shake`) | Correct answer celebration; incorrect shake effect |
| `Foundation` `Timer` | iOS 17 SDK | 1-second tick for elapsed time display | Drive `elapsedSeconds` counter in `GameViewModel` |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `AVAudioPlayer` | `SystemSoundServices` | System sounds are non-customizable; `AVAudioPlayer` allows custom audio clips |
| `.sensoryFeedback()` | `UIFeedbackGenerator` (UIKit) | `.sensoryFeedback()` is SwiftUI-native and declarative; UIKit API requires bridging |
| Pow animations | Custom SwiftUI `.matchedGeometryEffect` | Pow is faster to implement and visually higher quality for particle effects |
| `@Observable` | `ObservableObject` + `@Published` | `@Observable` has better granularity (only views that read a property re-render); iOS 17 only, which matches our minimum |

**Installation:**
```bash
# Pow via Swift Package Manager (Xcode: File > Add Package Dependencies)
# URL: https://github.com/EmergeTools/Pow
# Version: Up to Next Major, starting from 1.0.5
```

---

## Architecture Patterns

### Recommended Project Structure
```
Math Quiz/
├── App/
│   ├── MathQuizApp.swift        # @main App struct, ModelContainer setup
│   └── PrivacyInfo.xcprivacy   # Required at project setup
├── Models/
│   ├── MathOperation.swift      # enum: addition, subtraction, multiplication, division
│   ├── MathQuestion.swift       # value type: operand1, operand2, operation, correctAnswer
│   ├── GameConfig.swift         # value type: selected operations (Phase 1); extended in Phase 2
│   └── Schema/
│       ├── SchemaV1.swift       # VersionedSchema V1 (GameSession, QuestionResult models)
│       └── MigrationPlan.swift  # SchemaMigrationPlan (empty stages until Phase 2 adds a V2)
├── Services/
│   ├── QuestionGenerator.swift  # Pure function: MathQuestion from operation + config
│   └── SoundPlayer.swift        # Preloads AVAudioPlayer instances; play(correct:) / play(incorrect:)
├── ViewModels/
│   └── GameViewModel.swift      # @Observable; owns all session state
├── Views/
│   ├── ContentView.swift        # NavigationStack root, navigation path
│   ├── OperationPickerView.swift # GAME-01: select operations, start session
│   ├── GameView.swift           # GAME-02 through GAME-09: active session screen
│   ├── NumberPadView.swift      # GAME-03, GAME-04: custom number pad
│   ├── QuestionView.swift       # GAME-02: problem display
│   └── ScoreHeaderView.swift    # GAME-07, GAME-08: elapsed time + score display
└── Assets.xcassets/
    ├── Sounds/                  # correct.mp3, incorrect.mp3 (or .wav/.aiff)
    └── AppIcon.appiconset/
```

### Pattern 1: @Observable GameViewModel
**What:** A class marked `@Observable` that owns all transient session state. Views that read specific properties re-render only when those properties change. `@Observable` is NOT `@Model` — it is not persisted; it lives in memory for the duration of a session.
**When to use:** Any state that drives the active game session (current question, score, answer buffer, elapsed time, feedback state).

```swift
// Source: Apple Developer — Migrating from ObservableObject to @Observable
import Observation
import Foundation

@Observable
final class GameViewModel {
    // Session state
    var currentQuestion: MathQuestion?
    var answerBuffer: String = ""
    var score: Int = 0
    var elapsedSeconds: Int = 0
    var feedbackState: FeedbackState = .idle   // .idle / .correct / .incorrect
    var sessionActive: Bool = false

    // Speed scoring anchor
    private var questionStartDate: Date = .now
    private var sessionStartDate: Date = .now

    // Dependencies
    private let questionGenerator: QuestionGenerator
    private let soundPlayer: SoundPlayer

    init(questionGenerator: QuestionGenerator = .init(),
         soundPlayer: SoundPlayer = .init()) {
        self.questionGenerator = questionGenerator
        self.soundPlayer = soundPlayer
    }

    func startSession(config: GameConfig) { ... }
    func submitAnswer() { ... }
    func appendDigit(_ digit: String) { ... }
    func deleteLastDigit() { ... }
    func pauseSession() { ... }   // called on scenePhase = .background
    func resumeSession() { ... }  // called on scenePhase = .active
}
```

### Pattern 2: Wall-Clock Elapsed Timer with scenePhase Handling
**What:** Anchor elapsed time to a `Date` value, not a tick count. On every timer tick, compute `Int(Date.now.timeIntervalSince(sessionStartDate))`. When the app backgrounds, record the background-entry date; when it foregrounds, add the background duration to an offset. This survives clock-drift and tick-skip.
**When to use:** Any duration display that must survive app backgrounding.

```swift
// Source: Hacking with Swift — How to use a timer with SwiftUI
// Pattern: wall-clock anchor, scenePhase pause

// In GameView.swift:
private let timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

var body: some View {
    GameContent(viewModel: viewModel)
        .onReceive(timer) { _ in
            guard viewModel.sessionActive else { return }
            viewModel.tick()
        }
        .onChange(of: scenePhase) { _, newPhase in
            switch newPhase {
            case .background: viewModel.pauseSession()
            case .active:     viewModel.resumeSession()
            default:          break
            }
        }
}

// In GameViewModel:
private var backgroundEntryDate: Date?

func pauseSession() {
    backgroundEntryDate = .now
}

func resumeSession() {
    if let bg = backgroundEntryDate {
        // Shift the session start date forward by however long we were in background
        let backgroundDuration = Date.now.timeIntervalSince(bg)
        sessionStartDate = sessionStartDate.addingTimeInterval(backgroundDuration)
        backgroundEntryDate = nil
    }
}

func tick() {
    elapsedSeconds = Int(Date.now.timeIntervalSince(sessionStartDate))
}
```

### Pattern 3: Speed-Based Bonus Scoring
**What:** Record `questionStartDate = Date.now` when a new question appears. On correct answer submission, calculate elapsed seconds for that question. Faster answers earn more bonus points.
**When to use:** GAME-09 requirement.

```swift
// Source: derived from wall-clock Date pattern
func nextQuestion() {
    currentQuestion = questionGenerator.generate(config: config)
    questionStartDate = .now
}

func submitAnswer() {
    guard let question = currentQuestion,
          let answer = Int(answerBuffer) else { return }

    let elapsed = Date.now.timeIntervalSince(questionStartDate)

    if answer == question.correctAnswer {
        let basePoints = 10
        let bonus = max(0, 5 - Int(elapsed))   // 5-second window; 1 bonus pt/sec saved
        score += basePoints + bonus
        soundPlayer.playCorrect()
        feedbackState = .correct
    } else {
        soundPlayer.playIncorrect()
        feedbackState = .incorrect
    }

    answerBuffer = ""
    // After brief feedback delay, advance to next question
    Task { @MainActor in
        try? await Task.sleep(for: .milliseconds(600))
        feedbackState = .idle
        nextQuestion()
    }
}
```

### Pattern 4: Custom Number Pad
**What:** A `LazyVGrid` of `Button` views. Stateless — the view fires callbacks and the ViewModel owns the buffer. No `TextField`, no system keyboard, no paste vulnerability.
**When to use:** GAME-03, GAME-04.

```swift
// Source: Apple HIG touch targets; LazyVGrid pattern
struct NumberPadView: View {
    let onDigit: (String) -> Void
    let onDelete: () -> Void
    let onSubmit: () -> Void

    private let digits = ["1","2","3","4","5","6","7","8","9","⌫","0","→"]
    private let columns = Array(repeating: GridItem(.flexible(), spacing: 12), count: 3)

    var body: some View {
        LazyVGrid(columns: columns, spacing: 12) {
            ForEach(digits, id: \.self) { label in
                Button {
                    switch label {
                    case "⌫": onDelete()
                    case "→": onSubmit()
                    default:  onDigit(label)
                    }
                } label: {
                    Text(label)
                        .font(.title)
                        .frame(maxWidth: .infinity)
                        .frame(height: 70)   // GAME-04: 70pt minimum
                        .background(.tint.opacity(0.15))
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }
            }
        }
        .padding(.horizontal)
    }
}
```

### Pattern 5: VersionedSchema V1 Boilerplate
**What:** SwiftData models nested inside a `VersionedSchema` enum from the very first commit. The `ModelContainer` is initialized with the schema version, not raw model types. Adding new `@Model` classes in Phase 2 requires a V2 — but if V1 was never created, there is no migration path and data is lost on the first app update.
**When to use:** Must be done in Phase 1 at project setup, even though data is only written in Phase 2.

```swift
// Source: Hacking with Swift — How to create a complex migration using VersionedSchema
// Phase 1: only GameSession and QuestionResult are defined, but not yet written to

import SwiftData

enum SchemaV1: VersionedSchema {
    static var versionIdentifier = Schema.Version(1, 0, 0)
    static var models: [any PersistentModel.Type] {
        [GameSession.self, QuestionResult.self]
    }

    @Model
    final class GameSession {
        var date: Date = Date.now
        var score: Int = 0
        // Add more properties in Phase 2; must have defaults or be Optional
    }

    @Model
    final class QuestionResult {
        var session: GameSession?
        var wasCorrect: Bool = false
    }
}

enum MathQuizMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] { [SchemaV1.self] }
    static var stages: [MigrationStage] { [] }   // empty until V2 is needed
}

// typealias active models to current schema
typealias GameSession = SchemaV1.GameSession
typealias QuestionResult = SchemaV1.QuestionResult
```

```swift
// MathQuizApp.swift — ModelContainer initialization
@main
struct MathQuizApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(for: SchemaV1.models,
                        migrationPlan: MathQuizMigrationPlan.self)
    }
}
```

### Pattern 6: PrivacyInfo.xcprivacy
**What:** A property list file declaring API usage reasons. `NSUserDefaults` (used by `@AppStorage` in Phase 2) requires reason code `CA92.1`. This file must exist as a target resource from initial project setup or App Store submission will fail automated binary review (enforced since May 2024).
**When to use:** Add at project setup in Phase 1, before any other code.

```xml
<!-- PrivacyInfo.xcprivacy -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array>
        <dict>
            <key>NSPrivacyAccessedAPIType</key>
            <string>NSPrivacyAccessedAPICategoryUserDefaults</string>
            <key>NSPrivacyAccessedAPITypeReasons</key>
            <array>
                <string>CA92.1</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

### Pattern 7: Pow Answer Feedback Animation
**What:** Apply `.changeEffect(.spray { ... }, value: correctAnswerCount)` to the question view. The `spray` effect emits particle views (e.g., `Image(systemName: "star.fill")`) upward from the origin. For incorrect answers, `.changeEffect(.shake(rate: .fast), value: incorrectAnswerCount)` provides a shake.
**When to use:** GAME-05 — combined with the color flash feedback state for multi-modal feedback.

```swift
// Source: EmergeTools/Pow README — changeEffect modifier
import Pow

struct QuestionView: View {
    let question: MathQuestion
    let correctCount: Int    // incremented on correct answer
    let incorrectCount: Int  // incremented on incorrect answer

    var body: some View {
        VStack {
            Text(question.displayString)
                .font(.system(size: 48, weight: .bold))
        }
        .changeEffect(
            .spray(origin: .center) {
                Image(systemName: "star.fill").foregroundStyle(.yellow)
            },
            value: correctCount
        )
        .changeEffect(.shake(rate: .fast), value: incorrectCount)
    }
}
```

### Anti-Patterns to Avoid
- **TextField with `.numberPad` keyboard type:** Exposes paste menu; user can paste non-numeric content; system keyboard covers half the screen. Use `NumberPadView` exclusively.
- **Tick-counting for elapsed time:** `Timer` ticks can be skipped when app is backgrounded or the device is under load. Always anchor to `Date` and compute elapsed from wall clock.
- **`@Published` on `GameViewModel`:** The project targets iOS 17+, so there is no reason to use `ObservableObject`/`@Published`. Use `@Observable` only.
- **`withAnimation` wrapping `feedbackState` changes:** The feedback state transition (idle → correct → idle) should be a brief fixed-duration flash, not an interruptible animation that the user can confuse with game state.
- **Storing `GameViewModel` as `@State` in a child view:** `GameViewModel` must live at the screen level or higher. Pass it as an environment object or via `@Bindable` to avoid re-creation on view updates.
- **`@Model` classes used directly without VersionedSchema:** SwiftData permits this, but the first time a property is added and the app updates, the migration system has no schema history and crashes or silently drops data.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Particle / confetti animation on correct answer | Custom `Canvas` particle system | Pow `.spray` effect | Pow handles particle lifecycle, Z-ordering, and animation curve; building this correctly takes days |
| Haptic feedback sequencing | Manual `UIImpactFeedbackGenerator` calls | `.sensoryFeedback(.success/.error, trigger:)` | SwiftUI modifier handles threading; `.success` maps to correct system haptic profile for iOS 17+ |
| Audio engine | `AVAudioEngine` graph | `AVAudioPlayer` with `prepareToPlay()` | `AVAudioEngine` is for real-time DSP; for two preloaded sound clips, `AVAudioPlayer` is correct scope |
| Custom number pad grid math | Manual `HStack`/`VStack` row composition | `LazyVGrid` with `GridItem(.flexible())` | Grid handles spacing, column count, and responsive sizing across screen sizes automatically |
| Schema versioning | Raw `@Model` class top-level | `VersionedSchema` enum | Migration crash on first update is unrecoverable without a force-reinstall requirement |

**Key insight:** The three hardest problems in this phase — animations, haptics, and schema migrations — all have first-class solutions (Pow, `.sensoryFeedback()`, `VersionedSchema`). The hand-rolled versions of each introduce platform-specific bugs and multi-day implementation costs.

---

## Common Pitfalls

### Pitfall 1: Timer Drift on Background/Foreground
**What goes wrong:** The `Timer.publish(every: 1)` publisher stops firing when the app enters the background. On return, the elapsed time display shows the paused value instead of the correct wall-clock elapsed time.
**Why it happens:** `Timer` publishers on `.common` run loop mode do not fire in background execution. Tick-counting accumulates nothing during background.
**How to avoid:** Use `scenePhase` to record background-entry time. On foreground, shift `sessionStartDate` by the background duration (see Pattern 2 above).
**Warning signs:** Manually test by double-pressing Home mid-session, waiting 30+ seconds, returning. If elapsed display is wrong, the fix is missing.

### Pitfall 2: SpriteKit Boilerplate Causes Build Errors
**What goes wrong:** Leaving `AppDelegate.swift` (with `@main`) while adding a new SwiftUI `@main` App struct creates a compile-time "multiple @main entry points" error.
**Why it happens:** SpriteKit template creates a UIKit `@main AppDelegate`. SwiftUI requires its own `@main` App struct. Both cannot coexist.
**How to avoid:** Delete ALL SpriteKit files BEFORE writing any SwiftUI code. Confirm the build succeeds with a minimal SwiftUI `App` struct before proceeding.
**Warning signs:** "Multiple types conform to protocol App" compile error.

### Pitfall 3: VersionedSchema Not Wrapped from Day One
**What goes wrong:** Phase 2 adds new properties to `GameSession` and `QuestionResult`. Without `VersionedSchema`, SwiftData has no migration path; existing users lose all session history on their first update.
**Why it happens:** SwiftData permits top-level `@Model` classes, but that mode has no migration capability — it uses an implicit schema with no version history.
**How to avoid:** Create `SchemaV1` enum in Phase 1 even though nothing is written to the store until Phase 2. Cost is ~15 lines of boilerplate.
**Warning signs:** `ModelContainer` initialized with `for: [GameSession.self, QuestionResult.self]` directly instead of `Schema(versionedSchema: SchemaV1.self)`.

### Pitfall 4: PrivacyInfo.xcprivacy Missing at Submission
**What goes wrong:** App Store Connect automated binary analysis rejects the build with error ITMS-91053: "Missing API Declaration."
**Why it happens:** Since May 2024, any app using `UserDefaults` (accessed via `@AppStorage`) must declare the API and its reason code in `PrivacyInfo.xcprivacy`. The file must be a target resource, not just present in the file system.
**How to avoid:** Add `PrivacyInfo.xcprivacy` in Phase 1 setup (before any other code), declare `NSPrivacyAccessedAPICategoryUserDefaults` with reason `CA92.1`. Verify it appears in "Copy Bundle Resources" build phase.
**Warning signs:** File exists in project navigator but is not listed under the target's "Copy Bundle Resources" build phase.

### Pitfall 5: GameViewModel Referencing View Types
**What goes wrong:** `GameViewModel` imports `SwiftUI` and references `View`-specific types, making it impossible to unit-test without a running SwiftUI environment.
**Why it happens:** Convenience — it feels natural to put UI logic in the ViewModel.
**How to avoid:** `GameViewModel` imports only `Foundation` and `Observation`. All animation triggers flow out as plain value changes (`feedbackState = .correct`) that Views observe and translate to Pow effects or color changes.
**Warning signs:** `import SwiftUI` at the top of `GameViewModel.swift`.

### Pitfall 6: Division by Zero in QuestionGenerator
**What goes wrong:** Division questions can generate `0` as the divisor (e.g., "8 ÷ 0"), which has no valid integer answer and displays confusingly.
**Why it happens:** Naive random operand generation doesn't exclude zero for the denominator in division.
**How to avoid:** `QuestionGenerator` must clamp divisor range to `1...max` for division operations. Also ensure quotients are integers: generate the divisor and quotient first, then compute the dividend.
**Warning signs:** Unit test `testQuestionGenerator_division_noZeroDivisor` failing or missing.

---

## Validation Architecture

nyquist_validation is enabled in config.json.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Swift Testing (Xcode 16 built-in) + XCTest for UI tests |
| Config file | None — Swift Testing is built into Xcode 16; no config file needed |
| Quick run command | `xcodebuild test -scheme "Math Quiz" -destination "platform=iOS Simulator,name=iPhone 16" -only-testing "Math QuizTests"` |
| Full suite command | `xcodebuild test -scheme "Math Quiz" -destination "platform=iOS Simulator,name=iPhone 16"` |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| GAME-01 | Operation selection updates GameConfig correctly | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testOperationSelection"` | ❌ Wave 0 |
| GAME-02 | currentQuestion is non-nil when session is active | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testSessionProducesQuestion"` | ❌ Wave 0 |
| GAME-03/04 | NumberPadView renders 12 buttons each ≥ 70pt | UI | `xcodebuild test ... -only-testing "Math QuizUITests/NumberPadUITests/testPadButtonMinimumSize"` | ❌ Wave 0 |
| GAME-05 | feedbackState transitions correct → idle after 600ms | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testFeedbackStateTransition"` | ❌ Wave 0 |
| GAME-06 | SoundPlayer.playCorrect() / playIncorrect() do not crash | unit | `xcodebuild test ... -only-testing "Math QuizTests/SoundPlayerTests"` | ❌ Wave 0 |
| GAME-07 | elapsedSeconds increments on tick() | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testElapsedSecondsIncrement"` | ❌ Wave 0 |
| GAME-08 | score is displayed in GameView header | UI | `xcodebuild test ... -only-testing "Math QuizUITests/GameViewUITests/testScoreVisible"` | ❌ Wave 0 |
| GAME-09 | Fast answer earns more points than slow answer | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testSpeedBonusPoints"` | ❌ Wave 0 |
| TECH-02 | ModelContainer initializes with VersionedSchema without crashing | unit | `xcodebuild test ... -only-testing "Math QuizTests/SchemaTests/testModelContainerInitialization"` | ❌ Wave 0 |
| GAME-06 (pitfall) | QuestionGenerator never produces zero divisor for division | unit | `xcodebuild test ... -only-testing "Math QuizTests/QuestionGeneratorTests/testNoDivisionByZero"` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** Unit tests only — `xcodebuild test -scheme "Math Quiz" -destination "platform=iOS Simulator,name=iPhone 16" -only-testing "Math QuizTests"`
- **Per wave merge:** Full suite including UI tests
- **Phase gate:** Full suite green before `/gsd:verify-work`

### Wave 0 Gaps
- [ ] `Math QuizTests/GameViewModelTests.swift` — covers GAME-01, GAME-02, GAME-05, GAME-07, GAME-09
- [ ] `Math QuizTests/QuestionGeneratorTests.swift` — covers GAME-02 (question validity), division-by-zero pitfall
- [ ] `Math QuizTests/SoundPlayerTests.swift` — covers GAME-06
- [ ] `Math QuizTests/SchemaTests.swift` — covers TECH-02 (ModelContainer init)
- [ ] `Math QuizUITests/NumberPadUITests.swift` — covers GAME-03, GAME-04
- [ ] `Math QuizUITests/GameViewUITests.swift` — covers GAME-08
- [ ] Test target must be added to the Xcode project (SpriteKit template may have created one; verify it compiles after SpriteKit deletion)

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `ObservableObject` + `@Published` | `@Observable` macro | iOS 17 / WWDC23 | Property-level granularity; no `@Published` annotation; simpler syntax |
| `UIApplicationDelegate` lifecycle | SwiftUI `@Environment(\.scenePhase)` | iOS 14+ | No UIKit boilerplate for background/foreground; declarative |
| `UIImpactFeedbackGenerator` (UIKit) | `.sensoryFeedback()` SwiftUI modifier | iOS 17 | Native SwiftUI; no generator object management |
| Top-level `@Model` classes | Models nested in `VersionedSchema` | SwiftData initial release (iOS 17) | Migration safety; crash prevention on schema change |
| Manual Privacy manifest | `PrivacyInfo.xcprivacy` added to target | May 2024 (enforcement) | App Store required for any UserDefaults usage |

**Deprecated/outdated:**
- `ObservableObject` + `@Published`: Still works but deprecated in spirit for iOS 17+ apps; use `@Observable`.
- `UIKit AppDelegate` lifecycle for background handling: Replaced by `scenePhase` for SwiftUI apps.
- SpriteKit template entry point: The existing `AppDelegate.swift` and `GameViewController.swift` are UIKit boilerplate that must be removed entirely.

---

## Open Questions

1. **Pow minimum iOS version confirmation**
   - What we know: Package.swift declares `platforms: [.iOS(.v15)]` (confirmed via raw GitHub fetch)
   - What's unclear: Whether all effects we use (`.spray`, `.shake`) were added at v15 or a later Pow version
   - Recommendation: Add Pow via SPM at Phase 1 project setup and verify the effects compile and run on the iOS 17 Simulator immediately; don't defer this check

2. **Swift Testing + @MainActor @Observable ViewModel test performance**
   - What we know: A Swift Forums thread confirms that `@MainActor @Observable` ViewModels cause Swift Testing to serialize all tests onto the main actor, reducing parallelism
   - What's unclear: Whether this is significant enough to require XCTest fallback or a `nonisolated` workaround for GameViewModel tests
   - Recommendation: Start with Swift Testing for all unit tests; if test suite takes >10 seconds to run, evaluate moving timer-specific tests to XCTest

3. **Division question generation: integer-only quotients**
   - What we know: Naive random generation can produce non-integer quotients (e.g., "7 ÷ 3")
   - What's unclear: Whether the requirement spec assumes integer answers only (likely yes for kids)
   - Recommendation: `QuestionGenerator` for division should generate divisor and quotient first (both from the operand range), then compute dividend = divisor × quotient; this guarantees integer results

---

## Sources

### Primary (HIGH confidence)
- [Apple Developer — Migrating from ObservableObject to @Observable](https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro) — `@Observable` pattern
- [Apple Developer — VersionedSchema](https://developer.apple.com/documentation/swiftdata/versionedschema) — schema versioning API
- [Apple Developer — sensoryFeedback(_:trigger:)](https://developer.apple.com/documentation/swiftui/view/sensoryfeedback(_:trigger:)) — haptic feedback modifier
- [Apple Developer — SensoryFeedback types](https://developer.apple.com/documentation/swiftui/sensoryfeedback) — `.success`, `.error` feedback types
- [Apple Developer — AVAudioPlayer](https://developer.apple.com/documentation/avfaudio/avaudioplayer) — sound playback API
- [Apple HIG — Accessibility: Touch Targets](https://developer.apple.com/design/human-interface-guidelines/accessibility) — 70pt minimum target guidance
- [Apple Developer — Required Reason API / PrivacyInfo.xcprivacy](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files/describing_use_of_required_reason_api) — privacy manifest requirements
- [EmergeTools/Pow — Package.swift](https://github.com/EmergeTools/Pow/blob/main/Package.swift) — minimum iOS version: iOS 15 (compatible with iOS 17 target)
- [EmergeTools/Pow — README](https://github.com/EmergeTools/Pow) — `.spray`, `.shake` change effects documentation
- [Hacking with Swift — VersionedSchema migration](https://www.hackingwithswift.com/quick-start/swiftdata/how-to-create-a-complex-migration-using-versionedschema) — V1 boilerplate pattern

### Secondary (MEDIUM confidence)
- [Use Your Loaf — SwiftUI Sensory Feedback](https://useyourloaf.com/blog/swiftui-sensory-feedback/) — `.sensoryFeedback()` usage examples
- [Hacking with Swift — Timer with SwiftUI](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-a-timer-with-swiftui) — `Timer.publish` + `onReceive` pattern
- [Pol Piella — Configuring SwiftData in a SwiftUI app](https://www.polpiella.dev/configuring-swiftdata-in-a-swiftui-app) — `ModelContainer` setup in App struct
- [Antoine van der Lee — @Observable macro performance](https://www.avanderlee.com/swiftui/observable-macro-performance-increase-observableobject/) — `@Observable` vs `ObservableObject` comparison
- [Jacob Bartlett — Unit Test the Observation Framework](https://blog.jacobstechtavern.com/p/unit-test-the-observation-framework) — testing `@Observable` ViewModels without views
- [Jochen Holzer — Troubleshooting PrivacyInfo.xcprivacy](https://jochen-holzer.medium.com/required-reason-api-troubleshooting-your-ios-privacy-manifest-file-privacyinfo-xcprivacy-c81084dc9d51) — `CA92.1` reason code for UserDefaults

### Tertiary (LOW confidence — noted for awareness)
- [Swift Forums — Swift Testing performance with @MainActor @Observable](https://forums.swift.org/t/improving-swift-testing-performance-for-mainactor-observable-view-models/84733) — known test parallelism issue; may or may not affect this project's test count

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — All core technologies are first-party Apple (iOS 17 SDK); Pow is verified via Package.swift
- Architecture: HIGH — MVVM with `@Observable`, build order, and component boundaries are backed by official Apple documentation and multiple authoritative sources
- Pitfalls: HIGH — VersionedSchema and PrivacyInfo requirements verified against Apple official docs; SpriteKit deletion requirement confirmed by direct inspection of existing project files
- Code examples: HIGH — Patterns derived from official Apple documentation and Hacking with Swift (authoritative); Pow examples from official README

**Research date:** 2026-03-12
**Valid until:** 2026-06-12 (90 days — stable Apple frameworks; re-check PrivacyInfo required-reason list before App Store submission as Apple has amended it previously)
