# JARVIS Browser Chatbot

JARVIS Home AI OS의 브라우저 자동화 에이전트를 실행하는 웹 챗봇 애플리케이션.
가족 구성원이 웹 UI에서 자연어로 브라우저 자동화 작업을 요청하고, 실시간으로 진행 상황을 확인한다.

## 핵심 유스케이스

1. 웹 챗봇에서 "구글시트에서 OOO 찾아서 메타 비즈니스 답장해줘" 입력
2. 백엔드 LangGraph Router가 의도 파악 → 서비스 식별 → 브라우저 자동화 실행
3. 실시간 WebSocket으로 진행 상황 스트리밍 (스크린샷 포함)
4. 완료 시 결과를 챗봇에 표시 + Telegram 알림 발송

## 기술 스택

### Frontend (웹 챗봇)
- Next.js 15 (App Router)
- React 19 + TypeScript
- Tailwind CSS 4
- WebSocket (실시간 메시지 스트리밍)
- next-auth (가족 멀티유저 인증)

### Backend (자동화 엔진)
- Python 3.12, FastAPI (WebSocket + REST)
- LangGraph (StateGraph 기반 파이프라인)
- Playwright (CDP 연결, 브라우저 자동화)
- Browser Use (AI 브라우저 에이전트)
- LM Studio (localhost:1234, OpenAI 호환 API)
- python-telegram-bot v20+ (알림 전용)
- structlog (JSON 로깅)

## 아키텍처

```
┌──────────────┐     WebSocket      ┌──────────────────┐
│  Next.js UI  │◄──────────────────►│   FastAPI Server  │
│  (챗봇 웹)   │     REST API       │                  │
│              │◄──────────────────►│  LangGraph Router │
└──────────────┘                    │       │          │
                                    │       ▼          │
                                    │  BrowserManager  │
                                    │   ┌────┬────┐   │
                                    │   │9222│9223│   │  Telegram
                                    │   │Ggl │Meta│   │──► 알림
                                    │   └────┴────┘   │
                                    └──────────────────┘
```

### 멀티 Chrome 인스턴스

| 포트 | 프로필 | 서비스 | CDP 연결 |
|------|--------|--------|----------|
| 9222 | ~/chrome-profiles/google | Google Sheets, Gmail, Drive | `--remote-debugging-port=9222 --user-data-dir=...` |
| 9223 | ~/chrome-profiles/meta | Meta Business Suite | `--remote-debugging-port=9223 --user-data-dir=...` |
| 9224 | ~/chrome-profiles/general | Naver, Coupang 등 | `--remote-debugging-port=9224 --user-data-dir=...` |

### 파이프라인 흐름

```
웹 챗봇 메시지 (WebSocket)
  → FastAPI → LangGraph Router (의도 파악, 서비스 식별)
    → BrowserManager (해당 포트 Chrome에 CDP 연결)
      → CDP 초기화 (다운로드/권한/다이얼로그)
        → Task 실행 (Sheets 읽기, Meta 메시지 전송 등)
          → 실시간 상태 스트리밍 (WebSocket)
            → 완료 결과 → 챗봇 응답 + Telegram 알림
```

## 의사결정 이력

| 결정 | 선택 | 사유 |
|------|------|------|
| 브라우저 연결 | CDP (`--remote-debugging-port`) | 기존 인증 세션 재사용, 프로필 잠금 충돌 방지 |
| 프론트엔드 | Next.js 15 App Router | SSR, 라우팅, 인증 통합 |
| 실시간 통신 | WebSocket (FastAPI native) | 양방향 스트리밍 필수 |
| 인증 | next-auth + 가족 계정 | 멀티유저, 각자 다른 에이전트 프로필 |
| 로컬 LLM | LM Studio (localhost:1234) | OpenAI 호환, 비용 $0 |
| Telegram | 알림 전용 | 웹 챗봇이 메인 인터페이스 |
| `launch_persistent_context` | ❌ 제외 | 프로필 잠금 충돌 |
| Qwen3-8B tool calling | ❌ 불안정 (50-70%) | 브라우저 자동화에 부적합 |

## 디렉토리 구조

