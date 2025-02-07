---
title: Git Flow 개념 정리
date: 2025-02-07 12:50:58
categories: [Git&GitHub, GitFlow]
---
<img width="526" alt="Image" src="https://github.com/user-attachments/assets/f05fbe4b-358d-428d-b2b1-73db8fb37592" />그림 이해하기!

## 브랜치 종류 5개

- feature/{기능명}
	- 특정 기능을 이것저것 개발 중인 브랜치
	- develop 에서 브랜치를 생성하여 기능 개발이 완료되면 develop 브랜치로 병합(merge)후 삭제

- develop (= 개발 브랜치)
	- 다음 배포를 위한 통합 브랜치
	- 로컬환경에서 사용하는 브랜치
	- 배포가 필요할 때, develop에서 release/* 를 생성
	
- release/{버전} (= 배포 브랜치)
	- 배포 직전에 최종 확인용 브랜치
	- 여기서 문제가 없으면 main, develop 브랜치에 병합 후 삭제
	- 추가적인 기능 개발은 금지, 버그 수정 & 안정화 작업만 가능

- main (= 버전관리용 브랜치)
	- 실제 배포 가능하고, 최신 버전의 코드 형상을 유지
	- 운영환경에서 실행되는 안정적인 코드만 포함(최대한 오류 없게 만듦)
	- 절대 직접 수정하지 않고, release/* 또는 hotfix/* 을 통해 변경

- hotfix/{기능명}
	- main 에서 긴급한 수정 사항이 생겼을 때 만드는 브랜치
	- 수정이 완료되면 main, develop에 병합하고 삭제

<br><br><br><br>

---

<br><br><br><br>

## 브랜치 간의 흐름

- 가장 처음 리포지토리 만들자 마자,
	main 생성 후 -->  develop 생성 --> feature 생성

- 실제 기능 개발 및 배포 과정
	`feature/* 에서 기능 개발` --<font color="#92d050">기능 개발 완료</font>--> 
	`develop 에 병합` --<font color="#92d050">배포 준비</font>--> 
	`release/* 에 병합` --<font color="#92d050">서버에서 테스트 완료</font>-->
	`main 에 병합`

<br><br><br><br>

---

<br><br><br><br>


## 기능을 수정 하고 싶을 때
### 기능이 개발 중일 경우 - develop 브랜치 또는 feature/{기능명} 브랜치
<font color="#ff0000">feature/{기능명}</font> 브랜치에서 진행

### 배포 전일 경우 - release/{버전} 브랜치
<font color="#ff0000">release/{버전} </font>에서 직접 수정(단, 작은 수정만 큰 기능 수정은 develop 혹은 feature 에서 진행)

### 이미 배포된 기능일 경우 - main 브랜치
<font color="#ff0000">hotfix/{기능명}</font> 브랜치를 만들어서 사용