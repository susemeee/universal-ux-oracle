# Prompt: Update UX Oracle Spec with Task-Oriented Testing Philosophy

You are updating an existing "Universal UX Oracle Specification" document.
The current version evaluates UX quality at the **individual action level**
(tap, swipe, type). This is necessary but insufficient.

## Core Update Requirement

Add a **Task-Oriented Testing** layer on top of the existing action-level
oracles. The key insight is:

> **Human UX testing must be scoped to TASKS (end-to-end goals),
> not FEATURES (isolated functions).**

## Definitions

- **Feature**: A single capability (e.g., login, text input, button click)
- **Task**: A goal-directed sequence of actions spanning multiple features
  that produces a meaningful outcome for the user
  (e.g., "create a document, edit it, and export as PDF")

## What to update

### 1. Philosophy Section

- Add a new subsection explaining why task-oriented testing is superior
  to feature-unit testing
- Include anti-patterns:
  · "Login-and-stop" — testing login in isolation
  · "Single-prompt-and-stop" — sending one AI prompt and calling it done
  · "Button-click-and-stop" — verifying a click without following through
- Emphasize: always think about the system's ULTIMATE GOAL from the
  user's perspective

### 2. Interaction Model

- Extend the existing `Intent → Input → ACK → Progress → DONE` model
  to support multi-phase tasks:

```
Task = [Phase₁ → Phase₂ → ... → Phaseₙ] → Goal Achieved
Phase = [Action₁ → Action₂ → ... → Actionₘ] → Phase Outcome
Action = Intent → Input → ACK → Progress → DONE
```

- A task is only "done" when the user's real-world goal is met

### 3. Oracle Enhancements

- For EACH existing oracle (ACK, Extra-Action, Focus, Navigation,
  Async, Gesture, Visual Stability, Error, Accessibility, Content,
  Platform, Performance), add a "Task-level interpretation" that
  explains how the oracle applies across the full task journey,
  not just a single action
- Add new "Task Continuity" checks:
  · State preservation across phases
  · Progress persistence (auto-save, crash recovery)
  · Mode transition smoothness (AI ↔ GUI switching)
  · Context carry-over (AI and manual edits don't conflict)
  · Goal reachability (the final step actually works)

### 4. Exploration Strategy

- Update the exploration strategy to be task-first:
  · Phase 1: Discover what tasks the app supports
  · Phase 2: Execute each task end-to-end (happy path)
  · Phase 3: Execute with variations (different input modes,
  interruptions, errors)
  · Phase 4: Cross-task interference testing
  · Phase 5: Edge cases (long tasks, large data, rapid switching)
- Add a "Task Coverage" metric alongside screen coverage
- The agent should ask: "What would the user do NEXT?" at every step,
  and only stop when the goal is reached or clearly blocked

### 5. Scoring Model

- Add task-level scoring with phase weighting:
  · Later phases (refinement, finalization) get higher weight
  because user investment is greater and failure cost is higher
  · A task that fails at the last step should score LOWER than
  one that fails at the first step (sunk cost principle)
- Add task completion rate as a top-level metric

### 6. Reporting

- Add a "Task Journey" section to reports that shows the full
  task timeline with per-phase verdicts
- Include "Human Impact Statements" that describe the user's
  experience across the entire task, not just individual actions

### 7. Anti-Pattern Detection

- The agent should detect and flag when testing falls into
  feature-unit traps:
  · Dead-end screens with no forward path
  · Orphan features disconnected from any task
  · Premature "pass" verdicts after a single action
  · Missing "final mile" (last step of a task is broken)

## Constraints

- Do NOT remove or weaken existing action-level oracles — they
  remain the foundation
- Task-level evaluation is an ADDITIONAL layer that uses
  action-level verdicts as building blocks
- Keep the spec format consistent with the existing document
- All new content must maintain the "AI agent + timing device"
  architecture assumption
- Write in the same language as the existing document sections
  (Korean for Korean sections, English for English sections)

## Multi-Modal Task Awareness

Many modern apps combine AI and GUI interactions. The spec must
account for these patterns:

| Pattern     | Description                                     |
| ----------- | ----------------------------------------------- |
| AI-then-GUI | Prompt AI → get output → manually refine in GUI |
| GUI-then-AI | Create manually → use AI to enhance/polish      |
| Interleaved | Alternate between AI prompts and manual edits   |
| Pure GUI    | Traditional click/type interaction only         |
| Pure AI     | AI does everything, user only reviews           |

The oracle must verify that transitions BETWEEN these modes are
seamless and that user work is NEVER lost when switching modes.

## Output

Produce the updated specification as a single Markdown document,
preserving all existing sections and integrating the new
task-oriented content naturally. Mark new or significantly
changed sections with a "(Updated)" or "(New)" tag.
