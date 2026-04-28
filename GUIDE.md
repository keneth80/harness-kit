# Claude Code 검증 하네스 — 설치 & 실행 가이드

## 1. 전체 구조 이해

```
scaffold.sh 실행 → 프로젝트 폴더 생성
                      │
                      ▼
my-project/
├── CLAUDE.md                          ⬅ Claude Code가 자동 로드하는 프로젝트 컨텍스트
├── .gitignore
├── src/                               ⬅ 소스 코드 작성 위치
├── tests/                             ⬅ 테스트 코드
│
└── .claude/                           ⬅ Claude Code 설정 디렉토리 (핵심)
    ├── settings.json                  ⬅ Hook 설정 — 어떤 이벤트에 어떤 스크립트 실행할지
    │
    ├── hooks/                         ⬅ 검증 스크립트 3종 (자동 실행됨)
    │   ├── security_gate.py           │  PreToolUse  → Bash 명령 실행 전 위험 차단
    │   ├── code_reviewer.py           │  PostToolUse → 파일 수정 후 AI 코드 리뷰
    │   └── report_generator.py        │  Stop        → 세션 종료 시 HTML 리포트 생성
    │
    ├── verifier-scripts/              ⬅ Hook들이 공유하는 유틸리티
    │   └── llm_client.py              │  LM Studio OpenAI 호환 API 클라이언트
    │
    ├── agents/                        ⬅ 서브에이전트 정의 (8개)
    │   ├── ui-planner.md              │  goal.md/prd.md → spec.md 작성
    │   ├── architect.md               │  빌드 설정 + 도메인 엔티티 스캐폴딩
    │   ├── ui-designer.md             │  UI 와이어프레임 + ui-spec.md
    │   ├── backend-dev.md             │  백엔드 구현
    │   ├── frontend-dev.md            │  프론트엔드 구현
    │   ├── qa-tester.md               │  테스트 케이스 작성
    │   ├── qa-engineer.md             │  컴포넌트 boundary 검증 (progressive)
    │   └── code-verifier.md           │  단위 검증 + self-bias 제거 (Haiku)
    │
    ├── skills/
    │   └── orchestrator/              ⬅ 도메인 비종속 슬롯 템플릿
    │                                     scaffold가 <프로젝트명>-orchestrator로 인스턴스 생성
    │
    ├── commands/                      ⬅ 슬래시 커맨드 (8개, 모두 도메인 비종속)
    │   ├── plan-start.md              │  /plan-start → ui-planner
    │   ├── architect.md               │  /architect → 스캐폴딩 (선택)
    │   ├── ui-design.md               │  /ui-design → UI 설계
    │   ├── test-cases.md              │  /test-cases → 테스트 케이스
    │   ├── qa-boundary.md             │  /qa-boundary → qa-engineer (선택)
    │   ├── verify.md                  │  /verify → code-verifier
    │   ├── verify-report.md           │  /verify-report → 리포트 요약
    │   └── dev-start.md               │  /dev-start → 현황 분석
    │
    └── reports/                       ⬅ 검증 대시보드 HTML 출력
        └── latest.html → (심볼릭 링크)

_workspace/                            ⬅ 에이전트 작업 보고서 (부분 재실행용)
├── 00_context_snapshot.md             │  진행 상황 스냅샷
├── 01_architect_report.md             │  architect 산출물
├── 02_backend_report.md               │  backend-dev 모듈별 누적
├── 02_frontend_report.md              │  frontend-dev 화면별 누적
├── 03_qa_tester_report.md             │  qa-tester 테스트 케이스 요약
└── 04_qa_engineer_report.md           │  qa-engineer boundary 검증 누적
```


## 2. 검증 파이프라인 동작 흐름

```
당신이 Claude Code에서 작업 중...

┌─────────────────────────────────────────────────────────────┐
│  "이 파일 수정해줘" 또는 "이 스크립트 실행해줘"              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
         ┌─── Bash 명령인가? ───┐
         │ YES                  │ NO
         ▼                      │
  security_gate.py              │
  (regex 패턴 매칭)             │
  ┌──────────┐                  │
  │ rm -rf / │→ 🛑 차단         │
  │ chmod 777│→ 🛑 차단         │
  │ ls -la   │→ ✅ 통과         │
  └──────────┘                  │
         │                      │
         ▼                      ▼
    Claude가 도구 실행 (Write/Edit/Bash)
                       │
                       ▼
         ┌─── 파일 수정인가? ───┐
         │ YES                  │ NO
         ▼                      │
  code_reviewer.py              │
  ┌────────────────────┐        │
  │ Phase 1: 정적 분석 │        │
  │  - API 키 노출?    │        │
  │  - eval() 사용?    │        │
  │  - SQL injection?  │        │
  │                    │        │
  │ CRITICAL → 🛑 즉시 │        │
  │                    │        │
  │ Phase 2: AI 리뷰   │        │
  │  (LM Studio 로컬)  │        │
  │  - 보안 심층 분석   │        │
  │  - 성능 패턴 검사   │        │
  │  - 코드 맥락 평가   │        │
  │                    │        │
  │ should_block?      │        │
  │  true → 🛑 차단     │        │
  │  false → ✅ 통과    │        │
  └────────────────────┘        │
         │                      │
         ▼                      ▼
    결과를 audit.jsonl에 기록
                       │
                       ▼
    세션 종료 ("exit" 또는 Ctrl+C)
                       │
                       ▼
  report_generator.py (백그라운드)
  ┌────────────────────────────┐
  │ audit.jsonl 분석            │
  │ → HTML 대시보드 생성        │
  │ → .claude/reports/latest.html │
  └────────────────────────────┘
```


