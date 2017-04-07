---
title: 使用docker进行项目开发
date: 2017-04-07 15:46:39
tags:
---

*docker是目前大红大紫的云计算技术，它极大的简化项目的部署并提高计算资源的利用率；同样，它也能用于项目的开发，提高代码的开发效率。*

## 用docker进行项目开发的优点
1. 不会污染当前环境，包括环境变量和临时文件。
2. 方便得进行第三方库的版本管理，例如可以分别构建go1.7和go1.8的docker image，编译的时候只用切换docker image就可以实现用不同的go版本进行代码编译。
3. 团队合作中，可以方便地保证大家使用的编译环境是一样的，避免了很多不必要的问题。

## 用docker进行开发的具体实现步骤
以lowtea为例子，介绍如何用docker进行项目开发。

### 构建docker image
前端docker image的Dockerfile，构建出前端运行环境image。
``` Dockerfile
FROM hub.c.163.com/library/node:6.10.0
MAINTAINER zhongxuqi
RUN npm install -g bower && npm install -g gulp && mkdir /workspace
WORKDIR /workspace
```

后端docker image的Dockerfile，构建出后端运行环境的image。
``` Dockerfile
FROM hub.c.163.com/library/golang:1.7.4
MAINTAINER zhongxuqi
RUN mkdir /golang && export GOPATH=/golang && mkdir /workspace
WORKDIR /workspace
```

### 编写Makefile
``` Makefile
SHELL=/bin/bash

lowtea-frontend:
	cd lowtea/front && npm install && gulp serve

docker-lowtea-frontend:
	cd lowtea/front && npm install && gulp serve -p http://lowtea-backend:7070

lowtea-backend:
	cd lowtea && source env.sh && make run
```

### 编写docker-compose.yml
**base-dc.yml**
``` yml
version: "2"
services:
  mongodb:
    image: hub.c.163.com/library/mongo
    ports:
     - "27017"
```

**default-dc.yml**
``` yml
version: "2"
services:
  lowtea-backend:
    image: "hub.c.163.com/zhongxuqi/go-base:latest"
    ports:
     - "7070:7070"
    external_links:
     - mongodb
    volumes:
     - /Users/zhongxuqi/ProjectFiles/github:/workspace
    working_dir: /workspace
    command: bash -c "make lowtea-backend"
  lowtea-frontend:
    image: "hub.c.163.com/zhongxuqi/frontend-base:latest"
    ports:
     - "3000:3000"
     - "3001:3001"
    volumes:
     - /Users/zhongxuqi/ProjectFiles/github:/workspace
    working_dir: /workspace
    links:
     - lowtea-backend
    command: bash -c "make docker-lowtea-frontend"
```

### 启动服务
``` sh
docker-compose -f base-dc.yml up -d
docker-compose -f default-dc.yml up -d
```