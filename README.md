

# DockerCompose
使用Docker-compose快速部署整个项目环境

demo项目采用falsk + MySQL + Nginx

-------

## 目录

- [DockerCompose](#dockercompose)
  - [目录](#%E7%9B%AE%E5%BD%95)
  - [Docker安装(Linux)](#docker%E5%AE%89%E8%A3%85linux)
  - [项目文件结构](#%E9%A1%B9%E7%9B%AE%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84)
  - [Dockerfile](#dockerfile)
    - [falsk](#falsk)
        - [一个最小的falsk应用代码](#%E4%B8%80%E4%B8%AA%E6%9C%80%E5%B0%8F%E7%9A%84falsk%E5%BA%94%E7%94%A8%E4%BB%A3%E7%A0%81)
        - [配置项目依赖环境](#%E9%85%8D%E7%BD%AE%E9%A1%B9%E7%9B%AE%E4%BE%9D%E8%B5%96%E7%8E%AF%E5%A2%83)
        - [Dockerfile编写](#dockerfile%E7%BC%96%E5%86%99)
    - [nginx](#nginx)
        - [Dockerfile编写](#dockerfile%E7%BC%96%E5%86%99-1)
        - [配置nginx](#%E9%85%8D%E7%BD%AEnginx)
        - [配置https](#%E9%85%8D%E7%BD%AEhttps)
    - [MySQL](#mysql)
  - [docker-compose](#docker-compose)
  - [部署](#%E9%83%A8%E7%BD%B2)
  - [整合](#%E6%95%B4%E5%90%88)
  - [TODO](#todo)

-------

## Docker安装(Linux)

更新源

```
$ sudo apt-get update
```

从阿里云镜像安装docker

```
$ curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
```

安装docker-compose

```
$ sudo apt install docker-compose
```

开启docker

```
$ sudo systemctl enable docker
$ sudo systemctl start docker
```

DaoCloud镜像加速

```
$ curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://a2608a7f.m.daocloud.io
```

配置加速器后重启docker

```
$ sudo systemctl restart docker.service
```

输入`docker version`看到如下内容则配置成功
![](https://ws1.sinaimg.cn/large/006tNc79gy1fhh55zrd4rj30g0095dgj.jpg)

-------

## 项目文件结构

```
└── demo_compose                        项目文件夹
    ├── data                            MySQL volume
    ├── docker-compose.yml              docker-compose配置文件
    ├── flask                           flask项目文件夹
    │   ├── Dockerfile                  flask的Dockerfile
    │   ├── hello.py                    flask代码
    │   └── requirements.txt            依赖环境
    └── nginx                           nginx文件夹
        ├── Dockerfile                  nginx的Dockerfile
        └── nginx.conf                  部署时用的nginx配置
```
## Dockerfile
### falsk
##### 一个最小的falsk应用代码
*hello.py*

```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World! Docker!'

if __name__ == '__main__':
	# 配置服务器可外部访问 host='0.0.0.0'
    app.run(host='0.0.0.0')
```

```
$ python hello.py
* Running on http://127.0.0.1:5000/
```
将`127.0.0.1`替换为服务器ip地址就可以在看到flask的输出内容

##### 配置项目依赖环境

*requirements.txt*

```
Flask==0.12.1
```

*如何生成requirements.txt文件*

进入项目开发环境的pip(本机环境、pyenv、virtualenv)
`pip freeze > requirements.txt`

##### Dockerfile编写

在flask项目跟目录下创建`Dockerfile`(无后缀)

*Dockerfile*

```
FROM daocloud.io/python:3.6                         # 指定基础镜像
MAINTAINER L "liwep.x@gmail.com"
COPY . /app
WORKDIR /app
# 从阿里云镜像安装requirements.txt
RUN pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/                         
ENTRYPOINT ["python"]
CMD ["hello.py"]                                    # 运行flask应用
```

> [Dockerfile具体讲解](https://yeasy.gitbooks.io/docker_practice/content/image/build.html)


### nginx

##### Dockerfile编写
同样在nginx文件夹内创建`Dockerfile`

*Dockerfile*

```
FROM daocloud.io/nginx:latest                       # 指定基础镜像

MAINTAINER L "liwep.x@gmail.com"

# 从配置文件中删除默认配置
RUN rm -v /etc/nginx/nginx.conf

# 从当前目录复制配置文件
ADD nginx.conf /etc/nginx/

# 暴露端口
EXPOSE 80

```

##### 配置nginx

根据自己项目需求进行配置，demo里的功能是访问80端口映射为falsk的5000端口

*nginx.conf*

```
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    server {
        listen 80;
        charset utf-8;
        # server_name www.abc.cn abc.cn;    配置域名
        location / {
            #配置flask项目的端口  ip写为falsk是docker-compose的配置
            proxy_pass http://flask:5000/;
            proxy_set_header  X-Real-IP  $remote_addr;
        }
    }
}
```
##### 配置https
需要SSL证书文件 `1_www.domain.com_bundle.crt` 和私钥文件 `2_www.domain.com.key`

修改*nginx.conf*为

```
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    server {

        listen 80;
        listen 443;   # ssl端口
        charset utf-8;
        server_name www.domain.com; #填写绑定证书的域名
        # ssl配置
        ssl on;
        ssl_certificate 1_www.domain.com_bundle.crt;
        ssl_certificate_key 2_www.domain.com.key;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2; #按照这个协议配置
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;#按照这个套件配置
        ssl_prefer_server_ciphers on;
        location / {
            proxy_pass http://flask:5000/;
            proxy_set_header  X-Real-IP  $remote_addr;
        }
    }
}
```
同时修改*Dockerfile*

```
FROM daocloud.io/nginx:latest

MAINTAINER L "782096772@qq.com"

# 从配置文件中删除默认配置
RUN rm -v /etc/nginx/nginx.conf

# 从当前目录复制配置文件
ADD nginx.conf /etc/nginx/

# 复制ssl证书到nginx路径下
ADD 1_www.domain.com_bundle.crt /etc/nginx/
ADD 2_www.domain.com.key /etc/nginx/

# 暴露端口
# EXPOSE 80

```

修改 *docker-compose.yml* 中的nginx的ports为

```
ports:
    - "443:443"
```

配置全站加密，http自动跳转https
在http的server里增加`rewrite ^(.*) https://$host$1 permanent;`

### MySQL

配置在docker-compose中

-------


## docker-compose

通过一个单独的 docker-compose.yml 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

Compose 中有两个重要的概念：

* 服务（service）：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
* 项目(project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

    *docker-compose.yml*
    
```
version: '2'
services:
    nginx:
      build: ./nginx
      <!--与flask容器关联后可在nginx配置中将ip写为falsk-->
      links:
        - flask
      ports:
        - "80:80"
      restart: always
 
    mysql:
      image: daocloud.io/mysql:5.6
      <!--utf-8-->
      command: mysqld --character-set-server=utf8 --collation-server=utf8_unicode_ci --init-connect='SET NAMES UTF8;' --innodb-flush-log-at-trx-commit=0
      environment:
        <!--配置默认用户root的密码-->
        - MYSQL_ROOT_PASSWORD=root123
        <!--初始时自动创建数据库test-->
        - MYSQL_DATABASE=test
      volumes:
        <!--数据卷所挂载路径设置-->
        - /data/mysql:/var/lib/mysql
      ports:
        - "3306:3306"
        
    flask:
      build: ./flask
      ports:
        - "5000:5000"
      volumes:
        - .:/code
      expose:
        - "5000"
    
```


[更多MYSQL配置项](https://github.com/mysql/mysql-docker)
[yml模板配置项](https://yeasy.gitbooks.io/docker_practice/compose/yaml_file.html)

-------

## 部署

在服务器上cd到和`docker-compose.yml`同级目录下

```
$ docker-compose build
$ docker-compose up
```
也可`docker-compose up -d`在后台执行
有如下信息则build成功

![](https://ws2.sinaimg.cn/large/006tNc79gy1fhh5ecabuij30m309l75x.jpg)

up后访问服务器ip地址

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fhlrbt5amgj30am04fjrd.jpg)

不加5000端口可以看到输出信息则flask和nginx部署成功

demo项目没有用到MySQL，可以自行连接数据库确认是否创建成功
*username : root 
password : root123
port : 3306
dbname : test*

-------
## 整合
把初期环境搭建的命令整合为一个shell脚本,就可以通过一个命令安装初期需要的各种环境了

在目录创建setup.sh

*seup.sh*

```
#!/bin/bash
sudo apt-get update
# 从阿里源安装docker
curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -
# 安装deocker-compose
sudo apt install docker-compose
# 开启docker
sudo systemctl enable docker
sudo systemctl start docker
# 设置docker的DaoCloude镜像加速
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://a2608a7f.m.daocloud.io
sudo systemctl restart docker.service

# APT 自动安装 Python 相关的依赖包，如需其他依赖包在此添加
apt-get install -y python \
                       python-dev \
                       python-pip  \
    && apt-get clean \
    && apt-get autoclean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* 
```
赋予权限

`$ chmod +x setup.sh`

执行

`$ yes | ./setup.sh`


最终文件路径

```
.
|-- demo_compose
|   |-- data
|   |-- docker-compose.yml
|   |-- flask
|   |   |-- Dockerfile
|   |   |-- hello.py
|   |   `-- requirements.txt
|   `-- nginx
|       |-- Dockerfile
|       `-- nginx.conf
`-- setup.sh

```


## TODO
* [x] Nginx https配置

