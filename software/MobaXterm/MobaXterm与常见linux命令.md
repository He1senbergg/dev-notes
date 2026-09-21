# 1. MobaXterm

## 1.1 软件下载

网上随便找个MobaXterm_Pro_Installer_v23.2的破解版安装包，或者什么版本的，下个盗版的。

## 1.2 软件设置

### 1.2.1 使用右键粘贴

前置：在MobaXterm中，

- 默认滑动选中内容以后，即成功复制到剪贴板。
- 双击可直接选中一段独立的内容。

![image](../../png/engineering/27.png)

### 1.2.2 关闭X11（可选项）

目的：偶现MobaXterm卡死，关闭X11可缓解。详见 **<a href="./MobaXterm卡顿问题.md" style="color: deepskyblue">Markdown</a>**。

![image](../../png/engineering/31.png)

## 1.3 软件使用

### 1.3.1 连接服务器

![image](../../png/engineering/32.png)

![image](../../png/engineering/33.png)

![image](../../png/software/1.png)

在输入完成后`IP`和`username`之后，点击`OK`。

![image](../../png/software/2.png)

键入密码
（Linux机器中，在输入密码时，会不显示文字。输完回车就行。）

![image](../../png/software/3.png)

### 1.3.2 保存指令/密码

![image](../../png/software/4.png)

![image](../../png/software/5.png)

点击后出现上方的大字。

![image](../../png/software/6.png)

键入任意所需字符（含回车等操作）。

![image](../../png/software/7.png)

![image](../../png/software/8.png)

按下stop。

![image](../../png/software/9.png)

出现弹窗，自定义是否需要更名，点击ok。

![image](../../png/software/10.png)

单击左侧保存的按钮，即可观测到效果。

![image](../../png/software/11.png)

事后需要编辑，则在对应的按钮区域，按下鼠标右键，`edit`/`duplicate`/`delete`。

![image](../../png/software/12.png)

# 2. Linux命令

## 2.1 获得管理员权限

平常机器

```
su
```

云龙编译机

```
sudo su
```

## 2.2 刷新文件权限

详见 **<a href="../../engineering/linux/指令合集1.md" style="color: deepskyblue">Markdown</a>**。

每个数字代表一组权限：​**文件所有者**​、**文件所在的组** 和 ​**其他用户**​。每个数字可以是 0 到 7 之间的一个值，具体含义如下：

* **读（r）** 权限：值为 `4`
* **写（w）** 权限：值为 `2`
* **执行（x）** 权限：值为 `1`

这些权限可以加起来形成不同的值。例如：

* `7` = `4 (读) + 2 (写) + 1 (执行)`，表示读、写、执行权限都被赋予了。

**举例解释 `777`**

`777` 的每个数字都对应文件或目录的三组权限：

* ​**第一个 7**​：文件所有者的权限 (Owner)
* ​**第二个 7**​：文件所属组的权限 (Group)
* ​**第三个 7**​：其他用户的权限 (Others)

每个 `7` 代表文件或目录的读、写、执行权限都被赋予，因此 `777` 表示：

* ​**所有者**​：可以读、写、执行该文件或目录。
* ​**所属组**​：可以读、写、执行该文件或目录。
* ​**其他用户**​：可以读、写、执行该文件或目录。

常见命令

```
chmod 777 *
```

## 2.3 切换文件夹

到任意路径

```
cd 目标路径
```

到当前路径的上一级

```
cd ../
```

回到切换目录前的目录

```
cd -
```

## 2.4 过滤符

```
grep 目标内容 查找范围
```

```
zgrep 目标 查找范围 >目标文件路径
```

## 2.5 展示当前路径的下一级子文件(夹)

### 2.5.1 按行排列，并只展示文件(夹)

```
ls
```

### 2.5.2 按列排列，同时展示文件(夹)的详细信息

```
ll
```

## 2.6 执行bash文件

### 2.6.1 默认执行bash文件

```
bash example.sh
```

### 2.6.2 详细展示bash文件的执行过程

