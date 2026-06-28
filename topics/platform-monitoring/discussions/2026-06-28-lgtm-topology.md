# 다학제 토론 — LGTM 관측 토폴로지 (2026-06-28)

## chat.md 에서 옮겨온 맥락 (요약 인용)

- **환경**: uinus PaaS. Spring Boot 4.0.7/Java 25, gateway(SCG WebFlux)+server(Web MVC) 2서비스, actuator 포함. 개발=ECS Fargate(서비스별 독립, autoscaling, 호스트 없음). Cloud Map 구성됨. 데이터: MySQL(Hikari)·Redis(Lettuce)·S3·SQS.
- **방향**: CloudWatch 직접 구성 → **LGTM 스택 직접 구성**으로 전환. 보는 창 = Amazon Managed Grafana(AMG). Grafana Cloud는 선택지에서 제외.
- **기조**: "모든 신호가 Grafana에서 보이고 **상관관계(correlation)**로 엮여야 한다."
- **standing goal**: 로그/관측 **전용 AWS account** 분리 희망(Control Tower Log Archive 계정 패턴과 정합).
- **신호 정리**: AMP/Mimir=metrics(시스템+앱 모두), Loki=logs, Tempo=traces. AMP는 "시스템 전용"이 아님(앱 메트릭도 저장).
- **cross-account 멘탈모델**: VPC 배선 부담은 self-host 서비스에 네트워크로 도달해야 해서 생김. 네이티브(AMP/X-Ray)는 IAM/SigV4라 VPC 면제.

## 선택지 (1:1에서 도출)

| 옵션 | Metrics | Logs | Traces | cross-account VPC | 상태 |
|------|---------|------|--------|-------------------|------|
| ① 풀 self-host | Mimir | Loki | Tempo | 3신호 | 후보 |
| ② AMP 하이브리드 | **AMP** | Loki | Tempo | 2신호(로그·트레이스) | **잠정 우세** |
| ③ AMP+X-Ray+Loki | AMP | Loki | **X-Ray** | 1신호 | **탈락** |

**③ 탈락 사유**: X-Ray의 native 상관 파트너가 CloudWatch Logs → Tempo↔Loki native 상관(trace_id)이 깨짐 → "Grafana correlation" 기조 위반.

**①vs② 핵심 분기점**: AMP의 **exemplar**(메트릭→트레이스 점프) 지원 여부. Mimir는 지원 확실, AMP는 미확인.

---

## 발언 (페르소나별)

### 안건 1 — AMP exemplar 지원 여부 (담당: aws-observability-expert → 주치의 직접 1차자료 검증)

> 주: 주제 전용 `aws-observability-expert` 가 세션 Agent 레지스트리에 없어 호출 불가 → 주치의가 AWS 공식 docs 로 직접 검증.

**결론: AMP 는 exemplar 를 지원하지 않는다 (공식 문서 명시).**

- AWS 공식 문서 *"Configure exemplars — Amazon Managed Grafana"* 가 못 박음: **"Exemplars are not supported in Amazon Managed Service for Prometheus."** (Prometheus 2.26+ 필요 기능이나 AMP 는 미구현)
- AMG 의 exemplar 설정 페이지는 **일반 Prometheus 데이터소스용**이고, AMP 는 명시적으로 제외 대상.
- ingestion 단의 `exemplarCount=0` (troubleshooting 로그)은 remote_write 가 exemplar 필드를 **인지하되 저장/질의로 이어지지 않음**을 시사 (carry 되지 않음).
- **우회책**: AMG 데이터소스 설정에서 exemplar **링크(data link) URL 매크로**(`https://.../${__value.raw}`)를 수동 구성 가능. 단 이는 native exemplar(시계열에 박힌 trace_id 자동 점프)가 **아니라** 수동 링크 → 2급 상관.
- **온전한 상관**: Tempo↔Loki 의 trace_id 기반 trace↔log 점프는 exemplar 와 무관하게 **①②둘 다 그대로 작동**. 깨지는 건 metric→trace 점프 한 곳뿐.
- **①vs② 함의**: chat 에서 "①vs② 실질 분기점 = AMP exemplar 여부" 로 좁혔는데, 그 분기점의 답은 **"AMP 미지원"**. 즉 metric→trace 점프를 native 로 원하면 그 한 가지가 Mimir(①) 의 유일하게 단단한 명분.

