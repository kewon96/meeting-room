# 플랫폼 모니터링 설계

> 한 줄 요약: PaaS 로 찍어내는 Spring 템플릿 기반 플랫폼들에 **시스템 모니터링 / 애플리케이션 모니터링** 두 계층의 관측 항목을 설계한다.

## 배경
- 플랫폼 기본 템플릿: `~/workspace/uinus/backend/ubittz/spring-template/`
  - Spring Boot 4.0.7 / Java 25, **gateway(Spring Cloud Gateway WebFlux) + server(Web MVC)** 2개 서비스 구조
  - 이미 `spring-boot-starter-actuator` 포함 (gateway, server 둘 다)
  - 데이터: MySQL(JPA/QueryDSL/Hikari), Redis(MemoryDB/Lettuce), S3, SQS, Spring AI, Cognito, Cloud Map
- 인프라: `~/workspace/uinus/infra/ubittz-infra-module/terragrunt/`
  - **개발 = ECS Fargate**: gateway/server 각각 별도 ECS 서비스(`svc_gateway`, `svc_server`), autoscaling on
  - **스테이징 = ECS on EC2/ASG**: 단일 task(`td_platform`, EC2 launch type) + 공유 stage 클러스터의 capacity provider(ASG)
  - 모니터링 대상 범위: **Fargate 와 ASG 만** (사용자 명시)

---

## 범위
- 무엇을 관측할지(metric/log/trace) 항목 정의 → 수집 방식 → 알람 임계치 순으로.
- 데이터 계층(RDS/Redis) 자체 모니터링은 범위 밖(별도 상태로 관리). 단 **앱에서 본 DB/Redis 체감 지표**(커넥션풀, 호출 지연)는 애플리케이션 모니터링에 포함.

### 대상 클러스터 (2026-06-28 확정)
- **1순위 `develop-active`(Fargate)** → `develop-ubittz-cluster`(Fargate) → `stage-[]-cluster`(EC2/ASG). `develop-biz-guide-cluster` 제외.
- stage(EC2)는 **호스트 계층 메트릭** 포함(해당 단계에서). develop(Fargate)은 호스트 없음.

## 방향 (2026-06-28 결정)
- **풀 LGTM**(AMP+Loki+Tempo+AMG), 구현 순서 메트릭→로그→트레이스.
- **cross-account 직행** + **AWS Organizations 신설 + 관측/로그 전용 멤버계정**. 연결=VPC Peering. 수집=클러스터당 Alloy 중앙 수집 task.
- 상세: [artifacts/observability-cross-account-design.md](artifacts/observability-cross-account-design.md) · [minutes/2026-06-28.md](minutes/2026-06-28.md)

## 참여 페르소나
- 사용자
- aws-observability-expert(생성됨), critic, analyst
