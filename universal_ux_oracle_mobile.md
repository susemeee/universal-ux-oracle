# 📱 Generic Mobile App --- Universal UX Oracle Specification (v1.0)

## Purpose

Define a **universal UX oracle** for AI QA agents that detects UX issues
in mobile apps *without feature-specific test cases*. The goal is to
approximate human expectations about responsiveness, interaction
correctness, and usability.

------------------------------------------------------------------------

# 1. Design Philosophy

## Traditional QA Problems

-   Requires manual test-case writing
-   High maintenance cost
-   Poor detection of UX-quality regressions
-   Focused on correctness, not experience

## Universal UX Oracle Approach

Instead of testing features, we test **human interaction invariants**.

> If a human expects something to happen, the system must acknowledge it
> quickly and clearly.

Key Principles: - Feature-agnostic - Signal-driven - Evidence-first -
Timing-aware - Exploration-friendly

------------------------------------------------------------------------

# 2. Human Interaction Model

Every user interaction follows:

    Intent → Input → ACK → Progress → DONE

### Definitions

  Stage      Meaning
  ---------- ------------------------
  Intent     User expectation
  Input      Tap / gesture / typing
  ACK        Immediate feedback
  Progress   Visible processing
  DONE       Final state achieved

Failures occur when transitions break.

------------------------------------------------------------------------

# 3. Oracle Categories

## 3.1 Input Acknowledgement Oracle (ACK Oracle)

### Rule

User input must produce observable feedback quickly.

### Budget

-   Ideal: ≤100ms
-   Acceptable: ≤300ms
-   Warning: ≤1s
-   Fail: \>1s

### Acceptable ACK Signals

-   Visual change
-   Button disabled
-   Spinner appears
-   Navigation transition
-   Network request start
-   Haptic feedback
-   Focus change

### Failure Types

-   Ghost tap
-   Frozen UI
-   Ignored key press
-   Double-submit dependency

------------------------------------------------------------------------

## 3.2 Extra-Action Dependency Oracle

Detects cases like: \> "User must press twice"

### Method (Metamorphic Testing)

Execute variants: - tap once - tap twice - delayed tap - long press

If only modified input succeeds → FAIL.

------------------------------------------------------------------------

## 3.3 Responsiveness Oracle

Measures perceived responsiveness.

### Metrics

-   Input latency
-   First visual change
-   Animation start delay
-   Event loop stalls

### Thresholds

-   Input reaction \>300ms → degraded

-   1s → unresponsive

------------------------------------------------------------------------

## 3.4 Focus & Input Routing Oracle

Checks: - correct input receiver - keyboard focus validity - gesture
ownership

Failures: - keyboard appears but typing ignored - tap captured by
overlay - first input activates focus only

------------------------------------------------------------------------

## 3.5 Navigation Stability Oracle

Detect: - double navigation - accidental back jumps - transition
interruption

Signals: - route change timestamps - animation lifecycle

------------------------------------------------------------------------

## 3.6 Async Transparency Oracle

Long operations must show progress.

Rules: - \>1s operation requires feedback - \>3s requires progress
indicator

Fail if silent waiting occurs.

------------------------------------------------------------------------

## 3.7 Gesture Consistency Oracle

Validates: - swipe predictability - scroll inertia - gesture
cancellation

Detect: - inconsistent thresholds - accidental triggers

------------------------------------------------------------------------

## 3.8 Visual Stability Oracle

Detect layout instability.

Metrics: - layout shift magnitude - tap target displacement - animation
jitter

------------------------------------------------------------------------

## 3.9 Error Feedback Oracle

Errors must be: - visible - actionable - timely

Fail if: - silent failure - retry unclear - state unchanged without
explanation

------------------------------------------------------------------------

# 4. Required Instrumentation (Minimal)

Recommended lightweight signals:

    app_event("action_started", name)
    app_event("action_ack", name)
    app_event("action_done", name)

Optional: - network start/end - animation start/end - screen transitions

------------------------------------------------------------------------

# 5. Evidence Collection

Each failure must attach:

-   timeline (ms precision)
-   screenshots/video
-   UI tree snapshot
-   network log
-   focus state
-   gesture metadata

Example:

    t=0   tap submit
    t=420 no UI change
    t=1120 spinner appears
    → ACK timeout FAIL

------------------------------------------------------------------------

# 6. Scoring Model

UX Score per action:

    UX Score =
      ACK score (30)
    + Responsiveness (25)
    + Stability (15)
    + Feedback clarity (15)
    + Interaction consistency (15)

Grades: - 90--100 Excellent - 75--89 Good - 60--74 Degraded - \<60 UX
Failure

------------------------------------------------------------------------

# 7. AI Agent Responsibilities

## Agent SHOULD

-   explore UI graph
-   vary inputs
-   minimize repro steps
-   collect signals
-   explain failures

## Agent SHOULD NOT

-   guess correctness
-   rely on screenshots alone
-   invent expectations

------------------------------------------------------------------------

# 8. Exploration Strategy

Agent builds a behavior graph:

Nodes: UI states\
Edges: actions

Exploration: - breadth-first initially - anomaly-driven deep
exploration - metamorphic retries on failures

------------------------------------------------------------------------

# 9. Minimal MVP Deployment

1.  Enable interaction logging
2.  Implement ACK oracle
3.  Run automated exploration nightly
4.  Store evidence bundles
5.  Human review weekly

------------------------------------------------------------------------

# 10. Expected Outcomes

Detect automatically: - double taps required - delayed UI feedback -
frozen interactions - misleading progress states - gesture conflicts -
navigation glitches

Without: - writing test cases - defining expected UI results

------------------------------------------------------------------------

# 11. Future Extensions

-   Learned UX baseline per app
-   Personalized responsiveness expectations
-   Cross-version regression comparison
-   Reinforcement learning exploration

------------------------------------------------------------------------

# END
