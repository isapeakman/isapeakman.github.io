---
title: "项目部署过程涉及的Linux命令--以校友直聘为例"
description: Linux commands involved in project deployment – taking alumni direct recruitment as an example
date: 2026-06-16T21:35:23+08:00
categories:
    - 部署
    - 脚本
tags:
    - Linux
    - Maven
    - SpringBoot
    - Vue
    - Docker
    - Nginx
    - Mysql
    - Redis
    - WSL
    - ubantu
image: screen.jpg
math: 
license: 
comments: true
draft: false
build:
    list: always    # Change to "never" to hide the page from the list
---

## 前言

Linux脚本，从部署一个项目开始到结束用到的命令

以及包含docker命令

贴图表示真实。

采用window下的wsl+ubantu

username: chinese

password: chinese



部署校友直聘项目

部署nginx  + 前端项目

部署后端项目

日志输出

部署MySQL+redis

基于shell脚本部署项目？

基于docker部署

Docker Compose

dockerFile的用法

- 传统部署实战 ： nohup , systemd , nginx 相关命令    命令+脚本+使用外部配置文件(java -jar myapp.jar --spring.profiles.active=prod)

- Docker部署实战 ： docker build , docker run , docker logs 等命令
- Docker Compose实战 ： docker compose up/down/restart 等命令
- 数据库与Redis配置 ：环境变量注入、配置文件管理
- 日志与监控 ： tail , grep , top , htop 等命令
- 故障排查 ：常见问题及解决命令
- 总结与建议 ：不同场景下的部署方式选择

----

## 第一步：基础环境

当前本地环境是windows下通过WSL实现的子系统Ubuntu

在Ubuntu下

### 1.  更新系统包列表：

```bash
sudo apt update
```

> apt是**Ubuntu 及其衍生发行版** 下的**高级包管理命令行工具**，核心作用是管理软件包——包括查找、安装、升级、卸载软件
>
> ```bash
> sudo apt update               # 刷新本地软件列表（同步远程源信息）
> sudo apt upgrade			  #将所有已安装的包升级到最新版本
> ```

### 2.  安装 Java 运行环境 (JDK)

```bash
# 安装 OpenJDK 17
sudo apt install openjdk-17-jdk -y
# 验证安装
java -version
openjdk version "17.0.19" 2026-04-21                                                                     OpenJDK Runtime Environment (build 17.0.19+10-1-24.04.2-Ubuntu)                                           OpenJDK 64-Bit Server VM (build 17.0.19+10-1-24.04.2-Ubuntu, mixed mode, sharing)  
```

安装位置：`dpkg -L openjdk-17-jdk`

![image](1.png)

核心内容都在**`/usr/lib/jvm/java-17-openjdk-amd64/`**文件夹中。

> 是否需要配置环境变量：不需要，apt已经在/usr/bin/java、/usr/bin/javac等创建为软链接，并最终指向 JDK 的实际安装目录下的可执行文件（比如/usr/lib/jvm/java-17-openjdk-amd64/bin/java）

### 3. 安装 Nginx

Nginx 作为 Web 服务器和反向代理服务器。

```bash
sudo apt install nginx -y
# 验证安装
nginx -v
nginx version: nginx/1.24.0 (Ubuntu)
```

### 4. 安装数据库 (MySQL)：

查看是否安装过mysql `dpkg -l | grep mysql-server`

- 如果输出以 `ii` 开头（如 `ii mysql-server ...`），说明已经装过了。
- 如果输出以 `rc` 开头，说明配置文件还在，但软件已删除。
- 如果无任何输出，说明未安装。

```bash
# 安装 MySQL 服务器
sudo apt install mysql-server -y
# 安装后建议运行安全脚本
sudo mysql_secure_installation
```

### 5. 安装redis

```bash
# 1. 更新源并安装
sudo apt update
sudo apt install redis-server -y

# 2. 启动 Redis 服务并设为开机自启
sudo systemctl start redis-server
sudo systemctl enable redis-server

# 3. 检查运行状态
sudo systemctl status redis-server

# 4. 测试连接（默认监听 127.0.0.1:6379）
redis-cli ping
# 返回 PONG 即成功
```

#### redis供外部连接

- 配置文件：`/etc/redis/redis.conf`
- 默认只允许本地连接，若需远程访问，需注释 `bind 127.0.0.1`并将`protected-mode` yes改为 no 
- ![image](8.png)

##### 通过windows的cmd连接Redis

`telnet 172.31.19.238 6379`

