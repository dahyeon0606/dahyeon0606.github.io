---
title: "[Git&GitHub - 기본개념] Git 파일 상태(working directory, staging area, git directory)"
date: 2025-02-07 11:53:26 +0800
categories: [Git&GitHub, 기본개념]
---

## 시작
VS Code나 IntelliJ를 사용할 때 터미널에 깃 파일 상태가 표시되었지만, 어떤 의미인지 정확히 이해하지 못하고 사용해왔다. 이를 제대로 정리하기 위해 유튜브 강의를 찾아보고, 내용을 정리하게 되었다.

## git 파일 상태
### 1. working directory(local 환경)
- <font color="#ff0000">unrtracked(U)</font> : 깃이 추적하지 않는 파일, `git add` 명령어로 Staging Area에 추가하면 **Tracked** 상태로 전환됨
	`git add {파일경로}` : working directory 파일을 staging area로 이동시킴
- tracked : 깃이 추적 중인 파일, 이미 Git이 관리하는 파일로 세 가지 상태 중 하나에 속함
	- unmodified : 수정되지 않음, 마지막 커밋 이후 변경되지 않은 상태
	- modified(M) : 수정됨, 마지막 커밋 이후 파일이 변경된 상태
	- Staged (스테이지됨): `git add` 명령어로 변경 사항이 Staging Area에 저장된 상태

### 2. staging area
- 다음 커밋에 포함할 파일과 변경 사항을 임시로 저장하는 공간.
- Staging Area에 있는 파일만 `git commit` 명령어로 커밋 가능.

<blockquote class="prompt-tip">하나의 커밋은 하나의 수정 사항만 포함해야함</blockquote>


### 3. git directory
- **Git Repository**의 메타데이터와 객체 데이터가 저장되는 곳
- `.git` 폴더에 저장되며, 버전 기록과 설정 파일을 포함함

<br><br><br><br>

---

<br><br><br><br>

## GitHub와 상태 변화 흐름

1. **로컬 변경 (Local Changes)**
    - Working Directory에서 파일을 수정하면 **Modified** 상태로 변함.
2. **Staging (스테이징)**
    - `git add` 명령어로 변경된 파일을 Staging Area로 이동해 **Staged** 상태로 전환.
3. **Commit (커밋)**
    - `git commit` 명령어로 Staging Area의 변경 사항을 **Committed** 상태로 저장.
4. **Push (푸시)**
    - `git push` 명령어로 로컬 커밋을 원격 저장소(GitHub)로 전송.
    - GitHub에 저장된 상태는 다른 협업자와 공유 가능.

<br><br><br><br>

---

<br><br><br><br>

## 관련된 주요 명령어

- `git status`  : 트래킹, 커밋된 상태에 대해 출력
- `git log` : commit 로그를 출력
- `git diff`: 변경된 내용을 구체적으로 확인 (Staging Area와 Working Directory 간의 차이도 확인 가능)

## 느낀 점
Git에서 commit이나 add를 할 때 파일 색상이나 터미널 표시가 바뀌는 이유를 이제 알게 되었다. 터미널에 M으로 표시되는 것은 "Modified" 상태를 의미하며, 여기서 git add를 하면 변경된 내용이 Staging Area로 옮겨진다. 이후 git commit을 하면 비로소 "Committed" 상태로 저장된다는 흐름을 이해하게 되었다.