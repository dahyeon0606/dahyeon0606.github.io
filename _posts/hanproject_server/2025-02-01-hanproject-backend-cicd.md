---
title: "[HanProject] 스프링 부트 백엔드 aws 서버 배포 및 github action으로 ci/cd 환경 구성"
date: 2025-02-01 2:00:00 +0900
categories: [HanProject, Server]
# pin: true
# render_with_liquid: false
---

## 시작
HanProject 백엔드 서버를 AWS EC2에 배포하고, GitHub Actions를 활용하여 CI/CD 환경을 구성하기로 했다.
처음에는 수동으로 .jar 파일을 서버에 올리고 실행하는 방식으로 진행했지만, 변경사항이 생길 때마다 다시 빌드하고 업로드하는 과정이 너무 번거로웠다.
따라서 이를 자동화하고 배포 효율을 높이기 위해 GitHub Actions를 도입했다.
이 글은 AWS 서버 세팅부터 GitHub Actions를 통한 자동 배포까지, 그 과정을 정리한 기록이다.

## 1. 기본 서버 세팅

<blockquote class="prompt-info"> aws 인스턴스 생성

<img width="1094" alt="Image" src="https://github.com/user-attachments/assets/30011e66-5706-4a64-9bbc-5730b93bdbfc" />

<img width="591" alt="Image" src="https://github.com/user-attachments/assets/56cd19a3-5239-4811-8d20-2c9e3eb9c44e" /> 

펨키는 이때 밖에 다운로드 못하니 잘 간직하기

참고) 사진에는 handali-key가 펨키인데, 해당 인스턴스 지우고 다시 만들어서 handali-app-key라는 새로운 팸키를 사용중임

<img width="942" alt="Image" src="https://github.com/user-attachments/assets/940d472a-2059-4633-8faa-6ec866e150e6" />

인스턴스 유형 잘 고려해서 선택하기, 무료로 제공하는 건 정말 간단한 프로젝트에서만 사용 가능, 현재는 t2.medium 사용중 (인스턴스만 몇 번을 중지했다 시작했다, 없앴다 만들었는 지 모름 ㅎ)

<img width="930" alt="Image" src="https://github.com/user-attachments/assets/003e7968-8034-483e-a178-46cc85e6e823" />
</blockquote>


<br><br><br><br>


### 서버에 접속
퍼블릭 IPv4 주소: `43.201.250.84`

펨키 파일 명 : `handali-app-key.pem`

```
chmod 400 handali-app-key.pem
ssh -i handali-app-key.pem ubuntu@43.201.250.84
```

<br><br><br><br>

### 스프링부트 실행을 위해 필요한 파일 다운로드(기초작업)
```
sudo apt update
sudo apt install openjdk-17-jdk -y
sudo apt install git -y
```

<br><br><br><br>

### 1) mysql 설치 및 설정
```
//설치 및 실행
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl status mysql //active 상태 인지 확인

//보안 설정(안해도 됨)
sudo mysql_secure_installatio

//사용자 설정
sudo mysql -u root -p //mysql 루트 계정으로 실행, 첫 비밀번호는 엔터
ALTER USER 'root'@'localhost' IDENTIFIED WITH 'mysql_native_password' BY 'Hyeon01!m'; //루트 비밀번호 변경

CREATE USER 'handali_user'@'%' IDENTIFIED BY 'Handali1234!'; //새로운 사용자 등록(스프링부트 서버에서 사용함)
GRANT ALL PRIVILEGES ON handali_db.* TO 'handali_user'@'%'; //권한 부여
SELECT user, host FROM mysql.user; //사용자 생성된 것 확인

FLUSH PRIVILEGES; //위의 설정들을 적용

//테이블 생성(스프링부트 서버에서 사용하는 테이블명과 일치)
create database handali_db;
show databases; //생성된 것 확인
```

<br><br>

### 2) redis 설치 및 설정
```
sudo apt install redis-server -y
sudo systemctl start redis 
sudo systemctl status redis //active 상태 인지 확인
```

<br><br>

### 3) nginx 설치 및 설정
```
sudo apt install nginx -y
sudo nano /etc/nginx/sites-available/default
```

<br>
아래 내용 입력 - ip주소:80 으로 요청한 내용을 localhost:8080으로 변환해줌