```
bash -x example.sh
```

## 2.7 查询当前系统中进程的IP

```
netstat -tluWn
```

`-t`：显示 TCP 连接（t = TCP）
`-l` ：只显示**监听（Listening）状态**的端口（不是所有连接）
`-u` ：显示 UDP 连接（u = UDP）
`-W`：防止输出被截断（W = wide，显示完整地址/端口名，避免省略号）
`-n`：以**数字**显示地址和端口，而不是尝试解析成域名和服务名（更快）

![image](../../png/software/13.png)

## 2.8 查询当前的进程

```
ps -ef
```

![image](../../png/software/14.png)

## 2.9 压缩文件

详见 **<a href="../../engineering/linux/解压&压缩.md" style="color: deepskyblue">Markdown</a>**。

### 2.9.1 打包为`archive.tar`

将 `folder_name` 文件夹打包成一个名为` archive.tar` 的归档文件，并在过程中显示详细信息。

```
tar -cvf archive.tar folder_name
```

- c：创建新的归档文件 (create)。表示将文件打包成一个归档文件。
- v：显示详细的输出 (verbose)。在执行过程中，列出归档中的文件。
- f：指定归档文件的名称 (file)。后面跟着归档文件的名称 archive.tar。

### 2.9.2 压缩为`usr.tar.gz`

将 `/home` 目录打包并压缩成 `usr.tar.gz `文件，同时显示详细信息。

```
tar czvf usr.tar.gz /home
```

- c：创建新的归档文件 (create)。
- z：压缩归档文件 (gzip)。表示使用 gzip 压缩归档文件，因此生成的是 .tar.gz 格式的压缩文件。
- v：显示详细的输出 (verbose)。
- f：指定归档文件的名称 (file)。这里是 usr.tar.gz。

## 2.10 列出`.tar`包中的所有文件

将列出 all.tar 归档文件中的所有文件和目录，但不会解压或修改它。

```
tar -tf all.tar
```

- t：列出归档内容 (list)。表示查看归档文件中的内容，而不解压或打包。
- f：指定归档文件的名称 (file)。这里是 all.tar。

## 2.11 解压文件

详见 **<a href="../../engineering/linux/解压&压缩.md" style="color: deepskyblue">Markdown</a>**。

### 2.11.1 解压`.tgz`/`.tar.gz`

将`.tgz`/`.tar.gz`文件解压在当前目录

```
tar zxvf  MY_NAME.tgz
tar zxvf  MY_NAME.tar.gz
```

- z：表示解压一个 gzip 压缩的归档文件 (gzip)。这个选项告诉 tar 解压文件时需要使用 gzip 解压程序，通常用于 .tgz 或 .tar.gz 格式的文件。
- x：表示解压归档文件 (extract)。这是解包的基本命令，用于提取归档中的文件。
- v：显示详细的输出 (verbose)。在解压的过程中会列出正在解压的文件。
- f：指定归档文件的名称 (file)。后面紧跟着归档文件的名称 MY_NAME.tgz。

### 2.11.2 解压`tar`

将解压 file.tar 文件，并显示解压的详细过程。file.tar 这里没有压缩，因此不需要使用 z 选项。

```
tar -xvf file.tar
```

- x：表示解压归档文件 (extract)。用于从归档文件中提取文件。
- v：显示详细的输出 (verbose)。解压时列出归档文件中的文件。
- f：指定归档文件的名称 (file)。后面紧跟着归档文件的名称 file.tar。

### 2.11.3 解压`.gz`

```
gunzip 文件.gz
```

## 2.12 节点/机器间传输文件

### 2.12.1 发送方主动给

```
scp 文件在当前节点的路径 用户名@目标IP:目标路径
```

示例：

```
scp /skt/faker/six.champion mtuser@192.168.2.2:/rng/faker/
```

### 2.12.2 接受方去拿

```
scp 用户名@目标IP:目标路径 本节点的保存路径
```

示例：

```
scp  mtuser@192.168.2.1:/skt/faker/six.champion .
```

