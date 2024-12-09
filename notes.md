<!--
 * @Author: deep-machine-03 deep-machine-03@gmail.com
 * @Date: 2024-12-09 23:14:04
 * @LastEditors: deep-machine-03 deep-machine-03@gmail.com
 * @LastEditTime: 2024-12-10 01:19:39
 * @FilePath: /Identifier/notes.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->

## 用 conda 创建 python 环境

> conda create -n identifier python=3.10

> conda activate identifier

## 使用 docker 官方的 MySQL 镜像

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

## 配置数据库容器的网络和存储

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

## 构建容器

使用官方的 Python 基础镜像 + 工程代码构建一个新镜像
> docker build -t flask-app .

将容器和数据库配置在同一个网络中，创建之后直接进入容器的 bash 
> docker run -it --network identifier-net -p 10089:5000 flask-app bash

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

