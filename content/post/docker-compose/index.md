---
title: "基于Docker Compose部署项目--以校友直聘为例"
description: Deploying a project with Docker Compose – using Alumni Direct Hiring as an example
date: 2026-06-20T16:11:24+08:00
image: screen.jpg
categories:
    - 脚本
    - 部署
    - 容器
tags:
    - docker
    - docker-compose
    - Linux
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

## 前言

Docker部署相较于经典的Linux命令部署的好处

- 环境一致性：镜像保证了开发、测试、生产环境的高度一致。
  - 之前使用无容器的方式，需要依赖本地的JDK和环境变量，而基于容器则已将环境封装在镜像中。
- 可移植性：构建好的镜像可以运行在任何安装有 Docker 的机器上。

----

## Dockerfile

`Dockerfile` 就是以文本格式记录一系列简单的指令，目的是构建一个 Docker 镜像。

通过学习后，发现Dockerfile并不是一连串的执行命令（如Linux部署脚本那样按序执行指令 `mvn+scp+java`）

Dockerfile 是静态配置。更像是一份“镜像配方”，定义了最终要打包成什么样的应用容器。而 `docker build` 命令才去执行构建镜像。

### Dockerfile指令介绍

`FROM`: 基础镜像

`WORKDIR`: 设置当前的工作目录，后续的命令都在该目录执行

`COPY`: 将本地的文件复制到镜像的指定目录中，该指定目录默认是上述的工作目录

`ENV`: 容器运行的环境变量，比如Java的 `-Xms256m -Xmx512m`

`EXPOSE`： 容器运行时使用的端口号

`ENTRYPOINT`：容器启动的入口，也就是执行的命令，如 `["sh", "-c", "java -jar app.jar]`

DockerFile目的是为了构建镜像，设置工作目录的作用是什么？

构建成的镜像里包含该工作目录，jar包放入到工作目录中，方便后续运行。

### 构建后端项目镜像并运行

#### 编写Dockerfile

```dockerfile
# 1. 使用一个轻量级的 OpenJDK 17 作为基础镜像
FROM openjdk:17-jdk-slim

# 2. 设置容器内的工作目录
WORKDIR /app

# 3. 将本地构建好的 JAR 包复制到工作目录，并重命名为 app.jar
COPY target/alumni-direct-service-0.0.1-SNAPSHOT.jar app.jar

# 4. 声明容器运行时将使用 8080 端口
EXPOSE 8080

# 5. 设置 Java 运行时的环境变量（可选）
ENV JAVA_OPTS="-Xms256m -Xmx512m"

# 6. 容器启动时运行的命令
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar --spring.profiles.active=local"]
```

#### 构建镜像

本机拥有Docker Desktop，故启动后即可直接执行docker指令。若无，可将Jar包和Dockerfile文件上传到docker环境再去构建镜像即可。

在 `Dockerfile`所在的项目根目录执行构建指令，构建镜像 `alumni-backend:v1.0`

```bash
# -t 给镜像起个名字和标签，最后的 . 表示构建上下文是当前目录
docker build -t alumni-backend:v1.0 .
```

构建成功后可以看到Docker Desktop的镜像列表出现项目镜像，当然也可以通过 `docker images`查询

![image](1.png)

**注意点**：**基础镜像不会每次都被下载**，Docker 会复用本地缓存。

#### 镜像问题

在这个过程可能会出现各种镜像源问题，如 403 Forbidden/EOF/429 Too Many Requests/not found等各种问题。

可以通过**AI搜索最新可用的镜像源**，另外出现这些问题很可能跟我开了**梯子**有关，国内很多的镜像源不允许国外IP使用。以下是可用镜像源

```
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live"
  ]
}
```

#### 运行容器

```
# -p 将宿主机的 8080 端口映射到容器的 8080 端口
# -d 表示在后台运行
docker run -d -p 8080:8080 --name alumni-app alumni-backend:v1.0
```

## Docker-Compose的使用





----

## 参考文档

[万字长文带你看全网最详细Dockerfile教程-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2327632)
