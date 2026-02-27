# 📱 Generic Mobile App — Universal UX Oracle Specification (v2.0)

## Purpose

Define a **universal UX oracle** for AI QA agents that detects UX issues
in mobile apps _without feature-specific test cases_. The goal is to
approximate **human expectations** about responsiveness, interaction
correctness, visual quality, and overall usability — so that the app
always feels "good" from a real user's perspective.

> **핵심 전제**: AI 에이전트는 자체적으로 시간 경과를 인지하기 어렵다.
> 따라서 별도의 **타이밍 측정 장치**(instrumentation layer)가
> 밀리초 단위의 타임스탬프·이벤트 로그·프레임 데이터를 제공한다고 가정한다.
> AI는 이 데이터를 받아 오라클 규칙에 따라 **합격/불합격/경고**를 판정한다.

---

# 1. Design Philosophy

## Traditional QA Problems

- 수동 테스트 케이스 작성 필요
- 유지보수 비용이 높음
- UX 품질 퇴행(regression)을 잡아내기 어려움
- **정확성(correctness)**에만 집중, **경험(experience)**은 간과

## Universal UX Oracle Approach

기능이 아니라 **인간의 상호작용 불변식(Human Interaction Invariants)**을 테스트한다.

> 사람이 "당연히 이렇게 되겠지"라고 기대하는 것이 **빠르고, 명확하게** 실현되어야 한다.

### Key Principles

| 원칙                 | 설명                                                        |
| -------------------- | ----------------------------------------------------------- |
| Feature-agnostic     | 특정 기능이 아닌, 모든 UI에 공통 적용                       |
| Signal-driven        | 스크린샷 추측이 아닌 계측 신호 기반 판정                    |
| Evidence-first       | 모든 판정에 타임라인·증거 첨부                              |
| Timing-aware         | 외부 타이밍 장치가 제공하는 ms 데이터 활용                  |
| Exploration-friendly | 정해진 시나리오 없이 자율 탐색                              |
| **Human-centric**    | 최종 판정 기준은 "사람이 이 화면을 봤을 때 좋다고 느끼는가" |

---

# 2. Human Interaction Model

모든 사용자 상호작용은 아래 단계를 따른다:

```

Intent → Input → ACK → Progress → DONE

```

### Definitions

| Stage    | Meaning                     | 사람 관점 기대           |
| -------- | --------------------------- | ------------------------ |
| Intent   | 사용자가 머릿속에 품은 목적 | "이걸 누르면 ~가 되겠지" |
| Input    | 탭 / 제스처 / 타이핑 / 음성 | 물리적 행위              |
| ACK      | 즉각적 피드백               | "아, 눌렸구나"           |
| Progress | 처리 중 시각 표시           | "지금 하고 있구나"       |
| DONE     | 최종 상태 도달              | "됐다!"                  |

각 전환(transition)이 **끊기거나 지연**되면 사람은 불쾌감을 느낀다.

### 2-1. 전환 실패 유형 (추가)

| 끊김 구간            | 사용자 감정          | 오라클 분류             |
| -------------------- | -------------------- | ----------------------- |
| Input → ACK 없음     | "눌린 건지 모르겠다" | ACK Oracle Fail         |
| ACK → Progress 없음  | "뭘 하고 있는 거지?" | Async Transparency Fail |
| Progress → DONE 지연 | "언제 끝나?"         | Responsiveness Degraded |
| DONE 후 상태 불일치  | "뭐가 바뀐 거야?"    | Result Clarity Fail     |
| 어떤 전환도 없음     | "앱이 죽었나?"       | Frozen UI Fail          |

---

# 3. Oracle Categories

## 3.1 Input Acknowledgement Oracle (ACK Oracle)

### Rule

사용자 입력(탭, Enter, 드래그 등) 후 **X ms 내에 반드시** UI가 반응해야 한다.

### Budget

| 등급       | 시간       | 판정         |
| ---------- | ---------- | ------------ |
| Ideal      | ≤ 100 ms   | ✅ Excellent |
| Acceptable | ≤ 300 ms   | ✅ Pass      |
| Warning    | ≤ 1 000 ms | ⚠️ Degraded  |
| Fail       | > 1 000 ms | ❌ Fail      |

### Acceptable ACK Signals (AI가 탐지해야 하는 것)

| 신호                             | 감지 방법                                      |
| -------------------------------- | ---------------------------------------------- |
| Visual change (색상·크기·투명도) | DOM mutation / 렌더 프레임 diff                |
| Button disabled 상태 전환        | 속성(attribute) 변경 감지                      |
| Spinner / skeleton 출현          | 새 요소 삽입(DOM) 또는 visibility 변경         |
| Navigation transition 시작       | route change event / 화면 전환 애니메이션 시작 |
| Network request 시작             | HTTP 요청 이벤트 로그                          |
| Haptic feedback                  | OS haptic API 호출 로그                        |
| Focus 변경                       | focused element 변경 로그                      |
| `aria-busy="true"` 설정          | 접근성 트리 변경                               |
| Cursor / pointer 스타일 변경     | CSS computed style 변경                        |
| Sound effect 재생                | 오디오 API 호출 로그                           |

### Failure Types

| 유형                     | 설명                                    | 심각도   |
| ------------------------ | --------------------------------------- | -------- |
| Ghost tap                | 탭했지만 아무 반응 없음                 | Critical |
| Frozen UI                | 이벤트 루프 블로킹으로 전체 화면 무반응 | Critical |
| Ignored key press        | 키 입력이 어디에도 전달되지 않음        | Major    |
| Double-submit dependency | 한 번으로 충분한 액션에 두 번 탭 필요   | Major    |
| Delayed visual only      | 내부 처리는 시작했으나 시각 피드백 지연 | Minor    |

### 판정 로직 (AI 에이전트 의사코드)

