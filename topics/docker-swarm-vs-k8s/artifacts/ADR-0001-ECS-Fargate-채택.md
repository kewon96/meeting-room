# ADR-0001: 컨테이너 오케스트레이터로 ECS Fargate 채택

- **상태**: Accepted
- **날짜**: 2026-05-25
- **적용 범위**: 정부지원금 가산점 앱 (기업연구소 개발 앱) 백엔드 인프라
- **선행 환경**: EC2(t3.small) + docker-compose (2025년 중반까지) — 본 결정으로 대체

---

## 1. 컨텍스트

### 1.1 비즈니스 컨텍스트
- 회사 본업: **정부지원금 컨설팅**
- 앱 성격: 정부지원금 **가산점 항목** 으로 제출되는 기업연구소 개발 앱
- 앱 1세트 구성: **Spring Cloud Gateway + Spring Boot business server**
- 데이터: **AWS RDS (MySQL)** — 단일 리전·단일 인스턴스 + 세밀한 백업 정책
- 앱 수: 다수 (앱마다 고객·소유 분리)

### 1.2 핵심 제약 (Non-negotiable)
1. **회사 내부 계약**: 개발 완료 시점부터 **6개월간 서버 상시 가동** 의무. 비용·관리 부담을 근거로 협상 시도 → **거부됨**
2. **사용량**: 심사 외 기간 실사용량 극도로 낮음 (실서비스 목적 아님)
3. **앱 라이프사이클**: 짧음 (6개월 + 갱신). 폐기 가능성 높음
4. **운영 인력**: 작음 (백엔드 + 인프라 겸임)
5. **Fargate Spot**: 회사 정책상 **사용 불가**
6. **클라우드**: AWS 단일. 멀티클라우드 이전 가능성 **0**

### 1.3 이전 환경의 문제 (EC2 + docker-compose)
- **프로비저닝 신뢰성**: 폴더 누락·docker 설치 실패 등 간헐적 발생, 개별 검수 비용 과다
- **OS 패치 폭발**: 앱당 EC2 = 20+ 대 누적, 보안 대응 한계
- **확장 한계**: 앱 N 증가에 따라 운영 비용 선형 증가

---

## 2. 결정

**AWS ECS on Fargate** 를 채택하며, **앱(=고객) 단위로 ECS Cluster 를 분리** 한다. 인프라는 **Terraform IaC** 로 표준화한다.

---

## 3. 근거

### 3.1 왜 ECS 인가 (vs EKS / k8s self-hosted)

| 축 | 판단 |
|---|---|
| **Control plane 비용** | ECS = **무료**. EKS = $0.10/hr ≈ **$73/mo per 클러스터**. 앱당 클러스터 분리 정책과 결합 시 EKS 는 N × $876/yr 의 control plane 비용 단독 발생 → *실제 컴퓨팅 비용을 추월* 하는 비대칭 발생 |
| **AWS 네이티브 통합** | Task Role(IAM)·CloudWatch·ALB·EventBridge 와 즉시 호환. k8s 의 IRSA·Pod Identity 같은 별도 다리 불필요 |
| **운영 학습 곡선** | AWS 지식만으로 운영 가능. k8s 는 그 자체로 **1명의 풀타임 R&R** 을 요구 |
| **이식성 가치** | 정부지원금 앱은 AWS 영구 채류 → k8s 의 이식성 이점은 **가치 0** |
| **확장성 요구** | Operator/CRD 로 모델링할 도메인 없음. k8s 의 확장성 이점도 **가치 0** |

### 3.2 왜 Fargate 인가 (vs ECS on EC2)
- 호스트 OS 패치를 **AWS 에 위임** → 이전 환경의 가장 큰 운영 부담 직접 해소
- **Firecracker microVM** 단위 격리로 같은 클러스터 내에서도 강한 격리 자동 제공
- Task Definition + 이미지로 **환경 일관성** 확보 → 프로비저닝 신뢰성 문제 해소
- **0.25 vCPU + 0.5 GB** 최소 단위로 앱당 비용 예측 가능

### 3.3 추상화 레벨 정합성
도구의 추상화 레벨과 문제의 추상화 레벨이 일치해야 한다. 본 워크로드는:
- 단일 머신 자원으로 충분
- 워크로드 종류 단일 (스테이트리스 웹 + RDB)
- 멀티팀·멀티 클러스터·서비스 메시 요구 없음

→ **k8s 의 "데이터센터 OS" 추상화는 과잉**. ECS 의 "AWS 위 컨테이너 매니저" 추상화가 정합.

