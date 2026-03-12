# Feature Landscape

**Domain:** Kids math drill iOS app (ages 6-12, typed number input, solo practice)
**Researched:** 2026-03-12
**Confidence:** HIGH for table stakes / MEDIUM for differentiators

---

## Table Stakes

Features users expect. Missing = product feels incomplete or earns negative reviews.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Operation selection (+ - × ÷) | Every competitor offers it; defines the practice scope | Low | Already in requirements. Single ops or combined. |
| Immediate right/wrong feedback | Core learning loop; kids disengage without instant response | Low | Visual + sound. Wrong answers need correction, not just "X". |
| Score display during session | Kids watch the number go up; it is the motivational engine | Low | Show running total prominently during play. |
| Per-session timer | Creates urgency and pacing; missing makes drills feel formless | Low | Countdown or elapsed — countdown creates more tension. |
| Custom number pad (no system keyboard) | System keyboard is too small, wrong layout for math; every drill app builds its own | Medium | Large touch targets (min 44×44 pt per HIG, 60+ recommended for kids). |
| Encouraging positive feedback | Kids' apps require positive reinforcement; punitive tone causes abandonment | Low | Sound effects, color change, animation on correct. Avoid harsh sounds on wrong. |
| Stats across sessions | Parents and kids both want to see improvement over time | Medium | Accuracy %, total problems solved, correct count. Persist with SwiftData. |
| Daily streak tracking | Streak psychology drives return visits; Duolingo data shows 7-day streak = 2.4× retention | Medium | Requires date-aware persistence, streak break detection. |
| Operation difficulty levels | Basic (0-10), intermediate (0-20), advanced (multi-digit) — unlocking harder facts is expected | Medium | Per-operation or global. Affects problem generation range. |
| Session length choice | Kids have different attention spans; parents want control | Low | Short (10 problems), medium (25), long (50) or time-based options. |

---

## Differentiators

Features that set the product apart. Not universally expected, but create loyalty and word-of-mouth.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Smart problem weighting | Re-surface facts the child got wrong or answered slowly — targeted practice beats random drilling | High | Track per-fact accuracy + response time. Weight generation algorithm toward weak spots. Depends on: per-problem stats tracking. |
| Speed-based scoring | Bonus points for fast correct answers rewards both accuracy and fluency — more game-like than a raw count | Low-Med | Multiply score by time remaining fraction or add speed bonus. Creates excitement without over-rewarding rushing. |
| Personal bests / records | "Beat your best streak" or "fastest 20-problem run" drives replay without social features | Low-Med | Requires historical session storage. Depends on: session history. |
| Celebratory milestone moments | Crossing 100 correct answers, 7-day streak, first perfect session — brief celebrations feel earned | Low | SwiftUI animations + haptics. Duolingo-style milestone screen. |
| Problem set review after session | Show which problems were missed with the correct answer — closes the learning loop | Low-Med | End-of-session summary screen. Depends on: per-problem result logging. |
| Configurable number ranges | Let parents set max operand values (e.g., multiplication up to 12×12 only) | Low-Med | Useful for aligning with grade-level curriculum without requiring full adaptive engine. |
| Sound toggle | Kids use apps in school, in cars, at bedtime; mute is expected by parents | Low | Persist preference. Should be instantly accessible (not buried in settings). |

---

## Anti-Features

