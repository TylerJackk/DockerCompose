FROM daocloud.io/nginx:latest

MAINTAINER L "liwep.x@gmail.com"

# 从配置文件中删除默认配置
RUN rm -v /etc/nginx/nginx.conf

# 从当前目录复制配置文件
ADD nginx.conf /etc/nginx/

# Append "daemon off;" to the beginning of the configuration
# RUN echo "daemon off;" >> /etc/nginx/nginx.conf

# 暴露端口
EXPOSE 80
