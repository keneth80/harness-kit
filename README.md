# JARVIS Harness Kit

Claude Code 하네스 엔지니어링이 적용된 프로젝트 템플릿 킷.
`scaffold.sh`를 실행하면 완성된 하네스(에이전트 팀 + 스킬 + 검증 Hook + 슬래시 커맨드)가 포함된 프로젝트를 생성합니다.

---

## 킷 구조

```
jarvis-harness-kit/
├── scaffold.sh        ← 프로젝트 생성기
├── README.md          ← 이 파일
├── GUIDE.md           ← 상세 가이드 (검증 파이프라인, 커스터마이징)
├── template/          ← 도메인 비종속 하네스 템플릿 원본
│   ├── CLAUDE.md                 진입 문서 (메타블록 변수 포함)
│   ├── .claude/
│   │   ├── agents/    (8개)      ui-planner, architect*, ui-designer,
│   │   │                          backend-dev, frontend-dev,
│   │   │                          qa-tester, qa-engineer*, code-verifier
│   │   │                          (*) = 선택적 — 단순 프로젝트는 호출 안 해도 됨
│   │   ├── skills/    (1개)      orchestrator (도메인 비종속 슬롯 템플릿)
│   │   ├── hooks/     (3개)      security_gate, code_reviewer, report_generator
│   │   ├── commands/  (8개)      /dev-start, /plan-start, /architect,
│   │   │                          /ui-design, /test-cases, /verify,
│   │   │                          /qa-boundary, /verify-report
│   │   ├── rules/     (비어 있음) — 프로젝트/도메인별로 추가
│   │   └── settings.json         Hook 설정
│   ├── src/                      Next.js 15 프론트엔드 (선택)
│   ├── backend/                  FastAPI + LangGraph + Playwright (선택)
│   └── .env.example
└── examples/
    └── family-chatbot/           가족 챗봇 도메인 예제 (이전 잔재 보존)
        └── .claude/
            ├── commands/         browser-status
            ├── rules/            cdp-init, ws-protocol
            └── skills/           browser-automation, chatbot-ui, task-routing
```

---

## 에이전트 8개 — 언제 누구를 부르나

| 에이전트 | 호출 시점 | 모델 | 필수/선택 |
|---|---|---|---|
| **ui-planner** | 풀 사이클 시작 (goal/prd) | sonnet | 풀 사이클 필수 |
| **architect** | spec.md/prd.md 직후, 멀티스택 프로젝트 | sonnet | **선택** — 단일 스택 단순 프로젝트엔 과함 |
| **ui-designer** | spec.md 완료 후, UI 있으면 | sonnet | UI 프로젝트 필수 |
| **backend-dev** | 구현 시작 시 | inherit | 백엔드 있으면 필수 |
| **frontend-dev** | 구현 시작 시 | inherit | 프론트 있으면 필수 |
| **qa-tester** | spec.md 완료 후 (TDD) 또는 구현 직후 | haiku | 테스트 작성 필요 시 |
| **qa-engineer** | 모듈 완료 시마다 | haiku | **선택** — 풀스택(백엔드+프론트) 일 때만 가치 |
| **code-verifier** | 코드 변경 직후 자동 | haiku | 항상 필수 |

`*` 표시한 architect와 qa-engineer는 다음 조건일 때만 가치가 큽니다:

- **architect**: 멀티스택(예: Java + React) 또는 빌드 설정이 복잡한 경우. 단일 FastAPI나 단일 Next.js 프로젝트에서는 dev들이 직접 빌드 설정해도 충분.
- **qa-engineer**: 백엔드와 프론트엔드가 둘 다 있고, 사이의 인터페이스 일치가 깨질 위험이 있을 때. 백엔드 또는 프론트만 있는 프로젝트에는 호출 의미 없음.

**검증 3종 — 헷갈릴 때 결정 트리**

```
코드 변경 직후
       │
       ▼
code-verifier (자동) — 변경된 파일이 spec.md 명세대로 동작하는가?
                        ├─ 정적 분석 + 테스트 실행
                        └─ 실패하면 책임 dev에게 수정 요청

모듈 완료 보고
       │
       ▼
풀스택 프로젝트인가?
   ├─ YES → /qa-boundary (qa-engineer) — 백엔드 응답 ↔ 프론트 타입 비교
   └─ NO  → 건너뛰기

테스트 코드를 새로 만들어야 하는가? (spec.md 변경 또는 신규 기능)
       │
       ▼
qa-tester — test-cases.md + tests/ 코드 작성
```

---

## 워크플로우 모드

scaffold.sh는 세 가지 모드를 지원합니다:

| 모드 | 입력 | 첫 단계 |
|---|---|---|
| 1) 간단 | (없음) | `/dev-start` |
| 2) 풀 사이클 (goal) | 한 줄 목표 + 인터뷰 | `/plan-start` → ui-planner |
| 3) 풀 사이클 (PRD) | `docs/prd.md` 직접 입력 | `/architect` 직행 가능 |

PRD 진입 모드는 외부에서 PRD를 받아온 프로젝트나 사용자가 직접 명세를 작성한 경우 ui-planner의 인터뷰 단계를 건너뛸 수 있게 해 줍니다.