```

on user_input(event):
t0 = event.timestamp # 타이밍 장치가 제공
signals = collect_signals(until = t0 + ACK_BUDGET)

    if signals is empty:
        verdict = FAIL("Ghost tap — no ACK within budget")
    elif first_signal.time - t0 > WARN_THRESHOLD:
        verdict = WARNING("ACK delayed", latency = first_signal.time - t0)
    else:
        verdict = PASS(latency = first_signal.time - t0)

    attach_evidence(timeline, screenshot_diff, ui_tree)

```

---

## 3.2 Extra-Action Dependency Oracle

사용자가 한 번의 의도(intent)에 대해 **추가 입력 없이** 결과에 도달해야 한다.

> "왜 두 번 눌러야 돼?"는 가장 흔한 UX 불만 중 하나다.

### Method (Metamorphic Testing)

동일한 의도에 대해 여러 변형 입력을 실행한다:

| 변형                            | 설명                   |
| ------------------------------- | ---------------------- |
| Single tap                      | 정상 단일 탭           |
| Double tap                      | 빠른 연속 탭           |
| Delayed tap                     | 느린 탭 (300 ms+ 간격) |
| Long press                      | 길게 누르기 (500 ms+)  |
| Tap + keyboard Enter            | 탭 후 엔터             |
| Tap on different area then back | 다른 곳 탭 후 대상 탭  |

### 판정

- 단일 탭으로 완료 → **PASS**
- 수정된 입력(더블탭, 딜레이 등)에서만 성공 → **FAIL** (Extra-action dependency)
- 어떤 변형으로도 실패 → **BLOCKED** (기능 자체 결함 가능성 — 별도 보고)

### 추가: No-Extra-Action 범위 확장

| 시나리오         | 기대                     | 위반 예시                                 |
| ---------------- | ------------------------ | ----------------------------------------- |
| 폼 제출          | 제출 버튼 1회 탭         | 키보드 닫기 → 스크롤 → 제출 필요          |
| 검색             | 검색어 입력 + Enter/버튼 | 입력 후 포커스 재설정 필요                |
| 모달 닫기        | 닫기 버튼 또는 외부 탭   | 닫기 후 배경 스크롤 불가 (추가 조작 필요) |
| 리스트 항목 선택 | 항목 1회 탭              | 첫 탭은 하이라이트만, 두 번째 탭에 진입   |

---

## 3.3 Responsiveness Oracle

사용자가 **체감하는 속도**를 측정한다.

### Metrics (타이밍 장치 제공)

| 지표                      | 설명                             | 제공 소스             |
| ------------------------- | -------------------------------- | --------------------- |
| Input-to-first-change     | 입력 → 첫 시각 변화              | 렌더 프레임 diff      |
| Animation start delay     | 트리거 → 애니메이션 첫 프레임    | 애니메이션 API 로그   |
| Event loop stall          | 메인 스레드 블로킹 시간          | long task 모니터      |
| Frame drop count          | 연속 드롭 프레임 수              | FPS 카운터            |
| Time to Interactive (TTI) | 화면 전환 후 입력 가능까지       | 이벤트 루프 idle 시점 |
| Scroll jank               | 스크롤 중 16 ms 초과 프레임 비율 | 프레임 타이밍         |

### Thresholds

| 지표                   | Good     | Degraded        | Unresponsive    |
| ---------------------- | -------- | --------------- | --------------- |
| Input reaction         | ≤ 100 ms | ≤ 300 ms        | > 1 000 ms      |
| Animation start        | ≤ 50 ms  | ≤ 150 ms        | > 300 ms        |
| Event loop stall       | ≤ 50 ms  | ≤ 200 ms        | > 500 ms        |
| Continuous frame drops | 0        | ≤ 3 consecutive | > 5 consecutive |
| TTI after navigation   | ≤ 500 ms | ≤ 2 000 ms      | > 5 000 ms      |
| Scroll jank ratio      | < 5 %    | < 15 %          | ≥ 15 %          |

### Frozen UI 세부 판정

```

if event_loop_stall > 500ms:
FAIL("Frozen UI — main thread blocked {stall}ms")
if frame_drops > 5 consecutive:
WARNING("Jank — {n} frames dropped during {context}")

```

---

## 3.4 Focus & Input Routing Oracle

사용자가 의도한 곳에 입력이 전달되는지 확인한다.

### 검증 항목

| 항목                  | 기대                                            | 판정                                |
| --------------------- | ----------------------------------------------- | ----------------------------------- |
| 올바른 입력 수신자    | 탭한 요소가 이벤트를 수신                       | 다른 요소가 수신하면 FAIL           |
| 키보드 포커스 유효성  | 텍스트 필드 탭 → 키보드 표시 + 해당 필드 포커스 | 포커스 불일치 시 FAIL               |
| 제스처 소유권         | 드래그 시작 요소가 끝까지 이벤트 소유           | 중간에 다른 요소로 넘어가면 WARNING |
| 입력 직후 포커스 유지 | 버튼 탭 후 포커스가 논리적 다음 위치            | 엉뚱한 곳으로 이동하면 FAIL         |
| Enter 키 전달         | 입력창에서 Enter → 해당 입력창이 수신           | 다른 곳으로 전달되면 FAIL           |
| 오버레이/모달 격리    | 모달 표시 중 뒤쪽 요소 터치 불가                | 뒤쪽 요소가 이벤트 수신하면 FAIL    |

### Failure Examples

| 증상                            | 원인 추정                   | 심각도   |
| ------------------------------- | --------------------------- | -------- |
| 키보드 표시 + 타이핑 무시       | 포커스가 숨겨진 요소에 있음 | Critical |
| 탭이 오버레이에 잡힘            | 투명 오버레이 z-index 문제  | Critical |
| 첫 입력이 포커스 활성화만 수행  | focus-then-type 패턴 강요   | Major    |
| Enter가 폼 제출 대신 줄바꿈     | textarea vs input 혼동      | Minor    |
| 스크롤 중 탭이 다른 요소에 도달 | 레이아웃 시프트와 동시 발생 | Major    |

