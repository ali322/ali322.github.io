---
title: gitea + drone 搭建持续集成环境
date: 2022-12-06 16:02:36
tags: git
---

## 安装 gitea

```bash
wget -O gitea https://dl.gitea.io/gitea/1.17.1/gitea-1.17.1-linux-amd64
chmod +x gitea
./gitea web
```

注意设置 webhook

```bash
[webhook]
ALLOWED_HOST_LIST = 10.1.1.1
```

## 启动 drone

```bash
version: '3'

services:
  drone-server:
    image: drone/drone:2
    ports:
      - 2021:80
    restart: always
    volumes:
      - drone-data:/data:rw
    environment:
      - DRONE_AGENTS_ENABLED=true
      - DRONE_GITEA_SERVER=http://10.1.1.1:2020
      - DRONE_GITEA_CLIENT_ID=f1b59dd9-549a-4dcb-abde-e41465a74f2a
      - DRONE_GITEA_CLIENT_SECRET=PK4YvR8Sd65YQYVp46VbSX28DHnawySspq0bxYDczbgs
      - DRONE_RPC_SECRET=bigbrother
      - DRONE_SERVER_HOST=10.1.1.1:2021
      - DRONE_SERVER_PROTO=http
      - DRONE_USER_CREATE=username:root,admin:true
      - DRONE_GIT_ALWAYS_AUTH=true
  drone-agent:
    image: drone/drone-runner-docker:1
    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_RPC_PROTO=http
      - DRONE_RPC_HOST=drone-server
      - DRONE_RPC_SECRET=bigbrother
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_RUNNER_NAME=drone-docker-runner
    dns: 114.114.114.114


volumes:
  drone-data:
```

### 常用配置

- 前端项目

```yaml
kind: pipeline
type: docker
name: wrtc-frontend
steps:
- name: restore-cache  
  image: drillster/drone-volume-cache  
  settings:  
    restore: true  
    mount:  
      - ./.npm-cache  
      - ./node_modules  
  volumes:  
    - name: cache  
      path: /cache
- name: build
  image: node:14.19.0
  commands:
  - npm config set cache ./.npm-cache --global
  - npm config set registry https://registry.npm.taobao.org
  - npm install --silent
  - npm run build
  when:
    branch:
    - master
- name: deploy
  image: drillster/drone-rsync
  settings:
    hosts: [ "test.example.com" ]
    user: root
    key:
      from_secret: rsync_key
    port: 22
    source: ./dist
    target: /usr/share/nginx/html
    prescript:
      - rm -rf dist
    script:
      - cd /usr/share/nginx/html
      - ls -l
  when:
    branch:
    - master
- name: rebuild-cache  
  image: drillster/drone-volume-cache  
  settings:  
    rebuild: true  
    mount:  
      - ./.npm-cache  
      - ./node_modules  
  volumes:  
    - name: cache  
      path: /cache
volumes:  
  - name: cache  
    host:  
      path: /tmp/cache
```

- 后端项目

```yaml
kind: pipeline
type: docker
name: linux
platform:
  os: linux
  arch: amd64
steps:
- name: test
  image: golang:1.17
  commands:
  - go env -w GOPROXY=https://goproxy.cn,direct
  - go test -v ./...
  when:
    branch: dev
    event: push
- name: build
  image: golang:1.17
  volumes:
  - name: deps
    path: /go
  commands:
  - go env -w GOPROXY=https://goproxy.cn,direct
  - CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -ldflags "-X=main.version=${DRONE_TAG}" main.go
  - mv main wrtc
  when:
    event: tag
- name: release
  image: plugins/gitea-release
  settings:
    api_key: 
      from_secret: api_key
    base_url: http://10.1.1.1/git/
    files: wrtc
  when:
    event: tag
volumes:
- name: deps
  temp: {}
```
