# expert-claude 개념 설계 논의 기록

> 작성: 2026-06-27 · 최종 갱신: 2026-06-27
> 목적: expert-claude(멀티 에이전트 오케스트레이터) 설계 과정에서 정립한 핵심 개념과 결정 사항을 기록한다.

---

## 1. 핵심 개념 4분류 (Agent / Skill / Tool / Workflow)

가장 추상적인 층위에서는 모두 "에이전트의 능력을 확장하는 수단"이지만, 설계할 때는 다음과 같이 구분한다.

| 개념 | 답하는 질문 | 본질 | Claude Code 1급 요소? | 컨텍스트 |
|---|---|---|---|---|
| **Agent** (subagent) | **누가** 하나 | 독립 컨텍스트를 가진 전문 작업자 (위임 대상) | ✅ 예 (`.claude/agents/*.md`) | 별도 컨텍스트 윈도우 (격리) |
| **Skill** | **무슨 지식·방법**으로 하나 | 지식(SKILL.md) + 선택적 스크립트·리소스 묶음 | ✅ 예 (`.claude/skills/<name>/SKILL.md`) | 현재 에이전트 컨텍스트에 주입 (공유) |
| **Tool** | 어떤 **동작**을 | 단일 호출 가능 기능 (스키마 있음) | ✅ 예 (Read, Bash, MCP 등) | 호출 → 결과 반환 |
| **Workflow** | 어떤 **순서**로 | 단계의 순서·제어 흐름 | ❌ 별도 파일 타입 아님 (slash command·skill·오케스트레이션으로 구현) | 상위 제어 흐름 |

### 결정적 구분: Tool vs Skill

- **Tool = 호출(call)**: 모델이 `tool_use`로 부르고 결과를 받는 함수. 원자적.
- **Skill = 로드(load)**: 컨텍스트에 주입되어 행동을 바꾸는 지식 묶음. 값을 반환하지 않고, *어떤 tool을 어떻게 쓸지*까지 지시할 수 있다. → tool보다 한 층 위.

"skill은 추상적 도구다"라는 직관은 최상위 추상에서는 맞지만, **호출이냐 로드냐 / 값 반환이냐 행동 변경이냐 / 원자냐 복합이냐**의 층위 차이를 뭉갠다.

### 주방 비유

| 개념 | 비유 | 정체 |
|---|---|---|
| Tool | 칼, 도마, 불 | 단일 동작 (호출하면 결과) |
| Skill | "생선 손질법", "소스 유화법" | 특정 일을 *어떻게* (칼이라는 tool을 *이렇게* 써라) |
| Agent | 요리사, 목수 | 일을 *위임*받는 전문 주체 (자기 skill·tool 보유) |
| Workflow | 코스 요리 순서 (전채→메인→디저트) | *어떤 순서로* 거치나 |

> 같은 칼이라도 목공이냐 요리냐(skill)에 따라 쓰는 법이 다르다 — **tool은 고정, skill이 사용법을 결정한다.**

### expert-claude 대입

- **Tool**: Read / Bash / WebFetch / MCP
- **Skill**: verification 기준, git 규칙, venv 사용법, 문서 작성법 (특정 일을 *어떻게*)
- **Agent**: UIDesigner, TechnicalWriter, ReferenceBroker 등 (전문 역할)
- **Workflow**: 분석→설계→검수→문서 (orchestration_instruction에 기술하는 순서)

---

## 2. Agent vs Workflow

라이프사이클 단계(분석→계획→설계→검수)를 나열하면 workflow처럼 보이는 게 자연스럽다. 오케스트레이터의 일이 곧 "에이전트들을 순서로 엮는 것"이라, **에이전트를 잘 만들면 workflow는 emergent하게 나온다.**

판단 기준:
> **별도 agent로 둘 가치 = 고유한 {도구 / 모델 / 시스템프롬프트 / 컨텍스트}가 있는가?**

- 진짜 전문가 (도구·전문성이 다름) → **agent**
- 같은 두뇌의 다른 단계 (사고 모드) → **workflow의 step** 또는 한 agent 내부의 마이크로 워크플로

