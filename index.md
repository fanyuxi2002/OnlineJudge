#安装说明
首先选择一下安装目录，我选择的是`/home/`下，下面的操作不加特殊说明的话，都是相对这个目录的。

如果不是 root 用户，下面的命令需要的自行添加 sudo。

##下载源代码

```
git clone https://github.com/QingdaoU/OnlineJudge.git
```

##安装 Docker
因为国内特殊网络环境，Docker 的安装和使用并不方便，我们使用 DaoCloud 的安装镜像。

```
curl -sSL https://get.daocloud.io/docker | sh
```

## build 需要的镜像
安装后运行 `docker` ，如果看到一堆命令参数说明安装成功，接下来 build docker 镜像。

有两个，一个是 `oj_web_server`，另一个是判题用 `judger`。

```
docker build -t oj_web_server /home/OnlineJudge/dockerfiles/oj_web_server/
docker build -t judger /home/OnlineJudge/dockerfiles/judger/
```

我们还需要 MySQL 和 Redis 的镜像，直接 pull 下来就可以。

```
docker pull mysql
docker pull redis
```

运行 `docker images` 就能看到所有的镜像了。

```
oj_web_server       latest              e7b0b79b0866        28 minutes ago      951.9 MB
<none>              <none>              26486c6d3965        37 minutes ago      725.6 MB
python              2.7                 5b60519c8d3b        12 hours ago        676 MB
ubuntu              14.04               c4bea91afef3        3 days ago          187.9 MB
redis               latest              0c4334bed751        9 days ago          151.3 MB
mysql               latest              ea0aca21950d        3 weeks ago         360.3 MB
```


## 启动数据库
首先要启动 MySQL 和 Redis 的 container, 分别命名为 mysql 和 redis。因为 MySQL 和 Redis 的数据文件需要持久化, 可以使用 `-v` 参数进行目录映射。

我们使用/home/data/mysql 和 /home/data/redis 来存放数据文件, 先创建这两个文件夹

```
mkdir -p /home/data/mysql /home/data/redis
```

启动 MySQL 的命令

```
docker run --name mysql -v /home/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD={PASSWORD} -d mysql
```

里面的 `{PASSWORD}` 是数据库密码，请自行修改。

启动 Redis 的命令

```
docker run --name redis -v /home/data/redis:/data -d redis
```

然后 `docker ps -a` 就能看到这两个运行的 container 了

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
8a4e0f2511ec        redis               "/entrypoint.sh redis"   3 seconds ago       Up 2 seconds        6379/tcp            redis
968fd9d04a3d        mysql               "/entrypoint.sh mysql"   3 minutes ago       Up 3 minutes        3306/tcp            mysql
```

##创建数据库
接下来创建数据库，上一步可以看到 mysql 的 container id，运行`docker exec -i -t {CONTAINER ID} /bin/bash`

然后 `mysql -u root -p`，输入刚才的密码，然后执行下面两条 sql 语句，创建 oj 和 submission 数据库。

```
create database oj default character set utf8;
create database oj_submission default character set utf8;
```
然后 exit 退出 docker 容器

##启动 oj_web_server

启动之前需要创建 `test_case`、`log` 和 `upload` 三个文件夹。

```
mkdir -p /home/test_case /home/log /home/upload
```

把这几个文件夹修改为 nobody 所属的.

```
chown -R nobody:nogroup /home/test_case /home/log /home/upload
```

启动 `oj_web_server` 的命令如下

```
docker run --name oj_web_server -e oj_env=server -e smtp_password={SMTP_PASSWORD} -v /home/OnlineJudge:/code  -v /home/log:/code/log -v /home/upload:/code/upload -v /home/test_case:/code/test_case -d -p 127.0.0.1:8080:8080 --link=redis --link=mysql oj_web_server
```

注意 `{SMTP_PASSWORD}` 是发送密码找回邮件使用的，SMTP 设置在 `oj/server_settings.py`末尾可以看到。

##创建数据表

`docker ps -a` 可以看到这个容器是 running，然后复制 container id

运行
```
docker exec -i -t {CONTAINER ID} /bin/bash
```

然后容器中运行 

```
python manage.py migrate
python manage.py migrate --database=submission
```

执行完后运行 `python manage.py shell`，输入下面的命令创建管理员

```
>>> from account.models import User
>>> user = User.objects.create(username="root", admin_type=2)
>>> user.set_password("root")
>>> user.save()
```

这样就创建了账号和密码都是root 超级管理员。

然后运行`python tools/release_static.py`，会生成压缩版的js，可能会比较慢，耐心等候。

## 配置nginx

服务器上安装nginx,然后创建下面的配置文件

```
server {
    listen 80;

    # 如果没有域名,就写你当前的ip
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

## server安装成功
如果可以访问了,请在 `/login/` 处登录 root 用户,然后访问 `/admin/` 添加一道题目试试吧。但是现在还不能判题。

## 配置判题
运行

```
docker run --privileged=true -v /home/OnlineJudge:/var/judger/code  -v /home/test_case:/var/judger/test_case -v /home/log:/var/judger/code/judge/log -e rpc_token={rpc_token} -p 0.0.0.0:8085:8080 -d judger
```

其中 `{rpc_token}` 自行修改，后面会用到。

`docker ps -a`可以看到 judger 容器的 container id，复制出来，运行`docker inspect {CONTAINER ID}`，里面有一行是 IPAddress，记住这个 ip。

去后台左边的 `tab/判题服务器` ，填写 ip，token 和端口8080，创建一个判题服务器就好了。

#高级配置
todo
