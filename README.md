#启动redis方法
docker run -p 6379:6379 --name redis_test01 -v $PWD/redis.conf:/etc/redis/redis.conf -v /data/redis:/data -d redis:4.0 redis-server /etc/redis/redis.conf
#启动mysql方法
docker run -p 3308:3306 --name mysql_demo -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

