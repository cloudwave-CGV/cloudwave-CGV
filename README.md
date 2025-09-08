# CGV 통합 예매 플랫폼 (MSA)

## 0. 전체 아키텍처
<img width="2927" height="1806" alt="전체 아키텍처" src="https://github.com/user-attachments/assets/c72c78dc-0128-4747-914e-530441aeca98" />

## 1. 개요
‘CGV에 시집갈래말래’ 팀은 대규모 트래픽 상황과 재해 상황에서도 비즈니스 연속성을 확보하는 것을 목표로 MSA 기반 통합 예매 플랫폼을 설계·구축  
핵심 영역은 고가용성(HA), 재해 복구(DR), 모니터링, 보안, 자동화된 CI/CD 파이프라인 구축  

### 팀원

| 박민서(팀장) | **MSA 구축/개발, 모니터링 구축** |
| --- | --- |
| 강채운 | **CI/CD 파이프라인 구축, MSA 구축/개발** |
| 이태호 | **MSA 설계/구축, EKS 담당, 보안** |
| 조유성 | **DR(Terraform), 아키텍처 설계** |
| 진보현 | **EKS 담당, 모니터링 구축** |
| 최정현 | **API 연동, 모니터링 구축** |

---

## 2. MSA (마이크로서비스 아키텍처)

안정성과 개발 유연성 극대화를 위해 전체 시스템을 기능별로 분리된 독립적인 마이크로서비스로 설계  

### 서비스 구성
- **Booking 서비스**: Java Spring Boot 기반, 영화 예매, 대기열 처리, 공연 목록 조회 담당, 데이터베이스는 DynamoDB 사용  
- **Ticket 서비스**: Go 기반, 티켓 발급 및 조회 담당, 데이터베이스는 RDS 사용  
- **Live 서비스**: Python Django 기반, 라이브 채널 생성 담당, DynamoDB와 S3 사용  

각 서비스는 독립된 개발 스택과 데이터베이스를 사용해 장애 전파를 방지하고 기술 스택 유연성 확보  
Booking 서비스는 Amazon MSK 기반 대기열 시스템으로 대규모 트래픽 처리  
서비스 간 통신은 비동기 처리로 결합도 최소화 및 안정성 확보  

### EKS 기반 아키텍처
모든 마이크로서비스는 Amazon EKS 환경에서 컨테이너 기반으로 운영  
도커 이미지 추출 후 ECR 업로드, 이후 EKS 오케스트레이션  

요청 흐름: Cognito 인증 → API Gateway → NLB → EKS Ingress Controller → 해당 서비스  

데이터베이스는 Aurora, RDS 등 매니지드 서비스로 분리해 성능과 운영 효율성 극대화  

---

## 3. 보안
- **IAM**: 최소 권한 원칙, 역할 기반 그룹 운영, MFA 적용  
- **WAF/Shield**: CloudFront 전단 WAF 배치, OWASP Top 10 공격 차단, Shield Standard로 DDoS 방어  
- **알람 경로**: 방화벽 활성화 시 CloudFront–WAF–S3–EventBridge–Lambda–Slack 경로 알림 전송  
- **암호화/비밀 관리**: ACM 기반 HTTPS 암호화, KMS와 Secrets Manager로 자격증명 및 민감정보 관리  

---

## 4. 확장성 및 모니터링

### 확장성
- **HPA**: CPU 70% 기준 파드 자동 증감  
- **Karpenter**: 노드 자동 프로비저닝, Cluster Autoscaler 대비 빠른 대응, Spot 인스턴스 적극 활용으로 약 61% 비용 절감  

### 모니터링
- **도구**: CloudWatch, Datadog, Prometheus & Grafana  
- **주요 지표**
  - 인프라: EKS 클러스터 및 노드 CPU/메모리, 네트워크 I/O  
  - 컨테이너: 파드 상태, 파드별 리소스 사용량, 재시작 횟수  
  - 로그 관리: EKS 로그 → CloudWatch 수집 → Kinesis Firehose → S3 적재, S3 Lifecycle 정책 기반 장기 보관 최적화  
- **Datadog**: Agent 배포, Host Map/Dashboard 구성, 트래픽 이상 탐지  
- **CloudWatch**: 로그 집계 및 장기 보관, Kinesis Firehose–S3 파이프라인 구축, Lifecycle 기반 Intelligent-Tiering 및 Glacier Deep Archive 활용  

---

## 5. CI/CD
- **Tekton**: 쿠버네티스 네이티브 CI/CD 도구, YAML 기반 관리, Task/Pipeline 구조  
- **Kaniko**: 데몬리스 컨테이너 이미지 빌드  
- **SonarQube**: 정적 코드 분석  
- **Trivy**: 컨테이너 이미지 보안 스캔  
- **Argo CD**: GitOps 기반 manifest–클러스터 동기화  

### CI 파이프라인
1. GitHub Push → Tekton 파이프라인 자동 실행  
2. 빌드 → 테스트 → SonarQube Quality Gate 검증  
3. Kaniko 이미지 빌드 → Trivy 스캔 → ECR Push → 도쿄 리전 ECR 레플리카  
4. k8s manifest 저장소 업데이트  

### CD 파이프라인
- Argo CD가 manifest 저장소 변경 감지  
- 개발/운영/DR EKS 환경에 자동 동기화 배포  

### 알림
- 모든 파이프라인 성공/실패 결과는 Slack 통보  


## 6. DR(재해 복구)

### 6.1 전략 및 인프라
- 전략: Warm Standby, 보조 리전에 최소 자원 상시 준비  
- 데이터 계층: Aurora Global Database 기반, RPO < 10초 목표  
  - 일부 서비스는 RDS Cross-Region Read Replica 사용  
- 컴퓨트/네트워크: Terraform으로 보조 리전 DR 인프라 사전 구축  
- 트래픽 전환: AWS Global Accelerator 사용  

### 6.2 감지 및 오케스트레이션
- 알람: CloudWatch Composite Alarm으로 서비스 및 DB 상태 감시  
- 이벤트: EventBridge가 알람을 수신해 Step Functions 실행  
- 오케스트레이션: Step Functions가 트리거 유형에 따라 분기 처리  
  - 서비스 장애: GA 트래픽만 보조 리전으로 전환  
  - DB 또는 리전 장애: DB 승격과 전체 트래픽 전환 수행  
- 작업: Lambda로 DB 승격, GA 전환, EKS 확장 실행  

### 6.3 Failover 절차
1. 트리거 유형 판정 (Step Functions Choice 상태 사용)  
2. 보조 리전 Aurora 승격 후 90~120초 안정화 대기  
3. GA 다이얼 전환으로 트래픽 이동  
4. EKS 노드 및 파드 용량 확장  
- 목표 RTO ≤ 5분, 장애 유형과 복제 지연에 따라 변동 가능  

### 6.4 Failback
- 현재 상태머신은 Failover 전용, Failback은 수동 절차  
- RDS는 Replica 승격 시 원복 불가, 복구 시 Replica 재구성 필요  
