#          CentOS源码安装搭建LNMP全过程(包括nginx，mysql，php，svn)     

### 相关目录：所有软件都安装到/www/目录下，在www目录下新建web文件夹作为网站的根路径，www目录下新建wwwsvn作为svn的仓库地址。/www/software用来放nginx，mysql，php的安装包和源码。nginx运行分组和账户www:www

### 安装前的准备

```shell
yum -y install ntp make openssl openssl-devel pcre pcre-devel libpng libpng-devel libjpeg-6b libjpeg-devel-6b freetype freetype-devel gd gd-devel zlib zlib-devel gcc gcc-c++ libXpm libXpm-devel ncurses ncurses-devel libmcrypt libmcrypt-devel libxml2 libxml2-devel imake autoconf automake screen sysstat compat-libstdc++-33 curl curl-devel cmake
```

### 然后下载nginx ，mysql， php的源代码：

```shell
nginx：http://nginx.org/en/download.html
MySQL：http://dev.mysql.com/downloads/mysql/ 
php：http://php.net/downloads.php
将这三份tar.gz文件通过scp命令弄到服务器上/www/software目录下。
```

### 安装nginx

```shell
解压缩文件，然后进到nginx-1.8.0里，输入命令：

./configure --user=www --group=www --prefix=/www/nginx

然后make，make install就安装完毕了。

安装完后第一件事，创建www的用户和分组，否则会遇到http://blog.itblood.com/nginx-emerg-getpwnam-www-failed.html 的错误。

执行：

/usr/sbin/groupadd -f www
/usr/sbin/useradd -g www www

nginx命令在/www/nginx/sbin/下，拷贝到/etc/init.d/一份，接下来设置开机启动。

chmod 755 /etc/init.d/nginx

chkconfig --add nginx

chkconfig nginx on

然后

cd /etc/rc.d/init.d/ #目录下新建nginx，内容如下：

#!/bin/bash
# nginx Startup script for the Nginx HTTP Server
# it is v.0.0.2 version.
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
# It has a lot of features, but it's not for everyone.
# processname: nginx
# pidfile: /var/run/nginx.pid
# config: /usr/local/nginx/conf/nginx.conf
nginxd=/www/nginx/sbin/nginx
nginx_config=/www/nginx/conf/nginx.conf
nginx_pid=/www/nginx/logs/nginx.pid
RETVAL=0
prog="nginx"
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0
[ -x $nginxd ] || exit 0
# Start nginx daemons functions.
start() {
if [ -e $nginx_pid ];then
echo "nginx already running...."
exit 1
fi
echo -n $"Starting $prog: "
daemon $nginxd -c ${nginx_config}
RETVAL=$?
echo
[ $RETVAL = 0 ] && touch /var/lock/subsys/nginx
return $RETVAL
}
# Stop nginx daemons functions.
stop() {
echo -n $"Stopping $prog: "
killproc $nginxd
RETVAL=$?
echo
[ $RETVAL = 0 ] && rm -f /var/lock/subsys/nginx /www/nginx/logs/nginx.pid
}
reload() {
echo -n $"Reloading $prog: "
#kill -HUP `cat ${nginx_pid}`
killproc $nginxd -HUP
RETVAL=$?
echo
}
# See how we were called.
case "$1" in
start)
start
;;
stop)
stop
;;
reload)
reload
;;
restart)
stop
start
;;
status)
status $prog
RETVAL=$?
;;
*)
echo $"Usage: $prog {start|stop|restart|reload|status|help}"
exit 1
esac
exit $RETVAL




注意：如果nginx的安装路径不是在/www/nginx下，则适当修改就好。

chmod 775 /etc/rc.d/init.d/nginx #赋予执行权限
chkconfig nginx on #设置开机启动
/etc/rc.d/init.d/nginx restart
service nginx restart

至此nginx安装就ok了，但遗留两个问题：

1，是更改默认web根目录在/www/web的问题 

2，是与php的整合，默认nginx是不认php得

对于1，nginx默认web根目录在 nginx安装路径下的html文件夹，我们把他改到/www/web目录下。

进到/www/nginx/conf目录下，vim nginx.conf，将

       location / {
            root   html;
            index  index.php index.html index.htm;
        }
修改为：

        location / {

            root   /www/web;

            index  index.html index.php;

        }
注意，增加了对index.php的识别。

将

location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
修改为：

        location ~ \.php$ {

            root           /www/web;

            fastcgi_pass   127.0.0.1:9000;

            fastcgi_index  index.php;

            fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;

            #include        fastcgi_params;

            include fastcgi.conf;

       }
然后就ok了。

第二个问题跟php的整合，待安装完毕php后再整。
三，安装MySQL

解压缩并进到源码目录，执行：

#cmake -DCMAKE_INSTALL_PREFIX=/www/mysql
之后make make install安装。安装完毕后需要做以下几个事：

1，检查/etc/下是否存在my.conf, 如果有的话通过mv命令改名为

my.cnf.backup

ps：此步骤非常重要！！！

2，创建mysql用户和分组

 #/usr/sbin/groupadd mysql

#/usr/sbin/useradd -g mysql mysql 增加mysql用户和分组。

执行

cat /etc/passwd 查看用户列表
cat /etc/group  查看用户组列表

chown -R mysql:mysql /www/mysql修改mysql安装目录的权限。

3，进到/www/mysql,创建系统自带的数据库。

scripts/mysql_install_db --basedir=/www/mysql --datadir=/www/mysql/data --user=mysql

4，添加服务，启动MySQL

cp support-files/mysql.server /etc/init.d/mysql
chkconfig mysql on
service mysql start  --启动MySQL

5,设置root密码

为了让任何地方都能用mysql/bin下的命令，vim /etc/prifile

添加：

PATH=/www/mysql/bin:$PATH
export PATH

保存后source /etc/profile

执行：

mysql -uroot  
mysql> SET PASSWORD = PASSWORD('root');


设置root用户的密码为root。

6，为了支持远程访问数据库，执行；

mysql> grant all on *.* to xroot@"%" identified by "xroot”;

mysql> flush privileges; //更新权限

这样就创建了一个用户名为xroot，密码为xroot的用户，可以远程访问数据库。
四,安装php

解压并进入源码：

#./configure --prefix=/www/php --enable-fpm --with-fpm-user=www --with-fpm-group=www --with-openssl --with-libxml-dir --with-zlib --enable-mbstring --with-mysql=/www/mysql --with-mysqli=/www/mysql/bin/mysql_config --enable-mysqlnd --with-pdo-mysql=/www/mysql --with-gd --with-jpeg-dir --with-png-dir --with-zlib-dir  --with-freetype-dir --with-curl

然后make make install。接着需要做以下事：

1，整合nginx，启动php

进到cd /www/php/etc/ 目录下，拷贝php-fpm.conf.default 为php-fpm.conf。执行/www/php/sbin/php-fpm start 启动php－fpm。

2，配置php.ini 

将安装源码里的/www/software/php-5.6.14/php.ini-production 拷贝到php的安装目录的lib文件夹下。

3，如果需要安装curl扩展的话（上面的configure已经带了），进到源码ext/curl目录下，保证电脑上已经安装了curl和curl-devel,然后：

 a，/www/php/bin/phpize 以下，为了方便可以把这个目录加到/etc/profile里：

PATH=/www/php/bin:/www/mysql/bin:$PATH

export PATH
b,

./configure --with-curl --with-php-config=/www/php/bin/php-config

之后make make install，curl.so会生成在

/www/php/lib/php/extensions/no-debug-non-zts-20131226目录下，然后编辑php.ini找到extension_dir和extension修改即可。


五，安装svn配置post－commit

此步作用是代替ftp，方便开发人员开发并同步代码。可以直接通过yum安装即可。

# rpm -qa subversion  //检查是否自带了低版本的svn

＃yum remove subversion //卸载低版本的svn

# yum install httpd httpd-devel subversion mod_dav_svn mod_auth_mysql   //安装svn

通过# svnserve --version验证是否安装成功。接下来就是创建仓库并与web目录同步。

1，mkdir -p /www/wwwsvn  此文件夹就是svn仓库. svnadmin create /www/wwwsvn 创建仓库，执行上述命令后，可以发现里面有conf, db,format,hooks, locks, README.txt等文件，说明一个SVN库已经建立。（ps：此处可以先通过svnserve -d -r /www/svndata 建立svn版本库目录，然后svnadmin在svndata目录下新建仓库）

2，配置用户和密码

在wwwsvn下进到conf文件夹，里面有三个文件：authz  passwd  svnserve.conf均要编辑。

＃vim passwd //设置用户和密码

[users]
# harry = harryssecret
# sally = sallyssecret
wangning=wangning
yanzi=yanzi

#vim authz  //设置权限

[/]

wangning = rw

yanzi = rw

# &joe = r

# * =
#vim svnserve.conf

anon-access = none
auth-access = write
### The password-db option controls the location of the password
### database file.  Unless you specify a path starting with a /,
### the file's location is relative to the directory containing
### this configuration file.
### If SASL is enabled (see below), this file will NOT be used.
### Uncomment the line below to use the default password file.
password-db = passwd
### The authz-db option controls the location of the authorization
### rules for path-based access control.  Unless you specify a path
### starting with a /, the file's location is relative to the the
### directory containing this file.  If you don't specify an
### authz-db, no path-based access control is done.
### Uncomment the line below to use the default authorization file.
authz-db = authz
### This option specifies the authentication realm of the repository.
### If two repositories have the same authentication realm, they should
### have the same password database, and vice versa.  The default realm
### is repository's uuid.
realm = My First Repository

注意：上面这些有效行前面不能有空格。

3,启动及停止svn

#svnserve -d -r /www/wwwwvn   //启动svn

#killall svnserve    //停止

待启动svn后，可以在外面测试了。

svn checkout svn://192.1.15.222 --username xxx

4,配置post－commit

经过上述配置后，svn的仓库地址是/www/wwwsvn, 但是web的根目录是/www/web,两者不是一个目录，无法svn push上来就看到作用。

a，首先在server的终端里，＃svn co svn://192.1.15.222 /www/web

记得将/www/web目录权限修改为www:www。

chown -R www:www /www/web

b, # cd /www/wwwsvn/hooks/,然后cp post-commit.tmpl post-commit  

vim post-commit，在里面输入：

export LANG=zh_CN.UTF-8

svn up --username yanzi --password yanzi /www/web/

chown -R www:www /www/web/

然后就一切ok了，在外面svn commit看看web目录里有么有对应文件吧！
```

