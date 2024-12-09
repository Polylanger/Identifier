# 使用官方的 Python 基础镜像
FROM python:3.9-slim

# 设置工作目录
WORKDIR /app

# 复制应用的其他代码
# 只有应用代码发生变化时才会重新复制
COPY . /app

# 暴露 Flask 应用运行的端口
EXPOSE 5000

# 设置 Flask 环境变量
ENV FLASK_APP=app.py
ENV FLASK_ENV=production

# 启动 Flask 应用（这个只是测试，暂时不启动应用）
CMD ["bash"]
