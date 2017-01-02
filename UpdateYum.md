# UpdateYum

```shell
cd /etc/
mv /etc/yum.repos.d /etc/yum.repos.d.backup4comex
mkdir /etc/yum.repos.d
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all
yum makecache
```