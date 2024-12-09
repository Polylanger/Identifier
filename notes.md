<!--
 * @Author: deep-machine-03 deep-machine-03@gmail.com
 * @Date: 2024-12-09 23:14:04
 * @LastEditors: deep-machine-03 deep-machine-03@gmail.com
 * @LastEditTime: 2024-12-09 23:46:15
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

## 配置数据库网络和存储

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

