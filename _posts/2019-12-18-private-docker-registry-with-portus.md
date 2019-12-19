---
title: "Docker Registry와 Portus를 이용한 Private Docker Registry 구축"
date: 2019-12-18 20:17:15 +0900
categories: docker registry portus authorization ui
---
## 시작
Docker를 본격적으로 사용하기 위해 회사 내에서 사용할 Private Docker Registry 구축이 필요했다.
EC2서버에 S3를 저장소로 사용하는 [Docker Registry 2.0](https://hub.docker.com/_/registry) 설치는 다음 명령어로 손쉽게 가능하다.
```
docker run -d -p 5000:5000 --restart=always --name docker-registry \
  -v /etc/letsencrypt:/certs \
  -v /docker/docker-registry:/var/lib/registry \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/live/docker-registry.zwolf.org/fullchain.pem \
  -e REGISTRY_HTTP_TLS_KEY=/certs/live/docker-registry.zwolf.org/privkey.pem \
  -e REGISTRY_STORAGE=s3 \
  -e REGISTRY_STORAGE_S3_BUCKET=docker-registry \
  -e REGISTRY_STORAGE_S3_ACCESSKEY=<S3_ACCESSKEY> \
  -e REGISTRY_STORAGE_S3_SECRETKEY=<S3_SECRETKEY> \
  -e REGISTRY_STORAGE_S3_REGION=ap-northeast-2 \
  registry:2
```
하지만 Docker Registry는 간단한 htpasswd 사용자 인증 기능만 제공한다. 제대로된 사용자 인증 기능이 필요했다.

> 참고
> - [https://docs.docker.com/registry/](https://docs.docker.com/registry/)
> - [https://docs.docker.com/registry/storage-drivers/s3/](https://docs.docker.com/registry/storage-drivers/s3/)
> - [https://github.com/docker/distribution](https://github.com/docker/distribution)

## [Portus](http://port.us.org/)
오픈 소스이면서 간단히 Docker Registry의 사용자 인증 기능을 제공하며 LDAP 연동도 가능한 서비스 모듈을 찾아보았다. Portus가 제격인 것 같다. 다음은 Portus 홈페이지에 나와있는 설명이다.
> Portus is an open source authorization service and user interface for the next generation Docker Registry.

> 참고
> - [http://port.us.org/](http://port.us.org/)
> - [http://port.us.org/docs/first-steps.html](http://port.us.org/docs/first-steps.html)
> - [https://github.com/SUSE/Portus](https://github.com/SUSE/Portus)
> - [https://github.com/SUSE/Portus/tree/master/examples/compose](https://github.com/SUSE/Portus/tree/master/examples/compose)

### .env
```
MACHINE_FQDN=docker-registry.zwolf.org

PORTUS_SECRET_KEY_BASE=b494a25faa8d22e430e843e220e424e10ac84d2ce0e64231f5b636d21251eb6d267adb042ad5884cbff0f3891bcf911bdf8abb3ce719849ccda9a4889249e5c2
PORTUS_PASSWORD=12341234

DATABASE_PASSWORD='portus'

S3_BUCKET=docker-registry
S3_ACCESSKEY=S3ACCESSKEYHR6RCXSW3
S3_SECRETKEY=s3secretkeyupm6Xak587y3EdSBYrSuG5pOJ5DxA
S3_REGION=ap-northeast-2
```

### docker-compose.yml
```
version: "2"

services:
  portus:
    image: opensuse/portus:2.4
    environment:
      - PORTUS_MACHINE_FQDN_VALUE=${MACHINE_FQDN}
      # DB
      - PORTUS_DB_ADAPTER=mysql2
      - PORTUS_DB_HOST=db
      - PORTUS_DB_PORT=3306
      - PORTUS_DB_DATABASE=portus_production
      - PORTUS_DB_PASSWORD=${DATABASE_PASSWORD}
      - PORTUS_DB_POOL=5
      # Secrets
      - PORTUS_SECRET_KEY_BASE=${PORTUS_SECRET_KEY_BASE}
      - PORTUS_KEY_PATH=/certs/live/${MACHINE_FQDN}/privkey.pem
      - PORTUS_PASSWORD=${PORTUS_PASSWORD}
      # SSL
      - PORTUS_PUMA_TLS_KEY=/certs/live/${MACHINE_FQDN}/privkey.pem
      - PORTUS_PUMA_TLS_CERT=/certs/live/${MACHINE_FQDN}/fullchain.pem
      # etc
      - RAILS_SERVE_STATIC_FILES='true'
      - CCONFIG_PREFIX=PORTUS
    ports:
      - 3000:3000
    volumes:
      - /etc/letsencrypt:/certs:ro
    depends_on:
      - db
    links:
      - db

  background:
    image: opensuse/portus:2.4
    environment:
      - PORTUS_MACHINE_FQDN_VALUE=${MACHINE_FQDN}
      # DB
      - PORTUS_DB_ADAPTER=mysql2
      - PORTUS_DB_HOST=db
      - PORTUS_DB_PORT=3306
      - PORTUS_DB_DATABASE=portus_production
      - PORTUS_DB_PASSWORD=${DATABASE_PASSWORD}
      - PORTUS_DB_POOL=5
      # Secrets
      - PORTUS_SECRET_KEY_BASE=${PORTUS_SECRET_KEY_BASE}
      - PORTUS_KEY_PATH=/certs/live/${MACHINE_FQDN}/privkey.pem
      - PORTUS_PASSWORD=${PORTUS_PASSWORD}
      # etc
      - PORTUS_BACKGROUND=true
      - CCONFIG_PREFIX=PORTUS
    volumes:
      - /etc/letsencrypt:/certs:ro
    depends_on:
      - portus
      - db
    links:
      - db

  db:
    image: library/mariadb:10.0.23
    command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci --init-connect='SET NAMES UTF8;' --innodb-flush-log-at-trx-commit=0
    environment:
      - MYSQL_DATABASE=portus_production
      - MYSQL_ROOT_PASSWORD=${DATABASE_PASSWORD}
    volumes:
      - /docker/portus/mariadb:/var/lib/mysql

  registry:
    image: registry:2.7
    environment:
      # Authentication
      REGISTRY_AUTH_TOKEN_REALM: https://${MACHINE_FQDN}:3000/v2/token
      REGISTRY_AUTH_TOKEN_SERVICE: ${MACHINE_FQDN}:5000
      REGISTRY_AUTH_TOKEN_ISSUER: ${MACHINE_FQDN}
      REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE: /certs/live/${MACHINE_FQDN}/fullchain.pem
      # SSL
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/live/${MACHINE_FQDN}/fullchain.pem
      REGISTRY_HTTP_TLS_KEY: /certs/live/${MACHINE_FQDN}/privkey.pem
      # S3
      REGISTRY_STORAGE: s3
      REGISTRY_STORAGE_S3_BUCKET: ${S3_BUCKET}
      REGISTRY_STORAGE_S3_ACCESSKEY: ${S3_ACCESSKEY}
      REGISTRY_STORAGE_S3_SECRETKEY: ${S3_SECRETKEY}
      REGISTRY_STORAGE_S3_REGION: ${S3_REGION}
      # Portus endpoint
      REGISTRY_NOTIFICATIONS_ENDPOINTS: >
        - name: portus
          url: https://${MACHINE_FQDN}:3000/v2/webhooks/events
          timeout: 2000ms
          threshold: 5
          backoff: 1s
    ports:
      - 5000:5000
    volumes:
      - /etc/letsencrypt:/certs:ro
      - /docker/portus/registry:/var/lib/registry
    links:
      - portus
```

### 실행
```
$ docker-compose up -d
Creating network "portus_default" with the default driver
Creating portus_db_1 ... done
Creating portus_portus_1 ... done
Creating portus_background_1 ... done
Creating portus_registry_1   ... done
```

## 테스트
먼저 Portus에 접속해서 관리자 계정과 사용자 계정을 생성한다.
- https://docker-registry.zwolf.org:3000

![First Admin User Create](https://raw.githubusercontent.com/jongho/jongho.github.io/master/files/portus-screenshot-1.png)
![Normal User Create](https://raw.githubusercontent.com/jongho/jongho.github.io/master/files/portus-screenshot-2.png)

### Login & Push, Pull
```
## hello-world pull from Docker Hub
$ docker pull hello-world
Using default tag: latest
latest: Pulling from library/hello-world
1b930d010525: Pull complete 
Digest: sha256:4fe721ccc2e8dc7362278a29dc660d833570ec2682f4e4194f4ee23e415e1064
Status: Downloaded newer image for hello-world:latest
docker.io/library/hello-world:latest

## hello-world tag onto Private Docker Registry
$ docker tag hello-world:latest docker-registry.zwolf.org:5000/hello-world

## images 확인
$ docker images
REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
docker-registry.zwolf.org:5000/hello-world   latest              fce289e99eb9        11 months ago       1.84kB
hello-world                                     latest              fce289e99eb9        11 months ago       1.84kB

## hello-world push onto Private Docker Registry (인증 관련 오류 발생)
$ docker push docker-registry.zwolf.org:5000/hello-world
The push refers to repository [docker-registry.zwolf.org:5000/hello-world]
af0b15c8625b: Preparing 
denied: requested access to the resource is denied

## 생성한 사용자로 로그인을 해보자.
$ docker login docker-registry.zwolf.org:5000/hello-world
Username: zwolf
Password: 
WARNING! Your password will be stored unencrypted in /home/zwolf/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

## hello-world push onto Private Docker Registry (성공)
$ docker push docker-registry.zwolf.org:5000/hello-world
The push refers to repository [docker-registry.zwolf.org:5000/hello-world]
af0b15c8625b: Layer already exists 
latest: digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a size: 524
```
잘 된다.

## 나머지
- LDAP 연동
    - [docker-compose.ldap.yml 예제](https://github.com/SUSE/Portus/blob/master/examples/compose/docker-compose.ldap.yml)
- Nginx Proxy 설정
    - [docker-compose.yml 예제](https://github.com/SUSE/Portus/blob/master/examples/compose/docker-compose.yml)
    - [nginx.conf 예제](https://github.com/SUSE/Portus/blob/master/examples/compose/nginx/nginx.conf)