## 3. 설치 (3단계)

### Step 1: scaffold.sh 다운로드 & 실행

```bash
# scaffold.sh를 원하는 위치에 다운로드 후:
bash scaffold.sh my-project webapp

# 또는 도메인 지정:
bash scaffold.sh jarvis-browser automation
bash scaffold.sh my-api api
bash scaffold.sh side-project        # 기본(general)
```

### Step 2: LM Studio 서버 준비 (선택사항)

```bash
# LM Studio가 없거나 서버가 꺼져있으면 정적 분석만 동작 (문제 없음)
# AI 리뷰까지 원하면:

# 1. LM Studio 실행
# 2. 원하는 모델 로드 (예: qwen3-8b, GLM-4.7-9B 등)
# 3. Local Server 탭 > Start Server (기본 포트 1234)

# 모델명 확인:
curl -s http://localhost:1234/v1/models | python3 -m json.tool

# 모델명 변경하고 싶으면:
#   .claude/verifier-scripts/llm_client.py 에서
#   DEFAULT_MODEL = "qwen3-8b" 부분을 LM Studio에 로드한 모델명으로 변경
#   (단일 모델만 로드했으면 아무 값이나 넣어도 동작)
#
# 포트 변경 시:
#   LMSTUDIO_BASE_URL = "http://localhost:1234" 부분 수정
```

### Step 3: Claude Code 시작

```bash
cd my-project
claude
```

이 시점에서 벌써 검증이 작동합니다. Claude Code가 `.claude/settings.json`을 자동으로 읽고 Hook을 등록합니다.


## 4. Harness 적용 (에이전트 팀 구성)

검증 하네스 위에 revfactory/harness를 올려서 도메인 특화 에이전트 팀을 생성합니다.

```bash
# Claude Code 안에서:

# 1. Harness 플러그인 설치
/plugin marketplace add revfactory/harness
/plugin install harness@harness

# 2. 하네스 구성 요청
> 하네스 구성해줘
# 또는
> Build a harness for this project. 웹 애플리케이션 개발용으로
#   React + FastAPI 스택, 인증/결제/관리자 기능 포함.
```

Harness가 `.claude/agents/`와 `.claude/skills/`에 파일을 생성합니다.
6가지 아키텍처 패턴 중 프로젝트에 맞는 것을 자동 선택:

| 패턴 | 용도 |
|------|------|
| Pipeline | 순차 처리 (기획→개발→테스트→배포) |
| Fan-out/Fan-in | 병렬 작업 후 합치기 |
| Expert Pool | 전문가 에이전트 풀에서 적합한 에이전트 선택 |
| Producer-Reviewer | 생성→검증 반복 |
| Supervisor | 감독자가 하위 에이전트 조율 |
| Hierarchical | 계층적 위임 |

생성 후 프로젝트 구조:

```
.claude/
├── settings.json          ← Hook (검증)
├── hooks/                 ← 검증 스크립트
├── verifier-scripts/
├── agents/
│   ├── code-verifier.md   ← 검증 에이전트 (scaffold가 생성)
│   ├── frontend-dev.md    ← 프론트엔드 에이전트 (harness가 생성)
│   ├── backend-dev.md     ← 백엔드 에이전트 (harness가 생성)
│   └── qa-tester.md       ← QA 에이전트 (harness가 생성)
├── skills/
│   └── harness/
│       └── ...            ← 도메인 스킬 (harness가 생성)
├── commands/
│   ├── verify-report.md
│   └── verify-status.md
└── reports/
```


## 5. 실제 사용 시나리오

### 시나리오 A: 정상 개발 흐름

