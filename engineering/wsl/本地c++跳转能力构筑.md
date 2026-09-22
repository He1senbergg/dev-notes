**WSL->Ubuntu22.04**

---

# 0. VSCode设置复制|粘贴

按下`ctrl`+`,`，再键入下面的内容，开启选中即复制。

```
terminal copy
```

![image](../../png/engineering/57.png)

此时，选中即复制已经有了，但是右键不确定是否一定有。

再键入以下内容，观察一下是不是`copyPaste`。

```
Right Click Behavior
```

![image](../../png/engineering/58.png)

# 1. 下载并安装Ubuntu22.04

**安装Ubuntu时，注意路径，防止出现我在 `#17. wsl的Ubuntu迁移磁盘` 的窘境。**

----

[wsl-Ubuntu22.04](https://aka.ms/wslubuntu2204)

下载完，把`Ubuntu2204-221101.AppxBundlle`重命名为`Ubuntu.zip`，解压开。

![image](../../png/engineering/59.png)

安装Ubuntu_2204.1.7.0_x64.appx。

安装完成，点Launch，会弹出黑框，设置一下用户名、密码。

# 2. 下载wsl中的Ubuntu22.04所需的依赖

注：该步是因为，没能解决wsl中的Ubuntu22.04的联网问题，联网下载失败。

[Ubuntu22-04-Deb.zip](../../installer/linux/Ubuntu22-04-Deb.zip)

![image](../../png/engineering/60.png)

本地Windows下载这个zip，然后打开文件管理器(快捷键：`Win`+`E`)，键入

```
\\wsl$\Ubuntu\home\heisenberg\
```

其中，`heisenberg`改成自己设置的用户名。

安装这四个文件。

```
sudo dpkg -i openssh-client_8.9p1-3ubuntu0.13_amd64.deb
sudo dpkg -i openssh-sftp-server_8.9p1-3ubuntu0.13_amd64.deb
sudo dpkg -i libwrap0_*.deb
sudo dpkg -i openssh-server_8.9p1-3ubuntu0.13_amd64.deb
```

执行第四行之后，可能会出现高亮的新屏幕，选下面这个选项就行，没有就拉倒。

`keep the local version currently installed`

# 3. 设置wsl中的Ubuntu22.04的ssh

```
sudo vim /etc/ssh/sshd_config
```

输入密码，并键入内容。其中，Port改成自己要的端口，Allowusers改成设置的用户名

```
Port 10246
PasswordAuthentication yes
Allowusers heisenberg
Subsystem sftp /usr/lib/openssh/sftp-server
```

保存退出，启动ssh。

```
sudo service ssh start
```

验证是否启动成功。用户名(heisenberg)、端口(10246)，自己改一下。

```
sudo ss -tlnp | grep ssh
sftp -P 10246 heisenberg@127.0.0.1
```

出现下面的界面，ssh就设置成功了。

![image](../../png/engineering/61.png)

# 4. VSCode-huawei用wsl连接Ubuntu

![image](../../png/engineering/62.png)

![image](../../png/engineering/63.png)

# 5. 解决wsl联网问题

(**华为内部的解决方案，外部需要回归使用Fiddler**)

---

安装EasyProxy（外部没这个软件可用），以本地Windows跳板，给wsl中的ubuntu代理联网。

![image](../../png/engineering/64.png)

![image](../../png/engineering/65.png)

如果EasyProxy启动失败，就说明端口不行，换个端口就行，比如说18080。

在wsl的ubuntu终端键入

```
ip route | grep default
```

得到代理IP`default via 172.23.48.1 dev eth0 proto kernel`，即`172.23.48.1`。

![image](../../png/engineering/66.png)

根据EasyProxy开启的监听端口，拼接出代理IP。

```
172.23.48.1:3128
```

设置代理环境变量

```
sudo vim ~/.bashrc
```

打开文件后，键入以下内容。IP和端口修改成自己设置的。

```
export http_proxy=http://172.23.48.1:3128
export https_proxy=http://172.23.48.1:3128
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy

# ===== 内网不走代理 =====
export NO_PROXY="localhost,127.0.0.1,codehub-dg-y.huawei.com,.huawei.com,100.0.0.0/8"
export no_proxy="$NO_PROXY"
```

激活生效。

```
source ~/.bashrc
```

此时，git已经能连接到CodeHub，但是证书校验失败。

```
git clone https://codehub-dg-y.huawei.com/CoreMind/usrservice/IMInferService.git
```

![image](../../png/engineering/67.png)

把root用户里的联网也设置一下代理。

```
sudo su
sudo vim ~/.bashrc
```

打开文件后，键入以下内容。IP和端口修改成自己设置的。

```
export http_proxy=http://172.23.48.1:3128
export https_proxy=http://172.23.48.1:3128
export HTTP_PROXY=$http_proxy
export HTTPS_PROXY=$https_proxy

# ===== 内网不走代理 =====
export NO_PROXY="localhost,127.0.0.1,codehub-dg-y.huawei.com,.huawei.com,100.0.0.0/8"
export no_proxy="$NO_PROXY"
```

激活生效。

```
source ~/.bashrc
```

# 6. 解决wsl证书校验问题

打开Windows的PowerShell

```
$Out = "$pwd\all-root-certs.pem"
Remove-Item $Out -ErrorAction SilentlyContinue
Get-ChildItem Cert:\LocalMachine\Root | ForEach-Object {
"-----BEGIN CERTIFICATE-----" | Out-File -FilePath $Out -Append -Encoding ascii
[System.Convert]::ToBase64String($_.RawData, 'InsertLineBreaks') | Out-File -FilePath $Out -Append -Encoding ascii
"-----END CERTIFICATE-----`r`n" | Out-File -FilePath $Out -Append -Encoding ascii
}
```

它会在当前目录生成`all-root-certs.pem`。

把文件拷贝到wsl的路径下，再在wsl执行。

```
mkdir -p ~/ca_tmp
cd ~/ca_tmp
csplit -f cert- -b "%03d.crt" ../all-root-certs.pem '/-----BEGIN CERTIFICATE-----/' '{*}' >/dev/null 2>&1 || true

# 删掉可能为空的第一个分片

[ -s cert-000.crt ] || rm -f cert-000.crt
sudo cp cert-*.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

清空老环境变量

```
# 清空可能的 Git SSL CA 配置

git config --global --unset-all http.sslCAInfo 2>/dev/null || true
git config --global --unset-all http.sslCAPath 2>/dev/null || true
git config --global --unset-all http.sslVerify 2>/dev/null || true

# 清空环境变量（仅当前终端）

unset GIT_SSL_CAINFO GIT_SSL_CAPATH SSL_CERT_FILE SSL_CERT_DIR CURL_CA_BUNDLE REQUESTS_CA_BUNDLE
```

导入新证书的环境变量

```
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
```

Windows上，`Win`+`R`，再键入

```
certmgr.msc
```

依次点击界面：中级证书颁发机构->证书，找到`HWIT Enterprise CA 1`。右键，导出证书。选择`Base-64 编码 X.509 (.CER)`，保存为`HWIT-Enterprise-CA-1.cer`。

复制到wsl，再加入信任链。

```
sudo cp HWIT-Enterprise-CA-1.cer /usr/local/share/ca-certificates/HWIT-Enterprise-CA-1.crt
sudo update-ca-certificates
```

# 7. 更新Ubuntu工具

将ubuntu的镜像源替换为华为镜像源，先备份原始的，再`ggdG`清空老文件，粘贴下面的四行。

```
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo vim /etc/apt/sources.list
ggdG
```

```
deb [arch=amd64] https://mirrors.tools.huawei.com/ubuntu/ jammy main restricted universe multiverse
deb [arch=amd64] https://mirrors.tools.huawei.com/ubuntu/ jammy-updates main restricted universe multiverse
deb [arch=amd64] https://mirrors.tools.huawei.com/ubuntu/ jammy-backports main restricted universe multiverse
deb [arch=amd64] https://mirrors.tools.huawei.com/ubuntu/ jammy-security main restricted universe multiverse
```

添加官方源（用来支持arm64）

```
sudo tee /etc/apt/sources.list.d/ubuntu-official.list >/dev/null <<'EOF'
deb [arch=arm64] https://ports.ubuntu.com/ubuntu-ports jammy main restricted universe multiverse
deb [arch=arm64] https://ports.ubuntu.com/ubuntu-ports jammy-updates main restricted universe multiverse
deb [arch=arm64] https://ports.ubuntu.com/ubuntu-ports jammy-backports main restricted universe multiverse
deb [arch=arm64] https://ports.ubuntu.com/ubuntu-ports jammy-security main restricted universe multiverse
EOF
```

强制IPv4。

```
sudo tee /etc/apt/apt.conf.d/99force-ipv4 >/dev/null <<'EOF'
Acquire::ForceIPv4 "true";
EOF
```

apt配置本地Windows端口代理，此处IP与端口需使用**5. 解决wsl联网问题**中获得的那个。

```
sudo tee /etc/apt/apt.conf.d/99proxy >/dev/null <<'EOF'
Acquire::http::Proxy  "http://172.23.48.1:3128/";
Acquire::https::Proxy "http://172.23.48.1:3128/";
EOF
```

关闭apt的ssl验证

```
sudo nano /etc/apt/apt.conf.d/99ssl-ignore
```

加入以下内容后，保存并关闭。

```
Acquire::https::Verify-Peer "false";
Acquire::https::Verify-Host "false";
```

更新sudo apt工具

```
sudo apt update
```

安装`dos2unix`

```
sudo apt install dos2unix
```

安装c++编译所需工具

```
sudo apt install build-essential cmake make gcc g++ unzip
sudo apt install clang lld
sudo apt install clangd
```

下载并解包arm64的python

```
sudo apt-get download python3.10-dev:arm64
sudo apt-get download libpython3.10-dev:arm64
sudo mkdir -p /opt/arm64-sysroot
sudo dpkg-deb -x python3.10-dev_*_arm64.deb /opt/arm64-sysroot
sudo dpkg-deb -x libpython3.10-dev_*_arm64.deb /opt/arm64-sysroot
```

# 8. 构建编译条件

到构建工具目录，创建一下BiSheng_native。

```
cd /opt/buildtools
mkdir -p BiSheng_native
```

切root用户，这个目录
```
sudo su
```

下载BiSheng

```
wget -O BiShengCompiler-4.0.0-x86-linux.tar.gz https://mirrors.huaweicloud.com/kunpeng/archive/compiler/bisheng_compiler/BiShengCompiler-4.0.0-x86-linux.tar.gz
wget -O BiShengCompiler-4.0.0-x86-linux.tar.gz.sha256 https://mirrors.huaweicloud.com/kunpeng/archive/compiler/bisheng_compiler/BiShengCompiler-4.0.0-x86-linux.tar.gz.sha256
```

校验sha值

```
sha256sum -c BiShengCompiler-4.0.0-x86-linux.tar.gz.sha256
```

![image](../../png/engineering/68.png)

解压BiSheng

```
tar -xzf BiShengCompiler-4.0.0-x86-linux.tar.gz -C /opt/buildtools/BiSheng_native --strip-components=1
```

解压完成，检查一下。

```
/opt/buildtools/BiSheng_native/bin/clang++ --version
```

![image](../../png/engineering/69.png)

放开权限给非root用。

```
chmod -R 777 *
```

# 9. 构造交叉编译能力（X86编译ARM）

安装交叉编译必需包

```
sudo apt-get update
sudo apt-get install -y \
  gcc-aarch64-linux-gnu g++-aarch64-linux-gnu \
  binutils-aarch64-linux-gnu \
  libc6-dev-arm64-cross \
  pkg-config
```

看一下安装结果，大概率是`/usr/aarch64-linux-gnu/`。

```
dpkg -L libc6-dev-arm64-cross | head -n 20
```

![image](../../png/engineering/70.png)

创建交叉编译 CMake toolchain 文件。（修改其中的**用户名** / **仓名**）

```
mkdir -p /home/heisenberg/IMInferService/toolchains
cat > /home/heisenberg/IMInferService/toolchains/aarch64-gcc.cmake <<'EOF'
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER   /usr/bin/aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER /usr/bin/aarch64-linux-gnu-g++)

