1、项目总览：50-server.cnf 是数据库 db1 的配置文件；

docker-compose.yml 是 Docker compose 配置文件，定义了服务、容器及容器行为；

MDockerfile 用于构建 db1 容器的来源镜像，SDockerfile 用于构建 db2 容器的来源镜像；

sources.list 是ubuntu 的软件源配置，用于加速软件安装

2、50-server.cnf文件内容如下：

```yaml
[server] 
[mysqld] 
user = mysql 
pid-file = /run/mysqld/mysqld.pid 
socket = /run/mysqld/mysqld.sock 
basedir = /usr 
datadir = /var/lib/mysql 
tmpdir = /tmp 
lc-messages-dir = /usr/share/mysql 
query_cache_size = 16M 
log_error = /var/log/mysql/error.log 
server-id = 2 
log_bin = mysql-bin 
expire_logs_days = 10 
binlog_ignore_db = mysql 
character-set-server = utf8mb4 
collation-server = utf8mb4_general_ci 
[embedded] 
[mariadb] 
[mariadb-10.3]
```

50-server.cnf 是 db1 的主配置文件，其中定义了服务的端口、日志文件类型等配置。

3、docker-compose.yml文件内容如下：

```yml
version: "3.8" 
services: 
  db1: 
    image: ubuntu:db1 
    build: 
      context: . 
      dockerfile: MDockerfile 
    networks: 
      - db_network 
    ports: 
      - 33060:3306 
  db2: 
    image: ubuntu:db2 
    build: 
      context: . 
      dockerfile: SDockerfile 
    networks: 
      - db_network 
    ports: 
      - 33060:3306 
networks: 
  db_network: 
  driver: bridge
```

该配置文件中定义了 MySQL 主从同步集群的服务.。

集群中共两个服务，db1 和 db2。其中 db1 为主服务器，服务的来源镜像由 MDockerfile 构建，服务所在的网络是所定义的名为 db_network 的桥接网络。

4、MDockerfile文件内容如下：

```yml
FROM ubuntu:20.04 
MAINTAINER lxc 
COPY sources.list /etc/apt 
RUN apt update && apt dist-upgrade -y \ 
             && apt install mariadb-server -y 
COPY 50-server.cnf /etc/mysql/mariadb.conf.d/50-server.cnf 
RUN /etc/init.d/mysql start \ 
             && mysql -u root -e "create user 'root'@'%' identified by '123456'" \ 
             && mysql -u root -e "grant all on *.* to 'root'@'%'" 
EXPOSE 3306 
CMD /usr/bin/mysqld_safe 
```

MDockerfile 中定义了由 ubuntu 镜像生成所需的 db1 镜像的配置。

使用 COPY 将配置文件 50-server.cnf 拷贝到指定的位置，然后启动服务，执行创建用户和授予权限的 SQL 语句；

最后指定容器暴露 3306 端口，容器启动后需要要执行/usr/bin/mysqld_safe。

5、SDockerfile文件内容如下：

```yml
FROM ubuntu:20.04 
MAINTAINER lxc 
COPY sources.list /etc/apt 
RUN apt update && apt dist-upgrade -y \ 
             && apt install mariadb-server -y \ 
             && sed -i s/bind-address/#bind-address/g /etc/mysql/mariadb.conf.d/50-server.cnf \ 
             && /etc/init.d/mysql start \ 
             && mysql -u root -e "change master to master_host='db1',master_user='root',master_password='123456'" \ 
             && mysql -u root -e "start slave" 
EXPOSE 3306 
CMD /usr/bin/mysqld_safe
```

SDockerfile 用于构建镜像 db2，其中的操作与 MDockerfile 类似。

不同的是，其中执行了 change master 语句和 start slave 语句，前者的功能是指定主服务器地址及用户密码配置，后者的功能是启动主从服务。

6、构建镜像，使用指令 docker-compose build 根据配置文件构建镜像。

7、启动服务，使用指令 docker-compose up -d 启动服务并使服务在后台运行。

8、在宿主机使用端口 33061 登录数据库 db2，查看主从服务状态。

```sql
mysql -u root -h 127.0.0.1 -P33061 -p123456 -e "show slave status\G" 
```

9、在宿主机使用端口 33060 登录数据库 db1，创建数据库、数据表。

```sql
mysql -u root -h 127.0.0.1 -P33060 -p123456 
create database student_manager; 
use student_manager;
create table student( id int primary key auto_increment,name varchar(20) );
insert into student values(null, 'lxc'); 
```

10、在宿主机使用端口 33061 登录数据库 db2，查询数据库、数据表。

```sql
mysql -u root -h 127.0.0.1 -P33061 -p123456 
show databases;
use student_manager;

```
