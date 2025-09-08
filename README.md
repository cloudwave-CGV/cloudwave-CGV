
# CGV 통합 예매 플랫폼 (MSA)

## 1. 개요
‘CGV에 시집갈래말래’ 팀은 대규모 트래픽 상황과 재해 상황에서도 비즈니스 연속성을 확보하는 것을 목표로 MSA 기반 통합 예매 플랫폼을 설계·구축했다. 핵심 영역은 고가용성(HA), 재해 복구(DR), 모니터링, 보안, 자동화된 CI/CD 파이프라인을 구

---

## 팀원
| 이름 | 역할 |
|---|---|
| 박민서(팀장) | MSA 구축/개발, 모니터링 |
| 강채운 | CI/CD 파이프라인, MSA 구축/개발 |
| 이태호 | MSA 설계/구축, EKS, 보안 |
| 조유성 | DR(Terraform), 아키텍처 설계 |
| 진보현 | EKS, 모니터링 |
| 최정현 | API 연동, 모니터링 |

---

## 2. MSA 아키텍처
- **Booking 서비스**: Spring Boot. 예매/대기열/목록 조회. 데이터베이스는 **DynamoDB**.
- **Ticket 서비스**: Go. 티켓 발급/조회. 데이터베이스는 **RDS**.
- **Live 서비스**: Django. 라이브 채널 생성. **DynamoDB + S3** 사용.
- **대기열/비동기 처리**: 서비스 간 통신은 비동기 위주로 구성, Booking 대기열에 **Amazon MSK** 활용.
- **컨테이너 오케스트레이션**: 모든 서비스는 도커 이미지로 **ECR**에 저장하고 **Amazon EKS**에서 운영.

### 요청 흐름
`Cognito 인증 → API Gateway → NLB → (EKS) Ingress Controller → 각 마이크로서비스`

데이터 계층은 **Aurora/RDS** 등 매니지드 서비스로 분리해 성능과 운영 효율성을 확보했다.

---

## 3. 보안
- **IAM**: 최소권한 원칙. Developers/DevOps/Operators 등 역할 그룹 운영, **MFA** 적용.
- **WAF/Shield**: CloudFront 전단에 **AWS WAF**(OWASP Top 10 룰)과 **Shield Standard** 적용.
- **암호화/비밀관리**: **ACM(HTTPS)**, **KMS**, **Secrets Manager**로 자격증명 및 민감정보 관리.
- **이벤트 알림**: 방화벽 활성화 시 `CloudFront → WAF → S3 → EventBridge → Lambda → Slack` 경로로 알림.

---

## 4. 확장성 및 모니터링
- **확장성**
  - **HPA**: 목표 CPU 70% 기준 자동 파드 증감.
  - **Karpenter**: 노드 자동 프로비저닝. 스팟 적극 활용(비용 최적화).
- **모니터링**
  - **도구**: CloudWatch, Datadog, Prometheus/Grafana.
  - **로그 파이프라인**: EKS 로그를 CloudWatch 수집 → Kinesis Firehose → S3 적재, S3 Lifecycle로 장기 보관 비용 최적화.
  - **Datadog**: Agent 배포, Host Map/Dashboard로 이상 징후 조기 탐지.

---

## 5. CI/CD
- **구성요소**: Tekton(쿠버네티스 네이티브 CI/CD), Kaniko(데몬리스 이미지 빌드), SonarQube(정적분석), Trivy(이미지 스캔), Argo CD(GitOps 배포).
- **CI 흐름**
  1. GitHub Push(Webhook) → Tekton Pipeline 실행
  2. 빌드 → 테스트 → **SonarQube Quality Gate** 통과 검증
  3. **Kaniko**로 이미지 빌드 → **Trivy** 스캔 → **ECR Push**
  4. 서울 리전 ECR 이미지를 도쿄 리전 ECR로 레플리케이션
  5. k8s manifest 저장소 갱신
