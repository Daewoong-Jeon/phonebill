# 물리 아키텍처 설계서 - 개발환경

## 1. 개요

### 1.1 설계 목적
- 통신요금 관리 서비스의 **개발환경** 물리 아키텍처 설계
- MVP 단계의 빠른 개발과 검증을 위한 최소 구성
- 비용 효율성과 개발 편의성 우선

### 1.2 설계 원칙
- **MVP 우선**: 빠른 개발과 검증을 위한 최소 구성
- **비용 최적화**: Spot Instances, Pod 기반 백킹서비스 활용
- **개발 편의성**: 복잡한 설정 최소화, 빠른 배포
- **단순성**: 운영 복잡도 최소화

### 1.3 참조 아키텍처
- 마스터 아키텍처: design/backend/physical/physical-architecture.md
- HighLevel아키텍처정의서: design/high-level-architecture.md
- 논리아키텍처: design/backend/logical/logical-architecture.md
- 유저스토리: design/userstory.md

## 2. 개발환경 아키텍처 개요

### 2.1 환경 특성
- **목적**: 빠른 개발과 검증
- **사용자**: 개발팀 (5명)
- **가용성**: 95% (월 36시간 다운타임 허용)
- **확장성**: 제한적 (고정 리소스)
- **보안**: 기본 보안 (복잡한 보안 설정 최소화)

### 2.2 전체 아키텍처

📄 **[개발환경 물리 아키텍처 다이어그램](./physical-architecture-dev.mmd)**

**주요 구성 요소:**
- NGINX Ingress Controller → AKS 기본 클러스터
- 애플리케이션 Pod: Auth, Bill-Inquiry, Product-Change, KOS-Mock Service
- 백킹서비스 Pod: PostgreSQL (Local Storage), Redis (Memory Only)

## 3. 컴퓨팅 아키텍처

### 3.1 Azure Kubernetes Service (AKS) 구성

#### 3.1.1 클러스터 설정

| 설정 항목 | 값 | 설명 |
|-----------|----|---------| 
| Kubernetes 버전 | 1.29 | 안정화된 최신 버전 |
| 서비스 계층 | Basic | 비용 최적화 |
| Network Plugin | Azure CNI | Azure 네이티브 네트워킹 |
| Network Policy | Kubernetes Network Policies | 기본 Pod 통신 제어 |
| Ingress Controller | NGINX Ingress Controller | 오픈소스 Ingress |
| DNS | CoreDNS | 클러스터 DNS |

#### 3.1.2 노드 풀 구성

| 설정 항목 | 값 | 설명 |
|-----------|----|---------| 
| VM 크기 | Standard_B2s | 2 vCPU, 4GB RAM |
| 노드 수 | 2 | 고정 노드 수 |
| 자동 스케일링 | Disabled | 비용 절약을 위한 고정 크기 |
| 최대 Pod 수 | 30 | 노드당 최대 Pod |
| 가용 영역 | Zone-1 | 단일 영역 (비용 절약) |
| 가격 정책 | Spot Instance | 70% 비용 절약 |

### 3.2 서비스별 리소스 할당

#### 3.2.1 애플리케이션 서비스
| 서비스 | CPU Requests | Memory Requests | CPU Limits | Memory Limits | Replicas |
|--------|--------------|-----------------|------------|---------------|----------|
| Auth Service | 50m | 128Mi | 200m | 256Mi | 1 |
| Bill-Inquiry Service | 100m | 256Mi | 500m | 512Mi | 1 |
| Product-Change Service | 100m | 256Mi | 500m | 512Mi | 1 |
| KOS-Mock Service | 50m | 128Mi | 200m | 256Mi | 1 |

#### 3.2.2 백킹 서비스
| 서비스 | CPU Requests | Memory Requests | CPU Limits | Memory Limits | Storage |
|--------|--------------|-----------------|------------|---------------|---------|
| PostgreSQL | 500m | 1Gi | 1000m | 2Gi | 20GB (Azure Disk Standard) |
| Redis | 100m | 256Mi | 500m | 1Gi | Memory Only |

