# 1. wsl里拉起ssh映射

进入wsl

```
sudo apt update
sudo apt install -y openssh-server
```

看下用户名

```
whoami
```

启动ssh

```
sudo service ssh start
```

看一下ssh的进程ID

```
ps -ef | grep '[s]shd'
```

![image](../../png/engineering/2.png)

看一下拉起的是哪个端口

```
sudo /usr/sbin/sshd -T | grep -E '^(port|listenaddress|addressfamily)'
```

![image](../../png/engineering/3.png)

验证一下wsl的ssh开好了没。

```
ssh -p {port} {username}@127.0.0.1
```

示例

```
ssh -p 10246 heisenberg@127.0.0.1
```

# 2. 查看wsl的IP

打开powershell

```
$wslIp = (wsl hostname -I).Trim().Split()[0]
$wslIp
```

![image](../../png/engineering/4.png)

# 3. 本地windows发起wsl映射

打开cmd

```
ipconfig
```

![image](../../png/engineering/5.png)

![image](../../png/engineering/6.png)

现在有了wsl和windows的IP。打开IPOP。**本地地址**填windows的IP，端口随便写个，映射地址填**wsl地址**，端口填前面wsl里的端口。

填完点右边的增加，增加了以后，点右边下面的映射，发起映射。

![image](../../png/engineering/7.png)


#IPOP #ipop #映射 #wsl映射 #wsl #Ubuntu