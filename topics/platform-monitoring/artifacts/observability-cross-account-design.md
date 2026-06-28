# 관측 cross-account 설계 (LGTM, 전용 관측계정)

> 상태: **초안(설계 골격)** · 2026-06-28 · platform-monitoring 토픽
> 근거 결정: [minutes/2026-06-28.md](../minutes/2026-06-28.md), [discussions](../discussions/2026-06-28-lgtm-topology.md)

## 0. 확정된 결정 (전제)

| # | 결정 |
|---|------|
| D1 | 옵션 ③(X-Ray) 탈락 — Grafana correlation 기조 위반 |
| D2 | 메트릭 백엔드 = **AMP**(managed). Mimir self-host 보류 |
| D3' | **dev부터 cross-account 직행** (단계화 중 "같은 계정 시작" 번복). 근거: 이관 비용 > 초기 구축 비용 + log account는 모니터링 외 용도도 있음 |
| D4 | exemplar(metric→trace 점프) 갭 수용 — Tempo↔Loki 상관으로 충분 |
| 목표 | **풀 LGTM 3신호**(AMP+Loki+Tempo+AMG). 구현 순서 = 메트릭 → 로그 → 트레이스 점진 |
| 계정 | **AWS Organizations 신설 + 관측/로그 전용 멤버계정** |

## 1. 인프라 현황 (2026-06-28 스캔, 설계 출발점)

- **계정**: 단일 `897722674699`. dev·stage 동거. **Organizations·Control Tower 없음.**
- **네트워크**: dev·stage 같은 VPC `192.168.0.0/16`. **TGW·Peering·PrivateLink 없음.**
- **ECS(dev)**: Fargate, 플랫폼별 다중 클러스터(`develop-active`, `develop-ubittz-cluster`, `develop-biz-guide-cluster`). 각 `svc_gateway`(8282)·`svc_server`(8080). terragrunt에 52개 platform.
- **Service discovery**: Cloud Map private DNS `platform.local` + Service Connect `platform-services`(Envoy).
- **IAM**: task/exec role = `modules/standard-iam-task-roles` 로 플랫폼별 생성. cross-account role 없음.
- **계측(spring-template)**: actuator 4.0.7만 존재. `micrometer-registry-prometheus` **없음** → `/actuator/prometheus` 비활성. endpoint 노출 설정·커스텀 logback·JSON 로깅·trace_id MDC·micrometer-tracing **모두 없음**. (Boot 4.0.7 / Java 25 / Spring Cloud 2025.1.2)

## 2. 목표 토폴로지

```
┌─ Org 관리계정 (897722674699 승격 or 신규) ──────────┐
│  Organizations · (향후) SCP · 중앙 빌링            │
└────────────────────────────────────────────────────┘
        │ 멤버
        ▼
┌─ 관측/로그 전용 멤버계정 (신규) ────────────────────┐
│  AMP workspace ........ 메트릭 저장 (managed)        │
│  AMG workspace ........ Grafana 보는 창 (managed)    │
│  svc_observability(ECS Fargate)                      │
│     ├ Loki  컨테이너 ── 로그 저장 ┐                  │
│     └ Tempo 컨테이너 ── 트레이스   ┘→ S3(중앙 버킷)  │
└────────────────────────────────────────────────────┘
        ▲ 메트릭: IAM/SigV4 (VPC 불필요)
        ▲ 로그·트레이스: cross-account VPC(peering/TGW)
        │
┌─ dev 계정 897722674699 (기존) ──────────────────────┐
│  svc_gateway / svc_server (앱, Fargate, 다중 클러스터)│
│  수집 에이전트(Alloy):                               │
│     메트릭 → AMP remote_write  ... VPC ❌            │
│     로그   → Loki push         ... VPC ✅            │
│     트레이스 → Tempo OTLP      ... VPC ✅            │
└────────────────────────────────────────────────────┘
```

## 3. 단계화 로드맵

### Phase 0 — 기반 (병행 가능)
- **0a 계정**: AWS Organizations 생성 → 기존 897722674699를 관리(또는 멤버)로 편입 → 관측/로그 전용 멤버계정 신설.
- **0b 계측 백지 메우기** (cross-account와 무관, 선행 가능):
  - `micrometer-registry-prometheus` 의존성 추가(gateway·server).
  - `management.endpoints.web.exposure.include` 에 `health,info,prometheus` 노출. `management.metrics.tags.application` 등 공통 태그.
  - *Spring Boot 4.x 좌표·키는 Context7로 최신 확인 후 확정.*

### Phase 1 — 메트릭 (cross-account, VPC 공사 없음)
- 관측계정에 **AMP workspace** 생성.
- dev 앱 task role(`standard-iam-task-roles` 모듈)에 **cross-account `aps:RemoteWrite`** 권한(관측계정 AMP 리소스 대상, AssumeRole or resource policy + SigV4).
- 수집: 앱 task에 **Alloy 사이드카** → `/actuator/prometheus` scrape → 관측계정 AMP remote_write.
- 관측계정에 **AMG** 생성, AMP를 데이터소스로. 메트릭 대시보드 가동.
- ✅ 이 단계까지 **VPC peering 불필요** — "처음부터 cross-account" 충족.