set(CMAKE_AR      /usr/bin/aarch64-linux-gnu-ar)
set(CMAKE_RANLIB  /usr/bin/aarch64-linux-gnu-ranlib)
set(CMAKE_STRIP   /usr/bin/aarch64-linux-gnu-strip)

# 避免捡到主机(x86)库/头文件

set(CMAKE_FIND_ROOT_PATH
/usr/aarch64-linux-gnu
/usr/lib/gcc-cross/aarch64-linux-gnu
${CMAKE_SOURCE_DIR}
)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
EOF
```

# 10. 修改编译文件

修改最外层的`CMakeLists.txt`

```
# 如果外部已经指定了 CMAKE_TOOLCHAIN_FILE / CMAKE_CXX_COMPILER，就不要强行覆盖

# （这是交叉编译的关键）

if(NOT DEFINED CMAKE_TOOLCHAIN_FILE AND NOT DEFINED CMAKE_C_COMPILER AND NOT DEFINED CMAKE_CXX_COMPILER)
message("*** I am using the BiSheng compiler ***")
set(COMPILER_EXEC_PATH /opt/buildtools/BiSheng_native/bin)
set(CMAKE_C_COMPILER   ${COMPILER_EXEC_PATH}/clang)
set(CMAKE_CXX_COMPILER ${COMPILER_EXEC_PATH}/clang++)
set(GSP_PKG_LINKER     ${COMPILER_EXEC_PATH}/ld.lld)
else()
message(STATUS "Use external toolchain/compiler settings (cross-compile friendly).")
endif()
```

```
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
# 适配X86跳转岛
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-T${PROJECT_SOURCE_DIR}/build/ld.lds")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm|aarch64")
# 适配ARM跳转岛
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-T${PROJECT_SOURCE_DIR}/build/ld.lds")
else()
message(STATUS "Unknown platform: ${CMAKE_SYSTEM_PROCESSOR}")
endif()
```

修改内层的`CMakeLists.txt`，对于会报错Python头文件找不到的文件夹，给`target_include_directories()`添加一句。

```
PUBLIC /opt/arm64-sysroot/usr/include/python3.10
PUBLIC /opt/arm64-sysroot/usr/include
```

修改`compile.sh`

写死**aarch64**。

```
CPU_ARCH="aarch64"
export CPU_ARCH
echo "[local-build] force CPU_ARCH=$CPU_ARCH"
```

注释掉读取，使得前面写死的arm架构能生效，不然编译脚本里的if(arm)会进不去。

```
# CPU_ARCH=$(uname -p)
```

修改编译函数

```
function compile() {
    # 编译
    export BUILD_CPU="yes"
    cd $ROOT_PATH
    rm -rf build-arm
    cmake -S . -B build-arm 
        -DCMAKE_TOOLCHAIN_FILE=toolchains/aarch64-gcc.cmake 
        -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=bfd" 
        -DCMAKE_SHARED_LINKER_FLAGS="-fuse-ld=bfd" 
        -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    cmake --build build-arm -j

    # cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=${CUR_PATH} \
    #   -DCMAKE_C_COMPILER=/usr/bin/gcc \
    #   -DCMAKE_CXX_COMPILER=/usr/bin/g++ .
    # make -j 8 VERBOSE=1
    # make install
}
```

# 11. 执行编译

```
bash -x download.sh
bash -x compile.sh
```

# 12. 开启跳转能力

安装插件`Clangd`。

![image](../../png/engineering/71.png)

安装完成之后，`ctrl`+`shift`+`p`唤起快捷菜单，键入`setting`。

![image](../../png/engineering/72.png)

![image](../../png/engineering/73.png)

`setting.json`中键入内容

```
"clangd.arguments": [
"--compile-commands-dir=${workspaceFolder}/.vscode"
],
```

还有`CMake Tool`工具，安装一下。设置里也补一下，最后是：

```
{
    "cmake.buildDirectory": "${workspaceFolder}/.vscode",
    "cmake.configureSettings": {
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
    },
    "clangd.arguments": [
        "--compile-commands-dir=${workspaceFolder}/.vscode"
    ]
}
```

---

两种方式任选一种，完成之后，按下`shift`+`ctrl`+`p`，键入

```
reload windows
```

![image](../../png/engineering/74.png)

设置加载模式为编译模式。

![image](../../png/engineering/75.png)

完成之后，查看Git页面，会发现很多CMakeLists.txt的记录(因为前面修改了)，但是不希望提交，那就参考Wiki处理一下。

- [Git忽略本地修改](../../git/operation/Git忽略本地修改.md)
- [Git忽略本地新文件](../../git/operation/Git忽略本地新文件.md)

# 13. 宏定义强制跳转引用

VSCode的默认操作逻辑：

- `ctrl` + `left Click` -> 跳转至**定义** / **实现**
- `shift` + `F12` -> 跳转至**引用**

但CLion带来的习惯是，`ctrl` + `left Click` -> 跳转至**定义**/**实现**/**引用**。

安装AutoHotkey。

[AutoHotkey_2.0.21_setup.exe](../../installer/AutoHotkey_2.0.21_setup.exe)

随便找个地方，以右键新建一个文本文档，重命名为 `VSCode_GoToReferences.ahk`。后缀是`ahk`就行，名字随便。

编辑文件，写入如下内容，保存。

```
#Requires AutoHotkey v2.0