#### 3.2.3 스토리지 클래스 구성
| 스토리지 클래스 | 제공자 | 성능 | 용도 | 백업 정책 |
|----------------|--------|------|------|-----------|
| managed-standard | Azure Disk | Standard HDD | 개발용 데이터 저장 | 수동 백업 |
| managed-premium | Azure Disk | Premium SSD | 미사용 (비용 절약) | - |

## 4. 네트워크 아키텍처

### 4.1 네트워크 구성

#### 4.1.1 네트워크 토폴로지

📄 **[개발환경 네트워크 다이어그램](./network-dev.mmd)**

| 네트워크 구성요소 | 주소 대역 | 용도 | 특별 설정 |
|-----------------|----------|------|-----------|
| Virtual Network | phonebill-vnet-dev | 전체 네트워크 | Azure CNI 사용 |
| Public Subnet | 10.0.1.0/24 | Load Balancer, Ingress | 인터넷 연결 |
| Application Subnet | 10.0.2.0/24 | 애플리케이션 Pod | Private 통신 |
| Data Subnet | 10.0.3.0/24 | 데이터베이스, 캐시 | 제한적 접근 |
| Management Subnet | 10.0.4.0/24 | 모니터링, 관리 | 개발용 도구 |

#### 4.1.2 네트워크 보안

**기본 Network Policy:**
| 정책 유형 | 설정 | 설명 |
|-----------|------|---------|
| Default Policy | ALLOW_ALL_NAMESPACES | 개발 편의성을 위한 허용적 정책 |
| Complexity Level | Basic | 단순한 보안 구성 |

**Database 접근 제한:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 허용 대상 | Application Tier Pods | tier: application 레이블 |
| 프로토콜 | TCP | 데이터베이스 연결 |
| 포트 | 5432, 6379 | PostgreSQL, Redis 포트 |

### 4.2 서비스 디스커버리

| 서비스 | 내부 주소 | 포트 | 용도 |
|--------|-----------|------|------|
| Auth Service | auth-service.phonebill-dev.svc.cluster.local | 8080 | 사용자 인증 API |
| Bill-Inquiry Service | bill-inquiry-service.phonebill-dev.svc.cluster.local | 8080 | 요금 조회 API |
| Product-Change Service | product-change-service.phonebill-dev.svc.cluster.local | 8080 | 상품 변경 API |
| PostgreSQL | postgresql.phonebill-dev.svc.cluster.local | 5432 | 메인 데이터베이스 |
| Redis | redis.phonebill-dev.svc.cluster.local | 6379 | 캐시 서버 |

## 5. 데이터 아키텍처

### 5.1 데이터베이스 구성

#### 5.1.1 주 데이터베이스 Pod 구성

**기본 설정:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 컨테이너 이미지 | bitnami/postgresql:16 | 안정화된 PostgreSQL 16 |
| CPU 요청 | 500m | 기본 CPU 할당 |
| Memory 요청 | 1Gi | 기본 메모리 할당 |
| CPU 제한 | 1000m | 최대 CPU 사용량 |
| Memory 제한 | 2Gi | 최대 메모리 사용량 |

**스토리지 구성:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 스토리지 클래스 | managed-standard | Azure Disk Standard |
| 스토리지 크기 | 20Gi | 개발용 충분한 용량 |
| 마운트 경로 | /bitnami/postgresql | 데이터 저장 경로 |
| 백업 전략 | Azure Backup | 일일 자동 백업 |

**데이터베이스 설정값:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 최대 연결 수 | 100 | 동시 연결 제한 |
| Shared Buffers | 256MB | 공유 버퍼 크기 |
| Effective Cache Size | 1GB | 효과적 캐시 크기 |
| Work Memory | 4MB | 작업 메모리 |

#### 5.1.2 캐시 Pod 구성

