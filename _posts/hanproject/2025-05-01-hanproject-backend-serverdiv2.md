---
title: "[HanProject - Backend] <개발 서버, 운영 서버 분리-2> 프로파일 분리"  # 문서 제목 입력
date: 2025-05-30 21:21:07  +0800# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---


| 브랜치        | 프로파일   | 포트   | 도메인                         |
| ---------- | ------ | ---- | --------------------------- |
| `main`     | `prod` | 8080 | `handali20.duckdns.org`     |
| `release/` | `dev`  | 8081 | `dev-handali20.duckdns.org` |


## 개발용 데이터베이스 생성
로컬환경, 서버 양쪽에 생성한다.
```
create database handali_db_dev;

//사용자에게 권한 부여
GRANT ALL PRIVILEGES ON handali_db_dev.* TO 'handali_user'@'%';

//적용
FLUSH PRIVILEGES;
```

## application.yml 파일 작성

### application.yml
```yml
spring:  
  application:  
    name: handali  
  
  mvc:  
    pathmatch:  
      matching-strategy: ANT_PATH_MATCHER  
  
  jpa:  
    properties:  
      hibernate:  
        dialect: org.hibernate.dialect.MySQLDialect  
    hibernate:  
      ddl-auto: update  
  
  server:  
    servlet:  
      session:  
        timeout: 1h  
  
#  flyway:  
#    enabled: true  
#    locations: classpath:db/migration  
#    baseline-on-migrate: true  
  
logging:  
  level:  
    org.springframework: DEBUG  
    com.handalsali.handali: DEBUG  
  
jwt:  
  secret: ${JWT_SECRET}
```

### application-dev.yml
새로운 데이터베이스 `handali_db_dev` 를 연결하고, 포트를 8081로 설정한다.
```
spring:  
  config:  
    activate:  
      on-profile: dev  
  
  datasource:  
    url: jdbc:mysql://host.docker.internal:3306/handali_db_dev  
    username: ${DB_USERNAME}  
    password: ${DB_PASSWORD}  
    driver-class-name: com.mysql.cj.jdbc.Driver  
  
  data:  
    redis:  
      host: host.docker.internal  
      port: 6379  
  
server:  
  port: 8081
```

### application-prod.yml
기존 데이터베이스 `handali_db` 를 연결하고, 8080포트로 설정한다.
```
spring:  
  config:  
    activate:  
      on-profile: prod  
  
  datasource:  
    url: jdbc:mysql://host.docker.internal:3306/handali_db  
    username: ${DB_USERNAME}  
    password: ${DB_PASSWORD}  
    driver-class-name: com.mysql.cj.jdbc.Driver  
  
  data:  
    redis:  
      host: host.docker.internal  
      port: 6379  
  
server:  
  port: 8080
```
## docker 파일 작성

### dockerfile
```
# Amazon Corretto 17 베이스 이미지 사용  
FROM amazoncorretto:17  
WORKDIR /app  
  
COPY build/libs/handali-0.0.1-SNAPSHOT.jar app.jar  
  
# spring.profiles.active는 docker-compose에서 넘김  
CMD ["java", "-Dspring.profiles.active=${SPRING_PROFILE}", "-jar", "app.jar"]
```

### docker-compose.dev.yml
```
version: "3.8"  
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

## docker-compose.prod.yml
```
version: "3.8"  
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

## 환경 변수 설정

### .env.dev
```
SPRING_PROFILE=dev  
DB_USERNAME=
DB_PASSWORD=  
JWT_SECRET=
```

### .env.prod
```
SPRING_PROFILE=prod  
DB_USERNAME=
DB_PASSWORD=  
JWT_SECRET=
```


## 실행 및 종료
```
docker-compose -f docker-compose.prod.yml up --build -d
docker-compose -f docker-compose.dev.yml up --build -d
```
localhost:8080, localhost:8081 포트로 요청을 보내서 성공하면 된다.

```
docker-compose -f docker-compose.prod.yml down
docker-compose -f docker-compose.dev.yml down
```
확인했으면 컨테이너를 종료한다.

## 결론
운영 환경 분리는 Nginx + 프로파일만으로도 가능했지만, 온프레미스(macOS) 기반에서 장기적으로 운영해야 하는 프로젝트였기 때문에 실행 환경을 표준화하고, 배포 자동화 구조를 안정화하기 위해 Docker를 도입했다. 도커를 사용하면 어떤 서버에서 실행하든 결과물이 100% 동일해진다. 또, GitHub Actions에서 할 일이 단순해진다.
<br>
결과적으로 배포 과정이 단순해졌고, dev/prod 환경이 인프라 수준에서 완전히 분리되었으며,
서버 이전·확장·장애 대응도 훨씬 용이해졌다.
