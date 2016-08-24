# CentOS编译安装PHP开发环境
***

> 最近在安装服务器开发环境, 踩了不少坑, 这里总结下来.
> yum安装虽然简单, 却不灵活, 版本也比较老旧不合符设计中的选型, 因此只使用yum安装一些依赖库, 目标软件采用编译安装.

### 目录
* [安装PHP](#安装PHP)
* [安装PHP扩展](#安装PHP扩展)
* [安装Phalcon框架](#安装Phalcon框架)
* [安装MySQL](#安装MySQL)
* [安装MongoDB](#安装MongoDB)
* [安装Redis](#安装Redis)

## <a name=安装PHP></a>安装PHP
#### yum安装依赖库
```bash
yum install -y make cmake gcc gcc-c++ autoconf automake libpng-devel libjpeg-devel zlib libxml2-devel ncurses-devel bison \
 libtool-ltdl-devel libiconv libmcrypt mhash mcrypt pcre-devel openssl-devel freetype-devel libcurl-devel
```
#### 安装PHP7.0.9
```bash
#先下载PHP
cd /tmp
wget http://cn2.php.net/distributions/php-7.0.9.tar.gz
tar -zxvf php-7.0.9.tar.gz
cd php-7.0.9.tar.gz

#我们先配置下PHP的编译参数
./configure --prefix=/usr/local/php \ 
--with-mysql \ 
--with-mysqli \ 
--with-pdo_mysql \ 
--with-iconv-dir \ 
--with-zlib \
--with-libxml-dir \
--enable-xml \
--with-curl \
--enable-fpm \
--enable-mbstring \
--with-gd \
--with-openssl \
--with-mhash \
--enable-sockets \
--with-xmlrpc \
--enable-zip \
--enable-soap \
--with-freetype-dir=/usr/lib64

#编译 
make
make install clean

#复制php.ini
cp php.ini-development /usr/local/php/lib/php.ini
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf

#运行php-fpm
/usr/local/php/sbin/php-fpm

#将php命令加入到全局
vi /root/.bash_profile 

#将/usr/local/php/bin 加到后面，用:隔开
PATH=$PATH:$HOME/bin:/usr/local/mysql/bin:/usr/local/mysql/lib:/usr/local/php/bin

#重启
source /root/.bash_profile
```
#### php-fpm的启动脚本
PHP7安装包里自带了启动脚本:

```bash
cp /tmp/php-7.0.9/sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
cd /etc/init.d
chmod +x php-fpm
```
然后就可以启动,停止重启等操作了:

```bash
service php-fpm restart
```
## <a name=安装PHP扩展></a>安装PHP扩展
PHP扩展的安装都是利用phpize, 这里用redis举例
#### 下载扩展
```bash
http://pecl.php.net
```
#### 安装
```bash 
tar -zxvf redis-3.0.1.tgz  
cd redis-3.0.1  
phpize   
./configure --with-php-config=/usr/local/php/bin/php-config   
make && make install  
```
#### 修改php.ini
```bash
extension=redis.so
```
#### 重启php-fpm
```bash
kill `cat /var/run/php-fpm.pid ` && /usr/local/php/sbin/php-fpm 
```
## <a name=安装Phalcon框架></a>安装Phalcon框架
phalcon作为PHP的扩展存在, 理应可以使用phpize安装的, 但是试过几次, 依赖的东东比较难搞, 还是使用官方给出的安装器:

#### 编译安装
```bash 
git clone git://github.com/phalcon/cphalcon.git
cd cphalcon/build
sudo ./install
```
其中遇到gcc报错, 是因为虚拟机内存太小, 后来加了一个swap分区解决了.
#### 修改php.ini
```bash 
extension=phalcon.so
```

## <a name=安装Nginx></a>安装Nginx
#### 安装nginx的依赖包
nginx 依赖于 zlib pcre ssl 三个模块，安装之前要先安装它们

```bash
cd /lamp 
wget http://zlib.net/zlib-1.2.8.tar.gz
tar -zxvf zlib-1.2.7.tar.gz
cd zlib-1.2.7
./configure
make
make install 

wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.32.tar.gz
tar -zxvf pcre-8.21.tar.gz
cd pcre-8.21
./configure
make
make install 

wget http://www.openssl.org/source/openssl-1.0.2.tar.gz
tar zxvf openssl-1.0.2.tar.gz
cd openssl-1.0.2.tar.gz
./config  # 注意是config,不是configure
make
make install 
```
#### 安装nginx
```bash
# 下载
wget http://nginx.org/download/nginx-1.11.3.tar.gz
tar -zxvf nginx-1.11.3.tar.gz
cd nginx-1.11.3.tar.gz

# 编译
# 这3个扩展的目录是他们的源代码目录，不是安装目录，这点很容易搞错
./configure --prefix=/usr/local/nginx \
--sbin-path=/usr/local/nginx/nginx \
--conf-path=/usr/local/nginx/nginx.conf \
--pid-path=/usr/local/nginx/nginx.pid \
--with-http_ssl_module \
--with-pcre=/lamp/pcre-8.32 \
--with-zlib=/lamp/zlib-1.2.7 \
--with-openssl=/lamp/openssl-1.0.2
make && make install

#启动
/usr/local/nginx/nginx

#查看端口
netstat -tnl|grep 8080

#访问
curl http://localhost:8080
```
#### 开机自启动
开机启动的配置文件是：/etc/rc.local ，vi加入 /usr/local/nginx/nginx 即可

```bash
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/usr/local/apache/bin/apachectl start
/usr/local/bin/redis-server /etc/redis.conf
/usr/local/php/sbin/php-fpm
/usr/local/nginx/nginx

```
#### 开启nginx的iptables限制
我们在本地访问127.0.0.1:8080/index.php，是可以打开的。 但是如果，在另外一台机子上访问：http://192.168.155.128:8080/index.php 不能访问，可能是这个8080端口号没有加入到iptables的白名单，所以需要加一下：

(PS： 如果你的nginx端口号是80，应该是已经在iptables白名单中了，如果能访问就不需要加了)

iptables的配置文件在这：`/etc/sysconfig/iptables`

我们vi 打开下，然后在**倒数第二行**上面加入：
`-A INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT `
然后，重启下 iptables

```bash
service iptables restart
```
## <a name=安装MySQL></a>安装MySQL
```bash
cd /tmp
#先下载mysql 
http://dev.mysql.com/Downloads/mysql/mysql-5.7.14.tar.gz
tar zxvf mysql-5.7.14.tar.gz
cd mysql-5.7.14

#cmake配置下
cmake \ 
-DCMAKE_INSTALL_PREFIX=/usr/local/mysql \  #安装目录
-DMYSQL_DATADIR=/usr/local/mysql/data \    #数据库存放目录
-DDEFAULT_CHARSET=utf8 \                   #使用utf8字符
-DDEFAULT_COLLATION=utf8_general_ci \      #校验字符
-DEXTRA_CHARSETS=all \                     #安装所有扩展字符集
-DENABLED_LOCAL_INFILE=1                   #允许从本地导入数据

#编译安装
make && make install

#创建mysql用户和用户组
groupadd mysql
useradd -r -g mysql mysql

#给mysql目录设置目录权限
chown -R mysql:mysql /usr/local/mysql

#将mysql的启动服务添加到系统服务中
cd /usr/local/mysql/
cp support-files/my-default.cnf /etc/my.cnf

#创建系统数据库的表
cd scripts/
./mysql_install_db --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data/

#复制mysql管理脚本到系统服务目录
cd /opt/mysql/support-files
cp mysql.server /etc/rc.d/init.d/mysql

#添加mysql命令到系统服务命令
chkconfig --add mysql

#加入开机启动策略
chkconfig mysql on
service mysql start

#以后就可以调用service 命令来管理mysql
service mysql start
service mysql stop
service mysql restart

#将mysql命令加入全局可用
vi /root/.bash_profile
#在PATH=$PATH:$HOME/bin添加参数为：
PATH=$PATH:$HOME/bin:/usr/local/mysql/bin:/usr/local/mysql/lib
#重新生效
source /root/.bash_profile

#修改root密码
mysql -u root mysql
mysql>use mysql;
mysql>desc user;
mysql> GRANT ALL PRIVILEGES ON *.* TO root@"%" IDENTIFIED BY "root";　　//为root添加远程连接的能力。
mysql>update user set Password = password('12346') where User='root';
mysql>select Host,User,Password  from user where User='root'; 
mysql>flush privileges;
mysql>exit
#重新登录：
mysql -uroot -p123456
```

## <a name=安装MongoDB></a>安装MongoDB
#### 下载解压
```bash
wget http://fastdl.mongodb.org/linux/mongodb-linux-x86_64-2.6.4.tgz
tar -zxvf mongodb-linux-x86_64-2.6.4.tgz -C /usr/src
cd mongodb-linux-x86_64-2.6.4
```
#### 创建数据库和日志的目录
```bash
mkdir log
mkdir db
```
#### 以后台方式启动
```bash
./bin/mongod --dbpath=./db --logpath=./log/mongodb.log --fork --auth
```
#### 设置开机启动
```bash
echo "/usr/src/mongodb-linux-x86_64-2.6.4/bin/mongod --dbpath=/usr/src/mongodb-linux-x86_64-2.6.4/db --logpath=/usr/src/mongodb-linux-x86_64-2.6.4/log/mongodb.log --fork --auth" >> /etc/rc.local
```
#### 查看端口
```bash
netstat -nalupt | grep mongo
```

## <a name=安装Redis></a>安装Redis
#### 安装
```bash
#下载解压
wget http://download.redis.io/releases/redis-3.0.3.tar.gz
tar xzf redis-3.0.3.tar.gz
cd redis-3.0.3

#编译
make
make install
```
#### 修改redis.conf
```bash 
daemonize yes
loglevel notice
logfile /var/log/redis.log
dir ./
```
#### 设置系统的overcommit_memory, vi /etc/sysctl.conf
```bash 
vm.overcommit_memory = 1 
```
执行:

```bash
sysctl vm.overcommit_memory=1
```
#### 添加启动脚本
```bash
#!/bin/sh
#
# redis        Startup script for Redis Server
#
# chkconfig: - 90 10
# description: Redis is an open source, advanced key-value store. 
#
# processname: redis-server
# config: /etc/redis.conf
# pidfile: /var/run/redis.pid
 
REDISPORT=6379
EXEC=/usr/local/bin/redis-server
REDIS_CLI=/usr/local/bin/redis-cli
 
PIDFILE=/var/run/redis.pid
CONF="/usr/src/redis-3.0.3/redis.conf"
 
case "$1" in
    start)
        if [ -f $PIDFILE ]
        then
                echo "$PIDFILE exists, process is already running or crashed"
        else
                echo "Starting Redis server..."
                $EXEC $CONF
        fi
        if [ "$?"="0" ] 
        then 
              echo "Redis is running..."
        fi 
        ;;
    stop)
        if [ ! -f $PIDFILE ]
        then
                echo "$PIDFILE does not exist, process is not running"
        else
                PID=$(cat $PIDFILE)
                echo "Stopping ..."
                $REDIS_CLI -p $REDISPORT SHUTDOWN
                while [ -x ${PIDFILE} ]
               do
                    echo "Waiting for Redis to shutdown ..."
                    sleep 1
                done
                echo "Redis stopped"
        fi
        ;;
   restart|force-reload)
        ${0} stop
        ${0} start
        ;;
  *)
    echo "Usage: /etc/init.d/redis {start|stop|restart|force-reload}" >&2
        exit 1
esac
```
```bash
chmod +x /etc/init.d/redis
chkconfig --add redis
# 加入开机启动
chkconfig redis on
```