#HotIf WinActive("ahk_exe VSCode-huawei.exe")

; 使用 * 修饰符确保拦截更彻底，防止逻辑穿透
*^LButton::
{
; 1. 记录位置
MouseGetPos &x, &y

; 2. 【核心修改】强制解除物理 Ctrl 的按下状态信号
; 即使你手按着，也要让 Windows 认为 Ctrl 弹起来了
if GetKeyState("Ctrl")
    Send "{Blind}{Ctrl up}"

; 3. 执行纯粹的左键点击来定位光标
; 这里不再使用 ControlClick，因为它在某些 Electron 应用中不稳定
Click x, y

; 4. 等待光标定位完成（如果还闪现，可以调大到 100）
Sleep 50

; 5. 发送跳转引用的快捷键
Send "+{F12}"

; 6. 恢复 Ctrl 状态，否则你后续按 Ctrl+C/V 会失效
if GetKeyState("Ctrl", "P")
    Send "{Blind}{Ctrl down}"
}

#HotIf
```

双击启动。

-----

**在有需要的情况下，把宏定义设置为开机自启动。**

打开AutoHotkey Dash，点击Compile，它会安装编译`.ahk`文件的编译器。

![image](../../png/engineering/76.png)

Browse`.ahk`文件的路径，点击Convert即可，就会在`.ahk`文件的同级得到`.exe`文件。

![image](../../png/engineering/77.png)

右键图标栏的win标志，点击运行。

![image](../../png/engineering/78.png)

键入内容。

```
shell:startup
```

![image](../../png/engineering/79.png)

打开文件管理器之后，把刚刚的exe放进来。

![image](../../png/engineering/80.png)

# 14. wsl的Ubuntu完善git

```
git config --global credential.helper store
cd 仓名的文件夹
git pull
```

此时会弹窗，要求键入用户名（工号）和密码。Git会保存下来登录信息。

检查一下记录是否成功。

```
vi ~/.git-credentials
```

![image](../../png/engineering/81.png)

再pull一把看看是否不用再键入登录信息了。

```
git pull
```

如果图形界面里拉取git仓库的时候，出现网络报错504，那就是git走了啥特定的代理，别弄那些，就保持之前在wsl里设置的走本地Windows的代理就可以了。

```
git config --global http.https://codehub-dg-y.huawei.com.proxy ""
git config --global --get http.https://codehub-dg-y.huawei.com.proxy
```

这样子就正常了。

![image](../../png/engineering/82.png)

# 15. 后续的仓的跳转能力

**只需要执行一次的操作**

---

对于每个仓要做的操作。

## 15.1 手动版

```
cd ~/{代码仓名}
cmake -S . -B ./.vscode -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