---

## 풀 사이클 워크플로우

```
goal.md 또는 prd.md
       │
       ▼
ui-planner ─────► docs/spec.md (단일 진실 원천)
                       │
            ┌──────────┼──────────┐
            ▼          ▼          ▼
       architect   ui-designer  qa-tester
       (스캐폴딩)  (ui-spec.md) (test-cases.md)
            │          │          │
            └──────────┼──────────┘
                       ▼
       ┌──────── backend-dev / frontend-dev (병렬) ────────┐
       │                                                    │
       │ 모듈 하나 완료 시마다 ↓                              │
       │                                                    │
       │   qa-engineer (boundary 검증, progressive)          │
       │   code-verifier (단위 검증, 자동)                    │
       └────────────────────────────────────────────────────┘
                       │
                       ▼
              docs/verification-report.md
              _workspace/04_qa_engineer_report.md
```

각 에이전트는 `_workspace/{NN}_{agent}_report.md`에 보고서를 남깁니다 — 부분 재실행이나 이어하기가 가능합니다.

---

## 실행 가이드

### Step 1: 프로젝트 생성

```bash
cd jarvis-harness-kit
bash scaffold.sh <프로젝트명> [도메인]

# 예시
bash scaffold.sh youtube-pulse webapp
bash scaffold.sh family-chatbot automation   # examples/family-chatbot 룰/스킬 추가 권장
bash scaffold.sh my-api api
```

대화형 프롬프트에서 다음을 선택합니다:

| 항목 | 선택지 | 기본값 |
|------|--------|--------|
| DB | Supabase 로컬, Supabase 클라우드, PostgreSQL, SQLite, 없음 | 없음 |
| 모니터링 | agents-observe, Hook 로깅, 없음 | Hook 로깅 |
| 로컬 LLM | LM Studio, Ollama, 없음 | LM Studio |
| 워크플로우 모드 | 간단, 풀 사이클(goal), 풀 사이클(PRD) | 간단 |

풀 사이클을 선택하면 `docs/goal.md` 또는 `docs/prd.md`가 생성되고, `_workspace/` 디렉토리와 프로젝트 전용 오케스트레이터 슬롯(`.claude/skills/<프로젝트명>-orchestrator/`)이 함께 만들어집니다.

### Step 2: 첫 명령

```bash
cd <프로젝트명>
claude
```

CLAUDE.md가 자동으로 로드되어 하네스 메타블록(목표/트리거)을 통해 프로젝트 컨텍스트를 파악합니다.

- 풀 사이클(goal): `> /plan-start`
- 풀 사이클(PRD): `> /architect`
- 간단 모드: `> /dev-start`

---

## 슬래시 커맨드 (8개, 모두 도메인 비종속)

| 커맨드 | 설명 | 호출 에이전트 |
|--------|------|---|
| `/plan-start` | 기획 시작 (goal/prd 자동 분기) | ui-planner |
| `/architect` | 빌드 설정 + 도메인 엔티티 스캐폴딩 (선택) | architect |
| `/ui-design` | UI 와이어프레임 + ui-spec.md | ui-designer |
| `/test-cases` | 테스트 케이스 명세 + 테스트 코드 | qa-tester |
| `/qa-boundary` | 모듈 사이 boundary 검증 (선택, 풀스택용) | qa-engineer |
| `/verify` | 단위 검증 리포트 생성 | code-verifier |
| `/verify-report` | 최신 검증 대시보드 요약 | (없음, 파일 읽기) |
| `/dev-start` | 프로젝트 현황 분석 + 다음 작업 제안 | (없음, 메인 세션) |

도메인 특화 커맨드(`/browser-status` 등)는 `examples/family-chatbot/.claude/commands/`에 별도 보관.

---

## 검증 대시보드

세션 종료 시 `.claude/reports/latest.html`에 자동 생성됩니다.

```bash
open .claude/reports/latest.html
```

---

## 새 프로젝트 만들기

template/은 항상 원본으로 남아있으므로 같은 하네스 기반으로 다른 프로젝트를 생성 가능합니다:

```bash
cd jarvis-harness-kit
bash scaffold.sh video-factory automation
bash scaffold.sh youtube-pulse webapp
bash scaffold.sh side-project general
```

가족 챗봇 도메인(WebSocket + 브라우저 자동화)이라면 examples 사본을 추가:

```bash
cp -r examples/family-chatbot/.claude/rules/* my-project/.claude/rules/
cp -r examples/family-chatbot/.claude/skills/* my-project/.claude/skills/
```

---

## 로드맵

- [ ] **settings.json 플러그인화** — 현재는 Python 훅 3개를 직접 등록. `harness@harness-marketplace` 같은 플러그인으로 패키징하면 settings.json이 한 줄로 슬림화됨. (별도 마켓플레이스 인프라 필요)
- [ ] **scaffold.sh `--preset` 플래그** — `--preset=family-chatbot|saas|cli` 같은 도메인 프리셋 자동 주입.
- [ ] **examples 디렉토리 확장** — saas, e-commerce, data-pipeline 등 다양한 도메인 예제.
