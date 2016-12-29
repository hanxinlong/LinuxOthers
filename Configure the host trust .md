## 配置主机信任

## 方法一

### 简单步骤

```shell
#主机1上操作


#第一步：生成公钥和私钥
ssh-keygen #接着一路回车就行

~/.ssh/id_rsa		#私钥保存路径
~/.ssh/id_rsa.pub   #公钥保存路径


#主机2上操作

touch ~/.ssh/authorized_keys       #创建文件
vim ~/.ssh/authorized_keys		   #编辑authorized_keys文件
#打开上面的文件之后，将主机1生成的id_rsa.pub公钥文件的内容复制到authorized_keys里面，复制的时候一定要复制好；
```