### Phase 2 — 로그 (VPC 연결 신설 지점)
- 관측계정에 **`svc_observability` ECS 서비스 + Loki 컨테이너 + S3**.
- **cross-account VPC 연결 신설**(아래 미결점 ①). Loki 엔드포인트에 dev에서 도달.
- 앱 logback → JSON 구조화 + trace_id MDC(Phase 3 대비) → Alloy가 로그도 push.
- AMG에 Loki 데이터소스 추가. (log account의 별도 로그 기능과 버킷 전략 정합 — 사용자 메모)

### Phase 3 — 트레이스
- `svc_observability`에 **Tempo 컨테이너** 추가(같은 task/서비스, S3).
- 앱에 `micrometer-tracing` + OTLP exporter, gateway↔server trace 전파. Service Connect(Envoy) 경유 전파 확인.
- AMG에 Tempo 데이터소스 + **Tempo↔Loki trace_id 상관** 설정(이게 Grafana correlation 기조의 핵심).

## 4. 미결 설계점

1. ~~cross-account 연결 방식~~ → **확정: VPC Peering** (2026-06-28). cross-account peering 정식 지원, 단일 쌍이라 TGW보다 가볍고 시간당 고정비 無. 멀티계정 더 확장 시 TGW 재검토.
2. ~~수집 에이전트 배치~~ → **확정: Alloy 중앙 수집 task, 클러스터당 1개**. Cloud Map/ECS discovery 로 클러스터 내 서비스 `/actuator/prometheus` scrape → cross-account AMP remote_write. 서비스 수와 무관하게 Alloy = 클러스터당 1개라 경제적. 구현 시 Alloy ECS/DNS discovery 문법 Context7 확인.
3. ~~VPC CIDR 충돌~~ → **해결**: 관측계정 VPC = `10.0.0.0/N` 사용 예정 → dev `192.168.0.0/16`과 비중첩.
4. **S3 중앙 버킷 전략** — Loki·Tempo 블록을 관측계정 버킷에. log account의 별도 로그 기능과 버킷·수명주기 정책 통합 설계.
5. **AMG ↔ self-host Loki/Tempo 인증 경로** — AMG(관측계정)에서 같은 계정 내 Loki/Tempo는 VPC 내부, dev 메트릭은 AMP(IAM). 데이터소스별 인증 분리.

### 대상 클러스터 (2026-06-28 확정)

| 순서 | 클러스터 | 타입 | 수집 | 비고 |
|------|----------|------|------|------|
| **1 (지금)** | `develop-active` | Fargate | 중앙 Alloy 1 | 첫 대상 |
| 2 | `develop-ubittz-cluster` | Fargate | 중앙 Alloy 1 | |
| 3 | `stage-[]-cluster` | **EC2/ASG** | 중앙 Alloy 1 | **호스트 계층 메트릭 부활** (EC2 CPU·mem·disk, ASG capacity). Alloy 호스트 데몬 옵션 가능 |
| 제외 | `develop-biz-guide-cluster` | — | — | 대상 아님 |

> **범위 변경 주의**: chat 초반 "개발(Fargate)만"에서 **stage(EC2/ASG) 재포함**으로 확장. Fargate(develop)는 호스트 없음 → ECS task+앱 메트릭만. stage(EC2)는 **호스트 계층 메트릭이 관측 대상에 추가**됨. 단 stage는 3순위 → 현 설계는 develop-active 중심, stage 호스트 계층은 해당 단계에서 별도 설계.

## 5. 실행 이슈 (2026-06-28 GitHub 등록)

| Phase | 작업 | 레포 | 이슈 |
|-------|------|------|------|
| 0b | 메트릭 계측(prometheus registry+actuator) | spring-template | [#35](https://github.com/ubittz/spring-template/issues/35) |
| 2 | 로그 계측(JSON+trace_id MDC) | spring-template | [#36](https://github.com/ubittz/spring-template/issues/36) |
| 3 | 트레이스 계측(micrometer-tracing+OTLP) | spring-template | [#37](https://github.com/ubittz/spring-template/issues/37) |
| 0a | Organizations+관측 멤버계정 | infra-module | [#17](https://github.com/ubittz/ubittz-infra-module/issues/17) |
| 1 | 메트릭 cross-account(AMP+AMG+IAM+Alloy) | infra-module | [#18](https://github.com/ubittz/ubittz-infra-module/issues/18) |
| 2 | 로그 백엔드(Loki+S3+Peering) | infra-module | [#19](https://github.com/ubittz/ubittz-infra-module/issues/19) |
| 3 | 트레이스 백엔드(Tempo) | infra-module | [#20](https://github.com/ubittz/ubittz-infra-module/issues/20) |

**착수 권장 순서**: ST#35(계측 P0, 무관·재사용) ∥ INF#17(계정 P0) → INF#18(메트릭, VPC無) → ST#36+INF#19(로그+peering) → ST#37+INF#20(트레이스).
