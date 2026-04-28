---
name: plan-start
description: 프로젝트 기획 단계 시작. docs/goal.md를 읽고 ui-planner 에이전트를 호출하여 requirements.md와 spec.md를 작성한다. 풀 사이클 워크플로우의 첫 단계.
---

지금부터 이 프로젝트의 기획 단계를 시작합니다.

## 절차

1. `docs/goal.md`가 존재하는지 확인합니다.
   - 없으면 사용자에게 "이 프로젝트는 간단 모드입니다. 풀 사이클로 진행하려면 docs/goal.md를 먼저 작성하거나 scaffold.sh를 풀 사이클 모드로 다시 실행해주세요"라고 안내합니다.

2. `docs/goal.md`가 있으면 ui-planner 에이전트에게 위임:
   - "ui-planner를 호출하여 docs/goal.md 기반으로 requirements.md와 spec.md를 작성해주세요"

3. ui-planner가 사용자에게 추가 질문할 수 있습니다. 그 답변을 받아서 다시 ui-planner에게 전달.

4. spec.md 작성 완료 후 사용자에게 다음 단계 안내:
   - UI가 있는 프로젝트: "/ui-design 명령으로 UI 설계 단계로 진행하시겠습니까?"
   - UI 없는 프로젝트(CLI/API): "/test-cases 명령으로 테스트 케이스 작성 단계로 진행하시겠습니까?"