---

## 4. 고려한 대안

| 대안 | 채택 안 한 이유 |
|------|-----------------|
| **EKS** | Control plane $73/mo × 앱당 클러스터 → 비용 산수 불성립 |
| **k8s self-hosted** | 운영 부담이 데모성 앱 가치를 훨씬 초과 |
| **ECS on EC2** | 이전 환경의 OS 패치/프로비저닝 문제 회귀 |
| **Lambda + API Gateway** | (a) Spring Boot 콜드스타트 (b) Spring Cloud Gateway 패턴 부적합 (c) 6개월 상시 계약과 idle-0 모델의 가치 충돌 |
| **AWS App Runner** | 앱 단위 격리·관리 요구 + 6개월 상시 조건과 결이 안 맞음 |
| **Fargate Spot** | 회사 정책상 불가 |
| **EC2 t4g.nano + systemd** | OS 패치 부담으로 회귀 |

---

## 5. 운영 구성

### 5.1 토폴로지
- **VPC**: 1개 (공용)
- **NAT Gateway**: 1개 (공유) — 외부 호출(본인인증 메일·PG 사) 용도. 월 비용 7–10만원
- **ECS Cluster**: 앱당 1개
- **Service**: 클러스터 안 2개 (SCG + Spring Boot)
- **RDS**: MySQL 단일 리전·단일 인스턴스 + 백업 정책

### 5.2 IaC
- Terraform 으로 IAM·Tag·Naming Convention 표준화
- 앱 추가 시 모듈 재사용으로 N 확장 부담 최소화

### 5.3 비용 최적화
- **매일 00–6시 Service off** (Task `desired count = 0`) → 약 25% 절감
- **CloudWatch Logs retention 6개월** (계약 기간 align)
- **Container Insights off** (운영 모니터링 비용 폭탄 회피)

### 5.4 보안·운영
- 호스트 OS: AWS 위임 (Fargate)
- 컨테이너 이미지 CVE: 구동 중 앱 기준으로 런타임 패치 대응
- 격리: microVM (Firecracker) + IAM Task Role + 앱당 클러스터

### 5.5 라이프사이클·폐기
1. 6개월 도래 시점 → Service `desired count = 0`
2. 계약 갱신 안내 후 **1주일** 미응답 → 미갱신 간주, 자원 제거 단계 진입
3. 고객의 최종 폐기 요청 시 → `terraform destroy`

---

## 6. 결과 (Consequences)

### 6.1 긍정
- 호스트 OS 패치 부담 0
- microVM 단위 강한 격리 자동
- 앱당 비용 예측 가능 (월 단위 단순 산수)
- IaC 로 앱 N 확장 시 일관성·신뢰성 확보
- 폐기 시 자원 잔여 없는 깔끔한 정리 절차

### 6.2 부정 (감수)
- **AWS 락인** — 단, 이식성 가치가 0 이라 *비용 0* 으로 평가
- **Spot 미사용** 으로 잠재적 70% 절감 옵션 봉인 (회사 정책)
- **컨테이너 베이스 이미지 패치** 는 운영자 책임 영역 잔존

---

## 7. 재검토 트리거

다음 신호 중 다수가 누적되면 본 결정을 재검토:

- [ ] 앱 수가 ~50개 이상으로 폭증, IaC 자동화로도 운영 부담 가시화
- [ ] 워크로드 다양화 (배치·스트리밍·ML 등 추가)
- [ ] 실서비스 전환 (트래픽·SLA·다중 영역 요구)
- [ ] 멀티 클라우드 요구 발생
- [ ] Operator·CRD 로 모델링하고 싶은 도메인 등장
- [ ] 컨테이너 이미지 베이스 자동 갱신·CVE 대응이 운영 비용의 주된 항목으로 부상
- [ ] 회사 내부 계약 (6개월 상시) 조건 변동

---

## 8. 참고

### 검토 회의록
- 본 ADR 은 `topics/docker-swarm-vs-k8s/chat.md` (2026-05-25) 의 다단 검토에서 도출됨
- 주제 전문가: `topics/docker-swarm-vs-k8s/.claude/agents/container-orchestration-sme.md`

### 외부 자료 (배경)
- The New Stack — *How Kubernetes Became the New Linux*
- CNCF — *Why Kubernetes Has Emerged as the 'OS' of the Cloud*
- Joe Beda — *"Kubernetes is a platform for building platforms"* (DevOps Days Seattle 2018)
- Google — *Borg, Omega, and Kubernetes*
