---
title: "[HanProject - Backend] <맥북을 서버로 사용하기-2>  스프링부트 실행을 위한 환경 설정"  # 문서 제목 입력
date: 2025-03-25 21:20:49  +0900# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---
### 프로젝트를 위한 프로그램 설치 및 설정
#### jdk, git
```
//깃허브
brew update
brew install git

//jdk17
//이렇게 했는데 환경변수 설정 같은게 잘 안 돼서,,
brew install openjdk@17
echo 'export PATH="/usr/local/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
echo 'export JAVA_HOME="$JAVA_HOME_17"' >> ~/.zshrc
export JAVA_HOME_17="/opt/homebrew/opt/openjdk@17"

//이걸로 했더니 됨
brew install cask
brew install --cask temurin@17 
echo 'export JAVA_HOME=$(/usr/libexec/java_home -v 17)' >> ~/.zshrc
source ~/.zshrc
echo $JAVA_HOME

```

#### mysql
```
brew install mysql
brew services start mysql 
brew services list //mysql이 started상태이면 성공

sudo mysql -u root -p
CREATE USER 'handali_user'@'%' IDENTIFIED BY 'Handali1234!'; //새로운 사용자 등록(스프링부트 서버에서 사용함)
GRANT ALL PRIVILEGES ON handali_db.* TO 'handali_user'@'%'; //권한 부여
SELECT user, host FROM mysql.user; //사용자 생성된 것 확인

FLUSH PRIVILEGES; //위의 설정들을 적용

//테이블 생성(스프링부트 서버에서 사용하는 테이블명과 일치)
create database handali_db;
show databases; //생성된 것 확인
```

#### redis
```
brew install redis
brew services start redis
brew services list
```
