---
title: "[HanProject - Backend] <개발 서버, 운영 서버 분리-3> 서버에 파일 생성 및 도커 작동 여부 확인"  # 문서 제목 입력
date: 2025-05-30 21:21:07  +0900# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---
### 도커 데스크탑 설치
서버에 맥os 전용 도커 데스크탑을 설치한다.(일반적인 Linux 서버에서는 Docker Engine만 설치하면 됨)

### 기존 세션 종료
```
tmux kill-session -t handaliSession
```
서버에서는 Spring Boot가 tmux 세션을 통해 백그라운드에서 실행 중이었으며, 새로운 Docker 기반 환경을 적용하기 위해 이전 세션을 종료하였다.

### 관련 파일 생성
```
nano docker-compose.dev.yml
nano docker-compose.prod.yml
nano .env.dev
nano .env.prod
```


docker-compose.dev.yml
```
services:
  handali-dev:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: handali-dev
    ports:
      - "8081:8081"
    env_file:
      - .env.dev
    restart: unless-stopped
```

docker-compose.prod.yml
```
services:  
  handali-prod:  
    build:  
      context: .  
      dockerfile: Dockerfile  
    container_name: handali-prod  
    ports:  
      - "8080:8080"  
    env_file:  
      - .env.prod  
    restart: unless-stopped
```

```
nano Dockerfile
```

```
# Amazon Corretto 17 베이스 이미지 사용
FROM amazoncorretto:17
WORKDIR /app

COPY handali.jar app.jar

# spring.profiles.active는 docker-compose에서 넘김
CMD ["java", "-Dspring.profiles.active=${SPRING_PROFILE}", "-jar", "app.jar"]
```
COPY 명령은 Docker build의 콘텍스트 경로를 기준으로 하므로, JAR 파일을 Dockerfile과 동일한 디렉토리에 배치한 뒤 COPY 했다.

```
security unlock-keychain ~/Library/Keychains/login.keychain-db
```
macOS 서버에서 Docker가 Keychain을 통해 인증 정보를 가져올 때 비-인터랙티브 SSH 세션에서는 Keychain이 잠겨 오류가 발생한다. 이를 해결하기 위해 먼저 Keychain을 수동으로 해제하였다.

```
docker-compose -f docker-compose.dev.yml up --build -d
```
서버에서 위 명령어를 실행하여 도커가 정상동작하는 것 확인한다.

```
docker-compose -f docker-compose.dev.yml down
```
개발용 컨테이너를 정상적으로 확인한 뒤, 불필요한 리소스 점유를 방지하기 위해 down 명령으로 종료하였다.