## 2.13 echo

在终端中输出文本或变量的值。详见 **<a href="../../engineering/linux/echo.md" style="color: deepskyblue">Markdown</a>**。

```
echo "Hello, world!"
name="Alice"
echo "Hello, $name!"
```

得到结果

```
Hello, world!
Hello, Alice!
```

## 2.14 创建文件夹

```
mkdir -p 文件夹路径
```

- `-p`代表允许目标路径已存在。

## 2.15 vim操作

公司机器很多没法用`vim`，所以改用`vi`。

### 2.15.1 进文件

```
vi 文件名
```

### 2.15.2 删除文件内的所有内容

```
gg
dG
```

### 2.15.3 删除部分行

删除一行：

在命令模式下将光标移至要删除的行位置，按下dd

删除n行：

假设要删除5行，在命令模式下将光标移至要删除的行的开始位置，按下5dd

### 2.15.4 设置utf-8

```
:set encoding=utf-8
```

### 2.15.5 设置行号

```
set nu
```

### 2.15.6 编辑模式

按下键盘上的`Insert`，启用编辑模式。

按下键盘上的`Esc`，退出编辑模式。

### 2.15.7 退出文件

无操作后的退出

```
:q
```

舍弃修改后的强制退出

```
:q!
```

带有保存的退出

```
:wq
```

带有保存的强制退出

```
:wq!
```

## 2.16 sed -i

平常用的不多。详见 **<a href="../../engineering/linux/sed -i.md" style="color: deepskyblue">Markdown</a>**。

## 2.17 复制

```
cp -rf 原路径 目标路径
```

-r：表示递归复制。
-f：表示强制复制，如果目标文件已经存在，则会覆盖目标文件，不会提示确认。

## 2.18 移动

```
mv-rf 原路径 目标路径
```

-r：表示递归复制。
-f：表示强制复制，如果目标文件已经存在，则会覆盖目标文件，不会提示确认。

## 2.19 观测显存信息

直接查看一次

```
npu-smi info
```

独立窗口持续观测（`ctrl c`退出）

```
watch -n 1 npu-smi info
```

`watch` 是一个用于定期执行其他命令并显示其输出的工具。它的基本语法是：

```sh
watch [options] command
```

* `-n <seconds>`：指定每隔多少秒执行一次命令。例如，`-n 1` 表示每隔1秒执行一次。
* `command`：要定期执行的命令。

## 2.20 c++解码

服务死后的黑匣子、gdb的结果，这两处地方，全是各种函数语句的栈，需要解码为人类可读语句。

```
c++filt 编译符号
```

示例：

```
c++filt _ZSt8__invokeIRSt8functionIFvvEEJEENSt15__invoke_resultIT_JDpT0_EE4typeEOS5_DpOS6_
```

## 2.21 查看磁盘空间

显示目录或文件的磁盘使用情况，即显示目录或文件占用了多少磁盘空间。

```
du -sh /path/to/directory
```

- -h：以可读性更强的格式（如 KB、MB、GB）显示磁盘使用量。
- -s：显示指定目录的总大小，而不是递归列出每个文件和子目录。
- -a：列出目录和所有文件的大小，而不仅仅是目录。
- -c：显示所有列出的文件和目录的总和。

显示文件系统的磁盘空间使用情况。它显示的是整个文件系统的磁盘使用情况，包括已用空间、可用空间和挂载点等信息。

```
df -h
```

```
df -h .
```

- -h：以人类可读的格式（KB、MB、GB）显示。
- -T：显示文件系统的类型。
- -a：显示所有文件系统，包括 0 字节空间的挂载。
- -i：显示 inode 信息，而不是磁盘空间。

## 2.22 查找文件

```
find 查找范围 -name 关键词
```

示例1：

```
find / -name llmservice.log 2>/dev/null
```

# 3. Git指令

详见 **<a href="../../git/operation/常见git.md" style="color: deepskyblue">Markdown</a>**。

#linux命令总结 #总结帖 #命令总结 #linux #MobaXterm操作总结