- **CD 흐름**
  - **Argo CD**가 manifest 변경을 감지해 개발/운영/DR 각 **EKS** 환경에 동기화 배포
- **알림**
  - 파이프라인 성공/실패 결과는 Slack으로 통보

---

## 6. DR(재해 복구)

### 6.1 전략 및 인프라

- **전략**: Warm Standby. 평상시 도쿄 리전에 최소 자원만 상시 준비.
- **데이터 계층**
    - 핵심 DB는 **Aurora Global Database** 사용
        - 평시: **Seoul = Primary(Writer)**, **Tokyo = Secondary(Reader)**
        - 글로벌 복제 특성상 **RPO < 10초** 목표로 운영
    - 표준 **RDS**를 사용하는 경우엔 Cross-Region Read Replica를 사용(필요 서비스에 한함).
- **컴퓨트/네트워크**
    - **Terraform**으로 도쿄 리전에 DR 기본 인프라 사전 구축
        - 예: **EKS** `dr-eks-cluster` / 노드그룹 `dr-nodegroup`
    - 글로벌 트래픽 전환은 **AWS Global Accelerator(GA)** 사용
        - GA 제어는 **us-west-2** 고정(운영 노트)
- **리소스 식별(예시)**
    - Aurora: `reserve-db-prod`(Seoul) / `dr-aurora-cluster`(Tokyo)

### 6.2 감지 및 오케스트레이션

- **알람 → 이벤트 → 상태머신**
    - **CloudWatch Composite Alarm**: `dr-nlb-composite-trigger`
        - 서비스 레벨/DB 레벨 상태를 통합 판단(아래 헬스체크 람다 포함)
    - **EventBridge Rule**: `dr-failover-nlb-composite`
        - 알람 수신 시 상태머신 실행
    - **Step Functions**: `dr-failover-failback`
        - 현재 운용 로직은 **Failover 전용**으로 사용
- **Lambda 구성(핵심)**
    - **헬스체크**: `aurora-healthcheck-seoul`, `rds-healthcheck-tokyo`
    - **조치 작업**: `aurora-promote-tokyo`, `ga-switch-endpoint`, `eks-scale-dr`
    - **알림(선택)**: `sns-notify-dr`

### 6.3 Failover 절차(자동)

1. **트리거 분기**(Step Functions)
    - `svc-failure`: 애플리케이션 레벨 장애 시 **GA 트래픽만 Tokyo로 전환**
    - `db-failure` 또는 `region-failure`/`db+app-failure`: DB 및 인프라 레벨 장애 대응 경로로 진행
2. **DB 승격**
    - Aurora Global DB의 Tokyo 클러스터를 **Writer로 승격**
    - 승격 직후 **90~120초 대기**(Writer 전환 안정화 운영 기준)
3. **트래픽 전환**
    - **GA 다이얼 전환**으로 사용자 트래픽을 Tokyo로 이동
4. **용량 확장**
    - **EKS 노드그룹 스케일업**(Karpenter와 연동)으로 유입 트래픽 수용

> 목표 RTO ≤ 5분. 실제 RTO는 장애 유형/복제 지연/확장 속도에 따라 달라질 수 있으므로 주기적인 리허설을 전제
> 

### 6.4 Failback 운영 기준

- 현재 상태머신은 **Failover 전용**으로 운용. Failback은 **수동 절차**로 진행
- **표준 RDS**는 Read Replica를 승격하면 **되돌릴 수 없음**. 도쿄가 Writer가 된 이후 서울 복구 시 **Replica를 재구성**

### 6.5 테스트/운영 노트

- GA 제어 리전은 **us-west-2 고정**이라 IAM/권한 및 자동화 실행 리전을 일관성 있게 관리
- Aurora 승격 후 애플리케이션의 연결 문자열/시크릿이 Writer 엔드포인트를 참조하는지 확인
- 리허설 시엔 트리거 종류별(`svc-failure`, `db-failure`, `region-failure`) 분기 성공 여부와 대기 구간(90~120초)에서의 지표 변화를 확인

---
