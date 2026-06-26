# expert-claude 개념 설계 논의 기록

> 작성일: 2026-06-27
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

> 요리사(agent)가 생선 손질법(skill)에 따라 칼(tool)을 쓰고, 전체는 코스 순서(workflow)대로 흐른다.
> 같은 칼이라도 목공이냐 요리냐(skill)에 따라 쓰는 법이 다르다 — **tool은 고정, skill이 사용법을 결정한다.**

### expert-claude 대입

- **Tool**: Read / Bash / WebFetch / MCP
- **Skill**: verification 기준, git 규칙, venv 사용법, 문서 작성법 (특정 일을 *어떻게*)
- **Agent**: UIDesigner, TechnicalWriter, ReferenceBroker 등 (전문 역할)
- **Workflow**: 분석→설계→검수→문서 (orchestration_instruction에 기술하는 순서)

---

## 2. Agent vs Workflow — "이거 workflow 아냐?"

라이프사이클 단계(분석→계획→설계→검수)를 나열하면 workflow처럼 보이는 게 자연스럽다. 오케스트레이터의 일이 곧 "에이전트들을 순서로 엮는 것"이라, **에이전트를 잘 만들면 workflow는 emergent하게 나온다.**

판단 기준:
> **별도 agent로 둘 가치 = 고유한 {도구 / 모델 / 시스템프롬프트 / 컨텍스트}가 있는가?**

- 진짜 전문가 (도구·전문성이 다름) → **agent**
- 같은 두뇌의 다른 단계 (사고 모드) → **workflow의 step** 또는 한 agent 내부의 마이크로 워크플로

---

## 3. 분리 vs 통합 — 결합도(coupling)로 판단

여러 단계를 별도 agent로 쪼갤지, 한 agent에 묶을지는 **결합의 *형태*** 로 결정한다.

| 묶어라 (한 agent) | 쪼개라 (별도 agent) |
|---|---|
| **연속적 공유 상태** — 서로 살아있는 작업 컨텍스트를 계속 주고받음 | **artifact handoff** — A가 깔끔한 산출물을 내고 B가 그걸 소비 |

→ verification_instruction의 dependency 규칙과 직결된다:
- `high (tightly coupled) → single strong model` = 통합
- `low (independent) → fan out to subagents` = 분리

### 분리가 *유리한* 추가 이유

탐색 헤비 + 산출물로 떨어지는 단계는 결합도가 어중간해도 분리가 낫다:
1. **컨텍스트 위생** — 부푼 raw 컨텍스트를 하위에 격리, 상위엔 증류된 결과만 전달
2. **재사용** — 산출물이 여러 하류 agent에 공급되는 공용 상류 출력
3. **모델 프로파일 차이** — 탐색(haiku/Explore) vs 추론(opus)
4. **병렬화**

---

## 4. 에이전트 라인업 (현재 설계)

```
expert-claude (오케스트레이터)
├── Analyst       — 분석. 질문 프레이밍 + 수집·정제·종합 조율 + 최종 분석 산출물 책임
│   ├── explorer  — (내장 Explore 재사용) raw 정보 수집, 읽기전용·Haiku·병렬(fan-out)
│   ├── refiner   — select the core signal from gathered info, dropping noise (선택·축소, 새 주장 X)
│   └── analyzer  — derive new information by connecting facts across segments (추론·종합, 새 주장 O)
├── Architect     — 전략적 아키텍처 설계
│   └── Planner   — 설계·목표에 따라 task를 의미 있고 적절한 크기 단위로 분해
├── UIDesigner    — Interface and Appearance Designer
├── Examinator    — 요청대로 완료됐는지 검수
├── TechnicalWriter — 기술 문서 작성
└── ReferenceBroker — 외부 참조 중개
```

### Analyst 내부: 수집 → 정제 → 종합 사다리

| sub-agent | 작업 | 정보 단계 |
|---|---|---|
| explorer | 수집 (raw) | 데이터 |
| refiner | 정제·선택 (있는 것 추림) | 정보 |
| analyzer | 종합·추론 (없던 것 도출) | 통찰 |

