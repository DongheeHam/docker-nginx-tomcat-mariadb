# docker-nginx-tomcat-mariadb
도커로 nginx+tomcat+mariadb 웹서버 구축하기

사내 신규 웹서버를 docker와 docker-compose를 이용하여 구축할 기회가 생겨 기록합니다.

## 1. 시스템 아키텍처

![](https://images.velog.io/images/gkaehdgmlzi/post/c76b9f4c-9383-4f65-ba0e-f4b6344f804a/image.png)

## 2. 도커 이미지
도커로 tomcat+mariadb 웹서버를 구축하기 위해 
1. was가 포함됭 tomcat이미지를 구성하고
2. certbot과 nginx가 결합된 이미지를 구성하고
3. mariadb는 기본 이미지를 사용합니다.

### 2-1) nginx-certbot
nginx-certbot 이미지를 생성할 Dockerfile
```
FROM nginx:stable


ARG CERTBOT_EMAIL=info@domain.com
ARG DOMAIN_LIST

RUN  apt-get update \
      && apt-get install -y cron certbot python-certbot-nginx bash wget \
      && certbot certonly --standalone --agree-tos -m "${CERTBOT_EMAIL}" -n -d ${DOMAIN_LIST} \
      && rm -rf /var/lib/apt/lists/* \
# 2,4,6,8,10,12월 1일 새벽 4시마다 인증서 재발급
      && echo "0 4 1 2,4,6,8,10,12 * certbot renew --nginx >> /var/log/cron.log 2>&1" >/etc/cron.d/certbot-renew \
      && crontab /etc/cron.d/certbot-renew
VOLUME /etc/letsencrypt
CMD [ "sh", "-c", "cron && nginx -g 'daemon off;'" ]

```
이미지가 빌드될 때 신규 인증서를 발급받고 crontab을 사용해 주기적으로 자동 갱신됩니다.

이 이미지는 docker-compose에서 빌드하고 실행됩니다.

출처 : https://dzone.com/articles/nginx-and-https-with-lets-encrypt-certbot-and-cron

### 2-2) tomcat

#### 톰켓 이미지를 생성할 Dockerfile
```
FROM tomcat:8.5.47-jdk8-openjdk

# tomcat root 경로 삭제
RUN rm -Rf /usr/local/tomcat/webapps/ROOT 

# war파일 복사
# 왼쪽은 컨테이너를 실행할 host, 오른쪽은 컨테이너 내부임.
# host는 상대경로.
COPY war/ROOT.war /usr/local/tomcat/webapps/ROOT.war
COPY war/CONTEXT.war /usr/local/tomcat/webapps/CONTEXT.war

# was에서 필요한 추가 작업을 진행. 
# 이 예제에서는 마운트 할 폴더를 생성.
RUN mkdir -p /home/files/bin/ && mkdir -p /home/files/log/
# 프로그램 내부적으로 실행하는 파일을 복사, 실행 권한 변경.
COPY bin/execute.sh /home/files/bin/exceute.sh
RUN chmod 777 /home/files/bin/execute.sh

# 추가로 설정해야할 부분이 있다면 추가하면 되겠습니다.
```

위 Dockerfile은 톰켓 기본 이미지에 우리 프로그램에 필요한 세팅들을 한 예제입니다.
이 이미지를 조금더 편하게 생성하기위해 간단한 쉘스크립트를 짜봅시다.

#### build.sh
```
#!/bin/bash

echo "         Building Was image          "
rm -rf war
mkdir war
rm -rf bin
mkdir bin
cp -RL \
                /home/user/war/ROOT.war \
                /home/user/war/CONTEXT.war \
                                war
cp /home/user/bin/execute.sh bin

docker image build --tag tomcatwas:latest .

```

## 3. docker-compose

#### docker-compose.yaml
```

version: '3.1'

services:
	######################################
	# MariaDB : mariadb
	######################################
	mariadbhost:
		image: mariadb:10
		restart: always
		container_name: mariadb
		environment:
			- MYSQL_ROOT_PASSWORD=12341234
			- TZ=Asia/Seoul
		volumes:
			- ./mariadb/data:/var/lib/mysql
			- ./mariadb/config:/etc/mysql/conf.d
			# mariadb서버가 최초로 실행될 때 필요한 스크립트(sql)를
			# docker-entrypoint-initdb.d에 옮겨놓으면
			# 최초 실행시 알아서 실행시켜줍니다. ex) init.sql
			- ./mariadb/init/:/docker-entrypoint-initdb.d/
		ports:
			- 3306:3306
		# 컨테이너별 로그 저장 정책을 결정합니다.
		logging:
			driver: "json-file"
			options:
				max-file: "3"
				max-size: "300m"
	######################################
	# adminer : DB 관리 툴 
	######################################
	adminer:
		image: adminer
		restart: always
		container_name: adminer
		ports:
			- 38080:8080
	######################################
	# tomcatwas : tomcat was 
	######################################
	tomcatwas:
		image: tomcatwas
		restart: always
		container_name: tomcatwas
		environment:
			- HOME_PATH=/home
		# 컨테이너 내부에서 생산되는 데이터(파일,폴더)는 host에 마운트
		volumes:
			- /data/files:/home/files
		ports:
			- 8080:8080
		# 컨테이너별 로그 저장 정책을 결정합니다.
		logging:
			driver: "json-file"
			options:
				max-file: "20"
				max-size: "1g"
	######################################
	# nginx : web 서버
	######################################
	nginx:
		# docker-compose가 up될 때 이미지를 빌드.
		build:
			context: ../images/20-nginx-certbot/
			network: host
			args:
				# certbot 이메일
				- CERTBOT_EMAIL=email@email.com
				# 인증받을 도메인(list)
				- DOMAIN_LIST=myweb.com
		# 컨테이너별 로그 저장 정책을 결정합니다.
		logging:
		driver: "json-file"
		options:
			max-file: "3"
			max-size: "300m"
		restart: always
        container_name: nginx
        volumes:
        	- ./nginx/conf.d:/etc/nginx/conf.d
            - ./nginx/.static_root/:/static/
            # named volume (하단에 볼륨을 정의하지 않으면 오류)
            # 이 방식으로 볼륨을 마운트하면 실제 경로는
            # /var/lib/docker/volumes/compose-volumename/_data/아래에 마운트됩니다.
            - letsencrypt:/etc/letsencrypt
		ports:
        	- "80:80"
            - "443:443"
		depends_on:
        	- tomcatwas
volumes:
        letsencrypt:

```


우리 서버는 nginx에서 80,443을 받고 80이면 443으로 포워딩, 443은 tomcatwas로 포워딩합니다. compose를 띄우기 전에 사전에 nginx설정을 해야 합니다.

#### nginx/conf.d/default.conf
```
server {
    listen       80;
    server_name  myweb.com;
    access_log  /var/log/nginx/host.access.log  main;
    client_max_body_size 0;

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    return 301 https://$host$request_uri;
}

server {
	#adminer로 들어온 경우 adminer로,
    location /adminer/ {
        proxy_pass http://adminer:8080;
    }
    # 그 외 경로는 tomcatwas로 포워딩
    location / {
        proxy_pass http://tomcatwas:8080;
    }
    location /static/ {
        alias /static/;
    }
    listen 443 ssl;
    server_name localhost;

	# ssl key파일은 certbot의 경로를 지정
    ssl_certificate /etc/letsencrypt/live/mywab.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mywab.com/privkey.pem;
}

#include /etc/letsencrypt/options-ssl-nginx.conf;
#ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

```



모든 준비가 완료되었습니다.

이 예제를 기준으로 
1. build.sh 실행(tomcatwas 빌드)
2. docker-compose.yaml실행
을 하면 서버가 구축됩니다.