### Focus Sanity 판정 로직

```

on user_input(event):
expected_target = element_at(event.coordinates)
actual_target = event.received_by
post_focus = document.active_element (after event)

    if actual_target ≠ expected_target:
        FAIL("Input routing mismatch: expected={expected}, actual={actual}")
    if post_focus is unrelated to action context:
        FAIL("Focus jumped to unrelated element: {post_focus}")
    if input_type == 'keyboard' and post_focus ≠ text_input:
        FAIL("Keyboard input not reaching focused text field")

```

---

## 3.5 Navigation Stability Oracle

화면 전환이 **예측 가능하고 안정적**인지 확인한다.

### 검증 항목

| 항목                | 기대                                             | 위반                                 |
| ------------------- | ------------------------------------------------ | ------------------------------------ |
| 단일 내비게이션     | 1 탭 → 1 화면 전환                               | 같은 화면 2회 push (더블 네비게이션) |
| 뒤로 가기 일관성    | 뒤로 → 직전 화면                                 | 2단계 이전으로 점프                  |
| 전환 중 인터럽트    | 전환 완료 전 다른 탭 → 자연스러운 취소 또는 큐잉 | 화면 깨짐·중첩                       |
| 딥링크 진입         | 외부 링크 → 올바른 화면 + 뒤로 가기 스택 정상    | 뒤로 누르면 앱 종료                  |
| 탭 바 / 사이드 메뉴 | 탭 → 해당 섹션 루트 화면                         | 이전 상태 잔재                       |

### Signals (타이밍 장치 제공)

- route change 이벤트 + 타임스탬프
- animation lifecycle (start / end / cancel)
- navigation stack depth 변화
- 화면 전환 소요 시간

### 판정

```

if route_changes_within(500ms) > 1:
FAIL("Double navigation detected: {routes}")
if back_action.destination ≠ expected_previous_screen:
FAIL("Back jump anomaly: expected={prev}, actual={dest}")
if transition_interrupted AND visual_glitch_detected:
WARNING("Transition interrupt caused visual artifact")

```

---

## 3.6 Async Transparency Oracle

비동기 작업(네트워크, 파일 처리 등)이 진행 중임을 사용자에게 **투명하게** 알려야 한다.

### Rules

| 경과 시간 | 필수 피드백                      | 예시                                          |
| --------- | -------------------------------- | --------------------------------------------- |
| > 500 ms  | 미묘한 시각 힌트                 | 버튼 비활성화, 미세 로딩 표시                 |
| > 1 s     | 명확한 로딩 표시                 | Spinner, skeleton UI                          |
| > 3 s     | 진행률 표시 또는 예상 시간       | Progress bar, "약 10초 소요"                  |
| > 10 s    | 취소 옵션 제공                   | "취소" 버튼 활성화                            |
| > 30 s    | 상태 갱신 + 백그라운드 전환 안내 | "오래 걸리고 있습니다. 알림으로 알려드릴게요" |

### Failure Types

| 유형             | 설명                                    | 심각도                          |
| ---------------- | --------------------------------------- | ------------------------------- |
| Silent waiting   | 아무 표시 없이 지연                     | Critical (> 3 s), Major (> 1 s) |
| Infinite spinner | 스피너가 끝나지 않음 (실제론 완료·실패) | Critical                        |
| Progress 역행    | 진행률이 감소                           | Major                           |
| Stale progress   | 진행률이 장시간 변하지 않음             | Major                           |
| 취소 불가        | 장시간 작업에 취소 수단 없음            | Minor (< 10 s), Major (≥ 10 s)  |

### 판정 로직

```

on async_operation_start(op):
t_start = op.timestamp

    at t_start + 1s:
        if no_visual_feedback():
            FAIL("Silent waiting > 1s for {op.name}")

    at t_start + 3s:
        if no_progress_indicator():
            FAIL("No progress indicator > 3s for {op.name}")

    at t_start + 10s:
        if no_cancel_option():
            WARNING("No cancel option for long operation {op.name}")

    on op.complete OR op.fail:
        if spinner_still_visible(after 500ms):
            FAIL("Infinite spinner after {op.name} completed/failed")

```

---

## 3.7 Gesture Consistency Oracle

제스처 조작이 **예측 가능하고 일관적**인지 확인한다.

### 검증 항목

| 항목                 | 기대                                   | 위반                                                |
| -------------------- | -------------------------------------- | --------------------------------------------------- |
| Swipe 방향 일관성    | 같은 맥락에서 같은 방향 = 같은 동작    | 리스트에서 좌 스와이프가 삭제였다가 아카이브로 변경 |
| Scroll inertia       | 자연스러운 관성 스크롤                 | 즉시 정지 또는 과도한 관성                          |
| Gesture cancellation | 드래그 중 반대 방향 → 취소             | 원상복구 안 됨                                      |
| Pinch-to-zoom        | 확대/축소가 손가락 중심점 기준         | 화면 좌상단 기준으로 확대                           |
| Pull-to-refresh      | 당기기 → 새로고침 피드백 → 완료        | 당겨도 무반응 또는 이중 새로고침                    |
| 제스처 충돌          | 수평 스와이프 vs 수직 스크롤 명확 분리 | 스크롤하려는데 카드가 스와이프됨                    |

### Detection Signals

- 제스처 시작/종료 좌표 + 타임스탬프
- 제스처 velocity / acceleration
- 동일 제스처의 결과 비교 (metamorphic)
- 제스처 영역 겹침(conflict zone) 감지

### Thresholds