Features to deliberately NOT build. Each has a real cost in complexity, trust, or alignment with the project's values.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| In-app purchases / paywalls | Top complaint in App Store reviews for this genre; parents abandon apps with paywalls mid-child-session; explicitly out of scope | Ship all operations unlocked. One-time purchase at app level if monetization needed later. |
| Ads | "Billions of ads and 400 paywalls" is a recurring parent complaint; ads near number pad = accidental taps; COPPA implications for under-13 | No ads in v1. Period. |
| Subscription model | Confusing billing, hard-to-cancel, top friction point in reviews | Not applicable for v1. |
| Multiplayer / leaderboards | Adds infrastructure, privacy complexity, moderation; out of scope explicitly | Solo practice only. |
| Adaptive AI difficulty (fully automatic) | Adaptive engines require significant data and tuning to feel correct; a miscalibrated adaptive system frustrates kids more than a fixed one | Use manual difficulty tiers + smart problem weighting as a lightweight substitute. |
| Parent/teacher dashboard | Remote dashboards require accounts, backend, privacy compliance (COPPA), auth flows — out of scope for v1 | Keep all data on-device. Optional share/export could come later. |
| Countdown timers that punish time-out with "wrong" | Marking a question wrong on timeout when no answer was given is harsh; kills session morale | On timeout: advance to next question, count as skipped, not wrong. Or just count elapsed time. |
| Overly complex navigation | Kids ages 6-12 cannot handle deep menus; competitive apps fail when navigation requires more than 2 taps to start drilling | Max 2 taps from launch to first question. Settings accessible but not in the critical path. |
| Onboarding gatekeeping | Long tutorials or mandatory sign-up before first question cause abandonment | First launch: operation select → play. No account required. |
| Social sharing of scores | Adds complexity and creates social pressure on kids; not appropriate for this age + context | Skip. |
| Multiple user profiles | Adds UI complexity and profile selection friction; most drill apps in this category don't need it for v1 | Single device user. Can add profiles in v2 if family use case validates. |

---

## Feature Dependencies

```
Per-problem result logging
    → Post-session review screen
    → Smart problem weighting (weak-spot targeting)
    → Personal bests / records

Session history storage
    → Daily streak
    → Personal bests / records
    → Stats across sessions

Custom number pad
    → All game modes (everything feeds through it)

Timer (per question or per session)
    → Speed-based scoring (needs time value)
    → Session length choice
```

---

## MVP Recommendation

**Prioritize (must ship):**

1. Operation selection (addition, subtraction, multiplication, division)
2. Custom number pad with large touch targets
3. Immediate feedback (correct/incorrect, sound + color)
4. Per-session timer (countdown)
5. Score tracking during session
6. End-of-session summary (score, accuracy, time)
7. Daily streak with persistence
8. Stats history across sessions

**Add early (high value, low cost):**

9. Sound toggle (parent-friendly)
10. Session length choice
11. Post-session problem review (what did I miss?)

**Defer (valuable but higher complexity):**

- Smart problem weighting: requires per-problem tracking infrastructure — build after basic stats are solid
- Personal bests: depends on session history being reliable
- Speed-based scoring bonus: fun but can be tuned in v2
- Configurable number ranges: useful for parents, but operation + difficulty level covers most needs initially

---

## Sources

- [Best Math Apps for Kids in 2026 — Kidslox](https://kidslox.com/guide-to/math-apps-for-kids/)
- [Best Math Apps for Kids 2026 — Brighterly](https://brighterly.com/blog/best-math-apps-for-kids/)
- [Best Math Apps for Kids 2025 Edition — Funexpected](https://funexpectedapps.com/blog-posts/best-math-apps-for-kids-2025-edition)
- [6 Best Speed Math Apps — EducationalAppStore](https://www.educationalappstore.com/best-apps/5-best-speed-maths-apps-for-children)
- [UX Design for Children: Main Principles — Eleken](https://www.eleken.co/blog-posts/ux-design-for-children-how-to-create-a-product-children-will-love)
- [Duolingo Streak Psychology — JustAnotherPM](https://www.justanotherpm.com/blog/the-psychology-behind-duolingos-streak-feature)
- [How Duolingo Streak Builds Habit — Duolingo Blog](https://blog.duolingo.com/how-duolingo-streak-builds-habit/)
- [Duolingo Gamification Secrets — Orizon](https://www.orizon.co/blog/duolingos-gamification-secrets)
- [Dark Patterns of Cuteness Research — Springer](https://link.springer.com/chapter/10.1007/978-3-031-46053-1_5)
- [Dark Patterns in Kids Apps — Fairplay for Kids](https://fairplayforkids.org/wp-content/uploads/2021/05/darkpatterns.pdf)
- [Best Math Apps for Kids 2025 — iGeeksBlog](https://www.igeeksblog.com/best-ipad-math-app-for-kids/)