**기본 설정:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 컨테이너 이미지 | bitnami/redis:7.2 | 최신 안정 Redis 버전 |
| CPU 요청 | 100m | 기본 CPU 할당 |
| Memory 요청 | 256Mi | 기본 메모리 할당 |
| CPU 제한 | 500m | 최대 CPU 사용량 |
| Memory 제한 | 1Gi | 최대 메모리 사용량 |

**메모리 설정:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 데이터 지속성 | Disabled | 개발용, 재시작 시 데이터 손실 허용 |
| 최대 메모리 | 512MB | 메모리 사용 제한 |
| 메모리 정책 | allkeys-lru | LRU 방식 캐시 제거 |
| TTL 설정 | 30분 | 기본 캐시 만료 시간 |

### 5.2 데이터 관리 전략

#### 5.2.1 데이터 초기화

**Kubernetes Job을 통한 데이터 초기화:**
- 데이터베이스 스키마 생성: auth, bill_inquiry, product_change 스키마
- 초기 사용자 데이터: 테스트 계정 생성 (admin, developer, tester)
- 기본 상품 데이터: KOS 연동을 위한 샘플 상품 정보
- 권한 설정: 개발팀용 기본 권한 설정

**실행 절차:**
```yaml
# 데이터 초기화 Job
apiVersion: batch/v1
kind: Job
metadata:
  name: data-init-job
spec:
  template:
    spec:
      containers:
      - name: init-container
        image: bitnami/postgresql:16
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgresql-secret
              key: postgres-password
        command: ["/bin/bash"]
        args:
        - -c
        - |
          psql -h postgresql -U postgres -f /scripts/init-schema.sql
          psql -h postgresql -U postgres -f /scripts/sample-data.sql
      restartPolicy: OnFailure
```

**검증 방법:**
```bash
# 초기화 확인
kubectl exec -it postgresql-0 -- psql -U postgres -c "SELECT COUNT(*) FROM users;"
kubectl exec -it postgresql-0 -- psql -U postgres -c "SELECT COUNT(*) FROM products;"
```

#### 5.2.2 백업 전략

| 서비스 | 백업 방법 | 주기 | 보존 전략 | 참고사항 |
|--------|----------|------|-----------|----------|
| PostgreSQL | Azure Disk Snapshot | 일일 | 7일 보관 | 개발용 데이터 자동 백업 |
| Redis | 없음 | - | 메모리 전용 | 재시작 시 캐시 재구성 |
| Application Logs | Azure Monitor Logs | 실시간 | 14일 보관 | 디버깅용 로그 |

## 6. KOS-Mock 서비스

### 6.1 KOS-Mock 구성

#### 6.1.1 서비스 설정

**KOS-Mock 서비스 구성:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 컨테이너 이미지 | kos-mock:latest | 개발환경용 Mock 서비스 |
| 포트 | 8080 | HTTP REST API |
| 헬스체크 | /health | 서비스 상태 확인 |
| 데이터베이스 | PostgreSQL | Mock 데이터 저장 |

**제공 API:**
| API 경로 | 메소드 | 용도 | 응답 시간 |
|---------|--------|------|-----------|
| /api/v1/bill-inquiry | POST | 요금 조회 Mock | 100-500ms |
| /api/v1/product-change | POST | 상품 변경 Mock | 200-1000ms |
| /api/v1/customer-info | GET | 고객 정보 Mock | 50-200ms |
| /health | GET | 헬스 체크 | 10ms |

#### 6.1.2 Mock 데이터 설정

**Mock 응답 패턴:**
| 응답 타입 | 비율 | 지연시간 | 용도 |
|-----------|------|---------|------|
| 성공 응답 | 80% | 100-300ms | 정상 케이스 테스트 |
| 지연 응답 | 15% | 1-3초 | 타임아웃 테스트 |
| 오류 응답 | 5% | 100ms | 오류 처리 테스트 |

## 7. 보안 아키텍처

### 7.1 개발환경 보안 정책

#### 7.1.1 기본 보안 설정

