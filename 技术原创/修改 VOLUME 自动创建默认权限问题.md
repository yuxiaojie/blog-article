### VOLUME 默认权限问题

Dockerfile 中，如果需要将数据持久化到本地，则需要以 VOLUME 关键字挂载一个目录，然后这个目录下的文件操作，都会持久化到本地中

如果挂载的目录在本地中是不存在的，则 Docker 会自动帮助创建一个目录，而此时是使用 root 的身份完成这个动作的，所以创建的目录权限属于 root 用户 及 root 用户组，然后 Docker 中的执行服务如果是用其他用户身份执行的，那么对这个目录进行写入等操作的时候，就会提示用户权限不足

此时不改动 dockerfile 要快速解决的话

1. 可以直接将本地映射的目录通过 `chmod -R 777 your/path` 命令开放所有权限，这样操作并不安全，因为开放所有权限意味着任意用户都可以对这个目录进行任意的操作，可能会有误操作等等的安全问题

2. 找到这个容器对应的用户名，然后将目录通过命令 `chown`  把所有权转给找到的用户，这样对比上面的操作没有问题，但是操作会麻烦不少

   ```bash
   # 找到镜像的名称
   docker images
   
   # 通过交互模式在镜像中执行 bash 命令
   docker run -it api_demo_api bash
   
   # 查看镜像中的用户
   root@20fcb7c63dee:/home/www# cat /etc/passwd
   
   ......
   www:x:101:65534::/home/www:/bin/false
   
   # 对本地映射的目录，授权为镜像中查看到的用户
   chown -R 101 /data/apiDemo
   ```

   但是有个问题就是，这些操作都需要用户使用时进行，需要培训或提前文档告知，如果能够在编写 dockerfile 的时候直接解决这个问题，那无疑是最好的方式



### 为 flask 应用编写 Dockerfile

假定已经有了一个 python 的 flask 的应用，服务的项目结构如下：

```
./
├── Dockerfile   
├── README.md
├── docker-compose.yml
├── entrypoint.sh
└── src
    ├── app
    ├── gun.py
    ├── requirements.txt
    └── server.py
```

服务基于 Python 3.6.6 镜像进行编写，dockerfile 如下：

```dockerfile
FROM python:3.6.6

# 安装依赖环境，单独 copy 一个文件，如果不改动这个文件，这一层产生的镜像都可以命中缓存
# 安装库比较费时间
COPY ./src/requirements.txt /home/app/requirements.txt
RUN pip install --upgrade pip && pip install -r /home/app/requirements.txt

# 通过 ENTRYPOINT 关键字，在镜像服务启动之前执行一个脚本
COPY ./entrypoint.sh /usr/local/bin/
RUN chmod 755 /usr/local/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]

# 创建一个镜像内的用户 www 及用户组 www，并且将用户 www 配置进 www 用户组中
RUN addgroup www && adduser --system www && adduser www www

# 将整个项目源码都 copy 进工作目录下
WORKDIR /home/www
COPY ./src /home/www

# 在镜像中创建目录并且进行授权，并且将日志目录写入镜像的环境变量中后续使用
ENV LOG_DIR /log
RUN mkdir -p "$LOG_DIR" && chown -R www:www "$LOG_DIR"
VOLUME /log
```



### 在 entrypoint.sh 对目录进行授权

```shell
#!/bin/sh

# 如果执行用户是 root 则进行授权操作
if [ "$(id -u)" = '0' ]; then
    # !表示对结果取反，表示找出所有用户不是 www 的文件，最后的 + 表示将所有找出的文件一起执行 chown 命令
    find "$LOG_DIR" \! -user www -exec chown www '{}' +
fi

# 执行 docker 传递进行来的命令，如果没有这行，docker执行完 entrypoint.sh 就会直接退出
exec "$@"
```

> 注意: 
>
> 运行程序是用创建的用户进行运行，但是 Dockerfile 及 entrypoint.sh 脚本执行的过程，全部都是通过 root 用户来进行执行的，不然也没有权限进行授权，所以这里都没有使用 Dockerfile 的 `USER` 命令



### 使用 docker-compose 快速启动

```yml
version: '3.7'

services:
  api:
    build: ./
    command: gunicorn -c gun.py server:app
    ports:
      - 12345:12345
    environment:
      - SERVER_ENV=$SERVER_ENV
      - HOST_ID=$HOST_ID
    volumes:
      - "/data/apiDemo:/log"
```

> 注意，用户是通过 root 权限运行的，所以执行命令需要以其他用户权限执行的操作需要由执行命令来完成，我们这里使用 gunicorn 来作为容器启动服务的，所以就在 gunicorn 的配置文件 gun.py中进行配置，配置文件参考如下：

```python
import multiprocessing
import os

from app.config import LOG_PATH

# 指定运行用户身份
user = 'www'
group = 'www'

debug = False
deamon = False
loglevel = 'info'
bind = '0.0.0.0:12345'
max_requests = 50000
worker_connections = 50000

x_forwarded_for_header = "X-Real-IP"

# 启动的进程数
workers = multiprocessing.cpu_count()
# workers = 3
worker_class = "gevent"

# 日志写入目录配置为授权的目录日志目录
accesslog = os.path.join(LOG_PATH, 'access.log')
access_log_format = '%({X-Real-IP}i)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s"'
errorlog = os.path.join(LOG_PATH, 'error.log')

timeout = 60

```



### 运行查看效果



```shell
# 执行命令进行打包和运行
docker-compose build
docker-compose up

# 查看写入的日志文件
ll /data/
drwxr-xr-x  2               101 ssh_keys        4096 12月 17 15:09 apiDemo

ll /data/apiDemo/
-rw-r--r-- 1 101 ssh_keys  82 12月 17 15:08 access.log
-rw-r--r-- 1 101 ssh_keys  64 12月 17 15:08 api.log
-rw-r--r-- 1 101 ssh_keys 914 12月 17 16:12 error.log
```



完整项目的配置可以参考： [api_demo](https://github.com/yuxiaojie/api_demo)


