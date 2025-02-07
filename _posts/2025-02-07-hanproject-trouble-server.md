---
title: "[HanProject - Troubleshooting] 깃허브 액션으로 메인 브랜치에 푸시하면 코드는 서버에 잘 올라가는데, 새로 추가한 api는 404에러"
date: 2025-02-07 12:17:35 +0800
categories: [HanProject, Troubleshooting]
---


<img width="874" alt="Image" src="https://github.com/user-attachments/assets/a8ae4023-3eb8-4ecd-8f41-591a459188cf" />

### 문제
아래의 야물 파일에서 2번까지 실행이 잘되어서 코드는 가져옴
그러나, 이전에 실행 중이던 빌드 파일의 프로세스를 킬하지 못해서 계속 그 파일이 실행되고 있었던 것 따라서 해당 파일에는 추가된 기능이 없기 때문에 404 오류가 날 수 밖에 없음(아마 처음 서버에서 직접 실행시킨 빌드 파일의 pid를 알 수가 없어서 종료를 못시키고 있었던 것 같음)

.github/workflows/deploy.yml 파일 내용 중 일부
```
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

<br><br><br><br>

---

<br><br><br><br>

### 해결책

서버에 직접 접속해서 현재 실행 중인 프로세스의 pid 얻은 후,
```
ps aux | grep java
```

프로세스 죽이기
```
kill -9 {pid}
```

프로세스를 죽였으니, 메인 브랜치에서 다시 푸시 하면, 
```
git add .
git commit -m "배포 오류 점검용1"
git push origin main
```

새로운 api에 대해 정상동작하는 것을 확인 할 수 있음
<img width="869" alt="Image" src="https://github.com/user-attachments/assets/fba4ee89-a5f7-41d1-a04e-faa3e44e757e" />

서버에서도 프로세스가 작동 중이고, pid도 정확히 저장하고 있음
<img width="757" alt="Image" src="https://github.com/user-attachments/assets/3271e756-42ea-4ec3-a217-1dbbfae3027a" />

(재확인차)
다시 메인 브랜치에 푸시 하면,
```
git add .
git commit -m "배포 오류 점검용2"
git push origin main
```

방금전과 다른 프로세스가 작동 중이고, pid도 제대로 저장하고 있음
<img width="761" alt="Image" src="https://github.com/user-attachments/assets/f0b31c4c-0f87-4e48-91f8-1b835c07f31c" />


<blockquote class="prompt-tip">
kill 및 새로운 프로세스 실행 확인<br>
- commit : 배포 오류 점검용1<br>
- pid : 16464<br>
- 실행시간 : 09:37<br><br>
- commit : 배포 오류 점검용2<br>
- pid : 16748<br>
- 실행시간 : 09:42<br>
</blockquote>

