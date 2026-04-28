# {{PROJECT_NAME}} — Claude Code 진입 문서

이 프로젝트는 jarvis-harness-kit으로 생성되었습니다.
Claude Code가 시작될 때 이 문서를 읽어서 프로젝트 전체 맥락(워크플로우, 에이전트 역할, 명세 위치)을 파악합니다.

---

## 하네스 메타

> scaffold.sh가 풀 사이클 모드에서 자동으로 채웁니다. 간단 모드에서는 비어 있을 수 있습니다.

**하네스 이름:** {{HARNESS_NAME}}

**목표:** {{HARNESS_GOAL}}

**트리거:** {{HARNESS_TRIGGER_KEYWORDS}} 관련 작업 요청 시 `{{ORCHESTRATOR_SKILL_NAME}}` 스킬을 사용하라. 단순 질문은 직접 응답 가능.

**변경 이력:**

| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| {{CREATED_DATE}} | 초기 구성 | 전체 | 하네스 신규 구축 |

---

## 워크플로우 모드

이 프로젝트의 워크플로우 모드는 `.claude/settings.json`의 `workflow_mode` 필드로 결정됩니다.

- **simple** (기본): 사용자가 직접 코드 작성 요청. dev 에이전트가 바로 구현.
- **full_cycle**: 기획 → 스캐폴딩 → UI 설계 → 테스트 케이스 → 구현 → 검증 순으로 진행.

`docs/goal.md` 또는 `docs/prd.md`가 존재하면 풀 사이클 모드입니다.

### PRD 직접 진입 (단축 경로)

`docs/prd.md`가 이미 존재하는 프로젝트(외부에서 PRD를 받은 경우 등)는 ui-planner의 인터뷰 단계를 건너뛰고 바로 architect로 직행할 수 있습니다.

- `docs/prd.md` 있고 `docs/spec.md` 없음 → architect가 PRD에서 직접 데이터 모델/엔드포인트 추출
- `docs/prd.md` 있고 `docs/spec.md` 있음 → spec.md를 단일 진실 원천으로 사용 (PRD는 참조)
- `docs/goal.md`만 있음 → 기존 풀 사이클 흐름(ui-planner부터)

---

## 풀 사이클 워크플로우

### 단계 1: 목표 / PRD (scaffold.sh가 자동 생성하거나 사용자가 직접 배치)
- 산출물: `docs/goal.md` (한 줄 목표 + 답변) 또는 `docs/prd.md` (외부 PRD)
- 수정 금지

### 단계 2: 기획 (ui-planner 에이전트)
- 입력: `docs/goal.md` 또는 `docs/prd.md`
- 산출물: `docs/requirements.md`, `docs/spec.md`
- 트리거: 새 프로젝트 시작 직후, `/plan-start` 명령
- 모델: sonnet
- **PRD 직접 진입**: `docs/prd.md`가 있고 사용자가 동의하면 이 단계를 건너뛰고 단계 3으로

### 단계 3: 스캐폴딩 (architect 에이전트) — **신규**
- 입력: `docs/spec.md` 또는 `docs/prd.md`, `docs/ui-spec.md`(있으면)
- 산출물: 빌드 설정, 도메인 엔티티, 공유 타입(`shared/types/`), 빈 라우트/페이지 골격, `_workspace/01_architect_report.md`
- 트리거: spec.md 작성 완료 후 자동, 또는 PRD 직접 진입 시 첫 단계
- 모델: sonnet
- 종료 조건: 빌드/typecheck/install이 통과해야 함

### 단계 4: UI 설계 (ui-designer 에이전트, UI 있는 프로젝트만)
- 입력: `docs/spec.md`
- 산출물: `docs/mockup.md`, `docs/ui-spec.md`
- 트리거: spec.md 작성 완료 후 (architect와 병렬 가능)
- 모델: sonnet

### 단계 5: 테스트 케이스 (qa-tester 에이전트)
- 입력: `docs/spec.md`
- 산출물: `docs/test-cases.md`, `tests/` 디렉토리의 테스트 코드, `_workspace/03_qa_tester_report.md`
- 트리거: spec.md 완료 후 (구현 시작 전 권장)
- 모델: haiku