---

## 3. 분리 vs 통합 — 결합도(coupling)와 컨텍스트 위생

여러 단계를 별도 agent로 쪼갤지, 한 agent에 묶을지는 **결합의 *형태*** 로 결정한다.

| 묶어라 (한 agent) | 쪼개라 (별도 agent) |
|---|---|
| **연속적 공유 상태** — 서로 살아있는 작업 컨텍스트를 계속 주고받음 | **artifact handoff** — A가 깔끔한 산출물을 내고 B가 그걸 소비 |

→ verification_instruction의 dependency 규칙과 직결: `high (tightly coupled) → single model` = 통합 / `low (independent) → fan out` = 분리.

### 분리가 *유리한* 추가 이유 (결합도와 무관)

탐색 헤비 + 산출물로 떨어지는 단계는 결합도가 어중간해도 분리가 낫다:
1. **컨텍스트 위생** — 부푼 raw 컨텍스트를 하위에 격리, 상위엔 증류된 결과만 전달
2. **재사용** — 산출물이 여러 하류 agent에 공급되는 공용 상류 출력
3. **모델 프로파일 차이** — 탐색(haiku/Explore) vs 추론(opus)
4. **병렬화**

### 인라인 vs 분리 vs 하이브리드

전부-분리와 전부-인라인 사이의 **하이브리드**가 대개 sweet spot이다.

- **전부 인라인**: 단순·연속성. 탐색이 가벼우면 합리적.
- **전부 분리**: 격리·병렬·모델분리. 탐색이 무겁고 대규모일 때.
- **하이브리드 (수집만 분리)**: 분리의 핵심 이득(격리·병렬·저렴·재사용)이 가장 큰 **수집(gatherer)만** 밖으로, *정제·종합·도출*은 본체 인라인(연속 추론 유지). → **대부분의 경우 균형점.**

> **하이브리드의 load-bearing 가정**: gatherer가 *raw 덤프가 아니라 정제된 findings*를 반환해야 한다. 그래야 본체 인라인 정제·종합이 깨끗하다. (내장 Explore가 원래 excerpt만 읽고 결론을 반환하므로 자동 충족)
>
> 대규모 프로젝트는 드물고, 한다면 그에 맞는 별도 agent를 두는 게 낫다. 현재 agent는 활용 범위 극대화를 위해 하이브리드로 둔다.

---

## 4. 에이전트 라인업 (현재 설계)

```
expert-claude (오케스트레이터)
├── Analyst        — 요구사항 분석. 질문 프레이밍 + 정제·종합·도출 + 산출물 책임
│   └── gatherer   — (구 explorer) 정보 수집 전담. 정제된 findings 반환(raw 금지)
│                     ※ refine/synthesize/derive는 Analyst 본체 인라인 (하이브리드)
├── Architect      — 전략적 아키텍처 설계
│   └── Planner    — 설계·목표에 따라 task를 의미 있고 적절한 크기 단위로 분해
├── UIDesigner     — Interface and Appearance Designer
├── Examinator     — 요청대로 완료됐는지 검수 (Analyst의 Criteria로 대조)
├── TechnicalWriter — 기술 문서 작성
└── ReferenceBroker — 외부 참조 중개
```

### 라인업 결정 근거

- **Analyst는 하이브리드** — 수집(gatherer)만 분리, 정제·종합·도출은 인라인. (§3 참고)
  - 초기엔 explorer/refiner/synthesizer 3개 sub-agent로 구상했으나, refine/synthesize는 저용량 연속 추론이라 인라인이 유리. 수집만 컨텍스트 위생·병렬·저렴(Explore 재사용) 이득이 커서 분리.
  - 이름은 `explorer` → **`gatherer`** 로 변경.
- **Planner는 Architect 하위로 흡수** — plan↔design은 결합도가 높지만 "설계 이후 작업 분해"는 깔끔한 후속 단계라 sub-agent로 둘 수 있음.
- **sub-agent 표기는 개념적** — "병렬/분리 가능한 단위" 표시. 실제 구현에서 `tools: Agent(...)` 진짜 위임으로 떨어뜨릴지는 상세 단계 결정.

