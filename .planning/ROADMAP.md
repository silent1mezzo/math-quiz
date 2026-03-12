# Roadmap: Math Quiz

## Overview

Math Quiz delivers a kid-friendly iOS math drill app in three phases. Phase 1 establishes the technical foundation and builds a fully playable game session with a custom number pad, immediate feedback, and in-session scoring. Phase 2 wraps sessions with configuration options and a post-session summary so kids can choose what to practice and review what they missed. Phase 3 wires in persistent statistics, daily streaks, milestone celebrations, and reminder notifications so the app gives kids a reason to return every day.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Foundation and Core Game Loop** - Playable drill session with custom number pad, feedback, and in-session scoring
- [ ] **Phase 2: Session Configuration and Summary** - Operation/length/range setup before play and a full post-session review screen
- [ ] **Phase 3: Persistence, Streaks, and Notifications** - Saved stats, daily streak tracking, milestone celebrations, and reminder notifications

## Phase Details

### Phase 1: Foundation and Core Game Loop
**Goal**: Kids can play a complete math drill session with immediate feedback and see their score climb
**Depends on**: Nothing (first phase)
**Requirements**: TECH-01, TECH-02, TECH-03, TECH-04, GAME-01, GAME-02, GAME-03, GAME-04, GAME-05, GAME-06, GAME-07, GAME-08, GAME-09
**Success Criteria** (what must be TRUE):
  1. User can select one or more operations (addition, subtraction, multiplication, division) and a session starts
  2. A math problem appears on screen and the user can enter their answer using the in-app number pad with no system keyboard visible
  3. After submitting an answer, the app immediately shows a visual color response and plays a distinct sound — correct answers feel rewarding, incorrect answers feel clearly different
  4. The elapsed session time and current score are both visible throughout the session
  5. A fast correct answer awards bonus points, creating a visible score difference compared to a slow correct answer
**Plans**: 7 plans

Plans:
- [ ] 01-00-PLAN.md — Test stub scaffolding (Wave 0: all failing stubs before implementation)
- [ ] 01-01-PLAN.md — Project foundation: delete SpriteKit, SwiftUI App, SwiftData schema, PrivacyInfo, Pow
- [ ] 01-02-PLAN.md — Models and QuestionGenerator: MathOperation, MathQuestion, GameConfig, FeedbackState
- [ ] 01-03-PLAN.md — GameViewModel: @Observable session state, scoring, timer, answer validation
- [ ] 01-04-PLAN.md — SoundPlayer: AVAudioPlayer service with correct/incorrect audio assets
- [ ] 01-05-PLAN.md — Leaf views: NumberPadView, QuestionView, ScoreHeaderView
- [ ] 01-06-PLAN.md — Screens and navigation: GameView, OperationPickerView, ContentView wiring + human verify

### Phase 2: Session Configuration and Summary
**Goal**: Kids (and parents) can tune what gets practiced before a session and review exactly what was missed after
**Depends on**: Phase 1
**Requirements**: SESS-01, SESS-02, SESS-03, SESS-04, SUMM-01, SUMM-02, SUMM-03
**Success Criteria** (what must be TRUE):
  1. Before starting, user can set the operand number range (e.g., 1–12 for times tables)
  2. Before starting, user can choose session length by problem count (10 / 25 / 50) or by time (1 / 3 / 5 minutes)
  3. Sound can be toggled on/off at any time during or outside a session, and the preference persists after closing the app
  4. After a session ends, user sees their final score and accuracy percentage on a summary screen
  5. After a session ends, user can see every problem they got wrong with the correct answer shown
**Plans**: TBD

### Phase 3: Persistence, Streaks, and Notifications
**Goal**: The app remembers every session and uses that history to celebrate consistency and pull kids back daily
**Depends on**: Phase 2
**Requirements**: STAT-01, STAT-02, STAT-03, STAT-04, STAT-05, NOTF-01, NOTF-02
**Success Criteria** (what must be TRUE):
  1. After closing and reopening the app, session history (date, score, accuracy, operation, duration) is still visible on the stats screen
  2. The daily streak increments on the first session of a new calendar day and resets correctly if a day is skipped — including across midnight and timezone changes
  3. A stats screen shows the current streak, total problems solved, overall accuracy, and a list of past sessions
  4. Reaching a milestone (7-day streak, first perfect session, 100 problems solved) triggers a brief celebratory screen
  5. A daily local notification reminds the user to practice, and the user can enable or disable it from within the app
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation and Core Game Loop | 0/7 | Not started | - |
| 2. Session Configuration and Summary | 0/TBD | Not started | - |
| 3. Persistence, Streaks, and Notifications | 0/TBD | Not started | - |