```shell
ps:

1，svn up后面的名字和密码是之前设的svn用户。

2，上面up就是update的意思，按Git的意思来理解，就是有个仓库A，然后新建了个B去跟踪A，每次A有提交的时候，让B也pull一下过来。在svn里是update。

参考链接：

1. svn部分：http://www.cnblogs.com/davidgu/archive/2013/02/01/2889457.html 

http://www.cnblogs.com/zhoulf/archive/2013/02/02/2889949.html

2.lnmp部分：http://www.tuicool.com/articles/jqIb22

3.mysql部分：http://www.cnblogs.com/xiongpq/p/3384681.html

下载链接nginx ＋ mysql ＋ php：

http://yunpan.cn/cFQZxxhYeWxX8 （提取码：7f93）


关于SVN的补充说明：

1，一般建立目录/www/wwwsvn为一级目录，然后新建二级目录/www/wwwsvn/project1作为仓库放project1. 

命令：

svnadmin create /www/wwwsvn/project1

启动svn:

svnserve -d -r /www/wwwsvn

这样启动的svn根目录是wwwsvn,外侧project1的地址是：svn:192.xx.xx.x/project1  这样做的好处是可以建立多个project.在外面svn co svn:192.xx.xx.x/project1的时候也会新建project1文件夹。

结论：如果希望自由控制项目的文件名那就svnserve的时候直接把项目文件夹启动，这样svn checkout的时候先建好外面的文件夹，然后进来checkout就行了。如果想直接checkout下来就带有项目文件夹，那就svnserve的时候把外层目录启动。这样在任意地方checkout的时候自带project1文件夹。

2,关于post-commit

首先在服务器的/a/apps/zhujibao/manager/public/路径下，svn co svn:192.xx.xx.x/city52，这样city52文件夹就自动创建了。

其内容为：

export LANG=zh_CN.UTF-8

svn up --username yanzi --password yanzi /a/apps/zhujibao/manager/public/city52

chown -R www:www /a/apps/zhujibao/manager/public/city52


REPOS="$1"

REV="$2"


#mailer.py commit "$REPOS" "$REV" /path/to/mailer.conf

最后的那句一定要注释掉！另外，要记得给post-commit增加可执行权限。

3，如果不设置post-commit，那么你在外面提交了东西，在仓库地址下是什么东西都看不到的，因此这个post-commit是必须的！
```