| 지표                   | 정상                    | 비정상                   |
| ---------------------- | ----------------------- | ------------------------ |
| Swipe threshold 일관성 | 동일 뷰에서 ± 10 % 이내 | > 30 % 편차              |
| Scroll deceleration    | 자연 곡선 (ease-out)    | 선형 정지 또는 즉시 정지 |
| Gesture cancel 성공률  | ≥ 95 %                  | < 80 %                   |
| 충돌 오발생률          | < 3 %                   | ≥ 10 %                   |

---

## 3.8 Visual Stability Oracle

레이아웃이 **예기치 않게 흔들리지 않는지** 확인한다.

### Metrics (타이밍 장치 제공)

| 지표                          | 설명                             | 허용                            |
| ----------------------------- | -------------------------------- | ------------------------------- |
| Layout shift magnitude        | 요소 이동 거리 (px / viewport %) | < 5 % viewport                  |
| Cumulative Layout Shift (CLS) | 누적 레이아웃 이동량             | < 0.1                           |
| Tap target displacement       | 탭 직전 vs 직후 대상 위치 변화   | 0 px (탭 시점에 이동 금지)      |
| Animation jitter              | 애니메이션 프레임 간 위치 편차   | Smooth curve, 역행 없음         |
| Content reflow                | 콘텐츠 로드 후 기존 요소 재배치  | 최소화 (skeleton으로 공간 확보) |
| 이미지/미디어 로드 점프       | 이미지 로드 시 주변 요소 이동    | 사전 크기 지정 필수             |

### Failure Scenarios

| 시나리오                       | 사용자 경험                | 판정     |
| ------------------------------ | -------------------------- | -------- |
| 광고 배너 삽입으로 버튼 밀림   | "다른 버튼을 눌러버렸다"   | Critical |
| 이미지 지연 로드로 텍스트 점프 | "읽던 곳을 잃어버렸다"     | Major    |
| 폰트 로드 후 텍스트 리플로우   | 미묘한 레이아웃 변화       | Minor    |
| 키보드 표시 시 입력창 가려짐   | "내가 뭘 치는지 안 보인다" | Critical |
| 가로/세로 회전 후 콘텐츠 손실  | "입력한 내용이 사라졌다"   | Critical |

### 판정 로직

```

on layout_change(event):
if event.shift > 5% viewport AND user_recently_tapped(within 500ms):
FAIL("Tap-time layout shift: {shift}% — user may hit wrong target")
if cumulative_layout_shift > 0.1:
WARNING("High CLS: {cls} on screen {screen}")

```

---

## 3.9 Error Feedback Oracle

오류 발생 시 사용자에게 **명확하고 실행 가능한** 피드백을 제공해야 한다.

### Rules

오류 메시지는 반드시 아래 3가지를 충족해야 한다:

1. **Visible** — 사용자가 인지할 수 있는 위치·크기·색상
2. **Actionable** — "무엇을 해야 하는지" 안내 포함
3. **Timely** — 오류 발생 후 1 s 이내 표시

### Failure Types

| 유형            | 설명                                   | 심각도   |
| --------------- | -------------------------------------- | -------- |
| Silent failure  | 오류 발생했지만 아무 표시 없음         | Critical |
| Cryptic error   | 기술 용어만 표시 (예: "Error 500")     | Major    |
| Invisible error | 토스트가 너무 짧거나, 스크롤 밖에 표시 | Major    |
| Retry unclear   | 실패 후 다시 시도하는 방법 불명확      | Major    |
| State unchanged | 실패했지만 UI가 성공한 것처럼 보임     | Critical |
| Error 잔류      | 문제 해결 후에도 에러 메시지 남아 있음 | Minor    |
| 입력값 소실     | 에러 후 사용자가 입력한 값이 사라짐    | Critical |

### Error Message Quality Checklist (AI 판정 기준)

| 항목        | Pass 조건                                                          |
| ----------- | ------------------------------------------------------------------ |
| 가시성      | 에러 요소가 viewport 내 존재, 최소 14 sp 텍스트, 대비율 4.5:1 이상 |
| 지속시간    | Toast: ≥ 3 s 또는 사용자 dismiss까지. Inline: 해결까지 유지        |
| 언어 명확성 | 사용자 언어로, 기술 코드 미포함 (또는 부가 설명 동반)              |
| 행동 안내   | "다시 시도", "네트워크 확인", "입력값 수정" 등 구체적 가이드       |
| 입력 보존   | 에러 후 사용자 입력 필드 값 유지                                   |
| 복구 경로   | 재시도 버튼 또는 명확한 대안 제시                                  |

---

## 3.10 Accessibility & Readability Oracle _(신규)_

좋은 UI는 **모든 사용자**가 사용할 수 있어야 한다. 접근성은 법적 요구사항이기도 하지만,
일반 사용자에게도 가독성·조작성 측면에서 직접적인 UX 품질 요소이다.

### 검증 항목

| 항목                  | 기준                                           | 판정                                     |
| --------------------- | ---------------------------------------------- | ---------------------------------------- |
| 터치 타겟 크기        | ≥ 48 × 48 dp (Material), ≥ 44 × 44 pt (HIG)    | 미달 시 WARNING, ≤ 30 dp 시 FAIL         |
| 터치 타겟 간격        | ≥ 8 dp                                         | 미달 시 WARNING                          |
| 텍스트 대비율         | 일반 텍스트 ≥ 4.5:1, 큰 텍스트 ≥ 3:1 (WCAG AA) | 미달 시 FAIL                             |
| 최소 폰트 크기        | ≥ 12 sp (본문), ≥ 10 sp (보조)                 | 미달 시 WARNING                          |
| 색상만으로 정보 전달  | 색상 외 보조 수단(아이콘, 텍스트) 필요         | 색상만 사용 시 FAIL                      |
| 스크린 리더 라벨      | 모든 인터랙티브 요소에 접근성 라벨 존재        | 누락 시 WARNING                          |
| 동적 콘텐츠 알림      | 콘텐츠 변경 시 `aria-live` 또는 동등 장치      | 누락 시 WARNING                          |
| 시스템 폰트 크기 반영 | OS 글꼴 크기 설정 존중                         | 무시 시 WARNING                          |
| 가로/세로 모드        | 양방향 모두 사용 가능                          | 한 방향 고정 시 WARNING (특수 사유 제외) |

