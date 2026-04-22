---
name: orchestrator
trigger: "오케스트레이션|조율|에이전트 간|데이터 전달|파이프라인 연결|통합"
---

# Orchestrator Skill

에이전트 간 데이터 전달, 프론트/백 통합, WebSocket 메시지 프로토콜 지식.

## 에이전트 팀 아키텍처: Pipeline + Supervisor 혼합

```
┌────────────────────────────────────────────┐
│                Supervisor                   │
│  (요청에 따라 적절한 에이전트에 위임)          │
├──────────┬──────────┬──────────┬───────────┤
│frontend  │ backend  │ qa-tester│  verifier │
│  -dev    │  -dev    │          │           │
└──────────┴──────────┴──────────┴───────────┘
```

Pipeline 내부:
```
frontend-dev (UI 구현) → backend-dev (API 구현) → qa-tester (테스트 작성/실행)
```

## WebSocket 메시지 프로토콜 (프론트↔백 공통)

```typescript
interface WSMessage {
  type: string;              // 메시지 타입
  payload: Record<string, any>;  // 데이터
  userId: string;            // 가족 구성원 ID
  timestamp: string;         // ISO 8601
}
```

### 메시지 타입 목록

| 방향 | type | payload | 설명 |
|------|------|---------|------|
| Client→Server | `chat_send` | `{ content }` | 사용자 메시지 전송 |
| Server→Client | `chat_response` | `{ content, role }` | 어시스턴트 응답 |
| Server→Client | `task_start` | `{ taskId, service, description }` | 작업 시작 알림 |
| Server→Client | `task_progress` | `{ taskId, step, progress, screenshot? }` | 진행 상황 |
| Server→Client | `task_complete` | `{ taskId, result, duration }` | 완료 |
| Server→Client | `task_error` | `{ taskId, error, recoverable }` | 에러 |
| Server→Client | `service_status` | `{ services: {name, connected}[] }` | Chrome 상태 |

## 프론트/백 통합 체크리스트

- [ ] WebSocket URL: `ws://localhost:8000/ws/{userId}`
- [ ] REST API URL: `http://localhost:8000/api/v1/...`
- [ ] Next.js API Route에서 FastAPI로 프록시 (`/api/chat` → `localhost:8000`)
- [ ] CORS 설정: FastAPI에서 `http://localhost:3000` 허용
- [ ] 환경변수: `NEXT_PUBLIC_WS_URL`, `NEXT_PUBLIC_API_URL`

## 에러 전파 규칙

1. Backend에서 에러 발생 → WebSocket `task_error` 전송 + Telegram 알림
2. Frontend에서 WS 끊김 → 자동 재연결 + "연결 중..." UI 표시
3. Chrome 인스턴스 다운 → Backend health check 감지 → `service_status` 전송
4. 모든 에러에 `recoverable: boolean` 플래그 포함 → UI에서 재시도 버튼 표시 여부 결정
