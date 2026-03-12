# Architecture Patterns

**Domain:** iOS SwiftUI educational math drill game
**Researched:** 2026-03-12

## Recommended Architecture

MVVM with a thin service layer. Views own no logic. ViewModels own game state. SwiftData models own persistence. The key insight for this app: there are two distinct concerns — **transient session state** (timer, current question, score in progress) and **persistent history state** (sessions, stats, streaks). Keep them separated.

```
App
├── Navigation Root (ContentView)
│   ├── HomeView
│   │   └── HomeViewModel
│   ├── GameView
│   │   └── GameViewModel  ← owns timer, current question, session state
│   ├── ResultsView
│   │   └── ResultsViewModel
│   └── StatsView
│       └── StatsViewModel
│
├── Models (SwiftData @Model)
│   ├── GameSession
│   └── QuestionResult
│
├── Services
│   ├── QuestionGenerator
│   └── StreakManager
│
└── SwiftData Stack
    └── ModelContainer (app-level singleton)
```

### Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|---------------|-------------------|
| `HomeView` | Operation selection, difficulty, start button | HomeViewModel, navigation |
| `GameView` | Renders current question, number pad, timer bar, feedback | GameViewModel only |
| `GameViewModel` | Timer tick, question sequencing, answer validation, score accumulation | QuestionGenerator, navigation |
| `ResultsView` | Post-session summary: score, accuracy, streak info | ResultsViewModel, StatsViewModel |
| `StatsView` | Historical sessions, streak display | SwiftData @Query directly |
| `QuestionGenerator` | Produces MathQuestion values for given operation/difficulty | GameViewModel |
| `StreakManager` | Read/write daily streak, compare calendar dates | AppStorage or SwiftData |
| `GameSession` (@Model) | Persisted record of one completed session | SwiftData ModelContext |
| `QuestionResult` (@Model) | Persisted record of one answered question | GameSession (cascade) |

### Data Flow

```
User taps operation + Start
        │
        ▼
HomeViewModel creates GameConfig (value type: operation, difficulty, question count)
        │
        ▼
NavigationStack pushes GameView(config:)
        │
        ▼
GameViewModel.init(config:) called
  - QuestionGenerator produces first question
  - Timer starts
        │
        ▼
User enters digits on NumberPadView
  - Each tap → GameViewModel.appendDigit(_:)
  - NumberPadView reads displayAnswer from GameViewModel (one-way binding)
        │
        ▼
User taps Submit (or auto-advance on final digit)
  - GameViewModel.submitAnswer()
  - Validates: correct / incorrect
  - Records QuestionAttempt (in-memory, value type)
  - Shows feedback overlay (0.6s)
  - Advances to next question or ends session
        │
        ▼
Session ends (questions exhausted or timer runs out)
  - GameViewModel.finalizeSession()
  - Builds GameSession + [QuestionResult] objects
  - Saves via ModelContext
  - StreakManager.recordActivity(date: .now)
  - Navigates to ResultsView
        │
        ▼
ResultsView reads session summary from GameViewModel.completedSession
StatsView reads history via @Query(sort: \GameSession.date, order: .reverse)
```

## Data Models

### Value Types (in-memory, not persisted)

```swift
// Produced by QuestionGenerator, consumed by GameViewModel
struct MathQuestion {
    let operandA: Int
    let operandB: Int
    let operation: MathOperation
    var correctAnswer: Int { operation.compute(operandA, operandB) }
    var displayString: String { "\(operandA) \(operation.symbol) \(operandB) = ?" }
}

enum MathOperation: String, CaseIterable, Codable {
    case addition, subtraction, multiplication, division
}

struct GameConfig {
    let operations: Set<MathOperation>
    let difficulty: Difficulty     // controls operand range
    let questionCount: Int         // 10, 20, or unlimited
    let timerMode: TimerMode       // perQuestion(seconds:) or session(seconds:)
}

// Accumulated in-memory during a session, flushed to SwiftData on completion
struct QuestionAttempt {
    let question: MathQuestion
    let userAnswer: Int?
    let isCorrect: Bool
    let responseTime: TimeInterval
}
```

### Persisted Types (SwiftData @Model)

```swift
@Model
class GameSession {
    var date: Date
    var operation: String          // store raw value, not enum (SwiftData limitation)
    var totalQuestions: Int
    var correctCount: Int
    var durationSeconds: Double
    var accuracy: Double           // computed at save time, stored for query efficiency

    @Relationship(deleteRule: .cascade)
    var questionResults: [QuestionResult] = []

    // Computed helpers
    var scorePercent: Int { Int((accuracy * 100).rounded()) }
}

@Model
class QuestionResult {
    var questionText: String       // "7 x 8 = ?"
    var correctAnswer: Int
    var userAnswer: Int?
    var isCorrect: Bool
    var responseTime: TimeInterval
    var session: GameSession?
}
```

### Streak Model

```swift
// Stored via @AppStorage (simple, no query needed)
// Key: "streakData" → Codable JSON
struct StreakData: Codable {
    var currentStreak: Int = 0
    var longestStreak: Int = 0
    var lastActivityDate: Date?
}
```

