FROM redis:4.0 
MAINTAINER kingjjm

ENV TZ "Asia/Shanghai"
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
COPY redis.conf /etc/redis/redis.conf