![image](10.png)

其中 $5表示五个字节的意思

当前窗口使用 `quit`进行退出

##### 通过RedisDesktopManager连接

![image](9.png)

### 6. 安装minio

#### 1. 下载并启动

由于**MinIO 官方并没有将服务器版本提交到 Ubuntu/Debian 的默认 APT 源**，所以使用wget下载

```bash
# 1. 下载最新版 MinIO 二进制（64位 Linux）
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio

# 2. 创建数据目录
sudo mkdir -p /data/minio

# 3. 设置环境变量（凭据）
export MINIO_ROOT_USER=minioadmin
export MINIO_ROOT_PASSWORD=minioadmin

# 4. 启动服务（前台运行，默认 API 端口 9000，控制台端口在启动日志里看）
./minio server /data/minio
# 后台运行命令并将日志输出到minio.log中
nohup ./minio server /data/minio > minio.log 2>&1 &
```

#### 2. 查看minio进程

![image](14.png)

第一条为minio进程，第二条为grep进程。minio的进程号为3168

#### 3. 查看minio端口号

```
# 查看所有minio占用端口号，默认为9000
sudo ss -tlnp | grep minio
```

#### 4. 通过windows访问minio客户端页面

![image](15.png)

用户名和密码均为 `minioadmin`

#### 5. 可能遇到的文件权限访问问题

![image](13.png)

本质上是一个 **文件系统权限问题**：MinIO 进程试图在 `/data/minio/.minio.sys` 下做内部元数据维护（重命名临时目录），但被**拒绝了**。

可以考虑用以下命令

```
sudo chown -R $USER:$USER /data/minio  
#递归地把 /data/minio 目录及其内部所有文件和子目录的所有者，改为当前用户（并同时更改所属组为当前用户组）
```

执行完后再执行启动命令

### 7. 安装SSH 

ssh用于文件上传，模拟真实的Linux环境，虽然可以让ubantu直接访问本地文件

#### 1. 安装并修改SSH配置文件

```bash
sudo apt install openssh-server -y
# 修改 SSH 配置文件
sudo vim /etc/ssh/sshd_config
```

找到 `#PasswordAuthentication yes`，去掉前面的 # 注释，确保它是 yes。表示ssh需要密码授权

修改时可以用 `/`进行搜索 `/PasswordAuthentication`,使用`INSERT`进行修改

![image](2.png)

#### 2. 启动SSH服务

```
# 启动服务
sudo service ssh start
# 设置为开机自启（如果是 WSL 2，可能需要额外配置，但手动启动也方便）
sudo systemctl enable ssh
```

` sudo systemctl status ssh`查看ssh的状态

![image](3.png)

`sudo ss -tlnp | grep :22` 查看端口监听状态

> sudo netstat -tlnp | grep :22 中的`netstat` 命令在某些现代 Linux 发行版中默认不再预装，它被更先进的 `ss` 替代了

![image](4.png)





在window的cmd窗口进行文件上传

`scp -r dist路径 wsl用户名@地址:文件夹路径`

```
scp -r D:\a\alumni-direct\alumni-direct-ui\dist chinese@172.31.19.238:~/projects/frontend
```

![image](5.png)

## 第二步：打包并上传项目

### 1. 打包前端项目 (Vue)

1. **在本地构建项目**：
   进入Vue 项目根目录，执行构建命令：

   ```bash
   npm run build
   ```

   构建完成后，会在项目根目录生成一个 `dist` 文件夹，里面包含了所有生产环境的静态文件。

### 2. 在本地打包后端项目(Springboot)：
在 IDEA 或终端中，进入后端项目根目录，执行 Maven 打包命令：

对于common模块需要用 `mvn clean install -DskipTests`跳过`@SpringBootTest`测试并打包安装到本地仓库，后续才能被service模块引用

```
mvn clean package -DskipTests
```

成功后，会在 `target/` 目录下生成一个 `.jar` 文件，比如 `myapp-0.0.1-SNAPSHOT.jar`。

> 值得注意的是打包的jar时需要用到spring-boot-maven-plugin插件，这个jar会**包含正确的启动入口信息**。没有该插件打包出来的jar包没有包含入口信息，运行时会出现 `no main manifest attribute`
>
> ```html
> <build>
>      <plugins>
>          <plugin>
>              <groupId>org.springframework.boot</groupId>
>              <artifactId>spring-boot-maven-plugin</artifactId>
>              <version>2.7.0</version> <!-- 版本号通常由 parent 管理，无需指定 -->
>              <executions>
>                  <execution>
>                      <goals>
>                          <goal>repackage</goal>
>                      </goals>
>                  </execution>
>              </executions>
>          </plugin>
>      </plugins>
>  </build>
> ```
>
> 