### 안건 2 — ①vs② TCO + cross-account 운영부담 (담당: analyst)

**결론: dev 한정 ② AMP 하이브리드로 기운다 (강도 중~강). 근거는 요금이 아니라 cross-account 배선 축소 + 가용성 분리.**

- **cross-account 가 최대 변별점**: ② 는 metrics 가 IAM/SigV4 → VPC 면제 → cross-account 의존 신호가 **3→2** (logs·traces 만). metrics 가 네트워크 경로 장애와 무관해짐.
- **요금**: dev 저카디널리티에선 self-host Mimir 의 **상시 Fargate compute 고정비**(단일 task ~월 $40~45, HA 2개 ~$80~90, *단가 확인 필요*)가 AMP 사용량 과금(거의 0 수렴)보다 불리할 공산. 단 카디널리티 폭증 시 프로덕션에선 역전 가능.
- **운영 인건**: metrics 1신호 managed 위임 → self-host 운영 3신호→2신호 로 축소. 소조직에 실익.
- **가용성**: metrics 를 AWS managed 면으로 빼 "감시자가 대상과 함께 죽는" 리스크에서 분리.
- **lock-in**: 앱 계측(Micrometer→remote_write/PromQL 표준)이 백엔드 비종속 → AMP→Mimir 회귀 전환비 낮음. dev lock-in 과대평가 불필요.
- **프로덕션 재검토 트리거**: active series/ingestion 이 커져 AMP 월요금이 Mimir compute+S3 추정을 추월하거나, 멀티클라우드 이식·세밀 튜닝 요구 발생 시 ①(Mimir) 재평가.

### 안건 3 — 전용계정 + self-host LGTM = dev 과설계인가 (담당: critic)

**판정: dev 시작 시점에 한 번에 까는 것은 과설계. "방향"이 아니라 "순서"가 틀렸다.**

- **YAGNI**: dev 1개에 LGTM 3종 self-host + cross-account 배선의 운영부담이 정당화 안 됨. 사용자 본인이 chat 에서 두 번 말한 **"Mimir만 먼저"** 가 옳았고 토론이 그 단계화를 건너뛰며 스코프를 키웠다.
- **계정 분리 타이밍**: 격리 가치는 보호 대상 가치에 비례 → dev 는 죽어도 되는 환경. **검증 대상이 dev 하나뿐일 때** cross-account 를 까는 게 더 비쌈.
- **숨은 가정**: "dev 에도 3신호 풀 상관관계 필요"는 프로덕션 SRE 요구의 조기 투영. dev 1차 목적(떴나/OOM/느린가)은 metrics+평문로그면 충분.
- **자기모순 실패모드**: 전용 task 1개에 M·L·T 3컨테이너 → 자원경합으로 Mimir 까지 OOMKill = 정작 OOM 을 봐야 할 도구가 OOM 으로 죽음. 별도계정(B)로 피하려던 실패가 task 내부에서 재현.
- **놓친 이해관계자**: self-host LGTM+전용계정을 **유지보수할 운영 주체**가 토론 어디에도 정의 안 됨. 1인이면 이 야심은 그 사람의 야간 호출기.
- **단계화(가역성 기준)**:
  - **Phase 1 (지금, dev)**: 메트릭만. Micrometer→Alloy→**AMP**→AMG. **같은 계정** 안, cross-account 0. self-host 안 함.
  - **Phase 2 (로그가 실측으로 아쉬울 때)**: 같은 계정에 `svc_observability` + **Loki 1개** 추가(Cloud Map DNS). Tempo 는 trace 점프를 실제로 쓰는 게 관찰될 때까지 보류.
  - **Phase 3 (스테이징/프로덕션 승격 트리거)**: 그때 전용 관측 **계정 격리 + cross-account 배선**. dev 에서 토폴로지 검증 끝났으니 "신축"이 아니라 "이사".
