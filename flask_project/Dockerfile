# 使用官方的 Python 基础镜像
FROM python:3.9-slim

# 设置工作目录
WORKDIR /app

# 配置阿里云镜像源（Debian）
RUN echo "Types: deb\n\
URIs: https://mirrors.aliyun.com/debian/\n\
Suites: bookworm bookworm-updates\n\
Components: main non-free contrib\n\
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg\n\
\n\
Types: deb\n\
URIs: https://mirrors.aliyun.com/debian-security/\n\
Suites: bookworm-security\n\
Components: main\n\
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg" > /etc/apt/sources.list.d/debian.sources

RUN apt-get update
RUN apt-get upgrade -y
RUN apt-get install -y python3-dev default-libmysqlclient-dev build-essential pkg-config

# 复制 requirements.txt 文件到容器
# 这样只有在 requirements.txt 变化时才会重新安装依赖，避免每次都安装
COPY requirements.txt /app/

# 安装 Python 依赖
RUN pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/

# 复制应用的其他代码
# 只有应用代码发生变化时才会重新复制
COPY . /app

# 暴露 Flask 应用运行的端口
EXPOSE 5000

# 设置 Flask 环境变量
ENV FLASK_APP=app.py
ENV FLASK_ENV=production

# 启动 Flask 应用（这里只是测试，暂时不启动应用）
CMD ["python", "app.py"]