### 3. 上传 JAR 包到Linux

ubantu在用户主目录（`~`）创建项目文件夹

```
mkdir -p ~/projects/frontend 
mkdir -p ~/projects/backend 
```

在windows cmd运行命令进行文件上传

```
scp -r D:\a\alumni-direct\alumni-ui\dist chinese@172.31.19.238:~/projects/frontend
scp -r D:\a\alumni-direct\alumni-direct-service\target\alumni-direct-service-0.0.1-SNAPSHOT.jar chinese@172.31.19.238:~/projects/backend
```

## 第三步：配置 Nginx运行前端项目

这是连接前后端的关键。Nginx 需要做两件事：一是找到前端页面，二是把 API 请求转交给后端 Java 程序。

### 1. 编辑 Nginx 配置文件：

```
sudo vim /etc/nginx/sites-available/default
```

将文件内容修改或替换为以下配置。**将路径和域名替换**。

```
server {
    listen 80;                          # 监听 80 端口
    server_name your-domain.com;        # 替换为域名或服务器 IP

    # 前端静态文件配置
    location / {
        root /home/your-username/projects/frontend/dist; # 替换 dist 实际路径
        index index.html;
        try_files $uri $uri/ /index.html; # 解决 Vue Router 的 history 模式刷新 404 问题[reference:18]
    }

    # 后端 API 反向代理配置
    location /api/ {                    # 前端请求的 API 前缀
        proxy_pass http://127.0.0.1:8080/; # 假设后端运行在 8080 端口[reference:19]
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

### 2. 测试并重载 Nginx：

```bash
# 测试配置文件语法是否正确[reference:20]
sudo nginx -t      
语法正确输出：
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok                       
nginx: configuration file /etc/nginx/nginx.conf test is successful 
# 如果显示 successful，则重载 Nginx 使配置生效[reference:21]
sudo systemctl reload nginx
```

如果此时访问，会发现500错误

![image](7.png)

### 3. 使Nginx 能读取/dist

- **Nginx无法读取到/dist原因**

Nginx 默认以 `www-data` 用户和用户组运行。而前端文件 (`dist`) 位于用户 `chinese` 的主目录下，默认权限是 `750` (即 `rwxr-x---`)，这意味着 `www-data` 用户**无法进入** `/home/chinese/` 目录，更别说读取其子目录下的文件了

> `www-data` 是 **Ubuntu / Debian 系统中，Web 服务器默认使用的系统用户**。当安装 Nginx 或 Apache 时，它们的工作进程会以这个用户的身份运行。

- **转移/dist文件夹到nginx的存放文件目录下**

```bash
sudo mv ~/projects/frontend /var/www/html/alumni-frontend 
```

- **重新配置并重启**

重新配置nginx文件： root /var/www/html/alumni-frontend/dist;

重启nginx 后在windows访问  `172.31.19.238:80`即可访问

![image](6.png)

## 第四步：数据库脚本运行

这里可以通过可视化工具 `datagrip`或者 `navicat`又或者 `idea`连接数据库，从而建好库表之类的。但本次采用导入sql脚本到ubantu直接去执行脚本

### 1. 导出SQL脚本

使用idea导出对应的sql脚本，点击库，选择 **export with mysqldump**

可以在资源管理器找到mysqld.exe后查找文件地址从而找到本地Mysql的安装目录

![image](11.png)

`Out path`导出路径最好写上文件名，不然可能会显示

`mysqldump: Can't create/write to file 'C:\Users\chinese' (OS errno 13 - Permission denied)`这样的错误

![image](12.png)

上传文件后 `scp C:\Users\chinese\schema.sql chinese@172.31.19.238:~/projects/sql/`

### 2. 执行SQL脚本

登录Mysql

```bash
sudo mysql -u root -p
```

修改密码

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '123456'; 
```

导出的sql文件是没有建库语句的，在执行脚本前要先建库

```
CREATE DATABASE IF NOT EXISTS alumni_direct;
USE alumni_direct;
```

运行sql

```
SOURCE /home/chinese/projects/sql/schema.sql;
```

## 第五步：后端项目部署(SpringBoot)

一个nginx只能部署一个前端项目？

### 1. 基于命令启动后端项目

```bash
java -jar alumni-direct-service-0.0.1-SNAPSHOT.jar  \
  --spring.datasource.url=jdbc:mysql://localhost:3306/yourdb \
  --spring.datasource.username=root \
  --spring.datasource.password=密码 \
  --spring.redis.host=localhost \
  --spring.redis.port=6379
