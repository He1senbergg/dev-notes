# 1. TCPDUMP

下载，解压，安装一下。

[tcpdump-4.99.4-4.h21510.2.eulerosv2r15.aarch64.zip](../installer/Fiddler/tcpdump-4.99.4-4.h21510.2.eulerosv2r15.aarch64.zip)

[tcpdump-4.99.4-4.h21510.2.eulerosv2r15.x86_64.zip](../installer/Fiddler/tcpdump-4.99.4-4.h21510.2.eulerosv2r15.x86_64.zip)

安装

```
rpm -ivh tcpdump-4.99.4-4.h21510.2.eulerosv2r15.aarch64.rpm
```

```
tcpdump -i any tcp port 179 and host 110.11.3.3 -ent -vvv -nnn
```

- -i` 后面跟抓得接口
- `-i any` 表示不关心抓得是哪个接口的数据包
- `tcp port 179` 表示抓端口是179的tcp包，也就是bgp包
- `and host 110.11.3.3` 表示抓的源或者目的地址必须包含110.11.3.3
- `-ent` 展开二层数据
- `-vvv` 展开详情
- `-nnn` 直接展示原数据

# 2. 容器里抓包

简易使用示例

![image](../png/engineering/47.png)

```
tcpdump -i eth1 tcp port 27321 -w ./ddd.cap
```

`eth1`是用命令行看的网卡，`port`是端口，`-w`是生成的文件的位置，名字要以cap结尾，随便起个名字。

```
# 看网卡
ip a s
```

# 3. Wireshark

[官网下载](https://www.wireshark.org/download.html)

安装，选项全都默认，直接点下一步，弹出的附加软件要安装一下，不然主软件的安装进度会一直卡着。

把容器里抓包下载一下，在本地windows拖进Wireshark。