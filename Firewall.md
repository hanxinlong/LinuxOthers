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

iptables -I INPUT -s [源地址] --sport [来源端口] -p [协议] -d [目标地址] --dport [指定目标的端口] -j [指定动作ACCEPT/DROP/REJECT]

iptables -I INPUT -s 10.0.169.45 -j ACCEPT

iptables -I INPUT -s 10.0.169.45 -p tcp --dport 80 -j ACCEPT #来源自10.0.169.45的主机，访问我这台服务器的时候，允许使用tcp协议打开80端口
iptables -I INPUT -p tcp --dport 80 -j ACCEPT	#非常常用，允许所有的ip地址来访问我的80端口

#iptables定义规则的方式比较复杂:
#格式：iptables [-t table] COMMAND chain CRETIRIA -j ACTION
	#	-t table ：3个filter nat mangle
	#	COMMAND：定义如何对规则进行管理
	#	chain：指定你接下来的规则到底是在哪个链上操作的，当定义策略的时候，是可以省略的
	#	CRETIRIA:指定匹配标准
	#	-j ACTION :指定如何进行处理

	#比如：不允许172.16.0.0/24的进行访问。
	iptables -t filter -A INPUT -s 172.16.0.0/16 -p udp --dport 53 -j DROP
	#当然你如果想拒绝的更彻底：
	iptables -t filter -R INPUT 1 -s 172.16.0.0/16 -p udp --dport 53 -j REJECT
#1. iptables
#2. 操作 ，增加-A/-I，删除-D
#3. INPUT / OUPUT / FORWARD
#4. -s 来源地址
#5. --sport 来源端口
#6. -p 协议
#7. -d 目标地址
#8. --dport 目标端口
#9. -j 动作，ACCEPT / DROP /REJECT   ，允许 / 丢弃 / 拒绝

