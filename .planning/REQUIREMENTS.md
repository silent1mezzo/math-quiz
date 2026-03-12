# Requirements: Math Quiz

**Defined:** 2026-03-12
**Core Value:** Kids practice math facts quickly and repeatedly, with just enough game-feel to keep them coming back.

## v1 Requirements

### Core Game Loop

- [ ] **GAME-01**: User can select which operations to practice (addition, subtraction, multiplication, division — any combination)
- [ ] **GAME-02**: App presents one math problem at a time with operands visible
- [ ] **GAME-03**: User enters answer using a custom in-app number pad (not system keyboard)
- [ ] **GAME-04**: Number pad buttons are minimum 70pt for reliable kid touch targets
- [ ] **GAME-05**: App shows correct/incorrect feedback immediately after answer is submitted (visual + haptic)
- [ ] **GAME-06**: App plays a sound effect on correct answer and a different sound on incorrect answer
- [ ] **GAME-07**: Elapsed time is displayed during session (how long the session has been running)
- [ ] **GAME-08**: Current score is displayed prominently during session
- [ ] **GAME-09**: Speed-based bonus points are awarded for fast correct answers

### Session Configuration

- [ ] **SESS-01**: User can set a manual number range for operands (e.g., numbers 1–12 for multiplication tables)
- [ ] **SESS-02**: User can choose session length by problem count (10 / 25 / 50 problems)
- [ ] **SESS-03**: User can choose session length by time (1 / 3 / 5 minutes)
- [ ] **SESS-04**: Sound can be muted/unmuted at any time, preference persists across sessions

### Post-Session Summary

- [ ] **SUMM-01**: After session ends, user sees score and accuracy percentage
- [ ] **SUMM-02**: After session ends, user sees a review of all missed problems with the correct answers
- [ ] **SUMM-03**: After session ends, user sees their current streak and any milestone achieved

### Streaks & Statistics

- [ ] **STAT-01**: App tracks and saves a daily practice streak (consecutive days practiced)
- [ ] **STAT-02**: Streak updates correctly across midnight boundaries and timezone changes
- [ ] **STAT-03**: App stores session history (date, score, accuracy, operation, duration) persistently
- [ ] **STAT-04**: User can view a stats screen showing streak, total problems solved, overall accuracy, and session history
- [ ] **STAT-05**: App celebrates milestone moments (7-day streak, first perfect session, 100 problems solved) with a brief celebratory screen

### Notifications

- [ ] **NOTF-01**: App sends a daily local notification to remind the user to practice
- [ ] **NOTF-02**: User can enable or disable the daily reminder notification

### Technical Foundation

- [ ] **TECH-01**: App targets iOS 17 minimum
- [ ] **TECH-02**: SwiftData persistence is wrapped in VersionedSchema from the initial implementation
- [ ] **TECH-03**: App includes PrivacyInfo.xcprivacy manifest declaring UserDefaults usage
- [ ] **TECH-04**: No third-party SDKs that transmit device identifiers (Kids Category compliance)

## v2 Requirements

### Smart Practice

- **SMRT-01**: App tracks per-problem accuracy and response time to weight future question generation toward weak spots
- **SMRT-02**: User can see which specific facts (e.g., 7×8) they struggle with most

### Social & Progress

- **PROG-01**: Personal bests tracking (fastest session, longest streak ever, best accuracy)
- **PROG-02**: Multiple user profiles for family devices

### Extended Configuration

- **CONF-01**: Separate difficulty settings per operation
- **CONF-02**: Custom session length (arbitrary problem count or time)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Ads / in-app purchases | Top parent complaint in this genre; ruins trust |
| Third-party analytics | Kids Category compliance prohibits them |
| Multiplayer / leaderboards | Backend complexity, out of scope for solo drill app |
| Parent / teacher dashboards | Requires accounts + backend + COPPA compliance |
| Social sharing | Adds complexity, social pressure on kids — not appropriate |
| Countdown timer (per-session or per-question) | User chose elapsed-only — no time pressure |
| Preset difficulty levels (Easy/Medium/Hard) | User chose manual number range instead |
| System number pad keyboard | Wrong UX for math drills; custom pad required |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| GAME-01 | Phase 1 | Pending |
| GAME-02 | Phase 1 | Pending |
| GAME-03 | Phase 1 | Pending |
| GAME-04 | Phase 1 | Pending |
| GAME-05 | Phase 1 | Pending |
| GAME-06 | Phase 1 | Pending |
| GAME-07 | Phase 1 | Pending |
| GAME-08 | Phase 1 | Pending |
| GAME-09 | Phase 1 | Pending |
| SESS-01 | Phase 1 | Pending |
| SESS-02 | Phase 1 | Pending |
| SESS-03 | Phase 1 | Pending |
| SESS-04 | Phase 1 | Pending |
| SUMM-01 | Phase 1 | Pending |
| SUMM-02 | Phase 1 | Pending |
| SUMM-03 | Phase 1 | Pending |
| STAT-01 | Phase 2 | Pending |
| STAT-02 | Phase 2 | Pending |
| STAT-03 | Phase 2 | Pending |
| STAT-04 | Phase 2 | Pending |
| STAT-05 | Phase 2 | Pending |
| NOTF-01 | Phase 2 | Pending |
| NOTF-02 | Phase 2 | Pending |
| TECH-01 | Phase 1 | Pending |
| TECH-02 | Phase 1 | Pending |
| TECH-03 | Phase 1 | Pending |
| TECH-04 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 27 total
- Mapped to phases: 27
- Unmapped: 0 ✓

---
*Requirements defined: 2026-03-12*
*Last updated: 2026-03-12 after initial definition*
