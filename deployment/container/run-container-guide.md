# 백엔드 컨테이너 실행 가이드

## 실행 정보
- ACR명: acrdigitalgarage02
- VM 접속 정보:
  - KEY파일: ~/home/bastion-dg0507
  - USERID: azureuser
  - IP: 4.217.190.230

## 시스템 구조
- 시스템명: phonebill
- 서비스 구성:
  - api-gateway (포트: 8080)
  - user-service (포트: 8081)
  - bill-service (포트: 8082)
  - product-service (포트: 8083)
  - kos-mock (포트: 8084)

## VM 접속 방법

### 1. 터미널 실행
- Linux/Mac: 기본 터미널 실행
- Windows: Windows Terminal 실행

### 2. Private Key 권한 설정 (최초 한번만)
```bash
chmod 400 ~/home/bastion-dg0507
```

### 3. VM 접속
```bash
ssh -i ~/home/bastion-dg0507 azureuser@4.217.190.230
```

## 컨테이너 레지스트리 인증

### 1. ACR 인증 정보 확인
```bash
az acr credential show --name acrdigitalgarage02
```

### 2. Docker 로그인
```bash
docker login acrdigitalgarage02.azurecr.io -u {username} -p {password}
```
*참고: username과 password는 위의 az acr credential 명령 결과에서 확인*

## Git Repository 클론

### 1. 작업 디렉토리 생성
```bash
mkdir -p ~/home/workspace
cd ~/home/workspace
```

### 2. 소스 클론
```bash
git clone https://github.com/cna-bootcamp/phonebill.git
```

### 3. 프로젝트 디렉토리 이동
```bash
cd phonebill
```

## 애플리케이션 빌드 및 컨테이너 이미지 생성

`deployment/container/build-image.md` 파일의 가이드에 따라 각 서비스의 컨테이너 이미지를 생성합니다.

## 컨테이너 이미지 푸시

각 서비스별로 아래 명령을 실행합니다:

### API Gateway
```bash
docker tag api-gateway:latest acrdigitalgarage02.azurecr.io/phonebill/api-gateway:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/api-gateway:dg0507
```

### User Service
```bash
docker tag user-service:latest acrdigitalgarage02.azurecr.io/phonebill/user-service:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/user-service:dg0507
```

### Bill Service
```bash
docker tag bill-service:latest acrdigitalgarage02.azurecr.io/phonebill/bill-service:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/bill-service:dg0507
```

### Product Service
```bash
docker tag product-service:latest acrdigitalgarage02.azurecr.io/phonebill/product-service:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/product-service:dg0507
```

### KOS Mock
```bash
docker tag kos-mock:latest acrdigitalgarage02.azurecr.io/phonebill/kos-mock:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/kos-mock:dg0507
```

## 컨테이너 실행 명령

### 1. KOS Mock 실행
```bash
SERVER_PORT=8084

docker run -d --name kos-mock --rm -p ${SERVER_PORT}:${SERVER_PORT} \
-e SERVER_PORT=${SERVER_PORT} \
-e SPRING_PROFILES_ACTIVE=dev \
acrdigitalgarage02.azurecr.io/phonebill/kos-mock:latest
```

### 2. User Service 실행
```bash
SERVER_PORT=8081

docker run -d --name user-service --rm -p ${SERVER_PORT}:${SERVER_PORT} \
-e CORS_ALLOWED_ORIGINS=http://localhost:3000,http://4.217.190.230:3000 \
-e DB_HOST=20.249.70.6 \
-e DB_KIND=postgresql \
-e DB_NAME=phonebill_auth \
-e DB_PASSWORD=AuthUser2025! \
-e DB_PORT=5432 \
-e DB_USERNAME=auth_user \
-e DDL_AUTO=update \
-e JWT_ACCESS_TOKEN_VALIDITY=18000000 \
-e JWT_REFRESH_TOKEN_VALIDITY=86400000 \
-e JWT_SECRET=nwe5Yo9qaJ6FBD/Thl2/j6/SFAfNwUorAY1ZcWO2KI7uA4bmVLOCPxE9hYuUpRCOkgV2UF2DdHXtqHi3+BU/ecbz2zpHyf/720h48UbA3XOMYOX1sdM+dQ== \
-e REDIS_DATABASE=0 \
-e REDIS_HOST=20.249.193.103 \
-e REDIS_PASSWORD=Redis2025Dev! \
-e REDIS_PORT=6379 \
-e SERVER_PORT=${SERVER_PORT} \
-e SHOW_SQL=true \
-e SPRING_PROFILES_ACTIVE=dev \
acrdigitalgarage02.azurecr.io/phonebill/user-service:latest
```

