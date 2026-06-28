# 결정사항 인덱스 (Decisions Index)

회의실 전체에서 내려진 **결정**을 한 줄로 모아두는 인덱스. 상세는 각 주제의 `minutes/` 참고.

| 날짜 | 주제 | 결정 요약 | 상세 링크 |
|------|------|-----------|-----------|
| 2026-05-25 | docker-swarm-vs-k8s | ECS Fargate 채택 (정부지원금 가산점 앱) | [ADR-0001](../topics/docker-swarm-vs-k8s/artifacts/ADR-0001-ECS-Fargate-채택.md) |
| 2026-06-28 | platform-monitoring | dev 메트릭=AMP, 단계화(Phase1 같은계정 메트릭만) | [minutes](../topics/platform-monitoring/minutes/2026-06-28.md) |
| 2026-06-28 | platform-monitoring | 번복(D3 단계화): dev부터 cross-account 직행, 같은계정 시작 거부 | [minutes](../topics/platform-monitoring/minutes/2026-06-28.md) |

## 작성 규칙
- 결정이 **확정** 된 경우에만 등재 (잠정 합의는 등재하지 않음)
- 한 줄 요약은 **30자 이내** 권장
- 결정이 나중에 **번복** 되면 행을 지우지 말고 "번복: YYYY-MM-DD" 행을 추가하여 이력을 남김