**보안 계층별 설정값:**
| 계층 | 설정 | 수준 | 설명 |
|------|------|------|----------|
| L4 네트워크 보안 | Network Security Group | 기본 | 기본 Azure NSG 규칙 |
| L3 클러스터 보안 | Kubernetes RBAC | 기본 | 개발팀 전체 접근 권한 |
| L2 애플리케이션 보안 | JWT 인증 | 기본 | 개발용 고정 시크릿 |
| L1 데이터 보안 | TLS 1.2 | 기본 | Pod 간 암호화 통신 |

**관리 대상 시크릿:**
| 시크릿 이름 | 용도 | 순환 정책 | 저장 위치 |
|-------------|------|----------|----------|
| postgresql-secret | PostgreSQL 접근 | 수동 | Kubernetes Secret |
| redis-secret | Redis 접근 | 수동 | Kubernetes Secret |
| jwt-signing-key | JWT 토큰 서명 | 수동 | Kubernetes Secret |
| kos-mock-config | KOS-Mock 설정 | 수동 | Kubernetes ConfigMap |

#### 7.1.2 시크릿 관리

**시크릿 관리 전략:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 관리 방식 | Kubernetes Secrets | 기본 K8s 내장 방식 |
| 암호화 방식 | etcd 암호화 | 클러스터 레벨 암호화 |
| 접근 제어 | RBAC | 네임스페이스별 접근 제어 |
| 감사 로그 | Enabled | Secret 접근 로그 기록 |

### 7.2 Network Policies

#### 7.2.1 기본 정책

**Network Policy 설정:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| Policy 이름 | dev-basic-policy | 개발환경 기본 정책 |
| Pod 선택자 | app=phonebill | 애플리케이션 Pod 대상 |
| Ingress 규칙 | 동일 네임스페이스 허용 | 개발환경 편의상 허용적 정책 |
| Egress 규칙 | 외부 시스템 허용 | KOS-Mock 서비스 접근 허용 |

## 8. 모니터링 및 로깅

### 8.1 기본 모니터링

#### 8.1.1 Kubernetes 기본 모니터링

**모니터링 스택 구성:**
| 구성요소 | 도구 | 상태 | 설명 |
|-----------|------|------|----------|
| 메트릭 서버 | Metrics Server | Enabled | 기본 리소스 메트릭 수집 |
| 대시보드 | Kubernetes Dashboard | Enabled | 웹 기반 클러스터 관리 |
| 로그 수집 | kubectl logs | Manual | 수동 로그 확인 |

**기본 알림 임계값:**
| 알림 유형 | 임계값 | 대응 방안 | 알림 대상 |
|-----------|----------|-----------|----------|
| Pod Crash Loop | 5회 연속 재시작 | 개발자 Slack 알림 | 개발팀 |
| Node Not Ready | 5분 이상 | 노드 상태 점검 | 인프라팀 |
| High Memory Usage | 85% 이상 | 리소스 할당 검토 | 개발팀 |
| Disk Usage | 80% 이상 | 스토리지 정리 | 인프라팀 |

#### 8.1.2 애플리케이션 모니터링

**헬스체크 설정:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| Liveness Probe | /actuator/health/liveness | Spring Boot Actuator |
| Readiness Probe | /actuator/health/readiness | 트래픽 수신 준비 상태 |
| 체크 주기 | 30초 | 상태 확인 간격 |
| 타임아웃 | 5초 | 응답 대기 시간 |

**수집 메트릭 유형:**
| 메트릭 유형 | 도구 | 용도 | 보존 기간 |
|-----------|------|------|----------|
| JVM Metrics | Micrometer | 가상머신 성능 모니터링 | 7일 |
| HTTP Request Metrics | Micrometer | API 요청 통계 | 7일 |
| Database Pool Metrics | HikariCP | DB 연결 풀 상태 | 7일 |
| Custom Business Metrics | Micrometer | 비즈니스 지표 | 7일 |

### 8.2 로깅

#### 8.2.1 로그 수집