## 15.2 插件版

安装**CMake Tools**。

![image](../../png/engineering/83.png)

`ctrl`+`shift`+`p`唤起快捷菜单，键入`setting`。

![image](../../png/engineering/84.png)

![image](../../png/engineering/85.png)

`setting.json`中键入内容

```
"cmake.exportCompileCommandsFile": true,
"cmake.copyCompileCommands": "${workspaceFolder}/.vscode/compile_commands.json",
"cmake.buildDirectory": "${workspaceFolder}/build-arm",
```

---

两种方式任选一种，完成之后，VSCode就自己加载跳转链接能力了。

如果报错缺失python，就安装一下。

```
sudo apt-get update
sudo apt-get install -y python3.10-dev
```

安装完可以自检一下。

```
python3 -V
which python3
```

![image](../../png/engineering/86.png)

再重跑一下（或重载一下VSCode界面，等插件自己完成）。

```
rm -rf ./.vscode
cmake -S . -B ./.vscode -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

# 16. 后续的仓的本地编译能力

建议在`build`同级，直接复制一个`local-build`目录。在`local-build`去修改本地的编译脚本，去弄编译等行为。如果确定要git commit，再粘贴到`build`目录里的同名文件去。

修改`local-build`目录的`compile.sh`，看着增加一下，弄下面的代码。

**同架构**

```
set -e
CUR_PATH=$(dirname $(readlink -f $0))
ROOT_PATH=${CUR_PATH}/..
BUILD_DIR=${ROOT_PATH}/local-build

