## 防火墙相关操作

#### 防火墙英文：Firewall

##### Windows下面的防火墙服务叫做firewall.cpl

##### Linux下面用的是iptables，这个是软件的防火墙，一些大企业到采用的是硬件防火墙

```shell
#重启之后生效
chkconfig iptables on   #开启
chkconfig iptables off  #关闭

#即时生效，重启后失效
service iptables start   #开启
service iptables stop    #关闭

#防火墙5788
#selinux radhat或者centos的一种安全机制，很少有人用
#临时关闭selinux
setenforce 0    
#检查selinux是否关闭
getenforce

vim /etc/selinux/config  #修改这个文件永久关闭
SELINUX=disabled         #这个选项要改成这个值

```
当地的v上的v说的v的v