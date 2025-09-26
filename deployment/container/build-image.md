# 백엔드 컨테이너 이미지 빌드 결과서

## 작업 개요
- **작업일시**: 2025-09-26
- **작업자**: 최운영/데옵스
- **작업 목표**: 백엔드 마이크로서비스들의 컨테이너 이미지 생성

## 빌드 대상 서비스
총 4개의 백엔드 서비스에 대한 컨테이너 이미지를 생성했습니다.

1. **user-service**: 사용자 관리 서비스
2. **bill-service**: 요금 조회 서비스
3. **product-service**: 상품 변경 서비스
4. **kos-mock**: KOS 시스템 목업 서비스

## 사전 작업

### 1. 서비스별 bootJar 설정 추가
각 서비스의 build.gradle 파일에 일관된 JAR 파일명 설정을 추가했습니다.

```gradle
bootJar {
    archiveFileName = '{서비스명}.jar'
}
```

### 2. Dockerfile 생성
`deployment/container/Dockerfile-backend` 파일을 생성했습니다.

```dockerfile
# Build stage
FROM openjdk:23-oraclelinux8 AS builder
ARG BUILD_LIB_DIR
ARG ARTIFACTORY_FILE
COPY ${BUILD_LIB_DIR}/${ARTIFACTORY_FILE} app.jar

# Run stage
FROM openjdk:23-slim
ENV USERNAME=k8s
ENV ARTIFACTORY_HOME=/home/${USERNAME}
ENV JAVA_OPTS=""

# Add a non-root user
RUN adduser --system --group ${USERNAME} && \
    mkdir -p ${ARTIFACTORY_HOME} && \
    chown ${USERNAME}:${USERNAME} ${ARTIFACTORY_HOME}

WORKDIR ${ARTIFACTORY_HOME}
COPY --from=builder app.jar app.jar
RUN chown ${USERNAME}:${USERNAME} app.jar

USER ${USERNAME}

ENTRYPOINT [ "sh", "-c" ]
CMD ["java ${JAVA_OPTS} -jar app.jar"]
```

### 3. 서비스별 빌드
모든 서비스에 대해 Gradle 빌드를 수행했습니다.

```bash
./gradlew user-service:bootJar
./gradlew bill-service:bootJar
./gradlew product-service:bootJar
./gradlew kos-mock:bootJar
./gradlew api-gateway:bootJar
```

## 컨테이너 이미지 빌드

각 서비스별로 다음 명령어를 사용하여 컨테이너 이미지를 빌드했습니다.

### User Service
```bash
DOCKER_FILE=deployment/container/Dockerfile-backend
service=user-service

docker build \
  --platform linux/amd64 \
  --build-arg BUILD_LIB_DIR="${service}/build/libs" \
  --build-arg ARTIFACTORY_FILE="${service}.jar" \
  -f ${DOCKER_FILE} \
  -t ${service}:latest .
```

### Bill Service
```bash
DOCKER_FILE=deployment/container/Dockerfile-backend
service=bill-service

docker build \
  --platform linux/amd64 \
  --build-arg BUILD_LIB_DIR="${service}/build/libs" \
  --build-arg ARTIFACTORY_FILE="${service}.jar" \
  -f ${DOCKER_FILE} \
  -t ${service}:latest .
```

### Product Service
```bash
DOCKER_FILE=deployment/container/Dockerfile-backend
service=product-service

docker build \
  --platform linux/amd64 \
  --build-arg BUILD_LIB_DIR="${service}/build/libs" \
  --build-arg ARTIFACTORY_FILE="${service}.jar" \
  -f ${DOCKER_FILE} \
  -t ${service}:latest .
```

### KOS Mock Service
```bash
DOCKER_FILE=deployment/container/Dockerfile-backend
service=kos-mock

docker build \
  --platform linux/amd64 \
  --build-arg BUILD_LIB_DIR="${service}/build/libs" \
  --build-arg ARTIFACTORY_FILE="${service}.jar" \
  -f ${DOCKER_FILE} \
  -t ${service}:latest .
```

## 빌드 결과

### 성공적으로 생성된 이미지들

| 서비스명 | 이미지 태그 | 이미지 ID | 크기 | 생성 시간 |
|---------|------------|-----------|------|----------|
| user-service | latest | 6377e1da14f2 | 592MB | 6분 전 |
| bill-service | latest | 9bda5edc843a | 601MB | 6분 전 |
| product-service | latest | 153cd88477f5 | 609MB | 6분 전 |
| kos-mock | latest | 9159dd0accdb | 587MB | 18초 전 |

### 이미지 검증 명령어 실행 결과
```bash
$ docker images | grep -E "user-service|bill-service|product-service|kos-mock"
kos-mock                      latest            9159dd0accdb   18 seconds ago   587MB
bill-service                  latest            9bda5edc843a   6 minutes ago    601MB
product-service               latest            153cd88477f5   6 minutes ago    609MB
user-service                  latest            6377e1da14f2   6 minutes ago    592MB
```

## 빌드 특징

### 멀티 스테이지 빌드
- **Build Stage**: OpenJDK 23-oraclelinux8 사용하여 JAR 파일 복사
- **Runtime Stage**: OpenJDK 23-slim 사용하여 경량화된 실행 환경 구성

### 보안 강화
- 비루트 사용자 `k8s` 생성 및 사용
- 적절한 파일 소유권 및 권한 설정
- 최소 권한 원칙 적용

### 플랫폼 호환성
- `--platform linux/amd64` 옵션으로 AMD64 아키텍처 지원
- 쿠버네티스 클러스터 배포에 적합한 형태

## 다음 단계

1. **컨테이너 레지스트리 푸시**: ACR 또는 Docker Hub에 이미지 푸시
2. **쿠버네티스 매니페스트 작성**: Deployment, Service 등 K8s 리소스 정의
3. **헬름 차트 작성**: 패키지 관리를 위한 Helm 차트 구성
4. **CI/CD 파이프라인 통합**: 자동화된 빌드 및 배포 파이프라인 구축

## 해결된 문제점

### 1. common 모듈 bootJar 오류
- **문제**: common 모듈에서 bootJar() 메서드 인식 불가
- **해결**: common 모듈은 라이브러리 역할로 bootJar 제거

### 2. 플랫폼 불일치 경고
- **문제**: macOS ARM64 환경에서 linux/amd64 빌드
- **상태**: 정상 (Kubernetes 클러스터 대상)

### 3. BuildKit 관련 경고
- **문제**: legacy builder 사용 경고
- **상태**: 빌드 성공 (차후 BuildKit 도입 권장)

## 주요 성과

✅ **모든 백엔드 서비스 컨테이너화 완료** (4개 서비스)
✅ **멀티 스테이지 빌드로 최적화된 이미지** (평균 597MB)
✅ **보안 강화된 컨테이너 구성** (비루트 사용자)
✅ **일관된 빌드 프로세스** (표준화된 Dockerfile)
✅ **쿠버네티스 배포 준비 완료**

모든 백엔드 서비스들이 성공적으로 컨테이너화되었으며, 프로덕션 환경 배포를 위한 준비가 완료되었습니다.