### "workflow 아닌가?" 정리

분석·계획·설계는 *목적(요구사항 분석 등)* 위에 *메커니즘(수집→정제→종합)*이 포개진 구조. 순서는 workflow, 전문가는 agent, 지식은 skill로 갈래를 나눈다. analyzing은 산출물 핸드오프형이라 분리 가능(→ Analyst), planning은 design과 묶임(→ Architect 하위).

---

## 5. Analyst 상세 설계

`agent/analyst.md` 로 구현됨. 핵심 결정:

### 목적 — 요구사항 분석

추상적 "analyze things"가 아니라 **요구사항 분석**이 목적. 산출물(criteria/risk/assumption/edge…)이 하류의 *계약*이 된다. 특히 **criteria ↔ Examinator**가 검수 루프를 닫는다.

### 2층 출력 구조

- **Layer 1 (Analysis, 근거 base)**: Summary / Key findings(출처) / Synthesized insights(추론 표시) — 정보가 *무엇을 말하는가*
- **Layer 2 (Derivation, 프로젝트 함의)**: Requirements / Criteria / Missing questions / Guardrails / Risks / Assumptions / Edge cases / Open·unknowns — 그게 *프로젝트에 요구하는 것*
- **추적성**: L2 항목은 L1을 *참조*(재서술 금지) → 모든 함의가 근거로 추적됨

### Process (순서 정합)

Frame → **Gather(위임)** → Refine → **Reconcile** → **Synthesize** → Derive
- Reconcile(병합·충돌해소)이 Synthesize(교차 추론)보다 **앞**: 종합은 정합된 base를 입력으로 받아야 하므로.
- iteration: Synthesize 중 자료 부족 시 Gather로 되돌아감 — 단, 막는 gap만, 아니면 unknown.

### 외부 analyst 차용 (Yeachan-Heo)

우리 2층·추적성을 유지하며 *출력의 날카로움*을 흡수:
- **테스트 가능성 렌즈** ("Is this testable?", not "valuable?")
- Criteria = **measurable pass/fail**
- 능동적 gap 사냥 (Missing questions / Guardrails + **suggested bounds**)
- 핸드오프 **라우팅** (Planner/Architect)
- **반-나태 조항** ("content-free sign-off violates contract")

### 검토로 반영한 정밀화