### AI 판정 방법

```

for each interactive_element in ui_tree:
if element.size < 48dp × 48dp:
WARNING("Small touch target: {element} is {w}×{h}dp")
if element.accessibility_label is empty:
WARNING("Missing a11y label: {element}")

for each text_element in ui_tree:
contrast = calculate_contrast(text_color, background_color)
if contrast < 4.5:
FAIL("Low contrast: {contrast}:1 for '{text_preview}' on {screen}")

```

---

## 3.11 Content & State Consistency Oracle _(신규)_

화면에 표시되는 **콘텐츠와 상태**가 사용자 기대와 일치하는지 확인한다.

### 검증 항목

| 항목                        | 기대                                   | 위반                              |
| --------------------------- | -------------------------------------- | --------------------------------- |
| 빈 상태(empty state)        | 데이터 없을 때 안내 메시지 + 행동 유도 | 빈 화면만 표시                    |
| 로딩 → 콘텐츠 전환          | 로딩 완료 후 콘텐츠 즉시 표시          | 로딩 끝났으나 빈 화면             |
| 오프라인 상태               | 오프라인 시 명확한 안내                | 무한 로딩 또는 빈 화면            |
| 데이터 신선도               | 최신 데이터 반영 (pull-to-refresh 등)  | stale 데이터 고착                 |
| 이전 화면 복귀 시 상태 유지 | 뒤로 가기 → 스크롤 위치·탭 상태 유지   | 최상단 리셋                       |
| 입력 중 화면 전환 후 복귀   | 작성 중 내용 유지                      | 입력값 소실                       |
| 숫자·단위 일관성            | 통화·날짜·숫자 포맷 로캘 일치          | 혼재 (₩ / $ / ¥ 섞임)             |
| 텍스트 잘림                 | 긴 텍스트에 말줄임표(…) 또는 확장 수단 | 잘림 + 정보 손실 + 접근 수단 없음 |
| 이미지 비율                 | 원본 비율 유지                         | 찌그러짐 (stretch/squash)         |

### Failure Scoring

| 심각도   | 조건                                  |
| -------- | ------------------------------------- |
| Critical | 사용자 데이터 손실 (입력값 소실)      |
| Major    | 정보 접근 불가 (빈 화면, 텍스트 잘림) |
| Minor    | 미관 저하 (비율 미맞춤, 포맷 불일치)  |

---

## 3.12 Platform Convention Oracle _(신규)_

각 플랫폼(iOS / Android)의 **관례(convention)**를 준수하는지 확인한다.
사람은 플랫폼별 패턴에 익숙하며, 위반 시 "이상하다"고 느낀다.

### 검증 항목

| 항목              | iOS 기대                                       | Android 기대                   | 위반 시                          |
| ----------------- | ---------------------------------------------- | ------------------------------ | -------------------------------- |
| 뒤로 가기         | 좌상단 back chevron / edge swipe               | 시스템 back 버튼 / gesture     | 뒤로 가기 불가 → FAIL            |
| 탭 바 위치        | 하단                                           | 하단 (Material 3) / 상단 탭    | 플랫폼 반대편 → WARNING          |
| 알림 권한         | 사용 시점에 요청 (맥락 설명 포함)              | 마찬가지                       | 앱 시작 즉시 권한 폭탄 → WARNING |
| 시스템 설정 반영  | 다크모드, 글꼴 크기, 접근성                    | 마찬가지                       | 완전 무시 → WARNING              |
| 키보드 타입       | 이메일 → email keyboard, 전화번호 → number pad | 마찬가지                       | 일반 키보드만 → WARNING          |
| 스크롤 오버스크롤 | Bounce                                         | Edge glow / stretch            | 아무 효과 없음 → Minor           |
| 상태바 처리       | 콘텐츠가 상태바 아래로 겹치지 않음             | 마찬가지                       | 겹침 → FAIL                      |
| Safe area 준수    | 노치·Dynamic Island 영역 회피                  | 펀치홀·내비게이션 바 영역 회피 | 콘텐츠 가려짐 → FAIL             |

---

## 3.13 Performance Perception Oracle _(신규)_

실제 성능뿐 아니라 사용자가 **"빠르다고 느끼는지"**를 평가한다.

### Perception Techniques & 판정

| 기법                | 설명                            | 적용 여부 확인                           |
| ------------------- | ------------------------------- | ---------------------------------------- |
| Skeleton UI         | 콘텐츠 로드 전 구조 미리보기    | 빈 화면 대신 skeleton 존재?              |
| Optimistic UI       | 서버 응답 전 성공 가정 표시     | 좋아요/저장 등에서 즉시 반영?            |
| Progressive loading | 핵심 콘텐츠 우선 표시           | 전체 로드 완료 전 상호작용 가능?         |
| Animated transition | 화면 간 자연스러운 전환         | 즉시 교체(flicker) 대신 전환 애니메이션? |
| Preloading          | 예측 가능한 다음 화면 사전 로드 | 탭 후 즉시 표시?                         |
| Perceived progress  | 빠른 초기 진행 + 느린 후반      | 프로그레스 바가 선형 이하로 느림?        |

### 판정

- 위 기법 중 **해당 맥락에 적합한 것이 적용되지 않은 경우** → WARNING (개선 권고)
- 화면 전환 시 **빈 화면이 300 ms 이상** 노출 → FAIL ("White flash")
- 콘텐츠 영역이 **2 s 이상 비어 있다가** 한꺼번에 표시 → WARNING ("Content pop-in")