### 3. Bill Service 실행
```bash
SERVER_PORT=8082

docker run -d --name bill-service --rm -p ${SERVER_PORT}:${SERVER_PORT} \
-e CORS_ALLOWED_ORIGINS=http://localhost:3000,http://4.217.190.230:3000 \
-e DB_CONNECTION_TIMEOUT=30000 \
-e DB_HOST=20.249.175.46 \
-e DB_IDLE_TIMEOUT=600000 \
-e DB_KIND=postgresql \
-e DB_LEAK_DETECTION=60000 \
-e DB_MAX_LIFETIME=1800000 \
-e DB_MAX_POOL=20 \
-e DB_MIN_IDLE=5 \
-e DB_NAME=bill_inquiry_db \
-e DB_PASSWORD=BillUser2025! \
-e DB_PORT=5432 \
-e DB_USERNAME=bill_inquiry_user \
-e JWT_ACCESS_TOKEN_VALIDITY=18000000 \
-e JWT_REFRESH_TOKEN_VALIDITY=86400000 \
-e JWT_SECRET=nwe5Yo9qaJ6FBD/Thl2/j6/SFAfNwUorAY1ZcWO2KI7uA4bmVLOCPxE9hYuUpRCOkgV2UF2DdHXtqHi3+BU/ecbz2zpHyf/720h48UbA3XOMYOX1sdM+dQ== \
-e KOS_BASE_URL=http://localhost:8084 \
-e LOG_FILE_NAME=logs/bill-service.log \
-e REDIS_DATABASE=1 \
-e REDIS_HOST=20.249.193.103 \
-e REDIS_MAX_ACTIVE=8 \
-e REDIS_MAX_IDLE=8 \
-e REDIS_MAX_WAIT=-1 \
-e REDIS_MIN_IDLE=0 \
-e REDIS_PASSWORD=Redis2025Dev! \
-e REDIS_PORT=6379 \
-e REDIS_TIMEOUT=2000 \
-e SERVER_PORT=${SERVER_PORT} \
-e SPRING_PROFILES_ACTIVE=dev \
acrdigitalgarage02.azurecr.io/phonebill/bill-service:latest
```

### 4. Product Service 실행
```bash
SERVER_PORT=8083

docker run -d --name product-service --rm -p ${SERVER_PORT}:${SERVER_PORT} \
-e CORS_ALLOWED_ORIGINS=http://localhost:3000,http://4.217.190.230:3000 \
-e DB_HOST=20.249.107.185 \
-e DB_KIND=postgresql \
-e DB_NAME=product_change_db \
-e DB_PASSWORD=ProductUser2025! \
-e DB_PORT=5432 \
-e DB_USERNAME=product_change_user \
-e DDL_AUTO=update \
-e JWT_ACCESS_TOKEN_VALIDITY=18000000 \
-e JWT_REFRESH_TOKEN_VALIDITY=86400000 \
-e JWT_SECRET=nwe5Yo9qaJ6FBD/Thl2/j6/SFAfNwUorAY1ZcWO2KI7uA4bmVLOCPxE9hYuUpRCOkgV2UF2DdHXtqHi3+BU/ecbz2zpHyf/720h48UbA3XOMYOX1sdM+dQ== \
-e KOS_API_KEY=dev-api-key \
-e KOS_BASE_URL=http://localhost:8084 \
-e KOS_CLIENT_ID=product-service-dev \
-e KOS_MOCK_ENABLED=true \
-e REDIS_DATABASE=2 \
-e REDIS_HOST=20.249.193.103 \
-e REDIS_PASSWORD=Redis2025Dev! \
-e REDIS_PORT=6379 \
-e SERVER_PORT=${SERVER_PORT} \
-e SPRING_PROFILES_ACTIVE=dev \
acrdigitalgarage02.azurecr.io/phonebill/product-service:latest
```