```
jarvis-browser-chatbot/
├── CLAUDE.md                         # 이 파일
├── .claude/                          # 하네스 설정
│   ├── settings.json                 # Hook + 검증 파이프라인
│   ├── hooks/                        # 검증 스크립트 (security_gate, code_reviewer, report_generator)
│   ├── agents/                       # 서브에이전트 정의 (6개)
│   ├── skills/                       # 도메인 스킬 (4개)
│   ├── commands/                     # 슬래시 커맨드
│   ├── rules/                        # 프로젝트 규칙
│   ├── verifier-scripts/             # 검증용 LM Studio 클라이언트
│   └── reports/                      # 검증 대시보드 HTML
│
├── src/                              # Next.js Frontend
│   ├── app/
│   │   ├── layout.tsx                # Root layout
│   │   ├── page.tsx                  # 메인 챗봇 페이지
│   │   ├── api/                      # API Routes (Next.js → FastAPI 프록시)
│   │   │   ├── auth/[...nextauth]/   # next-auth
│   │   │   └── chat/                 # 챗 메시지 REST
│   │   ├── login/                    # 로그인 페이지
│   │   └── settings/                 # 사용자 설정
│   ├── components/
│   │   ├── chat/                     # ChatWindow, MessageBubble, InputBar
│   │   ├── browser/                  # BrowserPreview (실시간 스크린샷)
│   │   ├── status/                   # TaskProgress, ServiceStatus
│   │   └── layout/                   # Header, Sidebar, UserSwitch
│   ├── lib/
│   │   ├── ws.ts                     # WebSocket 클라이언트
│   │   ├── api.ts                    # REST API 클라이언트
│   │   └── auth.ts                   # next-auth 설정
│   ├── hooks/
│   │   ├── useChat.ts                # 챗 상태 관리
│   │   └── useWebSocket.ts           # WS 연결/재연결
│   └── types/
│       └── index.ts                  # 공유 타입 정의
│
├── backend/                          # Python Backend
│   ├── app/
│   │   ├── main.py                   # FastAPI 엔트리포인트
│   │   ├── core/
│   │   │   ├── browser_manager.py    # 멀티 Chrome CDP 관리
│   │   │   ├── cdp_initializer.py    # 다운로드/권한/다이얼로그 초기화
│   │   │   └── config_loader.py      # YAML 설정
│   │   ├── agents/
│   │   │   └── browser_agent.py      # Browser Use + LLM 에이전트
│   │   ├── tasks/
│   │   │   ├── base_task.py          # BaseTask ABC
│   │   │   ├── google_sheets.py      # Sheets 읽기/쓰기
│   │   │   ├── meta_business.py      # Meta DM/댓글
│   │   │   └── task_registry.py      # 태스크 등록/조회
│   │   ├── router/
│   │   │   └── graph.py              # LangGraph StateGraph
│   │   ├── websocket/
│   │   │   └── manager.py            # WS 연결 관리 + 브로드캐스트
│   │   └── telegram/
│   │       └── notifier.py           # 알림 발송 (결과/에러)
│   ├── config/
│   │   ├── browsers.yaml             # Chrome 인스턴스 설정
│   │   └── services.yaml             # 서비스별 URL/셀렉터
│   ├── scripts/
│   │   └── launch-chrome.sh          # Chrome 디버그 모드 실행
│   └── requirements.txt
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── .env.example
├── .gitignore
├── package.json
└── next.config.ts
```

## 코딩 규칙

### 공통
- 시크릿: 환경변수 또는 .env — 하드코딩 절대 금지
- 에러 핸들링: 구체적 예외 타입 사용, bare except 금지
- 커밋: conventional commits (feat:, fix:, refactor:, test:, docs:)

### Frontend (TypeScript)
- strict 모드 필수
- 컴포넌트: function 선언 (arrow 아님), Props 타입 명시
- 상태: React 19 use() + Zustand (필요 시)
- 스타일: Tailwind utility-first, cn() 헬퍼 사용
- 파일 네이밍: PascalCase (컴포넌트), camelCase (유틸)

### Backend (Python)
- 타입 힌트 필수 (Python 3.12: `list[str]`, `dict[str, Any]`)
- async/await 기반 (모든 I/O)
- 로깅: structlog JSON 포맷
- CDP/Playwright 호출: 반드시 try-except + 재시도

## 핵심 제약사항

1. Chrome 136+: `--remote-debugging-port`에 `--user-data-dir` 필수 동반
2. CDP 연결 시 초기화 필수: `Browser.setDownloadBehavior`, `Browser.setPermission`
3. Playwright CDP 연결: `connect_over_cdp("http://localhost:{port}")` 사용
4. WebSocket 메시지 포맷: `{ type, payload, userId, timestamp }` 통일
5. LM Studio API: `http://localhost:1234/v1/chat/completions` (OpenAI 호환)
6. 가족별 동시 접속 지원 — WebSocket 세션은 userId 기준으로 분리