---

# 4. Required Instrumentation (Minimal)

AI 에이전트가 시간을 직접 인지할 수 없으므로, 아래 **계측 레이어**가
밀리초 정밀도의 데이터를 제공해야 한다.

### 필수 이벤트

```

app_event("user_input", { type, target, timestamp })
app_event("action_ack", { name, signal_type, timestamp })
app_event("action_done", { name, result, timestamp })
app_event("error", { code, message, context, timestamp })
app_event("layout_shift", { magnitude, affected_elements, timestamp })
app_event("focus_change", { from, to, timestamp })

```

### 선택 이벤트 (품질 향상)

```

app_event("network_start", { url, method, timestamp })
app_event("network_end", { url, status, duration, timestamp })
app_event("animation_start", { name, timestamp })
app_event("animation_end", { name, timestamp })
app_event("screen_transition", { from, to, timestamp })
app_event("frame_timing", { frame_durations[], timestamp })
app_event("gesture", { type, start, end, velocity, timestamp })
app_event("long_task", { duration, stack_hint, timestamp })
app_event("accessibility", { element, label, role, timestamp })

```

### 외부 타이밍 장치가 제공하는 파생 데이터

| 데이터                | 설명                              |
| --------------------- | --------------------------------- |
| `ack_latency_ms`      | user_input → action_ack 차이      |
| `total_latency_ms`    | user_input → action_done 차이     |
| `frame_drop_count`    | 연속 16 ms 초과 프레임 수         |
| `cls_score`           | 누적 레이아웃 시프트 점수         |
| `event_loop_stall_ms` | 메인 스레드 블로킹 최대 지속 시간 |
| `scroll_jank_ratio`   | 스크롤 중 지연 프레임 비율        |

---

# 5. Evidence Collection

모든 판정(PASS / WARNING / FAIL)에는 아래 증거가 첨부되어야 한다.

### 필수 증거

| 항목                    | 형식                   | 설명                            |
| ----------------------- | ---------------------- | ------------------------------- |
| Timeline                | ms 단위 이벤트 로그    | 입력→ACK→DONE 전체 흐름         |
| Before/After screenshot | 이미지 pair            | 입력 직전 / 판정 시점           |
| UI tree snapshot        | 구조화된 트리 (JSON)   | 요소 위치·크기·상태·접근성 속성 |
| Network log             | HAR 또는 이벤트 로그   | 요청/응답 타이밍                |
| Focus state             | element ID + timestamp | 포커스 변화 이력                |
| Gesture metadata        | 좌표·velocity·duration | 제스처 관련 판정 시             |
| Device context          | 기기·OS·화면크기·방향  | 재현을 위한 환경 정보           |

### 증거 예시 (ACK Oracle Fail)

```

📋 Evidence Bundle — ACK_FAIL_2024_001
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Oracle : ACK Oracle
Verdict : ❌ FAIL
Screen : checkout/payment
Element : #btn-submit-order
Input type : tap

Timeline:
t=0ms user_input(tap, #btn-submit-order)
t=0ms [no DOM mutation]
t=300ms [no DOM mutation]
t=420ms [no DOM mutation]
t=1120ms action_ack(spinner, #loading-overlay)
→ ACK latency: 1120ms (budget: 300ms)

Attachments:

- screenshot_t0.png
- screenshot_t1120.png
- ui_tree_t0.json
- ui_tree_t1120.json
- network_log.har
- focus_log.json

Device: iPhone 14, iOS 17.2, portrait

```

---

# 6. Scoring Model

## 6.1 Action-level UX Score

개별 액션마다 100점 만점으로 산출한다.

| 카테고리        | 배점 | 평가 오라클                                 |
| --------------- | ---- | ------------------------------------------- |
| ACK 응답성      | 25   | 3.1 ACK Oracle, 3.3 Responsiveness          |
| 상호작용 정확성 | 20   | 3.2 Extra-Action, 3.4 Focus/Routing         |
| 시각 안정성     | 15   | 3.8 Visual Stability, 3.13 Perf. Perception |
| 피드백 명확성   | 15   | 3.6 Async Transparency, 3.9 Error Feedback  |
| 일관성 & 관례   | 10   | 3.7 Gesture, 3.12 Platform Convention       |
| 접근성 & 가독성 | 10   | 3.10 Accessibility                          |
| 콘텐츠 무결성   | 5    | 3.11 Content Consistency                    |

## 6.2 등급

| 점수     | 등급          | 의미                          |
| -------- | ------------- | ----------------------------- |
| 90 – 100 | 🟢 Excellent  | 사용자가 쾌적하게 느끼는 수준 |
| 75 – 89  | 🔵 Good       | 대부분 문제없으나 개선 여지   |
| 60 – 74  | 🟡 Degraded   | 사용자 불편 체감 시작         |
| 40 – 59  | 🟠 Poor       | 명백한 UX 결함                |
| < 40     | 🔴 UX Failure | 사용 포기 수준                |

## 6.3 Screen-level & App-level Score

```

Screen Score = weighted_avg(Action Scores on screen)
(가중치: Critical action × 2.0, Normal × 1.0)

App Score = weighted_avg(Screen Scores)
(가중치: 진입빈도 높은 화면 우선)

```

## 6.4 Severity → 감점 매핑

| 심각도   | 감점      | 적용                  |
| -------- | --------- | --------------------- |
| Critical | -20 ~ -30 | 해당 카테고리 배점 내 |
| Major    | -10 ~ -15 | 해당 카테고리 배점 내 |
| Minor    | -3 ~ -5   | 해당 카테고리 배점 내 |
| WARNING  | -1 ~ -2   | 누적 시 감점 증가     |

---