Streak storage rationale: streak is a single record with two integers and a date. SwiftData is overkill here. `@AppStorage` with a `Codable` struct (encoded to JSON string) is correct for this scope. Revisit only if multi-habit tracking is added.

## State Management

Use `@Observable` (iOS 17+ macro), not the older `ObservableObject` pattern. This is the current Apple standard as of 2025 and eliminates `@Published`, `@ObservedObject`, and `@StateObject` boilerplate.

```swift
@Observable
class GameViewModel {
    // Transient session state
    var currentQuestion: MathQuestion?
    var displayAnswer: String = ""
    var timeRemaining: Double = 0
    var score: Int = 0
    var questionIndex: Int = 0
    var feedbackState: FeedbackState = .none
    var sessionComplete: Bool = false

    // Injected
    private let config: GameConfig
    private let modelContext: ModelContext
    private var timer: Timer?
    private var questionAttempts: [QuestionAttempt] = []
}
```

Ownership pattern:
- `@State var viewModel = GameViewModel(...)` in the owning view (replaces `@StateObject`)
- Child views receive the model directly — no wrapper needed, SwiftUI tracks access automatically

## Timer Implementation

Use `Foundation.Timer` scheduled on `.main` runloop with a 0.1-second interval for smooth progress bar animation. Do not use 1-second intervals if you want a visual countdown bar — granularity matters for UX.

```swift
// Inside GameViewModel
func startTimer() {
    timer = Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { [weak self] _ in
        guard let self else { return }
        self.timeRemaining -= 0.1
        if self.timeRemaining <= 0 {
            self.timeExpired()
        }
    }
}

func stopTimer() {
    timer?.invalidate()
    timer = nil
}
```

Set `.tolerance = 0.05` on the timer to allow OS scheduling flexibility and reduce battery impact — acceptable for a countdown where exact milliseconds are not critical.

Cancel the timer in the view's `.onDisappear` or when the session ends. Forgetting this is the most common timer memory leak.

## Streak Tracking Logic

```
On session completion:
  1. Load StreakData from @AppStorage
  2. Compare lastActivityDate to today using Calendar.isDateInToday()
  3. If same day → already practiced today, no change
  4. If yesterday → streak continues, increment currentStreak
  5. If earlier than yesterday → streak broken, reset to 1
  6. Update longestStreak if currentStreak > longestStreak
  7. Set lastActivityDate = today
  8. Save StreakData back to @AppStorage
```

Use `Calendar.current.isDateInYesterday(lastActivityDate)` — do not compute raw TimeInterval differences. Calendar-aware comparison handles DST and midnight edge cases correctly.

## View Hierarchy

```
ContentView (NavigationStack)
├── HomeView
│   ├── OperationSelectorView    (toggle buttons for +, -, ×, ÷)
│   ├── DifficultyPickerView     (easy/medium/hard segmented control)
│   └── StartButton
│
├── GameView
│   ├── TimerBarView             (animates timeRemaining / totalTime)
│   ├── ScoreHeaderView          (score + question counter)
│   ├── QuestionCardView         (large question text)
│   ├── AnswerDisplayView        (shows digits as user types)
│   ├── FeedbackOverlayView      (correct/incorrect flash)
│   └── NumberPadView            (4x3 grid: 1-9, 0, backspace, submit)
│
├── ResultsView
│   ├── ScoreSummaryView         (score, accuracy, time)
│   ├── StreakBadgeView
│   └── ActionButtonsView        (play again, home, view stats)
│
└── StatsView
    ├── StreakHeaderView
    ├── AllTimeStatsView         (total sessions, average accuracy)
    └── SessionHistoryList       (@Query-fed list of GameSession)
```

## Number Pad Design

Build `NumberPadView` as a pure SwiftUI grid of `Button` views — do not use system keyboard. Reasons:
1. System `.numberPad` keyboard has no submit/enter key by default
2. Custom pad gives full control over layout and button sizing (critical for kid-friendly large targets)
3. Avoids UIKit workarounds needed to dismiss the system keyboard

The pad sends actions upward via a callback or directly into the ViewModel:
- Digit tapped → append to answer string
- Backspace → remove last character
- Submit → call `viewModel.submitAnswer()`

Cap answer string at 4 characters to prevent absurd inputs.

## Anti-Patterns to Avoid

### Anti-Pattern 1: Putting Business Logic in Views
**What goes wrong:** Timer logic, score calculation, and answer checking end up in SwiftUI View structs
**Why bad:** SwiftUI recreates view bodies frequently; logic fires unexpectedly; untestable
**Instead:** All logic lives in GameViewModel; views only read state and dispatch actions

### Anti-Pattern 2: Using ObservableObject on iOS 17+ targets
**What goes wrong:** Unnecessary `@Published` boilerplate; whole-object invalidation instead of property-level tracking; stale documentation patterns
**Instead:** Use `@Observable` macro; more performant, less code