```

不过由于当前项目本身拥有配置文件 `application-local.yml` 且配置项较多，采用基于配置文件运行

### 2. ⭐基于配置文件运行

```bash
java -jar alumni-direct-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
```

> - `application.yml` (基础公共配置)
> - `application-local.yml`：本地环境
> - `application-dev.yml`：开发环境
> - `application-test.yml`：测试环境
> - `application-staging`：灰度环境 
> - `application-prod.yml`:生产环境
>
> 在部署时，通过 `--spring.profiles.active` 参数来指定，如 `--spring.profiles.active=test`。这样，Spring Boot 就会加载 `application.yml` 和 `application-test.yml` 中的配置，且**后者会覆盖前者的同名配置**

项目启动成功

![image](16.png)

----

## 管理与监控

### 配置为 systemd 服务

#### systemd服务的优点

systemd 服务就是把一个应用程序（比如 Spring Boot JAR 包）包装成一个可以被操作系统管理的“服务”。可以通过 `systemctl` 命令来控制它的**启动、停止、重启、开机自启**等生命周期

- **开机自启**：服务器重启后，Java 应用会自动启动，无需手动干预。
- **统一管理**：可以像管理 Nginx、MySQL 一样，用 `systemctl start/stop/status` 命令来管理应用。
- **日志集成**：服务的标准输出日志会被 systemd 自动捕获，可以通过 `journalctl` 命令方便地查看。
- **进程守护**：即使 JAR 包意外崩溃，systemd 也可以配置为自动重启（需要额外设置）。

#### 将后端项目配置为Systemd服务

```bash
sudo vim /etc/systemd/system/alumni-backend.service
```

```
[Unit]
Description=Alumni Directory Backend Service
After=network.target mysql.service redis.service
Requires=mysql.service redis.service

[Service]
Type=simple
User=chinese
WorkingDirectory=/home/chinese/projects/backend
ExecStart=/usr/bin/java -Xms256m -Xmx512m -jar /home/chinese/projects/backend/alumni-direct-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

配置项说明：

- `After` 和 `Requires`：声明服务在**网络、MySQL、Redis** 之后启动，并依赖它们（依赖项启动失败则本服务不启动）。
- `User`：指定以哪个系统用户运行，**建议用普通用户名**，而不是 `root`。
- `WorkingDirectory`：JAR 包所在的目录
- `ExecStart`：**启动命令**，等同于手动执行的命令，但更完整。
- `Restart`：`on-failure` 表示如果进程异常退出（非正常退出码），系统会自动重启。
- `RestartSec`：自动重启前的等待时间（秒）。
- `StandardOutput` 和 `StandardError`：将输出交给 `journalctl` 管理。**将服务的标准输出（stdout）和标准错误（stderr）直接交给 systemd 的日志系统（journal）来统一管理**。它们并不会直接写入某个固定的文件，而是将日志内容发送到 systemd 的日志收集服务中。通过 `journalctl` 命令来集中查看。
- `WantedBy=multi-user.target`：系统启动到“多用户模式”时，服务自动启动

#### 启动后端项目的systemd服务

```bash
sudo systemctl daemon-reload  # 重新加载 systemd 配置，让它识别新文件
sudo systemctl start alumni-backend   # 启动
sudo systemctl stop alumni-backend    # 停止
sudo systemctl restart alumni-backend # 重启
sudo systemctl status alumni-backend  # 查看状态和最近日志
sudo systemctl enable alumni-backend  # 开启开机自启
sudo systemctl disable alumni-backend # 关闭开机自启

# 查看完整日志
sudo journalctl -u alumni-backend -f
sudo journalctl -u alumni-backend --no-pager # 一次性输出所有内容到终端，不分页
```

#### journalctl日志能否保证持久化

查看是否存在文件

```bash
ls -l /var/log/journal/            

输出内容：
total 4     # 四个字节                                                                                       
drwxr-sr-x+ 2 root systemd-journal 4096 Jul 19 14:42 735702d0f62642098c9931f52aedd603 
```

**所有systemd服务的日志（alumni-backend、minio、nginx、mysql、redis）都会被统一记录在这个目录下**,无法直接查看文件内容。

#### 将minio也systemd服务化

```bash
sudo vim /etc/systemd/system/minio.service
```

