ARG PHP_VERSION

FROM php:${PHP_VERSION}-fpm

MAINTAINER "twitf <qiangzaiying123@gmail.com>"

USER root

RUN bash

ARG PUID=1000
ARG PGID=1000

# 复制生成的公钥私钥 至缓存目录（使用之后会清除缓存）
COPY id_rsa /tmp/id_rsa
COPY id_rsa.pub /tmp/id_rsa.pub

# 设置用户密码 添加用户twitf 安装ssh 设置密码和秘钥登录皆可
RUN apt-get update -yqq && \
  pecl channel-update pecl.php.net && \
  groupadd -g ${PGID} twitf && \
  useradd -u ${PUID} -g twitf -m twitf -s /bin/bash && \
  echo 'twitf:twitf' | chpasswd && \
  echo 'root:root' | chpasswd && \
  apt-get install -y ssh && \
  mkdir /root/.ssh && \
  cat /tmp/id_rsa.pub >> /root/.ssh/authorized_keys && \
  cat /tmp/id_rsa.pub >> /root/.ssh/id_rsa.pub && \
  cat /tmp/id_rsa >> /root/.ssh/id_rsa && \
  rm -f /tmp/id_rsa* && \
  chmod 644 /root/.ssh/authorized_keys /root/.ssh/id_rsa.pub && \
  chmod 400 /root/.ssh/id_rsa && \
  cp -rf /root/.ssh /home/twitf && \
  chown -R twitf:twitf /home/twitf/.ssh && \
  sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config && \
  sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
  sed -i 's/#AuthorizedKeysFile/AuthorizedKeysFile/' /etc/ssh/sshd_config

# debian 9.x (stretch) 阿里云镜像源
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak && \
  echo "deb http://mirrors.aliyun.com/debian/ stretch main non-free contrib" >/etc/apt/sources.list && \
  echo "deb-src http://mirrors.aliyun.com/debian/ stretch main non-free contrib" >>/etc/apt/sources.list && \
  echo "deb http://mirrors.aliyun.com/debian-security stretch/updates main" >>/etc/apt/sources.list && \
  echo "deb-src http://mirrors.aliyun.com/debian-security stretch/updates main" >>/etc/apt/sources.list && \
  echo "deb http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib" >>/etc/apt/sources.list && \
  echo "deb-src http://mirrors.aliyun.com/debian/ stretch-updates main non-free contrib" >>/etc/apt/sources.list && \
  echo "deb http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib" >>/etc/apt/sources.list && \
  echo "deb-src http://mirrors.aliyun.com/debian/ stretch-backports main non-free contrib" >>/etc/apt/sources.list

RUN echo "export TZ='Asia/Shanghai'"  >> /etc/profile

RUN apt-get update -yqq && \
  apt-get install -y --no-install-recommends \
  curl \
  libmemcached-dev \
  libz-dev \
  libpq-dev \
  libjpeg-dev \
  libpng-dev \
  libfreetype6-dev \
  libssl-dev \
  libmcrypt-dev \
  && rm -rf /var/lib/apt/lists/*

RUN pecl install -o -f redis \
  &&  rm -rf /tmp/pear \
  &&  docker-php-ext-enable redis

# rabbitmq prepare
RUN apt-get update -yqq && \
  apt-get install librabbitmq-dev -y && \
  pecl install amqp && \
  docker-php-ext-enable amqp && \
  docker-php-ext-install sockets

RUN docker-php-ext-install exif

RUN docker-php-ext-install mysqli

RUN docker-php-ext-install pdo_mysql

RUN docker-php-ext-install mcrypt

RUN docker-php-ext-configure gd \
  --enable-gd-native-ttf \
  --with-jpeg-dir=/usr/lib \
  --with-freetype-dir=/usr/include/freetype2 && \
  docker-php-ext-install gd

# nghttp2 prepare
RUN apt-get update -yqq && \
  apt-get install -y --no-install-recommends \
  g++ make binutils autoconf automake autotools-dev libtool pkg-config \
  zlib1g-dev libcunit1-dev libxml2-dev libevent-dev libjansson-dev \
  libc-ares-dev libjemalloc-dev libsystemd-dev \
  cython python3-dev python-setuptools

ARG NGHTTP2_VERSION=1.35.1

# nghttp2
RUN curl -O https://github.com/nghttp2/nghttp2/archive/v${NGHTTP2_VERSION}.tar.gz -L && \
  tar zxvf v${NGHTTP2_VERSION}.tar.gz && \
  cd nghttp2-${NGHTTP2_VERSION} && \
  autoreconf -i && \
  automake && \
  autoconf && \
  ./configure && \
  make && make install && ldconfig

ARG HIREDIS_VERSION=0.14.0

# install hiredis
# PS: Swoole 4.2.6版本及以后不再需要手动安装和启用, 而是swoole自带
RUN if [ $(php -r "echo PHP_MAJOR_VERSION;") = "5" || $(php -r "echo PHP_MINOR_VERSION;") = "0" ]; then \
  curl -O https://github.com/redis/hiredis/archive/v${HIREDIS_VERSION}.tar.gz -L && \
  tar zxvf v${HIREDIS_VERSION}.tar.gz && \
  cd hiredis-${HIREDIS_VERSION} && \
  make -j && \
  make install && \
  ldconfig \
  ;fi

## install swoole
## 2.0.11 是最后一个支持php5的版本
## 2.2.0 php7.0
## 4.0.1或更高版本后不再需要设置 --enable-coroutine \
## 4.2.6版本及以后不再需要手动安装hredis和启用--enable-async-redis \, 而是swoole自带 php5和7.0 都使用
RUN if [ $(php -r "echo PHP_MAJOR_VERSION;") = "5" ]; then \
  curl -O https://github.com/swoole/swoole-src/archive/v2.0.11.tar.gz -L && \
  tar zxvf v2.0.11.tar.gz && \
  cd swoole-src-2.0.11 && \
  phpize && \
  ./configure \
  --enable-coroutine \
  --enable-openssl  \
  --enable-http2  \
  --enable-async-redis \
  --enable-mysqlnd && \
  make clean && make && make install \
  ;else \
  if [ $(php -r "echo PHP_MINOR_VERSION;") = "0" ]; then \
  curl -O https://github.com/swoole/swoole-src/archive/v2.2.0.tar.gz -L && \
  tar zxvf v2.2.0.tar.gz && \
  cd swoole-src-2.2.0 && \
  phpize && \
  ./configure \
  --enable-coroutine \
  --enable-openssl  \
  --enable-http2  \
  --enable-async-redis \
  --enable-mysqlnd && \
  make clean && make && make install \
  ;else \
  curl -O https://github.com/swoole/swoole-src/archive/v4.2.12.tar.gz -L && \
  tar zxvf v4.2.12.tar.gz && \
  cd swoole-src-4.2.12 && \
  phpize && \
  ./configure \
  --enable-openssl  \
  --enable-http2  \
  --enable-sockets \
  --enable-mysqlnd && \
  make clean && make && make install \
  ;fi \
  ;fi  && \
  docker-php-ext-enable swoole

# 清理
RUN apt-get clean && \
  rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
  rm /var/log/lastlog /var/log/faillog

## 指定RUN、CMD与ENTRYPOINT命令的工作目录
WORKDIR /var/www

##提供了容器默认的执行命令。 Dockerfile只允许使用一次CMD指令。 使用多个CMD会抵消之前所有的指令，只有最后一个指令生效
#CMD ["php-fpm"]

# 启动ssh
ENTRYPOINT ["/bin/bash","-c","php-fpm | service ssh start"]

## 指定容器在运行时监听的端口
EXPOSE 9000