# 7. AI Agent Responsibilities

## Agent SHOULD

| 책임                | 상세                                         |
| ------------------- | -------------------------------------------- |
| UI 그래프 자율 탐색 | 화면·액션을 노드·엣지로 매핑                 |
| 입력 변형 실행      | metamorphic testing으로 변형 입력 시도       |
| 최소 재현 경로 도출 | 실패 시 최소한의 재현 단계(repro steps) 정리 |
| 신호 기반 판정      | 타이밍 장치 데이터를 오라클 규칙에 대입      |
| 증거 수집 & 번들링  | 판정마다 Evidence Bundle 생성                |
| 사람에게 설명       | 판정 근거를 자연어로 설명                    |
| 심각도 분류         | Critical / Major / Minor / Warning 분류      |
| 회귀 비교           | 이전 실행 결과와 비교하여 개선·퇴행 판별     |
| **사람 관점 해석**  | "사용자는 이 상황에서 ~라고 느낄 것" 서술    |

## Agent SHOULD NOT

| 금지                     | 이유                                                  |
| ------------------------ | ----------------------------------------------------- |
| 정확성(correctness) 추측 | 기능 명세가 없으므로 맞다/틀리다 판단 불가            |
| 스크린샷만으로 판정      | 이미지 해석은 오탐 위험 → 반드시 계측 데이터 동반     |
| 기대값 임의 생성         | 오라클은 **불변식** 기반이지, 특정 결과값 기반이 아님 |
| 시간 직접 측정           | AI는 시간 인지 불가 → 외부 장치 데이터만 사용         |
| 단일 실패에 과잉 보고    | 동일 근본 원인은 그룹핑하여 1건으로 보고              |

---

# 8. Exploration Strategy

## 8.1 Behavior Graph

```

Node = UI State (screen + 주요 요소 상태)
Edge = Action (tap, swipe, type, navigate, ...)

```

## 8.2 탐색 전략

| 단계    | 전략                     | 목적                                |
| ------- | ------------------------ | ----------------------------------- |
| Phase 1 | Breadth-first            | 전체 화면 맵 구축                   |
| Phase 2 | Anomaly-driven deep dive | 오라클 경고 발생 지점 집중 탐색     |
| Phase 3 | Metamorphic retry        | 실패 지점에서 입력 변형 실행        |
| Phase 4 | Edge-case probing        | 극단값 입력, 빠른 연타, 동시 제스처 |
| Phase 5 | Regression comparison    | 이전 빌드 결과와 비교               |

## 8.3 탐색 우선순위

| 우선순위 | 기준                                            |
| -------- | ----------------------------------------------- |
| 1 (최고) | Critical 판정이 발생한 화면/액션                |
| 2        | 사용자 빈도 높은 주요 경로 (happy path)         |
| 3        | 입력 변형에 민감한 액션 (metamorphic 차이 발생) |
| 4        | 아직 미방문 화면                                |
| 5        | 이전 빌드에서 수정된 영역                       |

## 8.4 탐색 종료 조건

- 모든 도달 가능 화면 방문 완료
- 새로운 FAIL 발견 빈도가 임계값 이하로 감소
- 탐색 시간 예산 소진
- 특정 커버리지 목표 달성 (화면 커버리지 ≥ 90 %)

---

# 9. Reporting

## 9.1 보고서 구조

```

📊 UX Oracle Report — {App Name} v{Version}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📅 Date: {date}
📱 Device: {device}, {OS}
🏆 App Score: {score}/100 ({grade})

┌─────────────────────────────┐
│ Summary │
│ Critical: {n} │
│ Major: {n} │
│ Minor: {n} │
│ Warning: {n} │
│ Pass: {n} │
│ Coverage: {n}% screens │
└─────────────────────────────┘

🔴 Critical Issues

1. [ACK Oracle] #btn-submit — Ghost tap (1120ms, budget 300ms)
   → 사용자 영향: "결제 버튼을 눌렀는데 반응이 없어 불안함"
   → Evidence: bundle_001
   → Repro: 결제 화면 → 주문하기 탭

2. [Error Feedback] 네트워크 오류 시 빈 화면
   → 사용자 영향: "뭐가 잘못된 건지 모르겠다"
   → Evidence: bundle_002

🟠 Major Issues
...

📈 Trend (vs previous build)
Score: 72 → 68 (▼4)
New issues: 3
Resolved: 1
Regressed: 2

```

## 9.2 사람 관점 해석 (Human Impact Statement)

모든 FAIL/WARNING에는 **사용자가 느끼는 감정/행동**을 한 줄로 서술한다.

| 판정             | Human Impact Statement 예시                                    |
| ---------------- | -------------------------------------------------------------- |
| ACK Ghost tap    | "버튼이 안 눌리는 줄 알고 여러 번 탭하게 되어, 중복 주문 위험" |
| Silent failure   | "결제가 된 건지 안 된 건지 모르겠다"                           |
| Layout shift     | "읽던 기사가 갑자기 밀려서 다른 광고를 누르게 됨"              |
| Focus mismatch   | "비밀번호를 치는데 아이디 칸에 입력되고 있었다"                |
| Infinite spinner | "앱이 멈춘 줄 알고 강제종료했다"                               |

---

# 10. Minimal MVP Deployment

| 단계 | 작업                                      | 소요 (추정) |
| ---- | ----------------------------------------- | ----------- |
| 1    | 필수 이벤트 계측 코드 삽입                | 1–2일       |
| 2    | 타이밍 장치 연동 (이벤트 → ms 타임스탬프) | 1일         |
| 3    | ACK Oracle + Frozen UI Oracle 구현        | 2–3일       |
| 4    | 자동 탐색 에이전트 연결 (nightly 실행)    | 2–3일       |
| 5    | Evidence Bundle 저장소 설정               | 1일         |
| 6    | 보고서 생성 & 알림(Slack/Email)           | 1일         |
| 7    | 주간 사람 리뷰 프로세스 시작              | 즉시        |
| 8    | 추가 오라클 점진적 활성화                 | 주 단위     |

