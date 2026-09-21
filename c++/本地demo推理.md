# 1. 准备代码仓

## 1.1 在IDE中设置编译机器

比如说
IP：xxx.xx.xx.xxx
USER/PWD：root/xxxxxxxxxx

准备代码文件与依赖文件，上传到编译机。

# 2. 用容器隔离

拉起一个容器。

进入容器

```
docker ps
```

```
docker exec -itu 0 --privileged {docker_id} bash
```

# 3. 加载环境变量

## 3.1 部门910B机器特供

机器
IP：xx.xx.xxx.xx   端口xxxx
USER/PWD：xxxx/xxxxxxxx

### 3.1.1 修改

1> 上面有miniconda的软链接，需要去掉一下

```
sudo mv /usr/local/bin/python /usr/local/bin/python_miniconda_backup
```

2> 修改根目录下的`CMakeLists.txt`中的编译器选项

```
原始
set(COMPILER_EXEC_PATH /opt/buildtools/BiSheng_native/bin)

修改为
set(COMPILER_EXEC_PATH /home/usrname/BiSheng_arm64le_native/bin)
```

3> 在根目录下的`CMakeLists.txt`中添加python路径

```
include_directories(/home/usrname/llm/python/include/python3.11)
link_directories(/home/usrname/llm/python/lib)
```

4> 删除`demo/config.json`里的这行

```
"plugin_params": "{\"plugin_type\":\"prefix_cache\"}",
```

### 3.1.2 寻找python中的包

示例代码

```
/home/usrname/llm/python/bin/python3.11 -m pip show torchvision
```

### 3.1.3 导入环境变量

1> `npu-smi info`报错缺失`libdrvdsmi_host.so`，导入环境变量补上

```
export LD_LIBRARY_PATH=/usr/local/Ascend/driver/lib64/driver:$LD_LIBRARY_PATH
```

### 3.1.4 恢复

1> 恢复软链接

```
sudo mv /usr/local/bin/python_miniconda_backup /usr/local/bin/python
```

## 3.2 共用-310P机器-910B机器

```
export PYTHONIOENCODING=utf-8
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/llm/python/lib/

export PYTHONPATH=/home/usrname/llm/package/atb_llm:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/attrs:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/decorator:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/llm_manager:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/mindformers:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/mindie_llm:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/psutil:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/safetensors:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/mindspore:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/numba:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/prorobuf:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/pytorch:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/saie_posix:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/saie_pydantic:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/saie_packaging:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/saie_pytz:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/scipy:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/sympy:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/torch-npu:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/transformers:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/torchvision:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/einops:$PYTHONPATH
export PYTHONPATH=/home/usrname/llm/package/numpy:$PYTHONPATH

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/mpu/software/ascend-toolkit/8.3.RC1/aarch64-linux/lib64/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/mpu/software/ascend-toolkit/8.3.RC1/aarch64-linux/devlib/linux/aarch64/

source /home/usrname/mpu/software/ascend-toolkit/set_env.sh
source /home/usrname/mpu/software/nnal/atb/set_env.sh
source /home/usrname/llm/software/MindIE-llm/set_env.sh
source /home/usrname/llm/software/Atb-Models/set_env.sh


export PATH=$PATH:/home/usrname/llm/package/mindie_llm/bin/

```

**如下的这四个，需要改成自己刚刚在`1.3 登录编译机，获取依赖文件`中所传的文件的路径。**

```
export PYTHONPATH=/home/usrname/IMInferRT-llm/src/tokenizer/src:$PYTHONPATH
export PYTHONPATH=/home/usrname/IMInferRT-llm/src/llm/src:$PYTHONPATH
export PYTHONPATH=/home/usrname/IMInferRT-llm/src/utils/src:$PYTHONPATH
```

---

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/publish/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/jsoncpp/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/UMOMFrmCpp/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/siteai/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/pybind11/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/securec/
```

---

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/publish/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/jsoncpp/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/IMInferRT-llm/platform/UMOMFrmCpp/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/platform/siteai/
```

---

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/publish/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/platform/jsoncpp/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/platform/securec/lib/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/platform/siteai/
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/rt/platform/UMOMFrmCpp/lib/
```

```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/usrname/framework/platform/openssl/lib/

