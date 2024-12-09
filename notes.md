<!--
 * @Author: deep-machine-03 deep-machine-03@gmail.com
 * @Date: 2024-12-09 23:14:04
 * @LastEditors: deep-machine-03 deep-machine-03@gmail.com
 * @LastEditTime: 2024-12-10 02:51:00
 * @FilePath: /Identifier/notes.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->

# 系统部署指南

系统一键式部署docker大致分为6个步骤，在开始之前，需要准备一个linux操作系统作为宿主机
- 1. 用conda创建python环境，并在宿主机上配置web应用所需环境，直到成功运行最简单的登录、注册
- 2. 用docker创建一个mysql容器 identifier-mysql，进入容器的命令行，通过创建database/user验证容器是否创建成功
- 3. 创建一个 Docker 网络 identifier-net，以便容器之间能够相互通信。然后在网络中重新部署一个 identifier-mysql，并且设置端口映射 10088:3306。通过将宿主机中web应用的数据库地址设置为 localhost:10088 验证网络、MySQL容器是否配置成功
- 4. 使用官方的 Python 基础镜像+工程代码构建一个新镜像 web-app，并且进入容器的命令行，在容器中重新配置一遍 web 应用所需的环境。最后启动应用，在宿主机中可以通过 localhost:5000 访问成功
- 5. 把步骤4中的指令整理成 dockerfile，构建并且部署到 identifier-net 中，并且设置端口映射 10089:5000。此时mysql和web-app位于同一子网下，可以使用mysql容器名作为数据库主机名，即：web应用中的数据库地址配置为 identifier-mysql:3306。此时，在宿主机中可以通过 localhost:10089 访问web应用。
- 6. 整理所有容器创建指令，生成 docker-compose.yml，实现一键式构建系统，配置网络、存储，部署应用及依赖服务。

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
