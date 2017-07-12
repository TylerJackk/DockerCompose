FROM daocloud.io/python:3.6
MAINTAINER L "liwep.x@gmail.com"
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
ENTRYPOINT ["python"]
CMD ["hello.py"]