### 5. API Gateway 실행
```bash
SERVER_PORT=8080

docker run -d --name api-gateway --rm -p ${SERVER_PORT}:${SERVER_PORT} \
-e BILL_SERVICE_URL=http://localhost:8082 \
-e CORS_ALLOWED_ORIGINS=http://localhost:3000,http://4.217.190.230:3000 \
-e JWT_ACCESS_TOKEN_VALIDITY=18000000 \
-e JWT_REFRESH_TOKEN_VALIDITY=86400000 \
-e JWT_SECRET=nwe5Yo9qaJ6FBD/Thl2/j6/SFAfNwUorAY1ZcWO2KI7uA4bmVLOCPxE9hYuUpRCOkgV2UF2DdHXtqHi3+BU/ecbz2zpHyf/720h48UbA3XOMYOX1sdM+dQ== \
-e KOS_MOCK_URL=http://localhost:8084 \
-e PRODUCT_SERVICE_URL=http://localhost:8083 \
-e SERVER_PORT=${SERVER_PORT} \
-e SPRING_PROFILES_ACTIVE=dev \
-e USER_SERVICE_URL=http://localhost:8081 \
acrdigitalgarage02.azurecr.io/phonebill/api-gateway:latest
```

## 실행 확인

각 서비스별로 컨테이너가 정상 실행되었는지 확인:

```bash
docker ps | grep api-gateway
docker ps | grep user-service
docker ps | grep bill-service
docker ps | grep product-service
docker ps | grep kos-mock
```

또는 전체 확인:
```bash
docker ps
```

## 재배포 방법

### 1. 로컬에서 수정된 소스 Push
로컬 개발환경에서 수정사항을 Git에 Push

### 2. VM 접속
```bash
ssh -i ~/home/bastion-dg0507 azureuser@4.217.190.230
```

### 3. 소스 업데이트
```bash
cd ~/home/workspace/phonebill
git pull
```

### 4. 컨테이너 이미지 재생성
`deployment/container/build-image.md` 파일의 가이드에 따라 수행

### 5. 컨테이너 이미지 푸시
```bash
# 예: user-service 재배포
docker tag user-service:latest acrdigitalgarage02.azurecr.io/phonebill/user-service:latest
docker push acrdigitalgarage02.azurecr.io/phonebill/user-service:latest
```

### 6. 기존 컨테이너 중지
```bash
docker stop user-service
```

### 7. 컨테이너 이미지 삭제
```bash
docker rmi acrdigitalgarage02.azurecr.io/phonebill/user-service:latest
```

### 8. 컨테이너 재실행
위의 "컨테이너 실행 명령" 섹션의 해당 서비스 명령 재실행

## 주의사항

1. **CORS 설정**: 모든 CORS_ALLOWED_ORIGINS 환경변수에 프론트엔드 접근 주소(`http://4.217.190.230:3000`)가 포함되어 있는지 확인
2. **서비스 간 통신**: 각 서비스는 localhost를 통해 통신하므로 모든 서비스가 같은 VM에서 실행되어야 함
3. **데이터베이스 연결**: 각 서비스별로 별도의 데이터베이스를 사용하며, 실행 전 데이터베이스 접근 가능 여부 확인 필요
4. **Redis 연결**: 모든 서비스가 공통 Redis를 사용하지만 DATABASE 번호가 다름 (user-service:0, bill-service:1, product-service:2)
