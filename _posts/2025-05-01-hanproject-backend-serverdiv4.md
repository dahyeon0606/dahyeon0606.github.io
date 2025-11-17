---
title: "[HanProject - Backend] 개발 서버, 운영 서버 분리-4 GitHub Actions로 Mac 서버에 배포 자동화"
date: 2025-05-01 21:21:07 +0900
categories: [HanProject, Backend]
---

```yml
name: Deploy Spring Boot to Mac Server

on:
  push:
    branches:
      - release/* # release로 시작하는 모든 브랜치에서 배포 실행
      - main

jobs:
  build:
    runs-on: ubuntu-latest # GitHub Actions에서 실행될 환경

    steps:
      - name: Checkout Repository # github 저장소 코드 가져오기
        uses: actions/checkout@v3

      - name : Set up JDK 17
        uses : actions/setup-java@v3
        with :
          distribution : 'temurin'
          java-version : '17'
      
      - name : Build with gradle
        run : |
          cd $GITHUB_WORKSPACE/handali
          ./gradlew build

      - name : Upload build artifact
        uses : actions/upload-artifact@v4
        with : 
          name : spring-boot-app
          path : /home/runner/work/handali_back/handali_back/handali/build/libs/handali-0.0.1-SNAPSHOT.jar

  deploy:
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Download build Artifact
        uses: actions/download-artifact@v4
        with:
          name: spring-boot-app
          path: build/libs/

      - name: Set up ssh key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.MACBOOK_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ secrets.MACBOOK_HOST }} >> ~/.ssh/known_hosts

      # 브랜치에 따라 JAR 파일 이름을 결정하고 서버로 복사합니다.
      # 이제 운영용(app-prod.jar)과 개발용(app-dev.jar)이 분리됩니다.
      - name: Copy JAR file to Mac Server
        run: |
          if [[ "${{ github.ref_name }}" == "main" ]]; then
            TARGET_JAR="app-prod.jar"
          else
            TARGET_JAR="app-dev.jar"
          fi
          echo ">> Determined target JAR name: $TARGET_JAR"
          rsync -avz -e "ssh -i ~/.ssh/id_rsa" build/libs/handali-0.0.1-SNAPSHOT.jar ${{ secrets.MACBOOK_USER }}@${{ secrets.MACBOOK_HOST }}:~/server/${TARGET_JAR}

      # 서버에서 Docker Compose를 실행합니다.
      # 이제 docker-compose 파일이 빌드 인수를 통해 올바른 JAR 파일을 사용하게 됩니다.
      - name: Deploy to Server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.MACBOOK_HOST }}
          username: ${{ secrets.MACBOOK_USER }}
          key: ${{ secrets.MACBOOK_SSH_KEY }}
          script: |
            # 현재 브랜치 이름을 기반으로 프로필과 compose 파일을 결정합니다.
            BRANCH_NAME="${{ github.ref_name }}"
            if [[ "$BRANCH_NAME" == "main" ]]; then
              export SPRING_PROFILE=prod
              COMPOSE_FILE=docker-compose.prod.yml
            else
              export SPRING_PROFILE=dev
              COMPOSE_FILE=docker-compose.dev.yml
            fi
            
            echo ">> Deploying branch: $BRANCH_NAME"
            echo ">> Using profile: $SPRING_PROFILE"
            echo ">> Using compose file: $COMPOSE_FILE"
            
            # 서버 디렉토리로 이동
            cd ~/server

            # Docker Compose가 환경 변수를 사용할 수 있도록 export 합니다.
            export SPRING_PROFILE

            # Docker Compose를 사용하여 컨테이너를 재시작합니다.
            export PATH="/usr/local/bin:$PATH"
            docker-compose -f $COMPOSE_FILE down
            docker-compose -f $COMPOSE_FILE up --build -d
            
            # 사용하지 않는 Docker 이미지(dangling images)를 정리하여 서버 용량을 확보합니다.
            docker image prune -f
```