### Anti-Pattern 3: Saving GameSession in the Middle of a Session
**What goes wrong:** Partial sessions appear in stats; cascade deletes cause complexity if user quits mid-game
**Instead:** Accumulate `[QuestionAttempt]` (value types, in-memory) during play; write one `GameSession` + all `QuestionResult` records atomically on completion or explicit quit

### Anti-Pattern 4: Storing Timer State in SwiftData
**What goes wrong:** Unnecessary persistence I/O on every tick; awkward model lifecycle
**Instead:** Timer state is 100% transient; lives only in GameViewModel; never persisted

### Anti-Pattern 5: Raw TimeInterval for Streak Calculation
**What goes wrong:** 23:59 Monday to 00:01 Tuesday is 2 minutes (< 86400 seconds) — raw math says "same day"; but midnight roll-over from 11:59 PM to 12:01 AM next day correctly counts as consecutive
**Instead:** Always use `Calendar.current.isDateInYesterday()` and `Calendar.current.isDateInToday()`

### Anti-Pattern 6: NavigationLink inside GameView for Results
**What goes wrong:** Back navigation appears, child taps it, loses session data
**Instead:** Use programmatic navigation via `navigationPath` in a root NavigationStack; GameViewModel signals completion; root transitions to ResultsView with completed session data passed explicitly

## Suggested Build Order

Dependencies flow downward — build lower layers before what depends on them.

```
Layer 1 — Models and Data (no dependencies)
  1. MathOperation enum + MathQuestion struct
  2. GameConfig + TimerMode + Difficulty enums
  3. QuestionGenerator service
  4. GameSession + QuestionResult SwiftData models
  5. StreakData + StreakManager

Layer 2 — Core Logic (depends on Layer 1)
  6. GameViewModel (timer, question sequencing, scoring, session finalization)
     — wire to QuestionGenerator and ModelContext
     — testable without any views

Layer 3 — Leaf Views (depends on Layer 1 only, no ViewModel)
  7. NumberPadView (stateless, callback-based)
  8. TimerBarView (takes progress: Double)
  9. QuestionCardView (takes question: MathQuestion)
  10. FeedbackOverlayView (takes feedbackState)

Layer 4 — Screen Views (assemble Layers 2+3)
  11. GameView (connects GameViewModel to leaf views)
  12. HomeView + HomeViewModel (operation/difficulty selection)
  13. ResultsView (reads from completed GameSession)
  14. StatsView (@Query for history, StreakManager for streak)

Layer 5 — Navigation Shell
  15. ContentView with NavigationStack and path management
  16. App entry point with ModelContainer setup
```

**Why this order:**
- GameViewModel can be built and unit-tested before any views exist
- Leaf views are independently composable; Xcode Previews work immediately
- Stats and persistence tested with real SwiftData before wiring navigation
- Navigation is last because it requires all screens to exist

## Scalability Considerations

| Concern | Now (v1) | If adding difficulty levels | If adding multiple kids/profiles |
|---------|----------|----------------------------|----------------------------------|
| Question storage | Generated in-memory | Same; seed params change | Same |
| Session history | All in one SwiftData store | Add difficulty field to GameSession | Add Profile @Model; scope @Query with predicate |
| Streak | Single StreakData in @AppStorage | Same | Per-profile streaks → move to SwiftData |
| Stats queries | Fetch all, compute in-memory | Filter by difficulty in @Query | Filter by profile in @Query |

v1 design deliberately avoids profile complexity. The `GameSession` model should include a `profileID` field (optional, unused) as a forward-looking stub — cheap to add now, expensive to migrate later.

## Sources

- [@Observable vs ObservableObject (Medium, Nadeem Ali)](https://medium.com/@nadeem.ali/observable-vs-observableobject-what-swiftui-developers-need-to-know-0a8e83f33cc9) — MEDIUM confidence (verified against Apple iOS 17 release)
- [iOS 17+ SwiftUI State Management 2025 (Medium, YLabZ)](https://zoewave.medium.com/new-swiftui-state-management-3a6c9b737724) — MEDIUM confidence
- [SwiftData Architecture Patterns and Practices (AzamSharp, 2025)](https://azamsharp.com/2025/03/28/swiftdata-architecture-patterns-and-practices.html) — HIGH confidence (respected Swift community author, current date)
- [Designing a Daily Streak System in Swift (Luke Roberts)](https://blog.lukeroberts.co/posts/streak-system/) — HIGH confidence (code-level detail, Calendar API usage verified)
- [@AppStorage vs UserDefaults vs SwiftData (BleepingSwift)](https://bleepingswift.com/blog/appstorage-vs-userdefaults-vs-swiftdata) — MEDIUM confidence
- [SwiftUI Timer (Hacking with Swift)](https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-a-timer-with-swiftui) — HIGH confidence (Paul Hudson, authoritative Swift resource)
- [MVVM in SwiftUI for a Better Architecture (Matteo Manferdini)](https://matteomanferdini.com/swiftui-mvvm/) — MEDIUM confidence