검토 항목 번호(#) 기준:

- (#1) Process 순서 Reconcile↔Synthesize 교체 (논리 정합) — *높은 실익*
- (#2) Derivation 빈 버킷 강제 채움 방지 ("실질 있는 칸만, 부재는 점검 결과")
- (#3) Guardrails/Risks/Edge cases 구분 기준 한 줄 (+ 중복 금지)
- (#4) handoff path를 Inputs에서 *주입받음* + 없으면 요청 — *높은 실익(핸드오프 작동)*
- (#5) Layer 1/2 표기를 intro에서 정의(통일)
- (#6) "gatherer는 정제본 반환" 중복(step2↔블록쿼트)은 **스킵** — 프롬프트에선 약한 중복이 강조 효과
- (#8) tool 스코프 명시 — **Write = artifact 전용**, **Read = 명시 제공 context 전용**(open-ended 탐색 금지 → gatherer 우회 차단)

### tools

`Read, Write, Agent(gatherer)`
- Grep/Glob/WebFetch/Bash 제외는 **의도적** — 탐색은 gatherer, 외부는 ReferenceBroker, 실행 안 함.
- Write/Read 둘 다 Boundaries에서 용도·금지 대칭 잠금.

---

## 6. gatherer 상세 설계

`agent/gatherer.md` 로 구현됨. 핵심:

- `model: haiku`, `tools: Read, Grep, Glob` (read-only)
- **출력 계약**: *condensed findings, not raw material* — Analyst가 의존하는 load-bearing 가정을 글로 명시. 각 finding = 사실 1줄 + 출처(file:line)
- **경계**: 사실만 보고, *추론·연결·결론은 금지*(그건 Analyst의 refine/synthesize) → Layer 1 내 역할 분리
- Inputs: "한 번에 focused 한 영역" — Analyst의 병렬 fan-out과 맞물림

---

## 7. 모델 선택 매핑 (verification_instruction 연계)

- Architect → **opus** (고난도 추론)
- Analyst / UIDesigner / TechnicalWriter / Examinator → **sonnet** (단, Analyst는 도출 추론 비중으로 opus 채택)
- gatherer / ReferenceBroker → **haiku** (탐색·fetch 중심)
- **fable-5는 한국에서 사용 불가** → 커스텀 에이전트는 opus를 상한으로.

---

## 8. 전역 원칙 (basic_rule.md)

agent 설계 중 발견한 일반 원칙은 개별 agent가 아니라 `basic_rule.md`에 전역으로 둔다.

- **`<output_quality>`** (신규):
  - 실질 내용 있는 섹션만 보고, 빈 섹션은 boilerplate로 채우지 말 것
  - "Not applicable"/생략은 *진짜 점검의 결과*여야지 회피가 아닐 것
  - → analyst의 "빈 버킷 생략"을 일반화. 모든 agent 출력 템플릿에 적용.
- **핸드오프 = 파일** (프로젝트 결정): 산출 agent는 Write로 artifact 영속화, 하류가 파일을 읽음. 단 Write는 artifact 전용으로 스코프.
- **tool 스코프 패턴**: 각 agent의 Write/Read 등은 "용도 + 금지"를 Boundaries에 명시 (analyst 사례를 표준 패턴으로).

---

## 9. 미해결 / 추후 결정 (TODO)

- [ ] **`<orchestration_instruction>` 채우기** — CLAUDE.md의 유일한 빈 블록. 핸드오프 *레이아웃 규칙*(어느 경로에 무슨 이름)도 여기서 일괄 정의.
- [ ] **구현(coder/builder) 주체** 미정 — 오케스트레이터 본인 or general-purpose or 별도 `Implementer`? (미참조 `dev-code-workflow.md`가 이 자리)
- [ ] **나머지 agent 작성** — Architect(+Planner), UIDesigner, Examinator, TechnicalWriter, ReferenceBroker. analyst 패턴(frontmatter+6섹션, tool 스코프, 2층/계약) 재사용.
- [ ] **Examinator 네이밍** — 조어. `Examiner` 등 검토.
- [ ] **버킷 수 정리(선택)** — Derivation 8버킷이 많다는 신호. 줄인다면 `Guardrails`(과결정 버킷)가 1순위 제거 후보.
- [ ] **미참조 파일 연결** — dev-code-workflow.md, document_writing_guide.md(미완성).

---

## 부록: 커스텀 서브에이전트 파일 포맷 (참고)

위치: `.claude/agents/*.md` (프로젝트) / `~/.claude/agents/*.md` (사용자 전역). Markdown + YAML frontmatter.

| 필드 | 필수 | 비고 |
|---|---|---|
| `name` | ✅ | kebab-case 식별자 |
| `description` | ✅ | **언제 위임하나** — 오케스트레이터 판단 근거 (WHEN형) |
| `tools` | ❌ | 생략 시 전체 상속. `Agent(type1, type2)`로 하위 위임 화이트리스트 |
| `disallowedTools` | ❌ | denylist (tools보다 먼저 적용) |
| `model` | ❌ | sonnet/opus/haiku/fable/풀ID/`inherit` (기본 inherit) |
| `effort` | ❌ | low~max |
| `memory` | ❌ | user/project/local (세션 간 학습) |
| `isolation` | ❌ | worktree |

### 내장 에이전트 (재사용 가능)

| 에이전트 | 모델 | 도구 | 용도 |
|---|---|---|---|
| Explore | Haiku | 읽기전용 | 빠른 코드베이스 탐색·검색 (gatherer로 재사용) |
| Plan | 부모 상속 | 읽기전용 | plan 모드 리서치 |
| general-purpose | 부모 상속 | 전체 | 탐색+수정 복합 작업 |

출처: [Create custom subagents](https://code.claude.com/docs/en/sub-agents.md)