### 단계 6: 구현 (frontend-dev, backend-dev)
- 입력: `docs/spec.md`, `docs/ui-spec.md`, `docs/test-cases.md`, architect의 골격
- 산출물: 실제 코드, `_workspace/02_backend_report.md`, `_workspace/02_frontend_report.md`
- 트리거: 사용자 요청 또는 단계 5 완료 직후 병렬
- 모델: inherit

### 단계 7: Boundary 검증 (qa-engineer 에이전트) — **신규, progressive**
- 입력: 방금 완료된 모듈, `docs/spec.md`, `docs/ui-spec.md`
- 산출물: `_workspace/04_qa_engineer_report.md` (모듈마다 섹션 누적)
- 트리거: dev 모듈 하나가 끝날 때마다 호출 (전체 완료 후 한꺼번에 X)
- 모델: haiku

### 단계 8: 단위/통합 검증 (code-verifier 에이전트)
- 입력: 변경된 코드, `docs/test-cases.md`, `docs/spec.md`
- 산출물: `docs/verification-report.md`
- 트리거: 코드 변경 직후 자동
- 모델: haiku (메인 세션과 다른 모델로 self-bias 제거)

---

## 에이전트별 작업 보고서 컨벤션 (`_workspace/`)

각 에이전트는 작업이 끝나면 `_workspace/` 아래에 다음 형식으로 보고서를 남깁니다. 보고서 파일은 다음 실행 시 컨텍스트 체크에 사용되어 **부분 재실행/이어하기**가 가능합니다.

| 파일 | 작성자 | 형식 | 갱신 정책 |
|---|---|---|---|
| `_workspace/00_context_snapshot.md` | 메인 세션 (선택) | 진행 상황 요약 | 단계마다 덮어쓰기 |
| `_workspace/01_architect_report.md` | architect | 결정/생성 파일/빌드 결과/인계 | architect 실행 시마다 덮어쓰기 |
| `_workspace/02_backend_report.md` | backend-dev | 구현한 기능 ID, 엔드포인트, 미완 사항 | 모듈 완료 시마다 누적 |
| `_workspace/02_frontend_report.md` | frontend-dev | 구현한 화면 ID, 컴포넌트, 미완 사항 | 모듈 완료 시마다 누적 |
| `_workspace/03_qa_tester_report.md` | qa-tester | 작성한 테스트 케이스/코드 요약 | spec 변경 시 갱신 |
| `_workspace/04_qa_engineer_report.md` | qa-engineer | Boundary 검증 결과 (모듈별 섹션) | 검증 실행 시마다 누적 |
| `_workspace/05_session_log.md` | 메인 세션 (선택) | 세션별 의사결정 기록 | 세션 종료 시 덮어쓰기 |

`_workspace/`는 `.gitignore`에 포함되지 않습니다 — 팀이 공유해야 하는 진행 상황 기록입니다. 단, 각자 별도 작업한 임시 흔적이 쌓이는 게 부담이라면 `.gitignore`에 추가해도 무방.

---

## docs/ 디렉토리 — 단일 진실의 원천

모든 에이전트는 `docs/`의 문서들을 **단일 진실의 원천**으로 취급합니다.

| 문서 | 작성자 | 수정 가능자 | 읽는 자 |
|---|---|---|---|
| goal.md | scaffold.sh | (수정 금지) | ui-planner |
| prd.md | 사용자/외부 | (수정 금지, 외부 동기화) | ui-planner, architect |
| requirements.md | ui-planner | ui-planner (사용자 승인 시) | architect, dev들 |
| spec.md | ui-planner | ui-planner (사용자 승인 시) | 모든 에이전트 |
| mockup.md | ui-designer | ui-designer | frontend-dev |
| ui-spec.md | ui-designer | ui-designer | frontend-dev, qa-engineer |
| test-cases.md | qa-tester | qa-tester (spec 변경 시) | code-verifier, dev들 |
| verification-report.md | code-verifier | code-verifier (덮어쓰기) | 메인 세션, dev들 |