- explorer는 fan-out(병렬 map), refiner/analyzer는 그 뒤를 잇는 처리 단계 (map-reduce 형태).
- refiner(선택)와 analyzer(도출)는 다른 인지 작업 — refiner는 새 주장을 만들지 않고, analyzer는 추론으로 새 주장을 생성한다.

### 결정 사항 / 근거

- **Planner는 Architect의 하위로 흡수.** plan↔design은 결합도가 높지만, "설계 이후 작업 분해"는 비교적 깔끔한 후속 단계(artifact handoff)라 sub-agent로 둘 수 있다.
- **Analyst는 독립 유지.** 분석은 탐색 헤비 + 산출물 핸드오프라 컨텍스트 위생 측면에서 분리가 유리.
- **sub-agent 표기는 개념적** — "병렬/분리 가능한 단위"를 나타낸 것. 실제 구현에서 `tools: Agent(...)` 진짜 위임으로 떨어뜨릴지(격리 ✅, 비용↑), Analyst 내부 inline으로 둘지(가볍지만 격리 ✗)는 상세 정의 단계에서 결정.

---

## 5. 모델 선택 매핑 (verification_instruction 연계)

- Architect → **opus** (고난도 추론)
- Analyst / UIDesigner / TechnicalWriter / Examinator → **sonnet**
- explorer / ReferenceBroker → **haiku** (탐색·fetch 중심)
- **fable-5는 한국에서 사용 불가** → 커스텀 에이전트는 opus를 상한으로.

---

## 6. 미해결 / 추후 결정 (TODO)

- [ ] **구현(coder/builder) 주체** 미정 — 오케스트레이터 본인 or general-purpose or 별도 `Implementer`? (미참조 `dev-code-workflow.md`가 이 자리로 보임)
- [ ] **Analyst iteration** — 1패스 vs 루프(analyzer↔explorer). 권장: **start 1-pass → analyzer가 정보 부족으로 막히는 게 관측되면 그때 루프 추가** (단순하게 시작).
- [ ] **네이밍** — `Examinator`(조어)는 상세 정의 단계에서 `Examiner` 등 관례적 이름 검토.
- [ ] **description를 WHEN형으로** — 각 agent의 description은 "무엇인가(IS)"가 아니라 "언제 호출하나(WHEN)"로 기술해야 오케스트레이터가 위임 판단에 쓴다.
- [ ] **상세 frontmatter** — name/description/tools/model/effort, `tools: Agent(...)` 위임 화이트리스트 등 상세 정의.
- [ ] **언어 통일** — 프로필 설명은 영어로 통일.

---

## 부록: 커스텀 서브에이전트 파일 포맷 (참고)

위치: `.claude/agents/*.md` (프로젝트) / `~/.claude/agents/*.md` (사용자 전역). Markdown + YAML frontmatter.

| 필드 | 필수 | 비고 |
|---|---|---|
| `name` | ✅ | kebab-case 식별자 |
| `description` | ✅ | **언제 위임하나** — 오케스트레이터 판단 근거 |
| `tools` | ❌ | 생략 시 전체 상속. `Agent(type1, type2)`로 하위 위임 화이트리스트 가능 |
| `disallowedTools` | ❌ | denylist (tools보다 먼저 적용) |
| `model` | ❌ | sonnet/opus/haiku/fable/풀ID/`inherit` (기본 inherit) |
| `effort` | ❌ | low~max |
| `memory` | ❌ | user/project/local (세션 간 학습) |
| `isolation` | ❌ | worktree |

### 내장 에이전트 (재사용 가능)

| 에이전트 | 모델 | 도구 | 용도 |
|---|---|---|---|
| Explore | Haiku | 읽기전용 | 빠른 코드베이스 탐색·검색 (explorer로 재사용) |
| Plan | 부모 상속 | 읽기전용 | plan 모드 리서치 |
| general-purpose | 부모 상속 | 전체 | 탐색+수정 복합 작업 |

출처: [Create custom subagents](https://code.claude.com/docs/en/sub-agents.md)
