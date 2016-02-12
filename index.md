**如果您只需要简单的安装测试，可以先进行简易安装，见[简易安装教程](/easy_install.html)。**

#正式部署说明

首先选择一下安装目录，我选择的是 `/home/` 下，下面的操作不加特殊说明的话，都是相对这个目录的。

如果不是 root 用户，下面的命令需要的自行添加 sudo。

以下均是在 Ubuntu 14.04 64位系统上进行的测试。

##下载源代码

```
git clone https://github.com/QingdaoU/OnlineJudge.git
```

然后运行 `cp OnlineJudge/oj/custom_setting.example.py OnlineJudge/oj/custom_setting.py`，然后修改里面`SECRET_KEY`为一个随机值，其余的选项可以以后再修改。

##安装 Docker 和 docker-compose

因为国内特殊网络环境，Docker 的安装和使用并不方便，我们使用 DaoCloud 的安装镜像。但是有时候也会出现添加 key 失败的问题，这时候可以使用[官方的安装方案](https://docs.docker.com/engine/installation/linux/ubuntulinux/)。

```
curl -sSL https://get.daocloud.io/docker | sh
```

安装 docker-compose 

```
pip install docker-compose
```

##build 需要的镜像

以下操作可能需要很长时间，主要取决于您的网络状况，如果卡在 pull 镜像的过程中，可以尝试设置代理或者使用 [daocloud 的镜像](https://dashboard.daocloud.io/mirror)。

build 镜像需要 Dockerfile，有两个，一个是 `oj_web_server`，另一个是判题用 `judger`。

```
docker build -t oj_web_server /home/OnlineJudge/dockerfiles/oj_web_server/
docker build -t judger /home/OnlineJudge/dockerfiles/judger/
```

我们还需要 MySQL 和 Redis 的镜像，直接 pull 下来就可以。

```
docker pull mysql
docker pull redis
```

**注意: 在服务器上运行 MySQL 至少需要1G内存，否则很容易出现异常退出的问题。**

运行 `docker images` 就能看到所有的镜像了。

```
root@iZ62amcgrs2Z:/home# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
judger              latest              4253102444bd        3 minutes ago       969.5 MB
oj_web_server       latest              007d1f706df0        9 minutes ago       787.8 MB
redis               latest              26380e1ca356        13 days ago         151.3 MB
mysql               latest              a917fce37db8        2 weeks ago         360.3 MB
python              2.7                 31093b2dabe2        2 weeks ago         676.1 MB
ubuntu              14.04               3876b81b5a81        3 weeks ago         187.9 MB
```

##配置目录映射

docker 在运行的时候，如果没有配置目录映射，数据都是保存在 docker 容器里面的，如果 docker 容器被删除，数据就会丢失，所以需要将容器内的数据映射到服务器上存储。

我们使用下面的文件夹进行映射

* `/home/data/mysql` MySQL 的数据文件
* `/home/data/redis` Redis 的持久化文件
* `/home/test_case` 上传的测试用例
* `/home/log` web 日志
* `/home/upload` 上传的图片等

请自行创建相关的文件夹。然后 `chown -R nobody:nogroup /home/test_case /home/log /home/upload`

##配置 docker-compose

Docker Compose是在使用Docker容器部署分布式应用时的工具，可以定义哪个容器运行哪个应用。使用Compose，你只需定义一个多容器应用的yml文件，然后使用一条命令即可部署运行所有容器。

下面是一个 `docker-compose.yml` 的例子
先解释下部分参数

* `image` 使用的镜像
* `volumes` 目录映射，将冒号前面的服务器上的映射到冒号后面容器内的路径
* `env` 容器内环境变量
* `link` 两个容器的连接，进行互相访问
* `ports` 端口映射

```
redis:
    image: redis
    volumes:
        - /home/data/redis:/data
mysql:
    image: mysql
    volumes:
        - /home/data/mysql:/var/lib/mysql
    environment:
        - MYSQL_ROOT_PASSWORD=PASSWORD
oj_web_server:
    image: oj_web_server
    volumes:
        - /home/OnlineJudge:/code
        - /home/test_case:/code/test_case
        - /home/upload:/code/upload
        - /home/log:/code/log
    links:
        - redis:redis
        - mysql:mysql
    ports:
        - "127.0.0.1:8080:8080"
    environment:
        - oj_env=server
        - smtp_password=PASSWORD
        - MYSQL_ENV_MYSQL_USER=root
```

如果你配置的映射目录或者端口有变化，修改对应的行就行。

将上面的内容保存在 `docker-compose.yml` 中。

##创建数据库

运行 `docker-compose up -d mysql` 就可以启动数据库，然后 `docker ps -a` 

```
root@iZ62amcgrs2Z:/home# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
46ca3bb58134        mysql               "/entrypoint.sh mysql"   About a minute ago   Up About a minute   3306/tcp            home_mysql_1

```

看到 mysql 的 CONTAINER ID，这个例子中就是 `46ca3bb58134`，运行`docker exec -i -t {CONTAINER_ID} /bin/bash`

然后 `mysql -u root -p`，输入 `docker-compose.yml` 中的的密码，然后执行下面两条 sql 语句，创建 oj 和 submission 数据库。

```
create database oj default character set utf8;
create database oj_submission default character set utf8;
```
然后 `exit` 退出 docker 容器。

##启动所有的容器
运行 `docker-compose up -d`

`docker ps -a`可以看到所有的容器的运行状态。

##创建数据表

需要进入 `oj_web_server` 容器，`docker ps -a` 可以看到这个容器是 running，然后复制  CONTAINER ID

运行

```
docker exec -i -t {CONTAINER ID} /bin/bash
```

然后容器中运行 

```
python manage.py migrate
python manage.py migrate --database=submission
python manage.py initadmin
```
注意最后一个命令是创建了超级管理员用户，密码是随机生成的，请妥善保管。

运行 `python tools/release_static.py`，会生成压缩版的js，可能会比较慢，耐心等候。

然后 `exit` 退出 docker 容器。

##配置nginx

服务器上安装nginx,然后创建下面的配置文件，比如 `/etc/nginx/conf.d/oj.conf`

```
server {
    listen 80;

    # 如果没有域名,就写你当前的ip或者使用 Host 绑定
    server_name xxxxxxx.com;
    location /static/upload {
        expires max;
        alias /home/upload;
    }
    location /static {
        etag off;
        expires max;
        alias /home/OnlineJudge/static/release;
    }
    location / {
        client_max_body_size 100m;
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
    }
}

```

`nginx -t` 可以测试配置文件是否正确,然后 `service nginx restart` 即可。

##server安装成功
如果可以访问了,请在 `/login/` 处登录 root 用户,然后访问 `/admin/` 添加一道题目试试吧。但是现在还不能判题。

##配置判题
运行

```
docker run -v /home/OnlineJudge:/var/judger/code  -v /home/test_case:/var/judger/test_case -v /home/log:/var/judger/code/judge/log -e rpc_token={rpc_token} -p 0.0.0.0:8085:8080 -d judger
```

其中 `{rpc_token}` 自行修改，后面会用到。

`docker ps -a`可以看到 judger 容器的 CONTAINER ID，复制出来，运行`docker inspect {CONTAINER ID}`，里面有一行是 IPAddress，记住这个 ip。

在 admin 界面 `判题服务器` tab 里面，填写 ip、token 和端口8080，创建一个判题服务器就好了。

#高级配置
##多台服务器判题
todo
##服务器之间通过 rsync 同步测试用例
todo

