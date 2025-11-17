---
title: "[HanProject - Backend] <개발 서버, 운영 서버 분리-1> 도메인 분리 및 엔진엑스 설정"  # 문서 제목 입력
date: 2025-05-30 21:21:07  +0800# 자동으로 오늘 날짜와 시간을 입력
categories: [HanProject, Backend]
# pin: true
# render_with_liquid: false
---

## 시작
개인 맥북을 서버로 사용함으로써 서버비용을 절감할 수 있었고, 여기서 절감한 비용으로 구글 플레이 스토어 계정을 구매했다. 프론트엔드가 1차적으로 완성이 되어서 개발자 계정에 빌드한 파일(aab)을 업로드하고 비공개 테스터들에게 테스트를 진행했다. 이때, 하나의 환경에서 운영과 개발을 진행하다 보니 이전처럼 테스트 데이터의 삽입과 삭제가 자유롭지 못했고, 실수로 실제 사용자의 데이터를 삭제 및 수정할 수도 있겠다 싶었다. 그래서 생각한 것이 운영 환경과 개발 환경을 분리하는 것이었다. 

## 개발 서버용 도메인 발급
`dev-handali20.duckdns.org`<br>
새로운 도메인을 발급 했고, 아래의 과정을 거쳐 동적 ip 설정을 한다.

```
cd ~/duckdns
```

```
nano duck.sh
```

```
echo url="https://www.duckdns.org/update?domains=handali20,dev-handali20&token=여기에_토큰_입력&ip=" | curl -k -o ~/duckdns/duck.log -K -
```
dev-handali20 를 도메인에 추가해준다.

```
chmod 700 ~/duckdns/duck.sh
```

```
./duck.sh
```

```
cat duck.log
```
터미널창에 `OK` 가 뜬 것을 확인했다.

## nginx 설정

직접 관리하기 편하게 **서브 설정 파일을 분리** 했다.<br>
`nginx.conf`에 `include` 라인 추가

```
sudo nano /usr/local/etc/nginx/nginx.conf
```

```
http {
    include       mime.types;
    default_type  application/octet-stream;

    include /usr/local/etc/nginx/sites-enabled/*;
}
```


```
mkdir -p /usr/local/etc/nginx/sites-available
mkdir -p /usr/local/etc/nginx/sites-enabled
```
파일 생성하기

```
sudo nano /usr/local/etc/nginx/sites-available/dev-handali.conf
```

```
server {
    listen 80;
    server_name dev-handali20.duckdns.org;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name dev-handali20.duckdns.org;

    ssl_certificate     /etc/letsencrypt/live/dev-handali20.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dev-handali20.duckdns.org/privkey.pem;

    location / {
        proxy_pass http://localhost:8081; # 개발용 Spring Boot
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```
sudo ln -s /usr/local/etc/nginx/sites-available/dev-handali.conf /usr/local/etc/nginx/sites-enabled/dev-handali.conf
```

```
sudo nginx -t           # 설정 문법 테스트
sudo nginx -s reload    # 설정 다시 로드
```

```
https://dev-handali20.duckdns.org
```
로 접속해서 502 Bad Gateway 가 나오면 nginx가 제대로 설정된 것이다. 아직 8081 포트에서 실행 중인 빌드 파일이 없기 때문에 nginx 만 정상적으로 연동되면 된다.