function compile() {
    export BUILD_CPU="yes"
    mkdir -p "${BUILD_DIR}"
    cmake -S "${ROOT_PATH}" -B "${BUILD_DIR}" \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX="${CUR_PATH}"
    cmake --build "${BUILD_DIR}" -j 8 --verbose
    cmake --install "${BUILD_DIR}"
}
```

**异架构**

```
function compile() {
    # 编译
    export BUILD_CPU="yes"
    cd $ROOT_PATH
    rm -rf build-arm
    cmake -S . -B build-arm \
        -DCMAKE_TOOLCHAIN_FILE=toolchains/aarch64-gcc.cmake \
        -DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=bfd" \
        -DCMAKE_SHARED_LINKER_FLAGS="-fuse-ld=bfd" \
        -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    cmake --build build-arm -j
}
```

# 17. wsl的Ubuntu迁移磁盘

示例为迁移到 D 盘。

确认当前 WSL 名称，在 PowerShell 执行：

```powershell
wsl -l -v
```

你会看到类似：

```
NAME      STATE           VERSION
* Ubuntu    Running         2
```

记住 ​**NAME**​，假设是 `Ubuntu`（后面都以它为例）。

彻底关闭 WSL

```powershell
wsl --shutdown
```

必须执行，否则导出可能失败。

导出当前 Ubuntu（生成备份文件）

在 D 盘准备一个临时目录，比如：

```powershell
mkdir D:\WSL_Backup
```

然后导出：

```powershell
wsl --export Ubuntu D:\WSL_Backup\ubuntu.tar
```

说明：

* 这是完整系统镜像
* 可能需要几分钟
* 生成的 tar 文件大小≈你当前真实使用空间

完成后确认文件存在。

注销原 Ubuntu（⚠重要）

**确认 tar 文件已经生成后**执行：

```powershell
wsl --unregister Ubuntu
```

这一步会：

* 删除 C 盘的 ext4.vhdx
* 立即释放空间

执行完再运行：

```powershell
wsl -l -v
```

应该看不到 Ubuntu。

再在 D 盘重新导入，创建新的安装目录：

```powershell
mkdir D:\WSL\Ubuntu
```

然后执行：

```powershell
wsl --import Ubuntu D:\WSL\Ubuntu D:\WSL_Backup\ubuntu.tar --version 2
```

解释：

* 第一个 Ubuntu = 新实例名字
* 第二个路径 = 新系统存储位置
* 第三个是刚才导出的 tar

完成后执行：

```powershell
wsl -l -v
```

应该看到 Ubuntu，VERSION=2。

确认系统能正常启动：

```powershell
wsl
```

能进 Ubuntu 后，可以删除：

```
D:\WSL_Backup\ubuntu.tar
```

释放空间。

确认已经在 D 盘，检查：

```
D:\WSL\Ubuntu\
```

里面应该有：

```
ext4.vhdx
```

说明已经成功迁移。

迁移后效果

| 项目              | 状态     |
| ------------------- | ---------- |
| C 盘空间          | 立刻释放 |
| WSL 运行方式      | 完全一样 |
| 环境              | 完整保留 |
| Docker / 编译缓存 | 全保留   |

当前wsl的Ubuntu界面，应该是以root进入的。如果不是，root一下。

```
sudo su
```

再执行一下默认用户切换，在Ubuntu（root）后执行：

```
# 用户名修改成之前自己设置的
usermod -aG sudo heisenberg
```

退出Ubuntu，在PowerShell执行

```
# 用户名修改成之前自己设置的
ubuntu config --default-user heisenberg
```

此时，迁移磁盘之后的Ubuntu默认用户就已经修改好了。

#VSCode编译跳转能力 #VSCode跳转能力 #VSCode编译能力 #VSCode-huawei编译跳转能力 #VSCode-huawei跳转能力 #VSCode-huawei编译能力 #编译跳转能力 #编译能力 #跳转能力