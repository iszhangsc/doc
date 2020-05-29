# Docker安装常用软件



## Docke环境安装

```shell
[root@VM_0_4_centos /]# yum install docker
# 没有就创建，添加国内镜像加速
[root@VM_0_4_centos /]# vim /etc/docker/daemon.json
{
	"registry-mirrors": [
        "https://kfwkfulq.mirror.aliyuncs.com",
        "https://2lqq34jg.mirror.aliyuncs.com",
        "https://pee6w651.mirror.aliyuncs.com",
        "https://registry.docker-cn.com",
        "http://hub-mirror.c.163.com"
	],
	"dns": [
		"8.8.8.8",
		"8.8.4.4"
	]
}
[root@VM_0_4_centos /]# systemctl daemon-reload
[root@VM_0_4_centos /]# systemctl restart docker
```





## Redis环境安装

> 拉取镜像

```shell
sudo docker pull redis
```

> 创建容器

1. **挂载文件**

```shell
# 创建数据存放目录和配置文件目录
sudo mkdir -p /data/docker/redis/{conf,data}
```

2. **获取配置文件模板**

```shell
wget https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf -O /data/docker/redis/conf/redis.conf
# 下面三个 sed 主要是替换日志配置、密码配置、开启持久化
sed -i 's/logfile ""/logfile "access.log"/' conf/redis.conf
sed -i 's/# requirepass foobared/requirepass 123456/' conf/redis.conf
sed -i 's/appendonly no/appendonly yes/' conf/redis.conf
```





## ElasticSearch环境安装

> 拉取镜像

```shell
docker pull elasticsearch:7.4.2
docker pull kibana:7.4.2
```

>  创建容器

1. **ElasticSearch挂载文件**

```shell
# 配置文件目录
mkdir -p /data/docker/elasticsearch/config
# 数据文件目录
mkdir -p /data/docker/elasticsearch/data
# 插件文件目录
mkdir -p /data/docker/elasticsearch/plugins
# 设置所有IP都可以访问
echo "http.host: 0.0.0.0" >>/data/docker/elasticsearch/config/elasticsearch.yml
# 外网客户端访问必须开启这个，否则端口扫描不到9300
echo "transport.host: 0.0.0.0" >>/data/docker/elasticsearch/config/elasticsearch.yml
# 修改文件权限--ES需要使用非root用户运行.
chmod 777 -R /data/docker/elasticsearch/*
```

2. **启动命令**

``` shell
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
-v /data/docker/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /data/docker/elasticsearch/data:/usr/share/elasticsearch/data \
-v /data/docker/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.4.2
```

**参数说明**

- `-p 9200:9200` rest api 通信端口
- `-p 9300:9300` 分布式集群下使用的端口
- `-e "discovery.type=single-node"` 单节点运行
- `-e ES_JAVA_OPTS="-Xms512m -Xmx512m"` jvm设置(避免服务器内存太小，导致资源耗尽，卡死，可以根据服务器实际配置做出对应的更改)
- `-v ` 挂载数据存储目录、插件放置目录、配置文件放置目录。以后再宿主机上安装好插件重启即可
- `--restart=always` 开机自启动

3. 安装kibana

``` shell
docker run -it -d -e ELASTICSEARCH_HOSTS=http://34.80.13.149:9200 --name kibana -p 5601:5601 kibana:7.4.2
```





## Nginx环境安装

> 随便启动一个nginx实例，用于将容器中的nginx配置复制到宿主机中。

```shell
docker run -p 8087:80 --name nginx-test -d nginx
# 创建宿主机nginx目录文件
mkdir -p /data/docker/nginx
# 拷贝容器中的nginx配置文件到 /data/docker目录下
docker cp nginx-test:/etc/nginx /data/docker
# 删除测试的容器
docker stop nginx-test
docker rm nginx-test
```

> 修改文件挂载目录

```
cd /data/docker
mv nginx conf
mkdir nginx
mv conf/ nginx/
```

> 创建容器

```shell
docker run -p 80:80 --name nginx \
-v /data/docker/nginx/html:/usr/share/nginx/html \
-v /data/docker/nginx/logs:/var/log/nginx \
-v /data/docker/nginx/conf:/etc/nginx \
-d nginx
```

**参数说明**

- `-p 80:80` 将宿主机中的80端口映射到容器的80端口
- `-v /data/docker/nginx/html:/usr/share/nginx/html ` 映射静态文件放置位置

> PS：如果需要代理静态文件，可以将静态文件直接放置在html下.







## MySQL环境安装

> 拉取镜像

```shell
sudo docker pull mysql:5.7
```

> 创建文件挂载目录

```shell
[root@VM_0_4_centos bin]# mkdir -p /data/docker/mysql/{datadir,conf.d}
```

> 创建MySQL容器

```shell
sudo docker run -d --name=mysql5.7 \
--restart=always -p 3306:3306 \
-v /data/docker/mysql/datadir:/var/lib/mysql \
-v /data/docker/mysql/conf.d:/etc/mysql/conf.d \
-e TZ=Asia/Shanghai \
-e MYSQL_ROOT_PASSWORD=123456 mysql:5.7 \
--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

**参数说明:**

- `--restart=always`	**开机自启**
- `–privileged=true`	**提升容器内权限**
- `-v /data/docker/mysql/datadir:/var/lib/mysql`		**映射数据目录**
- `-v /data/docker/mysql/config/mysqld.cnf:/etc/mysql/mysql.conf.d/mysqld.cnf`**映射配置文件
- `-e TZ=Asia/Shanghai` **指定时区为亚洲/上海(中国时区)**
- `-e MYSQL_ROOT_PASSWORD=123456`	**指定用户密码为123456**
- `--character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci` **设置字符编码****







## SQLServer环境安装

> 拉取镜像(可以选择版本，具体看https://hub.docker.com/)

```shell
sudo docker pull microsoft/mssql-server-linux:2017-latest
```

> 创建容器

```shell
sudo docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=sqlserver123#' \
 -e 'MSSQL_COLLATION=Chinese_PRC_CI_AS' -e 'MSSQL_LCID=2052' \
 -p 1433:1433 --name mssql -v /data/docker/mssql:/var/opt/mssql \
 -d microsoft/mssql-server-linux:2017-latest
```

**参数说明**：

-  **-e 'ACCEPT_EULA=Y'** 设置此参数说明同意 SQL SERVER 使用条款 ,  否则无法使用 
-  **-e 'SA_PASSWORD=密码'** 此处设置 SQL SERVER 数据库 SA 账号的密码 
-  **-e MSSQL_COLLATION=Chinese_PRC_CI_AS** 设置中文排序规则(建议使用中文，否则会出现各种错误)
-  **-e  MSSQL_LCID=2052** 设置为中文(建议使用中文，否则会出现各种错误)
-  **-p 1433:1433** 将宿主机 1433 端口映射到容器的 1433 端口(可以自定义端口)
-  **--name mssql** 设置容器名为 mssql  
-  **-v /data/docker/mssql:/var/opt/mssql** 将宿主机 `/data/docker/mssql` 映射到容器 `/var/opt/mssql` , 方便备份数据



> 还原bak文件

```shell
# 创建容器中放置bak文件的目录
sudo docker exec -it mssql mkdir /var/opt/mssql/backup
# 将宿主机内的 bak 文件放置到容器对应的目录下
sudo docker cp JZFZ_CRM_backup.bak mssql:/var/opt/mssql/backup/
```