```
server {
    listen 80;
    server_name yourdomain.com; # 도메인을 설정하거나 EC2 퍼블릭 IP 사용

    location / {
        proxy_pass http://localhost:8080; # Spring Boot가 실행 중인 포트
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
<br><br>
nginx 재시작

```
sudo systemctl restart nginx
```
<br><br>
<blockquote class="prompt-tip"> 서버 설정 완료 </blockquote>
<br><br><br><br>

---

<br><br><br><br>

## 2. ci/cd 없이 개발 할 경우,,,,(기초)(매우 번거로워서 비추)
(근데 ci/cd yml 파일에 이 내용을 기반으로 작성하긴 함)

### 1) 빌드 및 빌드 파일 있는 폴더로 이동
```
./gradlew build //가장 처음 빌드 할때 
./gradlew clean build //처음 이후로 빌드 할때
```
<br><br>
.jar 파일이 있는 폴더로 이동

```
cd build/libs
```
<br><br>

###  2) 서버에 .jar 파일 올리기
```
scp -i [펨키 경로] [jar 파일 경로] ubuntu@[ip 주소] : [원격 저장소 경로(내맘대로지정)]
```
- command + option + c : 파일 경로 복사 단축키

<br><br>

###  3) 서버에 접속 후, jar 파일 실행

오류 없이 잘 동작하나 확인하기, 종료는 `control+c`
```
java -jar handali-0.0.1-SNAPSHOT.jar 
```


이후, 백그라운드 실행
```
nohup java -jar handali-0.0.1-SNAPSHOT.jar > application.log 2>&1 & 
```
<br><br>

### +) 프로세스 종료

백그라운드에서 실행 중인지 확인 할 수 있고, 해당 프로세서의 pid 확인가능
```
ps aux | grep java
```

프로세스 종료
```
kill -9 {pid}
```
<br><br>

코드가 수정 되었다면, 프로세스 종료 후, 1~3번 과정을 다시 반복해야 함..ㅎㅎ


<br><br>
<blockquote class="prompt-tip">백엔드 코드 배포 완료</blockquote>

<br><br><br><br>

---

<br><br><br><br>

## 3. ci/cd 셋팅

###  1) 서버 접속 후, 깃허브 리포지토리 클론
```
ssh -i {pem key} ubuntu@{ip}
```

```
git clone https://github.com/ParkJangHa/handali_back.git
```

<br><br>

### 2) ssh 키 생성 및 깃허브 액션에 키 등록

#### 2-1) 키 생성('로컬'에서 생성)
```
ssh-keygen -t rsa -b 4096 -C "github-actions" -f ~/.ssh/github-actions -N ""
```

<blockquote class="prompt-info">
Your identification has been saved in <font color="#ff0000">/home/ubuntu/.ssh/github-actions</font> (개인 키, 깃허브액션에서 사용)<br>
Your public key has been saved in <font color="#ff0000">/home/ubuntu/.ssh/github-actions.pub</font>(공개 키, ec2에 등록)<br>
The key fingerprint is:<br>
SHA256:cmpaL0HKv28fEcnP1Iyd4fiPoaljgRrKoxO/Wn8p6wo github-actions<br>
The key's randomart image is:<br>

</blockquote>

<br><br>

#### 2-2) 공개 키 출력 및 복사하고,
```
cat ~/.ssh/github-actions.pub
```
<blockquote class="prompt-info">  공개키
ssh-rsa ...생략 github-actions" >> ~/.ssh/authorized_keys
</blockquote>

서버에 접속 후, 공개 키를 ec2 서버에 등록
```
ssh -i {pem key} ubuntu@{ip}
```

```
echo "공개키" >> ~/.ssh/authorized_keys

chmod 700 ~/.ssh 
chmod 600 ~/.ssh/authorized_keys
```

<br>

참고) 공개키를 서버에 등록했으므로, 앞으로는 개인 키로도 서버에 접속이 가능해짐
```
ssh -i ~/.ssh/github-actions ubuntu@43.201.250.84
```

<br><br>

#### 2-3) 개인 키 출력(로컬 환경)하고,
```
cat ~/.ssh/github-actions
```

<br>
출력된 키 형식

```
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

깃허브에 등록
<blockquote class="prompt-warning">
깃허브 리포지토리 > settings > secrets and variables > actions > new repository secret<br>
 - ssh key 등록 <br>
	제목 : AWS_SSH_KEY<br>
	내용 : 복사한 개인 키<br><br>
 - host 등록<br>
	제목: AWS_HOST<br>
	내용: EC2 퍼블릭 Ip<br>