**로그 수집 방식:**
| 설정 항목 | 값 | 설명 |
|-----------|----|---------|
| 수집 방식 | stdout/stderr | 표준 출력으로 로그 전송 |
| 저장 방식 | Azure Container Logs | AKS 기본 로그 저장소 |
| 보존 기간 | 7일 | 개발환경 단기 보존 |
| 로그 형식 | JSON | 구조화된 로그 형식 |

**로그 레벨별 설정:**
| 로거 유형 | 레벨 | 설명 |
|-----------|------|----------|
| Root Logger | INFO | 전체 시스템 기본 레벨 |
| Application Logger | DEBUG | 개발용 상세 로그 |
| Database Logger | INFO | 데이터베이스 쿼리 로그 |
| External API Logger | DEBUG | 외부 시스템 연동 로그 |

## 9. 배포 관련 컴포넌트

| 컴포넌트 유형 | 컴포넌트 | 역할 | 설정 |
|--------------|----------|------|------|
| Container Registry | Azure Container Registry Basic | 이미지 저장소 | phonebilldev.azurecr.io |
| CI | GitHub Actions | 지속적 통합 | 코드 빌드, 테스트, 이미지 빌드 |
| CD | ArgoCD | GitOps 배포 | 자동 배포, 롤백 |
| 패키지 관리 | Helm | Kubernetes 패키지 관리 | values-dev.yaml 설정 |
| 환경별 설정 | ConfigMap | 환경 변수 관리 | 개발환경 전용 설정 |
| 시크릿 관리 | Kubernetes Secret | 민감 정보 관리 | DB 연결 정보 등 |

## 10. 비용 최적화

### 10.1 개발환경 비용 구조

#### 10.1.1 주요 비용 요소

| 구성요소 | 사양 | 월간 예상 비용 (USD) | 절약 방안 |
|----------|------|---------------------|-----------|
| AKS 클러스터 | 관리형 서비스 | $73 | 기본 서비스 계층 사용 |
| 노드 풀 (VM) | Standard_B2s × 2 | $60 | Spot Instance 적용 |
| Azure Disk | Standard 20GB × 2 | $5 | 개발용 최소 용량 |
| Load Balancer | Basic | $18 | 기본 계층 사용 |
| Container Registry | Basic | $5 | 개발용 기본 계층 |
| 네트워킹 | 데이터 전송 | $10 | 단일 리전 사용 |
| **총합** | | **$171** | **Spot Instance로 $42 절약 가능** |

#### 10.1.2 비용 절약 전략

**컴퓨팅 영역별 절약 방안:**
| 절약 방안 | 절약률 | 적용 방법 | 예상 절약 금액 |
|-----------|----------|----------|----------------|
| Spot Instances | 70% | 노드 풀에 Spot VM 사용 | $42/월 |
| 비업무시간 자동 종료 | 50% | 야간/주말 클러스터 스케일다운 | $30/월 |
| 리소스 Right-sizing | 20% | requests/limits 최적화 | $12/월 |

**스토리지 영역별 절약 방안:**
| 절약 방안 | 절약률 | 적용 방법 | 예상 절약 금액 |
|-----------|----------|----------|----------------|
| Standard Disk 사용 | 60% | Premium 대신 Standard 사용 | 이미 적용 |
| 스토리지 크기 최적화 | 30% | 사용량 모니터링 후 크기 조정 | $2/월 |

**네트워킹 영역별 절약 방안:**
| 절약 방안 | 절약률 | 적용 방법 | 예상 절약 금액 |
|-----------|----------|----------|----------------|
| Basic Load Balancer | 50% | Standard 대신 Basic 사용 | 이미 적용 |
| 단일 리전 배포 | 100% | 데이터 전송 비용 최소화 | $5/월 |

## 11. 개발환경 운영 가이드

### 11.1 일상 운영

#### 11.1.1 환경 시작/종료

