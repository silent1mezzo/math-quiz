# Math Quiz

## What This Is

Math Quiz is an iOS app built in Swift that helps kids practice math through drill-style questions. Players see a math problem, type their answer on a number pad, and get immediate feedback — all against a timer with scores and streaks tracked over time.

## Core Value

Kids practice math facts quickly and repeatedly, with just enough game-feel (timer, score, streaks) to keep them coming back.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can choose which operation(s) to practice (addition, subtraction, multiplication, division)
- [ ] App presents a math problem and user types the answer using a number pad
- [ ] A timer counts down per question or per session
- [ ] User earns a score based on correct answers
- [ ] App tracks and saves statistics (correct answers, accuracy, streaks) across sessions
- [ ] Streaks are maintained for daily practice
- [ ] App provides immediate feedback on correct/incorrect answers

### Out of Scope

- Multiplayer / head-to-head — focus is solo practice
- Teacher/parent dashboard — v1 is self-contained for the child
- In-app purchases or ads — keep it simple for v1
- SpriteKit game engine — switching to SwiftUI for cleaner UI and data management

## Context

- Xcode project already created with default SpriteKit template — will be replaced with SwiftUI
- iOS target, Swift
- Audience: kids (approx ages 6-12), so UI should be clear, large touch targets, encouraging feedback
- User has no framework preference; SwiftUI chosen for number pad input, statistics, and data persistence advantages over SpriteKit

## Constraints

- **Tech Stack**: iOS/Swift with SwiftUI — no cross-platform requirement
- **Data**: Persist stats locally on device (no backend/cloud sync for v1)
- **UI**: Kid-friendly — large text, big buttons, clear feedback, no complex navigation

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| SwiftUI over SpriteKit | SpriteKit is for physics/animation games; SwiftUI handles forms, number pads, and data far better | — Pending |
| Local persistence (SwiftData or UserDefaults) | No backend needed for v1, simpler to ship | — Pending |
| Number pad input (not multiple choice) | User explicitly requested typed answers for better learning reinforcement | — Pending |

---
*Last updated: 2026-03-12 after initialization*