### 에러
```
ERROR: error getting credentials - err: exit status 1, out: `error getting credentials - err: exit status 1, out: `keychain cannot be accessed because the current session does not allow user interaction. The keychain may be locked; unlock it by running "security -v unlock-keychain ~/Library/Keychains/login.keychain-db" and try again``
```
GitHub Actions에서 SSH로 macOS 서버에 접속해 docker-compose up 명령을 실행했을 때,
Docker가 이미지 풀(Pull) 과정에서 macOS Keychain에 저장된 자격 증명(credential)을 가져오려고 시도했다.
하지만 GitHub Actions에서 들어오는 SSH 세션은 비인터랙티브(non-interactive) 세션이기 때문에
macOS가 Keychain에 대한 접근을 막아버리고, 위와 같은 에러가 발생했다.

```
docker pull amazoncorretto:17
```
서버에 SSH 접속해서 수동으로 Amazon Corretto 이미지를 미리 가져온다. 이렇게 하면 이미지가 캐싱이 되어서 keychain을 사용할 필요가 없어지므로 오류가 해결된다.


## 초반 deploy.yml의 문제와 해결
### 어떤 문제가 발생했는가?
초기 GitHub Actions 배포 파이프라인은 운영(main)과 개발(release/*) 환경이 동일한 jar 파일(handali.jar)을 공유하는 구조였다. 이로 인해 다음 문제가 반복적으로 발생했다.<br>
개발 환경 배포 시 운영 환경 jar 파일이 덮어써짐<br>
Docker Compose가 동일한 context를 바라봐 dev/prod 이미지 충돌<br>
운영 DB까지 영향을 미치는 테스트 데이터 손상 위험<br>
즉, “포트와 도메인만 다른 단일 서버”였기 때문에 운영-개발 환경이 논리적으로는 분리된 것처럼 보이나 실제로는 완전히 결합된 상태였다.

```yml
name: Deploy Spring Boot to Mac Server  
  
on:  
  push:  
    branches:  
      - release/* # release로 시작하는 모든 브랜치에서 배포 실행  
      - main  
  
jobs:  
  build:  
    runs-on: ubuntu-latest # GitHub Actions에서 실행될 환경  
  
    steps:  
      - name: Checkout Repository # github 저장소 코드 가져오기  
        uses: actions/checkout@v3  
  
      - name : Set up JDK 17  
        uses : actions/setup-java@v3  
        with :  
          distribution : 'temurin'  
          java-version : '17'  
      - name : Build with gradle  
        run : |  
          cd $GITHUB_WORKSPACE/handali  
          ./gradlew build  
  
      - name : Upload build artifact  
        uses : actions/upload-artifact@v4  
        with :   
          name : spring-boot-app  
          path : /home/runner/work/handali_back/handali_back/handali/build/libs/handali-0.0.1-SNAPSHOT.jar  
    
  deploy:  
    runs-on: ubuntu-latest  
    needs: build  
  
    steps:  
      - name : Download build Artifact  
        uses : actions/download-artifact@v4  
        with :  
          name : spring-boot-app  
          path : build/libs/  
  
      - name : Set up ssh key  
        run : |  
           mkdir -p ~/.ssh  
           echo "${{ secrets.MACBOOK_SSH_KEY }}" > ~/.ssh/id_rsa  
           chmod 600 ~/.ssh/id_rsa  
           ssh-keyscan -H ${{ secrets.MACBOOK_HOST }} >> ~/.ssh/known_hosts  
  
      - name: Copy JAR file to Mac Server  
        run: |  
          scp -i ~/.ssh/id_rsa build/libs/*.jar ${{ secrets.MACBOOK_USER }}@${{ secrets.MACBOOK_HOST }}:~/server/temp/  
  
        
      - name: Deploy Spring Boot to Mac Server  
        uses: appleboy/ssh-action@v0.1.4  
        with:  
          host: ${{ secrets.MACBOOK_HOST }}  
          username: ${{ secrets.MACBOOK_USER }}  
          key: ${{ secrets.MACBOOK_SSH_KEY }}  
          script: |  
            
            BRANCH="${{ github.ref_name }}"  
            if [ "$BRANCH" = "main" ]; then  
              PROFILE=prod  
              COMPOSE_FILE=docker-compose.prod.yml  
            else  
              PROFILE=dev  
              COMPOSE_FILE=docker-compose.dev.yml  
            fi  
  
  
            # 서버 디렉토리로 이동  
            cd ~/server  
  
            # 기존 빌드 파일 백업  
            if [ -f handali.jar ]; then  
              mv handali.jar backup/handali-$(date +%Y%m%d%H%M%S).jar  
            fi  
  
            # 새로운 빌드 파일 ~/server/handali.jar 로 이동  
            mv ~/server/temp/*.jar handali.jar  
              
            # 컨테이너 재시작  
            export PATH="/usr/local/bin:$PATH"  
              
            docker-compose -f $COMPOSE_FILE down || true  
            docker-compose -f $COMPOSE_FILE up --build -d
```

### 어떻게 해결 했는가?
#### 1) JAR 파일을 환경별로 분리 (핵심 해결 포인트)
GitHub Actions에서 브랜치에 따라 배포 파일명을 분리하도록 수정했다.
```yml
if [[ "${{ github.ref_name }}" == "main" ]]; then
  TARGET_JAR="app-prod.jar"
else
  TARGET_JAR="app-dev.jar"
fi
```
운영/개발 실행 파일이 분리되면서 jar 파일 충돌이 사라짐.

#### 2) Docker Compose도 환경별로 독립 실행
```yml
if [[ "$BRANCH_NAME" == "main" ]]; then
  SPRING_PROFILE=prod
  COMPOSE_FILE=docker-compose.prod.yml
else
  SPRING_PROFILE=dev
  COMPOSE_FILE=docker-compose.dev.yml
fi
```
운영 환경은 prod DB / prod jar / prod 포트<br>
개발 환경은 dev DB / dev jar / dev 포트<br>
→ 완전 독립 실행<br>

#### 3) (추가 개선) rsync 사용으로 “전송 속도” 개선
기존 방식:
```
scp handali.jar → 매번 전체 파일 전송
```
새 방식:
```
rsync -avz
```
기존 배포 방식에서는 GitHub Actions가 생성한 JAR 파일을 scp로 서버에 복사하고 있었다.
하지만 scp는 매번 파일 전체를 다시 전송하는 방식이기 때문에, JAR 파일 용량이 30~60MB 수준이면 네트워크 환경에 따라 업로드 속도 차이가 크게 발생한다.<br>
이를 개선하기 위해 scp 대신 rsync를 사용하도록 변경했다. rsync는 단순 파일 복사가 아니라
"변경된 부분만 전송(delta transfer)" 하는 방식을 사용한다.<br>
즉, 로컬의 JAR 파일과 서버에 저장된 기존 JAR 파일을 비교한 후
변경된 바이트(chunk)만 네트워크로 전송한다.<br>
따라서, 전체 업로드 속도가 2~10배 이상 단축되는 효과가 있었다.

### 결과적으로…
운영·개발 서버가 완전히 분리된 독립 환경으로 전환되었고 서로 어떠한 간섭도 하지 않는 안정적인 구조가 됨<br>
운영 DB 손상 위험 완전 제거<br>
개발 환경에서 기능 검증 후 운영 환경 배포까지 선형적인 CI/CD 파이프라인 완성<br>
서비스 안정성이 높아지고 장애 원인 추적이 훨씬 명확해짐<br>
운영 서비스 품질이 눈에 띄게 향상됨<br>

## 결론
<img alt="Image" src='../../assets/images/hanproject-backend-serverdiv4.png' />

이번 작업을 통해 운영 환경과 개발 환경을 완전히 분리함으로써  
코드, 데이터베이스, 배포 프로세스가 서로 간섭하지 않는 구조를 구축할 수 있었다.

특히 기존 사용자 테이블과 운영 DB는 그대로 유지한 상태에서  
개발용 DB를 신규로 생성하여, 테스트 중 발생할 수 있는  
데이터 손상·변조 위험을 완전히 제거한 점이 가장 큰 성과였다.

또한 GitHub Actions 브랜치 전략(`main → prod`, `release/* → dev`)과  
Docker 기반 실행 환경을 조합함으로써  
**“개발 → 검증 → 운영 반영”**이라는 안정적인 배포 흐름을 확립할 수 있었다.

결과적으로 운영 서비스의 안정성을 높이고,  
기능 추가와 코드 변경에도 더 빠르고 안전하게 대응할 수 있는  
개발·운영 구조를 갖추게 되었다.
