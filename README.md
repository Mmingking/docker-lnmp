目录结构 github地址 https://github.com/jinjunming/docker-lnmp.git 

目录结构 

```
docker_lnmpr
├── mysql
│   └── Dockerfile
	└── mysqld.cnf
├── nginx
│   ├── Dockerfile
│   ├── nginx.conf
│   └── vhost
│       └── www.texixi.com.conf
├── php
│   ├── Dockerfile
│   ├── composer.phar
│   ├── php-fpm.conf
│   ├── php.ini
│   ├── redis-3.0.0.tgz
└── redis
    └── Dockerfile
    └── redis.conf
```

### 建立镜像与容器
```
# build
docker build -t centos/nginx:v1.12.1  ./nginx
docker build -t centos/mysql:v5.7   ./mysql
docker build -t centos/php:v7.0.12  ./php
docker build -t centos/redis:v4.0 ./redis

#备注：这里选取了172.172.0.0网段，也可以指定其他任意空闲的网段
docker network create --subnet=172.171.0.0/16 docker-at

# run
docker run --name mysql56  --restart=always --net docker-at --ip 172.171.0.9 -d -p 3306:3306 -v /data/mysql:/var/lib/mysql -v /data/logs/mysql:/var/log/mysql -v /data/run/mysqld:/var/run/mysqld  -e MYSQL_ROOT_PASSWORD=123456 -it centos/mysql:v5.6

docker run --name mysql57 --restart=always --net docker-at --ip 172.171.0.9 -d -p 3306:3306 -v /data/mysql:/var/lib/mysql -v /data/logs/mysql:/var/log/mysql -v /data/run/mysqld:/var/run/mysqld  -e MYSQL_ROOT_PASSWORD=123456 -it centos/mysql:v5.7


docker run --name redis326 --restart=always --net docker-at --ip 172.171.0.10 -d -p 6379:6379  -v /data:/data -it centos/redis:v3.2.6

docker run --name php7 --restart=always --net docker-at --ip 172.171.0.8 -d -p 9000:9000 -v /www:/www -v /data/php:/data --link mysql56:mysql56 --link redis326:redis326 -it centos/php:v7.0.12 
docker run --name nginx1121 --restart=always --net docker-at --ip 172.171.0.7 -p 80:80 -d -v /www:/www -v /data/nginx:/data --link php7:php7 -it centos/nginx:v1.12.1 



```

以上整合到 docker-compose.yml，如下：

```
mysql:
    build: ./mysql
    ports:
      - "3306:3306"
    volumes:
      - /data/mysql:/var/lib/mysql
      - /data/log/mysql:/var/log/mysql
      - /data/run/mysqlmysqld:/var/run/mysqld

    environment:
      MYSQL_ROOT_PASSWORD: 123456

redis:
    build: ./redis
    volumes:
      - /data/redis:/data
    ports:
      - "6379:6379"

php:
    build: ./php
    ports:
      - "9000:9000"
    links:
      - "mysql"
      - "redis"
    volumes:
      - /www:/www
      - /data/php:/data
      - 

nginx:
    build: ./nginx
    ports:
      - "80:80"
    links:
      - "php"
    volumes:
      - /www:/www
      - /data/nginx:/data


```


### 问题点整理
* 注意容器内是没写的权限的

* 需要注意php.ini 中的目录对应  mysql 的配置的目录需要挂载才能获取文件内容，不然php连接mysql失败

		```
		# php.ini
		mysql.default_socket = /data/run/mysql/mysqld.sock
		mysqli.default_socket = /data/run/mysql/mysqld.sock
		pdo_mysql.default_socket = /data/run/mysql/mysqld.sock
		
		# mysqld.cnf
		pid-file       = /var/run/mysqld/mysqld.pid
		socket         = /var/run/mysqld/mysqld.sock
		
		```



### 使用php连接不上redis 

	```
	# 错误的
	$redis = new Redis;
	$rs = $redis->connect('127.0.0.1', 6379);
	
	```
	
	php连接不上，查看错误日志
	```
	PHP Fatal error:  Uncaught RedisException: Redis server went away in /www/index.php:7
	```
	考虑到docker 之间的通信应该不可以用127.0.0.1 应该使用容器里面的ip，所以查看redis 容器的ip
	```
	[root@iZ287mq5dooZ php]# docker ps 
	CONTAINER ID        IMAGE                  COMMAND                   CREATED             STATUS              PORTS                         NAMES
	5fb4b1904f1c        centos/nginx:v1.11.5   "/usr/local/nginx/sbi"    About an hour ago   Up About an hour    0.0.0.0:80->80/tcp, 443/tcp   nginx11
	2bf7ad9f44f9        centos/php:v7.0.12     "/usr/local/php/sbin/"    About an hour ago   Up About an hour    0.0.0.0:9000->9000/tcp        php7
	4b84858ea4e4        centos/redis:v3.2.6    "/bin/sh -c '\"/usr/lo"   18 hours ago        Up About an hour    0.0.0.0:6379->6379/tcp        redis326
	158c67aa178c        centos/mysql:v5.7      "docker-entrypoint.sh"    6 days ago          Up About an hour    0.0.0.0:3306->3306/tcp        mysql57
	[root@iZ287mq5dooZ php]# docker inspect 4b84858ea4e4
	
	```
	结果是为 192.168.0.4，测试连接，成功
	```
	$redis = new Redis;
	$rs = $redis->connect('192.168.0.4', 6379);
	```
	问题是重启容器ip为动态的，解决该问题
	第一步：创建自定义网络
	```
	#备注：这里选取了172.172.0.0网段，也可以指定其他任意空闲的网段
	docker network create --subnet=172.171.0.0/16 docker-at
	docker run --name redis326 --net docker-at --ip 172.171.0.10 -d -p 6379:6379  -v /data:/data -it centos/redis:v3.2.6
	```
	
	连接redis 就可以配置对应的ip地址了，连接成功
	```
	$redis = new Redis;
	$rs = $redis->connect('172.171.0.10', 6379);
	```

以上情况虽然容器之间关联了，但是容器之间的通讯需要用搭建的网段的连接。
假设：只有mysql 的容器，我们机器挂载了3306的端口，我们本地可以127.0.0.1去连接mysql容器服务
但是假设php服务也在容器里面，这时就不可以这么连接，因为是php容器去连接mysql容器，所以需要一个连接的ip。