```
You: 사용자 로그인 API를 만들어줘

Claude: (Write 도구로 src/auth/login.py 생성)
        ↓
        code_reviewer.py 자동 실행
        ├─ 정적: 시크릿 없음 ✅
        └─ AI: 보안 OK, 성능 OK, 맥락 OK ✅
        ↓
        파일 생성 완료!

You: exit
        ↓
        report_generator.py 자동 실행 (백그라운드)
        → .claude/reports/latest.html 생성

You: (브라우저에서 리포트 확인)
```

### 시나리오 B: 위험 감지 & 차단

```
You: DB를 초기화하는 스크립트 실행해줘

Claude: (Bash 도구로 "DROP TABLE users" 실행 시도)
        ↓
        security_gate.py 자동 실행
        🛑 차단: "DB 삭제 명령"
        ↓
Claude: "보안 게이트가 이 명령을 차단했습니다.
         DROP TABLE은 직접 실행할 수 없습니다.
         마이그레이션 스크립트를 통해 안전하게 처리할까요?"
```

### 시나리오 C: AI가 시크릿 하드코딩 시도

```
Claude: (Edit 도구로 config.py에 API_KEY = "sk-abc123..." 추가 시도)
        ↓
        code_reviewer.py 자동 실행
        Phase 1 정적: 🔴 CRITICAL: OpenAI API키 하드코딩
        🛑 즉시 차단 (LLM 호출 안 함)
        ↓
Claude: "코드 검증에서 차단되었습니다.
         API 키를 환경변수로 변경하겠습니다."
        ↓
        (Edit로 os.environ.get("OPENAI_API_KEY") 변경)
        ↓
        code_reviewer.py: 정적 ✅, AI ✅ → 통과
```


## 6. 커스터마이징

### 검증 강도 조절

```python
# .claude/hooks/code_reviewer.py 에서

# AI 리뷰 프롬프트 수정 (REVIEW_SYSTEM 변수)
# → "should_block=true 기준을 medium 이상으로 변경" 등

# 정적 분석 패턴 추가
SECRET_PATTERNS = [
    # 기존 패턴...
    (r'AKIA[0-9A-Z]{16}', "AWS Access Key"),  # ← 추가
]
```

### 특정 파일/폴더 스킵

```python
# code_reviewer.py의 SKIP_PATTERNS에 추가
SKIP_PATTERNS = [
    # 기존...
    "migrations/",    # DB 마이그레이션은 스킵
    "fixtures/",      # 테스트 데이터 스킵
]
```

### 모델 교체

```python
# .claude/verifier-scripts/llm_client.py

# 모델명은 LM Studio에서 로드한 모델의 identifier
# Local Server 탭 또는 curl http://localhost:1234/v1/models 로 확인
DEFAULT_MODEL = "qwen3-8b"        # 기본값 (LM Studio 로드 모델명)

# 포트 변경 시:
LMSTUDIO_BASE_URL = "http://localhost:1234"  # LM Studio 기본 포트
# LMSTUDIO_BASE_URL = "http://localhost:8080"  # 커스텀 포트
```

### Telegram 알림 연동 (JARVIS용)

```python
# report_generator.py main() 끝에 추가:
import requests
TELEGRAM_BOT_TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
TELEGRAM_CHAT_ID = os.environ.get("TELEGRAM_CHAT_ID")
if TELEGRAM_BOT_TOKEN and stats["blocked"] > 0:
    msg = f"🛑 검증 리포트: {stats['blocked']}건 차단, {stats['passed']}건 통과"
    requests.post(
        f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage",
        json={"chat_id": TELEGRAM_CHAT_ID, "text": msg}
    )
```


## 7. 새 프로젝트 시작 체크리스트

### 풀 사이클 (goal 기반)
```
□ bash scaffold.sh <이름> <도메인>      # 모드 [2] 풀 사이클(goal)
□ cd <이름>
□ LM Studio에서 모델 로드 & 서버 시작 (선택)
□ claude
□ /plan-start                          # ui-planner → spec.md
□ /architect                           # 빌드 설정 + 도메인 엔티티
□ /ui-design                           # UI 있으면
□ /test-cases                          # 테스트 케이스
□ "백엔드 F1 모듈 구현해줘"             # backend-dev
□ /qa-boundary                         # 모듈 완료마다 progressive
□ "프론트엔드 ScreenA 구현해줘"         # frontend-dev
□ /qa-boundary                         # 또 호출
□ /verify                              # 최종 단위 검증
□ 세션 종료 후 .claude/reports/latest.html 확인
```

### 풀 사이클 (PRD 직접 진입)
```
□ docs/prd.md 미리 작성 또는 외부에서 받기
□ bash scaffold.sh <이름> <도메인>      # 모드 [3] PRD
□ cd <이름>
□ claude
□ /architect                           # ui-planner 건너뛰고 직행
□ (이후 동일)
```