```




#### 基本知识

​	目前市面上比较常见的有3、4层的防火墙，叫网络层的防火墙，还有7层的防火墙，其实是代理层的网关。

​	对于TCP/IP的七层模型来讲，我们知道第三层是网络层，三层的防火墙会在这层对源地址和目标地址进行检测。但是对于七层的防火墙，不管你源端口或者目标端口，源地址或者目标地址是什么，都将对你所有的东西进行检查。所以，对于设计原理来讲，七层防火墙更加安全，但是这却带来了效率更低。所以市面上通常的防火墙方案，都是两者结合的。而又由于我们都需要从防火墙所控制的这个口来访问，所以防火墙的工作效率就成了用户能够访问数据多少的一个最重要的控制，配置的不好甚至有可能成为流量的瓶颈。



5个位置，

    1.内核空间中：从一个网络接口进来，到另一个网络接口去的

    2.数据包从内核流入用户空间的

    3.数据包从用户空间流出的

    4.进入/离开本机的外网接口

    5.进入/离开本机的内网接口

​	从上面的发展我们知道了作者选择了5个位置，来作为控制的地方，但是你有没有发现，其实前三个位置已经基本上能将路径彻底封锁了，但是为什么已经在进出的口设置了关卡之后还要在内部卡呢？ 由于数据包尚未进行路由决策，还不知道数据要走向哪里，所以在进出口是没办法实现数据过滤的。所以要在内核空间里设置转发的关卡，进入用户空间的关卡，从用户空间出去的关卡。那么，既然他们没什么用，那我们为什么还要放置他们呢？因为我们在做NAT和DNAT的时候，目标地址转换必须在路由之前转换。所以我们必须在外网而后内网的接口处进行设置关卡。        

​	

 这五个位置也被称为五个钩子函数（hook functions）,也叫五个规则链。

​		1.PREROUTING (路由前)

​		2.INPUT (数据包流入口)

​		3.FORWARD (转发管卡)

​		4.OUTPUT(数据包出口)

​		5.POSTROUTING（路由后）

        这是NetFilter规定的五个规则链，任何一个数据包，只要经过本机，必将经过这五个链中的其中一个链。      





**3.防火墙的策略**

​	防火墙策略一般分为两种，一种叫“通”策略，一种叫“堵”策略，通策略，默认门是关着的，必须要定义谁能进。堵策略则是，大门是洞开的，但是你必须有身份认证，否则不能进。所以我们要定义，让进来的进来，让出去的出去，所以通，是要全通，而堵，则是要选择。当我们定义的策略的时候，要分别定义多条功能，其中：定义数据包中允许或者不允许的策略，filter过滤的功能，而定义地址转换的功能的则是nat选项。为了让这些功能交替工作，我们制定出了“表”这个定义，来定义、区分各种不同的工作功能和处理方式。

​	我们现在用的比较多个功能有3个：

​		1.filter 定义允许或者不允许的

​		2.nat 定义地址转换的 

                3.mangle功能:修改报文原数据

​	我们修改报文原数据就是来修改TTL的。能够实现将数据包的元数据拆开，在里面做标记/修改内容的。而防火墙标记，其实就是靠mangle来实现的。

 

小扩展:

​	对于filter来讲一般只能做在3个链上：INPUT ，FORWARD ，OUTPUT

​	对于nat来讲一般也只能做在3个链上：PREROUTING ，OUTPUT ，POSTROUTING

​	而mangle则是5个链都可以做：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

 

​	iptables/netfilter（这款软件）是工作在用户空间的，它可以让规则进行生效的，本身不是一种服务，而且规则是立即生效的。而我们iptables现在被做成了一个服务，可以进行启动，停止的。启动，则将规则直接生效，停止，则将规则撤销。 

​	iptables还支持自己定义链。但是自己定义的链，必须是跟某种特定的链关联起来的。在一个关卡设定，指定当有数据的时候专门去找某个特定的链来处理，当那个链处理完之后，再返回。接着在特定的链中继续检查。

注意：规则的次序非常关键，谁的规则越严格，应该放的越靠前，而检查规则的时候，是按照从上往下的方式进行检查的。

 **三．规则的写法:**

​	 iptables定义规则的方式比较复杂:

​	 格式：iptables [-t table] COMMAND chain CRETIRIA -j ACTION

​		 -t table ：3个filter nat mangle

​		 COMMAND：定义如何对规则进行管理

​		 chain：指定你接下来的规则到底是在哪个链上操作的，当定义策略的时候，是可以省略的

​		 CRETIRIA:指定匹配标准

​		 -j ACTION :指定如何进行处理

​	 比如：不允许172.16.0.0/24的进行访问。

​	 iptables -t filter -A INPUT -s 172.16.0.0/16 -p udp --dport 53 -j DROP

​	 当然你如果想拒绝的更彻底：

​	 iptables -t filter -R INPUT 1 -s 172.16.0.0/16 -p udp --dport 53 -j REJECT

​	 iptables -L -n -v	#查看定义规则的详细信息



**四：详解COMMAND:**

**1.链管理命令（这都是立即生效的）**

​	-P :设置默认策略的（设定默认门是关着的还是开着的）

​		默认策略一般只有两种

​		iptables -P INPUT (DROP|ACCEPT)  默认是关的/默认是开的

​		比如：

​		iptables -P INPUT DROP 这就把默认规则给拒绝了。并且没有定义哪个动作，所以关于外界连接的所有规则包括Xshell连接之类的，远程连接都被拒绝了。

        -F: FLASH，清空规则链的(注意每个链的管理权限)

​	    iptables -t nat -F PREROUTING

​	    iptables -t nat -F 清空nat表的所有链

        -N:NEW 支持用户新建一个链

            iptables -N inbound_tcp_web 表示附在tcp表上用于检查web的。

        -X: 用于删除用户自定义的空链

            使用方法跟-N相同，但是在删除之前必须要将里面的链给清空昂了

        -E：用来Rename chain主要是用来给用户自定义的链重命名

            -E oldname newname

         -Z：清空链，及链中默认规则的计数器的（有两个计数器，被匹配到多少个数据包，多少个字节）

            iptables -Z :清空

 

**2.规则管理命令**

         -A：追加，在当前链的最后新增一个规则

         -I num : 插入，把当前规则插入为第几条。

            -I 3 :插入为第三条

         -R num：Replays替换/修改第几条规则

            格式：iptables -R 3 …………

         -D num：删除，明确指定删除第几条规则

        

**3.查看管理命令 “-L”**

​	 附加子命令

​	 -n：以数字的方式显示ip，它会将ip直接显示出来，如果不加-n，则会将ip反向解析成主机名。

​	 -v：显示详细信息

​	 -vv

​	 -vvv :越多越详细

​	 -x：在计数器上显示精确值，不做单位换算

​	 --line-numbers : 显示规则的行号

​	 -t nat：显示所有的关卡的信息