# Universal UX Oracle — AI-Driven QA for Human-Centric UI Quality

An open specification for AI agents that automatically detect UX issues in mobile apps — without writing a single test case.

## What is this?

Traditional QA asks: _"Does the feature work correctly?"_

Universal UX Oracle asks: **"Does it feel good to a real human?"**

Instead of feature-specific test scripts, we define **Human Interaction Invariants** — rules that every good UI must follow regardless of what the app does:

- Tapped a button? Something must react within 300 ms.
- Submitted a form? One tap should be enough.
- Waiting for data? Show me _why_ I'm waiting.
- Got an error? Tell me what to do next.

An AI agent explores the app autonomously, collects signals from an instrumentation layer, and judges every interaction against these universal rules.

## Core Idea

```

Intent → Input → ACK → Progress → DONE

```

Every user interaction follows this cycle. If any transition **breaks or stalls**, the user feels friction. The oracle detects exactly where and why.

## Oracle Categories

| #   | Oracle                          | Key Question                              |
| --- | ------------------------------- | ----------------------------------------- |
| 1   | **ACK**                         | Did the UI respond to my input?           |
| 2   | **Extra-Action Dependency**     | Can I do it in one tap?                   |
| 3   | **Responsiveness**              | Does it feel fast?                        |
| 4   | **Focus & Input Routing**       | Is my input going to the right place?     |
| 5   | **Navigation Stability**        | Are screen transitions predictable?       |
| 6   | **Async Transparency**          | Do I know why I'm waiting?                |
| 7   | **Gesture Consistency**         | Do gestures behave as expected?           |
| 8   | **Visual Stability**            | Does the layout stay still?               |
| 9   | **Error Feedback**              | Can I understand and recover from errors? |
| 10  | **Accessibility & Readability** | Can everyone use it?                      |
| 11  | **Content & State Consistency** | Is what I see correct and complete?       |
| 12  | **Platform Convention**         | Does it feel native to iOS / Android?     |
| 13  | **Performance Perception**      | Does it _feel_ fast (not just _be_ fast)? |

## Key Design Decisions

- **No feature knowledge required** — oracles are universal across any app
- **Signal-driven, not screenshot-guessing** — verdicts are based on measurable instrumentation data (DOM mutations, timestamps, frame timings)
- **AI can't perceive time** — a separate timing device provides millisecond-precision event logs; the AI agent only applies rules to that data
- **Every verdict includes evidence** — timeline, before/after screenshots, UI tree snapshots, and a **Human Impact Statement** explaining what the user would feel

## How It Works

```
┌──────────────┐      ┌────────────────────┐      ┌──────────────────┐
│              │      │                    │      │                  │
│  Mobile App  │─────▶│  Instrumentation   │─────▶│  AI QA Agent     │
│  (any app)   │      │  Layer             │      │                  │
│              │◀─────│  · events          │      │  · Oracle rules  │
│              │      │  · ms timestamps   │      │  · Verdicts      │
│              │      │  · frame data      │      │  · Evidence      │
│              │      │                    │      │  · Scoring       │
└──────────────┘      └────────────────────┘      └────────┬─────────┘
                                                           │
                                                           ▼
                                                  ┌──────────────────┐
                                                  │  UX Report       │
                                                  │                  │
                                                  │  Score : 78/100  │
                                                  │  Critical : 2    │
                                                  │  Major    : 5    │
                                                  │  Warning  : 12   │
                                                  └──────────────────┘
```

**Mobile App** → **Instrumentation Layer** → **AI QA Agent** → **UX Report**

1. The **Instrumentation Layer** hooks into the app and emits timestamped events
   (DOM mutations, input events, frame timings, network logs, etc.)

2. The **AI QA Agent** receives these signals, autonomously explores the app,
   and evaluates every interaction against 13 universal oracle rules.

3. A **UX Report** is generated with per-action scores, severity classifications,
   evidence bundles, and Human Impact Statements.

## Scoring

Every interaction is scored out of 100:

| Score  | Grade        | Meaning                                  |
| ------ | ------------ | ---------------------------------------- |
| 90–100 | 🟢 Excellent | Feels smooth and delightful              |
| 75–89  | 🔵 Good      | Mostly fine, minor improvements possible |
| 60–74  | 🟡 Degraded  | Users start noticing friction            |
| 40–59  | 🟠 Poor      | Obvious UX defects                       |
| < 40   | 🔴 Failure   | Users will abandon the app               |

## Quick Start (MVP)

1. **Instrument** — Add minimal event logging to your app (~1–2 days)
2. **Connect** — Wire the timing device to the AI agent
3. **Run** — Let the agent explore your app autonomously
4. **Review** — Read the UX report with scored verdicts and evidence bundles

Recommended oracle rollout:

```

Week 1: ACK + Frozen UI
Week 2: + Extra-Action + Focus Routing
Week 3: + Responsiveness + Visual Stability
Week 4: + Error Feedback + Async Transparency
Week 5+: + Accessibility, Gesture, Platform Convention, ...

```

## What You Don't Need

- ❌ Manual test case authoring
- ❌ Expected UI result definitions
- ❌ Feature-specific verification scripts
- ❌ Pixel-perfect screenshot baselines

## Documentation

See [`universal_ux_oracle_mobile.md`](./universal_ux_oracle_mobile.md) for the full specification including:

- Detailed oracle rules with thresholds and pseudocode
- Evidence collection requirements
- Scoring model with severity-to-deduction mapping
- Exploration strategy for the AI agent
- Reporting format with Human Impact Statements
- Glossary and quick-reference card

## Contributing

Contributions are welcome! Whether it's new oracle ideas, threshold refinements, or instrumentation adapters — feel free to open an issue or PR.

## License

[MIT](./LICENSE.md)
