FROM registry.cn-hangzhou.aliyuncs.com/kingjjm/mysql:5.7 
MAINTAINER kingjjm 

ENV TZ "Asia/Shanghai"
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
COPY mysqld.cnf /etc/mysql/mysql.conf.d/mysqld.cnf
