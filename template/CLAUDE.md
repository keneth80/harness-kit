# {{PROJECT_NAME}} — Claude Code 진입 문서

이 프로젝트는 jarvis-harness-kit으로 생성되었습니다.
Claude Code가 시작될 때 이 문서를 읽어서 프로젝트 전체 맥락(워크플로우, 에이전트 역할, 명세 위치)을 파악합니다.

---

## 워크플로우 모드

이 프로젝트의 워크플로우 모드는 `.claude/settings.json`의 `workflow_mode` 필드로 결정됩니다.

- **simple** (기본): 사용자가 직접 코드 작성 요청. dev 에이전트가 바로 구현.
- **full_cycle**: 기획 → UI 설계 → 테스트 케이스 → 구현 → 검증 순으로 진행.

`docs/goal.md`가 존재하면 풀 사이클 모드입니다.

---

## 풀 사이클 워크플로우

### 단계 1: 목표 (scaffold.sh가 자동 생성)
- 산출물: `docs/goal.md`
- 사용자가 scaffold.sh 실행 시 입력한 원본 (수정 금지)

### 단계 2: 기획 (ui-planner 에이전트)
- 입력: `docs/goal.md`
- 산출물: `docs/requirements.md`, `docs/spec.md`
- 트리거: 새 프로젝트 시작 직후, `/plan-start` 명령, 또는 사용자가 "기획부터 시작"이라 요청
- 모델: sonnet

### 단계 3: UI 설계 (ui-designer 에이전트, UI 있는 프로젝트만)
- 입력: `docs/spec.md`
- 산출물: `docs/mockup.md`, `docs/ui-spec.md`
- 트리거: spec.md 작성 완료 후
- 스킵: CLI/API 프로젝트는 mockup.md에 "UI 없음" 명시 후 종료
- 모델: sonnet

### 단계 4: 테스트 케이스 (qa-tester 에이전트)
- 입력: `docs/spec.md`
- 산출물: `docs/test-cases.md`, `tests/` 디렉토리의 테스트 코드
- 트리거: spec.md 완료 후 (구현 시작 전 권장)
- 모델: haiku

### 단계 5: 구현 (frontend-dev, backend-dev)
- 입력: `docs/spec.md`, `docs/ui-spec.md`, `docs/test-cases.md`
- 산출물: `src/`, `backend/` 등 실제 코드
- 트리거: 사용자 요청 또는 단계 4 완료 직후
- 모델: inherit (메인 세션과 동일)

### 단계 6: 검증 (code-verifier 에이전트)
- 입력: 변경된 코드, `docs/test-cases.md`, `docs/spec.md`
- 산출물: `docs/verification-report.md`
- 트리거: 코드 변경 직후 자동
- 모델: haiku (메인 세션과 다른 모델로 self-bias 제거)

---

## docs/ 디렉토리 — 단일 진실의 원천

모든 에이전트는 `docs/`의 문서들을 **단일 진실의 원천**으로 취급합니다.

| 문서 | 작성자 | 수정 가능자 | 읽는 자 |
|---|---|---|---|
| goal.md | scaffold.sh | (수정 금지) | ui-planner |
| requirements.md | ui-planner | ui-planner (사용자 승인 시) | ui-designer, dev들 |
| spec.md | ui-planner | ui-planner (사용자 승인 시) | 모든 에이전트 |
| mockup.md | ui-designer | ui-designer | frontend-dev |
| ui-spec.md | ui-designer | ui-designer | frontend-dev |
| test-cases.md | qa-tester | qa-tester (spec 변경 시) | code-verifier, dev들 |
| verification-report.md | code-verifier | code-verifier (덮어쓰기) | 메인 세션, dev들 |

### 변경 파급 규칙

상류 문서가 바뀌면 하류 문서/코드를 다시 만들어야 합니다:

- **spec.md 변경** → ui-spec.md, test-cases.md 재검토 필요
- **ui-spec.md 변경** → frontend 코드 재구현 필요
- **test-cases.md 변경** → tests/ 코드 재생성 필요

이 의존성을 무시하면 명세와 코드 사이에 drift가 발생합니다.

---

## 에이전트 모델 라우팅

| 에이전트 | 모델 | 역할 |
|---|---|---|
| 메인 세션 | Sonnet 또는 Opus | 전체 조율, 사용자 대화 |
| ui-planner | sonnet | 기획, 사용자 추가 질문, spec.md 작성 |
| ui-designer | sonnet | UI 와이어프레임, ui-spec.md |
| frontend-dev | inherit | 프론트엔드 구현 |
| backend-dev | inherit | 백엔드 구현 |
| qa-tester | haiku | test-cases.md, 테스트 코드 작성 |
| code-verifier | haiku | 다층 검증 + self-bias 제거 |

### 검증 분리 원칙

code-verifier는 반드시 메인 세션과 다른 모델(Haiku)을 사용합니다. 같은 모델이 작성한 코드를 같은 모델로 검증하면 self-bias로 인해 문제를 놓치는 경향이 있습니다. Haiku는 메인이 Sonnet이든 Opus든 충분히 다른 시각을 제공합니다.

---

## 슬래시 커맨드

### 풀 사이클 모드용
- `/plan-start` — ui-planner 호출하여 docs/spec.md 작성
- `/ui-design` — ui-designer 호출 (spec.md 완료 후)
- `/test-cases` — qa-tester 호출하여 test-cases.md 작성
- `/verify` — code-verifier 호출하여 검증 리포트 생성

### 공통
- `/dev-start` — 프로젝트 현황 분석 + 다음 작업 제안
- `/verify-report` — 최신 검증 리포트 요약

---

## 간단 모드에서의 동작

`docs/goal.md`가 없는 간단 모드 프로젝트에서는:
- ui-planner, ui-designer는 호출되지 않음
- dev 에이전트가 사용자 요청을 직접 수행
- code-verifier는 docs/ 문서 없이 정적 분석 + 보안 검토만 수행
- 풀 사이클로 전환하려면 `docs/goal.md`를 직접 작성하면 됨

---

## 에이전트 모델 라우팅 정책

이 프로젝트는 비용/품질/속도 균형을 위해 sub-agent별로 모델을 분리한다.

| 에이전트 | 모델 | 이유 |
|---|---|---|
| 메인 세션 | Sonnet 또는 Opus | 아키텍처 판단, 통합 |
| frontend-dev | inherit (메인과 동일) | 구현 품질 중요 |
| backend-dev | inherit (메인과 동일) | 구현 품질 중요 |
| qa-tester | haiku | 단순 테스트 작성, 비용 절감 |
| code-verifier | haiku | self-bias 제거 + 빠른 피드백 |

### 검증 분리 원칙

code-verifier는 반드시 메인 세션과 다른 모델을 사용한다. 같은 모델이 작성한 코드를 같은 모델로 검증하면 self-bias로 인해 문제를 놓치는 경향이 있다. Haiku는 메인이 Sonnet이든 Opus든 충분히 다른 시각을 제공한다.

코드 작성 직후 자동으로 code-verifier에 위임하여 검증 결과를 받은 뒤 다음 작업으로 넘어간다.

## 첫 실행 가이드

### 풀 사이클 모드인 경우

```
claude
> /plan-start
```

ui-planner가 goal.md를 읽고 추가 질문 후 spec.md를 작성합니다.
사용자 승인 후 다음 단계(ui-design 또는 test-cases)로 진행됩니다.

### 간단 모드인 경우

```
claude
> /dev-start
```

기존 jarvis-harness-kit 동작과 동일합니다.
