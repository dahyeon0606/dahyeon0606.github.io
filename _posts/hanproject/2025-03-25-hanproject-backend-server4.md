---
title: "[HanProject - Backend] <맥북을 서버로 사용하기-4> 다른 노트북에서 맥북으로 ssh 접근"  # 문서 제목 입력
date: 2025-03-25 21:21:07  +0800# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---


### ​1. 키 생성
```
ssh-keygen -t ed25519 -C "본인이름"
```
- 파일 경로 묻는 질문에 : 엔터 1번
- 비밀번호 설정 질문에 : 엔터 2번

### 2. 공개키 출력 및 복사
```
cat ~/.ssh/id_ed25519.pub
```
공개키를 출력하고, 복사한다. 
좋은 방법은 아니지만,,,, 다른 노트북에서 맥북으로 원격 접속이 불가하니 서버로 사용하려는 맥북으로 복사한 공개키를 메일로 보낸다.

### 3. 맥북 서버 접속 및 공개키 등록
```
echo "공개키" >> ~/.ssh/authorized_keys
```
서버로 사용하는 맥북에 접속하여 메일에서 공개키를 복사해 위 명령어에 삽입한다.

> 이제 다른 노트북에서도 맥북 서버로 ssh 접근이 가능해진다.<br>
> ssh {user}@{hostname}
{: .prompt-tip}