### MVP 오라클 활성화 순서 (권장)

```

Week 1: ACK Oracle + Frozen UI
Week 2: + Extra-Action Dependency + Focus/Routing
Week 3: + Responsiveness + Visual Stability
Week 4: + Error Feedback + Async Transparency
Week 5+: + Accessibility + Gesture + Platform Convention + ...

```

---

# 11. Expected Outcomes

## 자동 감지 가능한 UX 문제

| 문제                              | 담당 오라클                         |
| --------------------------------- | ----------------------------------- |
| 두 번 탭해야 작동하는 버튼        | Extra-Action                        |
| 느린 UI 피드백                    | ACK, Responsiveness                 |
| 화면 멈춤                         | Frozen UI                           |
| 거짓 진행 표시 (스피너가 안 끝남) | Async Transparency                  |
| 제스처 충돌                       | Gesture Consistency                 |
| 내비게이션 오류                   | Navigation Stability                |
| 레이아웃 점프로 인한 오탭         | Visual Stability                    |
| 에러 안내 없는 실패               | Error Feedback                      |
| 작은 터치 타겟                    | Accessibility                       |
| 키보드 가림                       | Focus/Routing, Platform Convention  |
| 빈 화면 (empty state 미처리)      | Content Consistency                 |
| 플랫폼 관례 위반                  | Platform Convention                 |
| 느리게 느껴지는 화면 전환         | Performance Perception              |
| 입력값 소실                       | Content Consistency, Error Feedback |

## 불필요한 것

- ❌ 수동 테스트 케이스 작성
- ❌ 예상 UI 결과값 정의
- ❌ 기능별 검증 스크립트

---

# 12. Future Extensions

| 확장                       | 설명                                                        |
| -------------------------- | ----------------------------------------------------------- |
| Learned UX baseline        | 앱별 정상 응답 시간 학습 → 이상치 자동 탐지                 |
| Personalized expectations  | 사용자 세그먼트별 기대 수준 차별화                          |
| Cross-version regression   | 빌드 간 UX 점수 자동 비교 & 알림                            |
| RL exploration             | 강화학습으로 탐색 효율 극대화                               |
| Multi-device matrix        | 다양한 기기·OS·화면 크기 동시 테스트                        |
| Real user session replay   | 실제 사용자 세션 데이터와 오라클 판정 비교                  |
| Competitive benchmarking   | 경쟁 앱의 UX 점수와 비교                                    |
| Natural language test spec | "결제 흐름이 매끄러운지 확인해줘" → 오라클 자동 선택 & 실행 |
| Continuous monitoring      | CI/CD 파이프라인에 통합, PR마다 UX 점수 게이트              |
| Emotion inference          | 사용자 행동 패턴(분노 탭, 반복 뒤로가기)으로 감정 추정      |

---

# Appendix A. Oracle Quick Reference Card

| #    | Oracle              | 핵심 질문                       | 주요 임계값                  |
| ---- | ------------------- | ------------------------------- | ---------------------------- |
| 3.1  | ACK                 | 입력 후 반응했나?               | ≤ 300 ms                     |
| 3.2  | Extra-Action        | 한 번에 되나?                   | 단일 입력으로 완료           |
| 3.3  | Responsiveness      | 빠르게 느껴지나?                | Input < 300 ms, TTI < 2 s    |
| 3.4  | Focus/Routing       | 올바른 곳에 입력되나?           | 의도한 요소 = 실제 수신자    |
| 3.5  | Navigation          | 화면 전환이 안정적인가?         | 1탭 = 1전환, back = 직전     |
| 3.6  | Async Transparency  | 기다리는 이유를 아나?           | > 1 s → 로딩, > 10 s → 취소  |
| 3.7  | Gesture             | 제스처가 예측 가능한가?         | 일관된 threshold, < 3 % 충돌 |
| 3.8  | Visual Stability    | 화면이 흔들리지 않나?           | CLS < 0.1                    |
| 3.9  | Error Feedback      | 오류를 이해하고 복구할 수 있나? | 1 s 내 표시, 행동 안내 포함  |
| 3.10 | Accessibility       | 누구나 쓸 수 있나?              | 48 dp, 4.5:1 대비            |
| 3.11 | Content Consistency | 화면 내용이 맞나?               | 빈 상태 안내, 값 보존        |
| 3.12 | Platform Convention | 플랫폼답게 느껴지나?            | iOS/Android 가이드 준수      |
| 3.13 | Perf. Perception    | 빠르다고 느끼나?                | 빈 화면 < 300 ms             |

---

# Appendix B. Glossary

| 용어                   | 정의                                                   |
| ---------------------- | ------------------------------------------------------ |
| ACK                    | Acknowledgement — 입력에 대한 즉각적 피드백            |
| CLS                    | Cumulative Layout Shift — 누적 레이아웃 이동량         |
| Evidence Bundle        | 판정에 첨부되는 증거 묶음                              |
| Ghost tap              | 사용자가 탭했지만 시스템이 무시한 경우                 |
| Human Impact Statement | 판정 결과를 사용자 감정/행동으로 번역한 서술           |
| Metamorphic Testing    | 동일 의도에 입력 변형을 가해 불변 관계 검증            |
| Oracle                 | 기대 결과 없이 합격/불합격을 판정하는 규칙             |
| Skeleton UI            | 콘텐츠 로드 전 구조를 미리 보여주는 플레이스홀더       |
| TTI                    | Time to Interactive — 화면이 상호작용 가능해지는 시점  |
| Timing 장치            | AI 대신 ms 단위 타임스탬프를 제공하는 외부 계측 레이어 |

---

# END