```
[Unit]
Description=MinIO Object Storage Server
After=network.target
Wants=network.target

[Service]
Type=simple
User=chinese
Group=chinese
WorkingDirectory=/home/chinese/projects
ExecStart=/home/chinese/projects/minio server /data/minio
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```
# 重新加载 systemd 配置
sudo systemctl daemon-reload
# 立即启动 MinIO
sudo systemctl start minio
# 设置开机自启
sudo systemctl enable minio
# 查看服务状态
sudo systemctl status minio
sudo journalctl -u minio -f   # 查看实时日志
```

#### ⭐查询所有的systemd服务及运行状态

```bash
systemctl list-units --type=service --all | grep -E 'alumni|minio|nginx|mysql|redis'
```

![image](17.png)

### ⭐后台执行并输出日志文件



日志文件配置，如何一天一个文件



### 包管理

如果mysql\redis\minio\项目 放在各种包里，切换目录很繁琐吧

像/etc 之类的目录有存放规范吗？



### ⭐项目启动和组件启动脚本

```
ps -ef | grep java
```

```
# 停止进程 (将 PID 替换为查到的数字)
kill -9 PID
```

- **将后端配置为系统服务 (Systemd)**：
  使用 `nohup` 管理进程不够优雅。更好的做法是创建一个 Systemd 服务，让系统来管理 Java 进程的启动、停止和开机自启。

  1. 创建一个服务文件：`sudo vim /etc/systemd/system/myapp.service`

  2. 写入以下内容（修改路径）：

     ini

     ```
     [Unit]
     Description=My Spring Boot Application
     After=syslog.target
     
     [Service]
     User=your-username
     ExecStart=/usr/bin/java -jar /home/your-username/projects/backend/myapp.jar
     SuccessExitStatus=143
     
     [Install]
     WantedBy=multi-user.target
     ```

     

  3. 启动并启用服务：

     ```
     sudo systemctl daemon-reload
     sudo systemctl start myapp
     sudo systemctl enable myapp  # 设置开机自启
     ```



### 监控

#### ip地址

`ip addr show eth0 | grep inet`  在WSL中查看ip地址，如果是正常的Linux系统则使用 `ifconfig`

输出：

```bash
inet 172.31.19.238/20 brd 172.31.31.255 scope global eth0                                                   inet6 fe80::215:5dff:fe75:5d60/64 scope link    
```

#### ⭐进程查询

```
ps aux          # 显示所有用户的所有进程，BSD 风格
ps -ef          # 显示所有进程，Unix 风格
ps -ef | grep nginx   # 结合 grep 过滤特定进程
常用组合：

aux：a 所有终端进程，u 用户格式输出，x 包含无终端的进程。

-ef：-e 所有进程，-f 全格式列表。

top   实时监控进程的CPU、内存占用， 按P则按CPU排序，按M则按内存排序，q退出
pgrep nginx           # 返回匹配进程的 PID
pidof nginx           # 返回进程的 PID（精确匹配可执行文件名）
```

#### 查看应用占用的端口号

```
# 查看所有监听中的 TCP 端口，过滤 minio 或 9000
sudo ss -tlnp | grep minio
# 或指定端口
sudo ss -tlnp | grep :9000
```

### ⭐日志文件的快捷命令

查看文件内容 

`tail`

cat





### 文件上传



### 文件管理

```
rm -r 文件夹名  递归删除文件夹
```

文件内的指令

`/`搜索，如何下一个？





### 权限管理

```
-rwxr-xr-x
```

拆成四段：

| 第1位             | 第2-4位    | 第5-7位    | 第8-10位   |
| :---------------- | :--------- | :--------- | :--------- |
| `-` 文件 `d` 目录 | 拥有者权限 | 同组人权限 | 其他人权限 |

x对于文件夹来说是不能cd进去查看文件和里面的文件操作

rwxr-xr-x     表示用户对文件/文件夹的权限，拥有者拥有rwx的权限 读写执行，同组用户拥有读执行 权限...

```
7 = 读(4) + 写(2) + 执行(1) = rwx
5 = 读(4) + 执行(1)         = r-x
0 = 啥都没有                  = ---
```

三个数字分别代表**拥有者、同组、其他人**：

```
chmod 755 文件名
```





## 参考文档

[企业项目常见部署方式全梳理：从安装、启动、托管到发布一次讲清_公司都是怎么项目部署的-CSDN博客](https://blog.csdn.net/m0_57021623/article/details/159247016)
