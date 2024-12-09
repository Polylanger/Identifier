<!--
 * @Author: deep-machine-03 deep-machine-03@gmail.com
 * @Date: 2024-12-09 23:14:04
 * @LastEditors: deep-machine-03 deep-machine-03@gmail.com
 * @LastEditTime: 2024-12-10 03:03:15
 * @FilePath: /Identifier/notes.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->

# 系统部署指南

系统一键式部署docker大致分为6个步骤，在开始之前，需要准备一个linux操作系统作为宿主机
- 1. 在宿主机上创建并激活 Python 环境，安装 Web 应用所需的依赖，确保能够成功运行最简单的登录和注册功能。
- 2. 拉取官方 MySQL 镜像并创建容器 `identifier-mysql`，验证数据库容器是否创建成功，并通过命令行创建数据库和用户。
- 3. 创建一个 Docker 网络 `identifier-net` 使容器能够相互通信。然后在该网络下重新部署 `identifier-mysql` 容器，并设置端口映射 `10088:3306`。通过配置宿主机 Web 应用的数据库地址为 `localhost:10088` 来验证网络和 MySQL 容器是否配置正确。
- 4. 使用官方 Python 基础镜像及项目代码构建新镜像 `web-app`，并进入容器配置所需环境。最后启动应用，通过 `localhost:5000` 验证 Web 应用是否运行正常。
- 5. 将步骤 4 中的操作写入 Dockerfile，以便通过构建并部署 Web 应用容器。配置国内镜像源，并优化 Docker 构建缓存机制，确保只有在 `requirements.txt` 文件变化时才重新安装依赖。将 `web-app` 容器和 `mysql` 容器置于同一子网 `identifier-net` 下，并使用容器名作为数据库主机名（即 `identifier-mysql:3306`）。此时，在宿主机中可以通过 localhost:10089 访问web应用。
- 6. 整理所有容器创建指令，生成 docker-compose.yml，实现一键式构建系统，配置网络、存储，部署应用及依赖服务。

## 使用说明

git 仓库中包含四个带有标签（tag）的提交，分别对应步骤 3/4/5/6。按顺序切换到每个标签的代码，逐步复现实验，最终目标是成功实现一个基于 Flask 框架的 Web 应用，并使用 MySQL 作为数据库。可以利用 Docker Compose 实现一键式部署。

## 1 用 conda 创建 python 环境

> conda create -n identifier python=3.10

> conda activate identifier

## 2 使用 docker 官方的 MySQL 镜像

拉取镜像
> docker pull mysql

创建容器
> docker run --name identifier-mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql

查询 mysql IP 地址
> docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' identifier-mysql

验证数据库容器
> docker exec -it identifier-mysql mysql -uroot -p
```sql
CREATE DATABASE flask_db;
CREATE USER 'flask_user'@'%' IDENTIFIED BY 'flaskpassword';
GRANT ALL PRIVILEGES ON flask_db.* TO 'flask_user'@'%';
FLUSH PRIVILEGES;
```

## 3 配置数据库容器的网络和存储

创建一个 Docker 网络，以便容器之间能够相互通信
> docker network create \
  --driver bridge \
  --subnet 172.22.0.0/16 \
  --gateway 172.22.0.1 \
  identifier-net


重建 identifier-mysql 容器，分配【静态IP地址】
> docker rm -f identifier-mysql
> docker run -d \
  --name identifier-mysql \
  --network identifier-net \
  --ip 172.22.0.2 \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=flask_db \
  -e MYSQL_USER=flask_user \
  -e MYSQL_PASSWORD=password \
  -p 10088:3306 \
  -v ./data:/var/lib/mysql \
  mysql:latest

验证数据库IP
> docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' identifier-mysql

## 4 构建容器

使用官方的 Python 基础镜像 + 工程代码构建一个新镜像，此时不对环境做任何配置
> docker build -t flask-app .

将容器和数据库配置在同一个网络中，创建之后直接进入容器的 bash 
> docker run -it --network identifier-net -p 10089:5000 flask-app bash

以下在容器中执行，配置系统环境：

配置镜像源
```shell
cat <<EOL > /etc/apt/sources.list.d/debian.sources
Types: deb
URIs: https://mirrors.aliyun.com/debian/
Suites: bookworm bookworm-updates
Components: main non-free contrib
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.aliyun.com/debian-security/
Suites: bookworm-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOL
```

安装开发工具和库
```shell
apt update
apt upgrade
apt-get install -y python3-dev default-libmysqlclient-dev build-essential pkg-config
```

安装 python 包
> pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

启动应用
> python app.py

## 5 编写 dockerfile 

见 flask_project/Dockerfile
- 为 apt, pip 配置国内的镜像源
- 利用 docker 的缓存机制，先复制 requirements.txt 文件到容器，这样只有在 requirements.txt 变化时才会重新安装依赖，避免每次修改代码都安装依赖包

构建+部署
> docker build -t flask-app .
> docker run -it --network identifier-net -p 10089:5000 flask-app

## 6 编写 docker-compose.yml

根据 3 中创建网络和mysql容器的步骤，以及 5 中创建 flask-app 的指令，编写 docker-compose.yml。结果见：./docker-compose.yml

启动所有服务
> docker compose up --build -d

关闭所有服务
> docker compose down