**환경 시작 절차:**
```bash
# 클러스터 스케일업
az aks scale --resource-group phonebill-dev-rg --name phonebill-dev-aks --node-count 2

# 애플리케이션 시작
kubectl scale deployment auth-service --replicas=1
kubectl scale deployment bill-inquiry-service --replicas=1
kubectl scale deployment product-change-service --replicas=1

# 백킹 서비스 시작
kubectl scale statefulset postgresql --replicas=1
kubectl scale deployment redis --replicas=1

# 상태 확인
kubectl get pods -w
```

**환경 종료 절차 (야간/주말):**
```bash
# 애플리케이션 종료
kubectl scale deployment --replicas=0 --all

# 백킹 서비스는 데이터 보존을 위해 유지
# 클러스터 스케일다운 (비용 절약)
az aks scale --resource-group phonebill-dev-rg --name phonebill-dev-aks --node-count 1
```

#### 11.1.2 데이터 관리

**개발 데이터 초기화:**
```bash
# 데이터 초기화 Job 실행
kubectl apply -f k8s/jobs/data-init-job.yaml

# 초기화 진행 상황 확인
kubectl logs -f job/data-init-job

# 데이터 초기화 확인
kubectl exec -it postgresql-0 -- psql -U postgres -c "SELECT COUNT(*) FROM users;"
```

**개발 데이터 백업:**
```bash
# 데이터베이스 백업
kubectl exec postgresql-0 -- pg_dump -U postgres phonebill > backup-$(date +%Y%m%d).sql

# Azure Disk 스냅샷 생성
az snapshot create \
  --resource-group phonebill-dev-rg \
  --name postgresql-snapshot-$(date +%Y%m%d) \
  --source postgresql-disk
```

**데이터 복원:**
```bash
# SQL 파일로부터 복원
kubectl exec -i postgresql-0 -- psql -U postgres phonebill < backup.sql

# 스냅샷으로부터 디스크 복원
az disk create \
  --resource-group phonebill-dev-rg \
  --name postgresql-restored-disk \
  --source postgresql-snapshot-20250108
```

### 11.2 트러블슈팅

#### 11.2.1 일반적인 문제 해결

| 문제 유형 | 원인 | 해결방안 | 예방법 |
|-----------|------|----------|----------|
| Pod Pending | 리소스 부족 | 노드 추가 또는 리소스 조정 | 리소스 사용량 모니터링 |
| Database Connection Failed | PostgreSQL Pod 재시작 | Pod 로그 확인 및 재시작 | Health Check 강화 |
| Service Unavailable | Ingress 설정 오류 | Ingress 규칙 확인 및 수정 | 배포 전 설정 검증 |
| Out of Memory | 메모리 한계 초과 | Memory Limits 증대 | 메모리 사용 패턴 분석 |
| Disk Full | 로그 파일 과다 | 로그 정리 및 보존 정책 수정 | 로그 순환 정책 설정 |

**문제 해결 절차:**
```bash
# 1. Pod 상태 확인
kubectl get pods -o wide
kubectl describe pod <pod-name>

# 2. 로그 확인
kubectl logs <pod-name> --tail=50

# 3. 리소스 사용량 확인
kubectl top pods
kubectl top nodes

# 4. 서비스 연결 확인
kubectl get svc
kubectl describe svc <service-name>

# 5. 네트워크 정책 확인
kubectl get networkpolicy
kubectl describe networkpolicy <policy-name>
```

## 12. 개발환경 특성 요약

**핵심 설계 원칙**: 빠른 개발 > 비용 효율 > 단순성 > 실험성  
**주요 제약사항**: 95% 가용성, 제한적 확장성, 기본 보안 수준  
**최적화 목표**: 개발팀 생산성 향상, 빠른 피드백 루프, 비용 효율적 운영

이 개발환경은 **통신요금 관리 서비스의 빠른 MVP 개발과 검증**에 최적화되어 있으며, Azure의 관리형 서비스를 활용하여 운영 부담을 최소화하면서도 실제 운영환경과 유사한 아키텍처 패턴을 적용했습니다.