### 변경 파급 규칙

상류 문서가 바뀌면 하류 문서/코드를 다시 만들어야 합니다:

- **prd.md / spec.md 변경** → architect 재실행 (스캐폴딩 갱신), ui-spec.md, test-cases.md 재검토 필요
- **ui-spec.md 변경** → frontend 코드 재구현 필요, qa-engineer로 boundary 재검증
- **test-cases.md 변경** → tests/ 코드 재생성 필요

이 의존성을 무시하면 명세와 코드 사이에 drift가 발생합니다.

---

## 에이전트 모델 라우팅

| 에이전트 | 모델 | 역할 |
|---|---|---|
| 메인 세션 | Sonnet 또는 Opus | 전체 조율, 사용자 대화 |
| ui-planner | sonnet | 기획, 사용자 추가 질문, spec.md 작성 |
| architect | sonnet | 빌드 설정, 도메인 엔티티, 공유 타입 스캐폴딩 |
| ui-designer | sonnet | UI 와이어프레임, ui-spec.md |
| frontend-dev | inherit | 프론트엔드 구현 |
| backend-dev | inherit | 백엔드 구현 |
| qa-tester | haiku | test-cases.md, 테스트 코드 작성 |
| qa-engineer | haiku | 컴포넌트 boundary 검증 |
| code-verifier | haiku | 다층 검증 + self-bias 제거 |

### 검증 분리 원칙

code-verifier와 qa-engineer는 반드시 메인 세션과 다른 모델(Haiku)을 사용합니다. 같은 모델이 작성한 코드를 같은 모델로 검증하면 self-bias로 인해 문제를 놓치는 경향이 있습니다.

- **code-verifier**: 단일 코드의 명세 일치성 (spec.md 기준)
- **qa-engineer**: 여러 코드 사이의 인터페이스 일치성 (boundary 기준)

두 에이전트는 보완 관계입니다 — 둘 다 통과해야 모듈이 완료된 것입니다.

---

## 슬래시 커맨드

### 풀 사이클 모드용
- `/plan-start` — ui-planner 호출. `docs/prd.md` 있으면 PRD 직접 진입 옵션 제공
- `/architect` — architect 호출하여 스캐폴딩 생성
- `/ui-design` — ui-designer 호출 (spec.md 완료 후)
- `/test-cases` — qa-tester 호출하여 test-cases.md 작성
- `/verify` — code-verifier 호출하여 검증 리포트 생성
- `/qa-boundary` — qa-engineer 호출하여 boundary 검증

### 공통
- `/dev-start` — 프로젝트 현황 분석 + 다음 작업 제안
- `/verify-report` — 최신 검증 리포트 요약

---

## 간단 모드에서의 동작

`docs/goal.md`도 `docs/prd.md`도 없는 간단 모드 프로젝트에서는:
- ui-planner, architect, ui-designer, qa-engineer는 자동 호출되지 않음
- dev 에이전트가 사용자 요청을 직접 수행
- code-verifier는 docs/ 문서 없이 정적 분석 + 보안 검토만 수행
- 풀 사이클로 전환하려면 `docs/goal.md` 또는 `docs/prd.md`를 직접 작성하면 됨

---

## 첫 실행 가이드

### 풀 사이클 모드 (goal.md 기반)

```
claude
> /plan-start
```

ui-planner → architect → ui-designer → qa-tester → dev들 → qa-engineer/code-verifier 순으로 진행됩니다.

### 풀 사이클 모드 (PRD 직접 진입)

`docs/prd.md`를 미리 배치한 경우:

```
claude
> /architect
```

architect가 PRD를 읽고 빌드 설정과 도메인 엔티티를 만든 뒤 dev 팀에 인계합니다.

### 간단 모드

```
claude
> /dev-start
```

기존 jarvis-harness-kit 동작과 동일합니다.

---

## 초기 설정 (scaffold가 자동 채움)
- 도메인: {{DOMAIN}}
- 생성일: {{CREATED_DATE}}
- DB: {{DB_TYPE}}
- 모니터링: {{MONITORING}}
- 로컬 LLM: {{LLM_TYPE}}
