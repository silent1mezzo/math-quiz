---
phase: 1
slug: foundation-and-core-game-loop
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-12
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Swift Testing (Xcode 16 built-in) + XCTest for UI tests |
| **Config file** | None — Swift Testing is built into Xcode 16 |
| **Quick run command** | `xcodebuild test -scheme "Math Quiz" -destination "platform=iOS Simulator,name=iPhone 16" -only-testing "Math QuizTests"` |
| **Full suite command** | `xcodebuild test -scheme "Math Quiz" -destination "platform=iOS Simulator,name=iPhone 16"` |
| **Estimated runtime** | ~30 seconds (unit tests only) / ~90 seconds (full suite) |

---

## Sampling Rate

- **After every task commit:** Run quick unit test command (unit tests only)
- **After every plan wave:** Run full suite including UI tests
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** ~30 seconds (unit), ~90 seconds (full)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| GAME-01 | 01 | 1 | GAME-01 | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testOperationSelection"` | ❌ Wave 0 | ⬜ pending |
| GAME-02 | 01 | 1 | GAME-02 | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testSessionProducesQuestion"` | ❌ Wave 0 | ⬜ pending |
| GAME-03/04 | 01 | 1 | GAME-03, GAME-04 | UI | `xcodebuild test ... -only-testing "Math QuizUITests/NumberPadUITests/testPadButtonMinimumSize"` | ❌ Wave 0 | ⬜ pending |
| GAME-05 | 01 | 2 | GAME-05 | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testFeedbackStateTransition"` | ❌ Wave 0 | ⬜ pending |
| GAME-06 | 01 | 2 | GAME-06 | unit | `xcodebuild test ... -only-testing "Math QuizTests/SoundPlayerTests"` | ❌ Wave 0 | ⬜ pending |
| GAME-07 | 01 | 2 | GAME-07 | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testElapsedSecondsIncrement"` | ❌ Wave 0 | ⬜ pending |
| GAME-08 | 01 | 3 | GAME-08 | UI | `xcodebuild test ... -only-testing "Math QuizUITests/GameViewUITests/testScoreVisible"` | ❌ Wave 0 | ⬜ pending |
| GAME-09 | 01 | 2 | GAME-09 | unit | `xcodebuild test ... -only-testing "Math QuizTests/GameViewModelTests/testSpeedBonusPoints"` | ❌ Wave 0 | ⬜ pending |
| TECH-02 | 01 | 1 | TECH-02 | unit | `xcodebuild test ... -only-testing "Math QuizTests/SchemaTests/testModelContainerInitialization"` | ❌ Wave 0 | ⬜ pending |
| DIV-ZER | 01 | 1 | GAME-02 | unit | `xcodebuild test ... -only-testing "Math QuizTests/QuestionGeneratorTests/testNoDivisionByZero"` | ❌ Wave 0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `Math QuizTests/GameViewModelTests.swift` — stubs for GAME-01, GAME-02, GAME-05, GAME-07, GAME-09
- [ ] `Math QuizTests/QuestionGeneratorTests.swift` — stubs for GAME-02 (question validity), division-by-zero check
- [ ] `Math QuizTests/SoundPlayerTests.swift` — stubs for GAME-06
- [ ] `Math QuizTests/SchemaTests.swift` — stubs for TECH-02 (ModelContainer init)
- [ ] `Math QuizUITests/NumberPadUITests.swift` — stubs for GAME-03, GAME-04
- [ ] `Math QuizUITests/GameViewUITests.swift` — stubs for GAME-08
- [ ] Verify test targets exist and compile after SpriteKit deletion

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Haptic feedback fires on correct answer | GAME-05 | `.sensoryFeedback()` cannot be detected programmatically | Run on physical device; submit correct answer; verify vibration |
| Elapsed time pauses correctly on background | GAME-07 | Timer background behavior requires real app lifecycle | Submit correct answer, double-press Home, wait 30s, return — elapsed time should advance by ~30s |
| Number pad touch targets feel natural for kids | GAME-04 | Subjective ergonomics; requires human tester | Test with a 7-8 year old or simulate with full finger width on device |
| Sound plays audibly | GAME-06 | Audio output cannot be automated in Simulator | Test on physical device with volume up; verify distinct correct/incorrect sounds |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s (unit) / 90s (full)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