### 간단 모드
```
□ bash scaffold.sh <이름> <도메인>      # 모드 [1] 간단
□ cd <이름>
□ claude
□ /dev-start
□ 자유 개발 → Hook이 알아서 검증
```

---

## 8. 에이전트 선택 가이드 (중복 같지만 다른 일)

비슷해 보이는 에이전트들이 정말 다른 일을 하는지, 그리고 언제 호출해야 하는지를 명확히 합니다.

### 검증 3종 — qa-tester / qa-engineer / code-verifier

| 항목 | qa-tester | qa-engineer | code-verifier |
|---|---|---|---|
| 무엇을 보는가 | 함수가 명세대로 동작하는가 | 모듈 사이가 맞물리는가 | 단일 파일이 spec과 일치하는가 |
| 검증 단위 | 단위(함수) | 경계(인터페이스) | 변경된 파일 |
| 도구 | Write (테스트 코드 작성) | Read/Grep (비교만) | Read/Grep/Bash (린트+테스트 실행) |
| 호출 시점 | 구현 전(TDD) 또는 직후 1회 | 모듈 완료 시마다 progressive | 코드 변경 직후 자동 |
| 산출물 | docs/test-cases.md, tests/ | _workspace/04_qa_engineer_report.md | docs/verification-report.md |
| 모델 | haiku | haiku | haiku (메인이 sonnet/opus여도 self-bias 차단) |

세 에이전트를 한 명에게 통합하지 않는 이유:

- 작성과 검증을 같은 모델이 하면 self-bias가 끼어 결함을 놓칩니다.
- "각 함수가 동작하는가"와 "여러 함수가 맞물리는가"는 다른 시각입니다 — 단위 테스트 100% 통과해도 boundary는 깨질 수 있습니다.
- code-verifier는 자동 호출 부담이 작아야 해서 boundary처럼 무거운 비교는 빼야 합니다.

### 기획/설계 2종 — ui-planner / architect

| 항목 | ui-planner | architect |
|---|---|---|
| 결과물 | docs/spec.md (무엇을 만들지) | 빌드 설정 + 빈 코드 골격 |
| 산출물 형태 | 문서 | 코드 (단, 비즈니스 로직 없음) |
| 필수성 | 풀 사이클 필수 | **선택** — 단일 스택이면 dev들이 직접 |

architect를 부를지 결정 트리:

```
멀티스택인가? (예: Java 백엔드 + React 프론트)
   ├─ YES → architect 권장 (공유 타입 + 빌드 설정 동기화 가치)
   └─ NO  → 단일 스택?
            ├─ Next.js만 / FastAPI만 → architect 건너뛰기, dev가 직접
            └─ 빌드 설정이 복잡(Gradle 멀티모듈 등) → architect 권장
```

### qa-engineer를 부를지 결정 트리

```
프로젝트가 백엔드 + 프론트 둘 다 있는가?
   ├─ NO  → qa-engineer 호출 의미 없음 (boundary가 없음)
   └─ YES → 매 모듈 완료마다 /qa-boundary 호출
            (전체 완료 후 한꺼번에 X — progressive로!)
```

### 정리: 프로젝트 유형별 권장 에이전트 셋

| 프로젝트 유형 | 호출하는 에이전트 |
|---|---|
| 단일 백엔드 (FastAPI 단독) | ui-planner → backend-dev → qa-tester → code-verifier |
| 단일 프론트 (Next.js 단독) | ui-planner → ui-designer → frontend-dev → qa-tester → code-verifier |
| 풀스택 단일언어 (Next.js + Route Handlers) | ui-planner → ui-designer → frontend-dev → qa-tester → code-verifier |
| 풀스택 멀티언어 (FastAPI + Next.js) | ui-planner → **architect** → ui-designer → qa-tester → backend-dev / frontend-dev → **qa-engineer** → code-verifier |
| CLI / API 전용 | ui-planner → backend-dev → qa-tester → code-verifier |

architect와 qa-engineer는 풀스택 멀티언어에서만 강력 권장. 그 외엔 호출 안 해도 무방합니다.

---

## 9. 로드맵 (settings.json 플러그인화)

현재는 Python 훅 3개(`security_gate.py`, `code_reviewer.py`, `report_generator.py`)를 직접 등록합니다. 향후 `harness@harness-marketplace` 형태로 플러그인화하면:

- 사용자 프로젝트의 `settings.json`이 `{ "enabledPlugins": { "harness@...": true } }` 한 줄로 슬림화됨
- 훅 업데이트 시 모든 프로젝트가 일관되게 갱신 (수동 동기화 불필요)
- 도메인 프리셋(`@harness/family-chatbot`, `@harness/saas`)을 마켓플레이스에서 받기 가능

이 작업은 별도 마켓플레이스 인프라가 필요하므로 단계적으로 진행합니다.
