---
title: "[HanProject - Backend] <맥북을 서버로 사용하기-1> 맥북과 iptime 공유기 설정"  # 문서 제목 입력
date: 2025-03-25 21:20:38  +0800# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---

### 시작
이전에 AWS 서버를 이용해 프로젝트를 진행했지만, 시간이 지나면서 서버 비용이 부담스럽게 느껴졌다.
그래서 "차라리 내 노트북을 서버로 써보자!"는 생각이 들었고, 이 과정을 정리해 아래 글로 남기게 되었다.

### 구성하려는 시스템 구조도

깃허브에 푸시하면, 깃허브 액션이 이를 빌드+테스트하여 빌드된 파일을 서버로 넘기고 서버에서 실행 되도록 한다.<br>
서버에는 스프링부트 빌드 .jar 파일과 Mysql 이 실행 중이고, 프론트로는 리엑트 네이티브를 사용하고 있다.<br>

<img width="843" alt="Image" src="https://github.com/user-attachments/assets/89cfcdbe-57f4-4c70-bc43-d740876f201a" />

### 잠자기 모드 비활성화
```
sudo pmset -a sleep 0
sudo pmset -a disablesleep 1
```

### ​ssh 접근 설정
시스템 설정 > 공유 > 원격 로그인 켬(i버튼 누른 후, '모든 사용자' 접근 허용), 원격 관리 켬(i버튼 누른 후, '모든 사용자' 접근 허용)

+) 위의 설정창에서 원격 호스트명도 변경할 수 있음

```
ssh {사용자명}@{원격 호스트명}
ssh hadahyeon@dahyeon-server.local
```
위의 명령어로 맥os 를 사용하는 컴퓨터는 접근 가능하지만, 윈도우는 불가함, 윈도우도 가능하게 하는 방법은 이어지는 포스트에 있음

<br><br><br><br>

---

<br><br><br><br>

### 공유기 설정 - iptime
`http://192.168.0.1` 로 접속

#### DHCP 서버의 고정 IP 설정
현재 노트북의 ip 주소를 찾아, 사용중인 ip 주소 정보에서 찾아 선택 > 수동 등록 버튼 > 저장 버튼
<img width="886" alt="Image" src="https://github.com/user-attachments/assets/db0eac2d-bb58-4369-9b64-d1dc4a7ec0be" />

#### 포트포워드 설정 
- `규칙 이름` : 내마음대로 설정 (스프링부트 서버로 사용하기 위해 -> spring boot)
- `내부 ip 주소` : 방금 설정한 고정 ip 주소
- `프로토콜` : TCP
- `외부포트` : 80~80 (요청할때 `http://도메인` 뒤에 :8080 쓰기 번거로워서 사진과 달리, 80으로 변경)
- `내부포트` : 8080~8080
위와 같이 설정한 후,
새 규칙 추가 > 저장
<img width="882" alt="Image" src="https://github.com/user-attachments/assets/9565bc93-361a-4bab-97d0-43ac44104ea1" />

- `규칙 이름` : 내마음대로 설정 (ssh 접근 허용하기 위해 -> ssh server)
- `내부 ip 주소` : 방금 설정한 고정 ip 주소
- `프로토콜` : TCP
- `외부포트` : 22~22
- `내부포트` : 22~22
위와 같이 설정한 후, 저장
<img width="876" alt="Image" src="https://github.com/user-attachments/assets/0d83b8d7-5859-442e-854c-368779320c93" />

### DDNS 설정
**인터넷에서 집이나 회사처럼 "고정 IP가 없는" 네트워크에 도메인 이름을 붙여주는 서비스**

아래 사진대로 등록을 하면, 2-3시간 정도 기다리면 '정상 등록 된다.'
<img width="876" alt="Image" src="https://github.com/user-attachments/assets/7ed45257-0b04-42e5-871e-ab24efb38ca9" />