</blockquote>

<br>
호스트와 키를 등록함으로써, 깃허브 액션이 ssh로 해당 호스트에 키를 가지고 접근할 수 있어짐
ssh -i ~/.ssh/github-actions ubuntu@43.201.250.84 위에 참고로 달아둔, 이 접근을 하는 것임

<br><br>
<blockquote class="prompt-danger">**ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain** 에러
</blockquote>
<br>
해결법

```
sudo nano /etc/ssh/sshd_config //ssh 설정 파일 열기
```

//4줄 추가

```
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
PubkeyAcceptedKeyTypes +ssh-rsa
```

```
sudo systemctl restart ssh //재시작
sudo systemctl status ssh //active 이면 성공
```

<br><br>

### 3) 깃허브 액션 워크플로우 파일 작성

로컬 환경에서 리포지토리의 루트 경로로 터미널 열기(.git 파일 있는 경로), 루트 경로에 아래의 파일이 생성 되어야 함

```
mkdir -p .github/workflows
nano .github/workflows/deploy.yml
```


.github/workflows/deploy.yml 파일 내용

```
name: Deploy to AWS EC2

on:
  push:
    branches:
      - main  # main 브랜치에 Push될 때 실행

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v3

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v0.1.4
        with:
          host: ${{ secrets.AWS_HOST }}
          username: ubuntu
          key: ${{ secrets.AWS_SSH_KEY }}
          script: |
            
            # 1. main 브랜치 코드 가져오기 
            cd /home/ubuntu/handali_back 
            git pull origin main
            
            
            # 2. 코드 빌드하기
            cd handali 
            ./gradlew clean build
            
            
            # 3. 권한 설정하기
            chmod +x gradlew 

            # 4. 기존 애플리케이션 프로세스 종료 (PID 파일을 이용)
            if [ -f app.pid ]; then
              old_pid=$(cat app.pid)
              echo "Stopping existing process with PID $old_pid"
              kill -9 $old_pid || true
              sleep 2
              rm -f app.pid
            fi
            
            
            #4. 빌드된 파일 백그라운드 실행하기
            echo "==== Running application in background ===="
            setsid java -jar build/libs/handali-0.0.1-SNAPSHOT.jar > app.log 2>&1 &
            echo $! > app.pid
            
            echo "==== Deployment Finished! ===="
```

yml 파일을 원격 저장소에 푸시

```
 git add .github/workflows/deploy.yml 
 git commit -m "Add GitHub Actions workflow for deployment"
 git push origin main 
```

<br><br>

### 4) main에 푸시할 경우, 자동으로 배포가 되는지 확인

내용 수정 이후, 

```
git add .
git commit -m "자동배포 테스트"
git push origin main
```

깃허브 액션에서 확인가능
<img width="922" alt="Image" src="https://github.com/user-attachments/assets/14564b50-5c6a-4566-81ad-504f81a1e0e9" />


작동 중인 java -jar 프로세서가 있는 지 확인하고, 포스트맨으로 요청 날렸을 때 정상 동작 하면 성공!
```
ps aux | grep java
```
(혹은, 서버에 접속해서 내용 수정한 부분이 제대로 적용 되었나 확인 해봐도 됨)

<br><br>
<blockquote class="prompt-tip">ci/cd 환경 구성 완료</blockquote>

## 느낀 점
이번에 AWS EC2에 서버를 배포하고 GitHub Actions로 자동 배포 환경을 구축하면서 CI/CD의 필요성과 중요성을 제대로 체감할 수 있었다.
특히 서버와 통신할 수 있도록 SSH 키를 설정하고, GitHub에 비밀 키를 등록해 워크플로우를 구성하는 과정이 처음에는 낯설었지만, 하나하나 과정을 밟아가며 이해할 수 있었다.
수동 배포보다 훨씬 빠르고 안정적으로 서비스를 업데이트할 수 있게 되어 앞으로 다른 프로젝트에서도 CI/CD를 기본 세팅으로 가져가야겠다는 생각이 들었다.
물론 중간에 SSH 에러나 권한 문제 등 다양한 시행착오가 있었지만, 이런 경험들이 나만의 서버 운영 노하우로 쌓이는 것 같아 뿌듯하다.