# 안건 (Agenda) — LGTM 관측 토폴로지 결정

> 2026-06-28 다학제 소집. 1:1(chat.md)에서 선택지가 3개로 좁혀진 뒤 토론 모드 전환.

## 배경 한 줄
LGTM 스택으로 플랫폼 모니터링 직접 구성. 보는 창은 AMG, 기조는 "모든 신호가 Grafana에서 상관관계로 엮여야 한다". 선택지 ③(AMP+X-Ray+Loki)는 X-Ray↔CloudWatch 종속으로 Grafana correlation 기조 위반 → 이미 탈락. 남은 ①풀 self-host vs ②AMP 하이브리드.

## 완료 (2026-06-28 결정)
- [x] **안건 1**: AMP가 exemplar를 지원하는가? → **미지원**(AWS 공식 문서). metric→trace 점프만 영향 — aws-observability-expert/주치의
- [x] **안건 2**: ① vs ② TCO + cross-account 운영부담 → **dev 한정 ②(AMP)** — analyst
- [x] **안건 3**: 전용계정+self-host = 과설계? → **dev 일괄 도입은 과설계, Phase 1=AMP 메트릭만** — critic
- [x] **안건 4**(종합): → **D1~D4 + Phase 1/2/3 단계화** — 주치의 종합

## 결정 등재 완료
- [minutes/2026-06-28.md](minutes/2026-06-28.md) + shared/decisions.md 등재 완료
