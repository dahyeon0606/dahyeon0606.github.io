---
title: "[Git&GitHub - 기본개념] 이전 커밋으로 돌아가기_reset,revert"
date: 2025-01-15 13:10:00 +0800
categories: [Git&GitHub, 기본개념]
# pin: true
# render_with_liquid: false
---


## Reset
<blockquote class="prompt-info">커밋 히스토리를 되돌리고, 특정 커밋 시점의 상태로 돌아가게 함</blockquote>

1. `git log` : 돌아갈 커밋의 해시를 복사
	 
2. `git reset --hard {돌아갈 커밋의 해시}` : 해당 커밋을 했던 시점의 코드로 변경됨, 해당 커밋 이후의 커밋은 사라짐(초기화)
		
        -- hard: working directory의 변경 사항도 모두 초기화
		-- soft: 커밋만 되돌리고, staging area&working directory는 유지
		-- mixed: 커밋&staging area 되돌리고, working directory는 유지

<br><br><br><br>

## Revert
<blockquote class="prompt-info">특정 커밋을 되돌리되, 기존 커밋 히스토리를 보존하면서 새로운 커밋을 생성</blockquote>

1. `git log` : 취소하고 싶은 커밋의 해시를 복사
2. 
	`git revert {특정 커밋 해시}` : **특정  커밋의 변경 사항**을 되돌리는 새로운 커밋을 생성
	협업 중인 프로젝트에서 안전하게 커밋을 되돌릴때 사용

<br><br><br><br>

## Revert 실습

<blockquote class="prompt-info"> 실습 목표<br>
첫번째 커밋: a.txt 파일 생성 "나는 첫번째 글" 추가<br>
두번째 커밋: a.txt 파일 "나는 두번째 글" 추가<br>
세번째 커밋: a.txt 파일 "나는 세번째 글" 추가<br>
네번째 커밋: b.txt 파일 생성 "1" 추가<br><br>
두번째 커밋으로 revert 했을 때, 결과는 ??
</blockquote>

- 첫번째 커밋
	- `git add .`
	- `git commit -m "first"`
<img width="1093" alt="스크린샷 2025-01-15 오후 1 00 58" src="https://github.com/user-attachments/assets/6a13abd4-04e5-4b41-876d-d47a73c7679e" />

- 두번째 커밋
	- `git add .`
	- `git commit -m "add second line"`
<img width="1093" alt="스크린샷 2025-01-15 오후 1 01 07" src="https://github.com/user-attachments/assets/3a52fd3c-5b8a-4c56-9cfd-8308653e64d0" />

- 세번째 커밋
	- `git add .`
	- `git commit -m "add third line"`
<img width="1092" alt="스크린샷 2025-01-15 오후 1 01 17" src="https://github.com/user-attachments/assets/3102a890-7df0-4a73-bf60-9ade2667fbb5" />

- 네번째 커밋
	- `git add .`
	- `git commit -m "add b.txt"`
<img width="1094" alt="스크린샷 2025-01-15 오후 1 01 28" src="https://github.com/user-attachments/assets/d5c07a4d-4b69-4f4b-af9b-6cad30ad29eb" />


- 두번째 커밋으로 revert
	- `git log` : 두번째 커밋의 해시값 얻기
	- `git revert d3256309b1f4d45d01d` : 두번째 커밋의 해시값 일부 복사하여 revert
	<img width="1090" alt="스크린샷 2025-01-15 오후 1 01 48" src="https://github.com/user-attachments/assets/184b77c7-295f-457c-a25d-9f625829d8f9" />

<blockquote class="prompt-tip"> 결과<br>
두번째 커밋의 변경 사항을 되돌리는 것이기 때문에 "나는 첫번째 글" 만 남는다<br>
b.txt 파일의 변경사항은 두번째 커밋과 관련이 없기 때문에 제거되지 않고 남아있게 된다.<br>
</blockquote>