```

910B机器上不需要下面这行。310P是否需要这行，不知道，没验证过。

```
export PYTHONPATH=$PYTHONPATH:/usr/local/lib64/python3.9/site-packages/
```

# 4. demo文件位置

在代码仓的根目录一级，有个文件夹叫`demo`，其中有三个文件。

## 4.1 修改文件

### 4.1.1 修改`demo\CMakeLists.txt`

![image](../png/c++/3.png)

### 4.1.2 修改根目录一级的CMakeLists


![image](../png/c++/4.png)

![image](../png/c++/5.png)

### 4.1.3 修改`demo\config.json`

将模型位置改为实际使用的位置

![image](../png/c++/6.png)

### 4.1.4 修改`build\package.sh`

将这些打包的过程全部注释、去掉。（因为已在`1.3 登录编译机，获取依赖文件`中完成下载）

![image](../png/c++/8.png)

### 4.1.5 修改`demo\main.cpp`

将`llmInstanceConfig.configPath`修改为自己的代码仓路径里的demo文件夹下的`config.json`路径。

![image](../png/c++/9.png)

### 4.1.6 可选修改项

注释掉键入再回车启动的过程

![image](../png/c++/10.png)

添加辅助定位的注释

![image](../png/c++/11.png)

修改显卡调用

`infer_model.cpp`中注释掉

```
InferRtCustomConfig::Instance()->GetDeviceId(deviceIds);
````

把`deviceIds`直接赋值

```
std::vector<std::set<size_t>> deviceIds = {{7}};
```

![image](../png/c++/12.png)

# 5. 编译代码

```
cd {代码仓位置}/build
./local-build.sh
cd ../demo
```

# 6. 执行demo

## 6.1 执行

```
./inferrt_demo
```

或以gdb形式启动

```
gdb ./inferrt_demo
键入 run并回车
当出现报错停顿时，键入bt
```

gdb退出为`ctrl d`，再`y`

## 6.2 预期结束时现象

![](../png/c++/13.png)

# 7. 报错信息指南

## 7.1 没加NPU环境变量

问题现象

![](../png/c++/14.png)

```
npu-smi info
```

```
npu-smi: error while loading shared libraries: libdrvdsmi_host.so: cannot open shared object file: No such file or directory
```

执行

```
export LD_LIBRARY_PATH=/usr/local/Ascend/driver/lib64/driver:$LD_LIBRARY_PATH
```

![](../png/c++/15.png)

## 7.2 mindie_llm_backend failed: Permission denied

![](../png/c++/16.png)

![](../png/c++/17.png)

```
ls -l /home/usrname/llm/package/mindie_llm/bin/mindie_llm_backend
```

会看到

```
-rw-r--r--
```

执行

```
chmod +x /home/usrname/llm/package/mindie_llm/bin/mindie_llm_backend
```

## 7.3 CMake报错

### 7.3.0 问题现象

![](../png/c++/17.png)

### 7.3.1 根因定位

```
python3 -c "import sys; print(sys.executable)"
python3 -c "import sysconfig; print(sysconfig.get_config_var('LIBDIR'))"
python3 -c "import sysconfig; print(sysconfig.get_config_var('LIBPL'))"
```

```
/usr/local/bin/python3
/usr1/jenkins/MPUDeployTool/dependency/python/lib
/usr1/jenkins/MPUDeployTool/dependency/python/lib/python3.11/config-3.11-aarch64-linux-gnu
```

夹杂了错误的`/usr1/jenkins/MPUDeployTool/dependency`

### 7.3.2 解决方案

去云龙流水线的机器，随便找个，打包它的3.11.4的python

```
cd /opt/buildtools
tar czf python311_clean.tar.gz python-3.11.4
```

复制到新机器

```
scp pipeline:/opt/buildtools/python311_clean.tar.gz /opt/buildtools/
cd /opt/buildtools
rm -rf python-3.11.4
tar xzf python311_clean.tar.gz
```

```
alias python3=/opt/buildtools/python-3.11.4/bin/python3.11
export PATH=/opt/buildtools/python-3.11.4/bin:$PATH
```

到根目录，删除`CMakeCache.txt`

```
rm -rf CMakeFiles/ CMakeCache.txt Makefile cmake_install.cmake shm_name.txt
```

----

#本地demo #demo #Demo