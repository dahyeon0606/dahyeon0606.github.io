---
title: "[HanProject - Backend] <맥북을 서버로 사용하기-3>  ssh 키 원격접속 설정 및 깃허브 액션으로 자동 배포"  # 문서 제목 입력
date: 2025-03-25 21:21:01 +0800 # 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---

### ssh 키를 이용한 서버 접속

#### 1. SSH 키 생성 (로컬에서 실행)
**서버로 사용하지 않는 PC에서**
```
ssh-keygen -t ed25519 -C "your_email@example.com"

//rsa 방식은 지원하지 않으므로 사용하지 않도록 주의!
ssh-keygen -t rsa -b 4096 -C "github-actions"
```

- 파일 경로 묻는 질문에 : 엔터
- 비밀번호 설정 질문에 : 엔터 2번


<img width="532" alt="Image" src="https://github.com/user-attachments/assets/b0141712-add7-41b1-b89c-8bb0b86994aa" />

키를 생성하면, 개인키와 공개키의 경로가 나옴
개인키는 /Users/dahyeon/.ssh/id_rsa
공개키는 /Users/dahyeon/.ssh/id_rsa.pub

#### 2. GitHub Secrets에 개인 키 등록

개인키 출력 및 복사
```
cat ~/.ssh/id_rsa
```

GitHub → **리포지토리 → Settings → Secrets and variables → Actions → New secret**

- `MACBOOK_SSH_KEY` : `id_rsa` 파일의 내용 (개인 키)
- `MACBOOK_HOST` : 맥북의 공인 IP 또는 DDNS (`dahyeon-server.iptime.org`)
- `MACBOOK_USER` : 맥북 사용자명 (`whoami` 로 확인 가능)


#### 3. 맥북 서버에 공개 키 등록
공개키 서버에 전송
```
ssh-copy-id -i {공개키 경로} {호스트명}
ssh-copy-id -i ~/.ssh/id_ed25519.pub hadahyeon@dahyeon-server.iptime.org
```

#### 4. SSH 서버 설정 확인 (`sshd_config`)
```
sudo nano /etc/ssh/sshd_config
```

아래 내용 추가
```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
```

서버 재시작
```
sudo launchctl stop com.openssh.sshd
sudo launchctl start com.openssh.sshd
```


#### 5. 개인키를 이용한 서버 접속
```
ssh -i ~/.ssh/id_ed25519 hadahyeon@dahyeon-server.iptime.org
```

<img width="594" alt="Image" src="https://github.com/user-attachments/assets/9547280c-c9da-47f8-a29a-2753897ad48d" />

서버로 원격 접속이 성공한 것을 확인할 수 있다.

<br><br><br><br>

---

<br><br><br><br>

### 서버에서 실행시킬 프로젝트의 깃허브 리포지토리에 deploy.yml 파일 추가

리포지토리의 루트디렉토리에 `.github/workflows/deploy.yml` 경로로 파일을 만들고,
깃허브의 release/로 시작하는 브랜치에 푸시하면, 맥북 서버로 코드가 자동으로 배포될 수 있도록 설정한다.


deploy.yml
```yaml
name: Deploy Spring Boot to Mac Server

on:
  push:
    branches:
      - release/* # release로 시작하는 모든 브랜치에서 배포 실행

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
      
      - name : Build with gradle # 자바 코드에 테스트 코드가 없을 당시이기 때문에, 테스트는 건너뜀
        run : |
          cd $GITHUB_WORKSPACE/handali
          ./gradlew build -x test 

      - name : Upload build artifact
        uses : actions/upload-artifact@v4 # 현재 v3는 지원되지 않으므로 주의
        with : 
          name : spring-boot-app
          path : /home/runner/work/handali_back/handali_back/handali/build/libs/handali-0.0.1-SNAPSHOT.jar # 상대경로로 설정했을때, 업로드에 실패해서 절대 경로로 지정
  
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

            # `tmux`가 있는 경로를 `$PATH`에 추가
            export PATH="/usr/local/bin:$PATH"


            # 서버 디렉토리로 이동
            cd ~/server

            # 기존 빌드 파일 백업
            if [ -f handali.jar ]; then
              mv handali.jar backup/handali-$(date +%Y%m%d%H%M%S).jar
            fi

            # 새로운 빌드 파일 ~/server/handali.jar 로 이동
            mv ~/server/temp/*.jar handali.jar

            # 기존 애플리케이션 종료
            if tmux has-session -t handaliSession 2>/dev/null; then
               tmux kill-session -t handaliSession
            fi

            # 새로운 애플리케이션 실행
            tmux new-session -d -s handaliSession "java -jar ~/server/handali.jar > ~/server/server.log 2>&1"

            # 실행 확인
            sleep 2
            tmux ls

```
