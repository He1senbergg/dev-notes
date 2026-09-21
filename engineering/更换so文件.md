# 1. 起特权容器

```
docker exec -it --privileged -u 0
```

# 2. 到框架so路径(`/opt/coremind/lib`)的上一级

把需要更换的so移动到`/opt/coremind`

**给目录和文件加上写权限后再移动：**

```
cd /opt/coremind
sudo chmod u+w lib
sudo chmod u+w lib/libcm_framework_cms_plugin.so
sudo mv libcm_framework_*.so lib/
```

**移动完后可以恢复安全权限：**

```
sudo chmod u-w lib
sudo chmod u-w lib/libcm_framework_cms_plugin.so
```

# 3. 更改权限

```
cd lib
ll
```

会发现新拷入的so文件，与原始的其它so文件的读写权限不一致

![image](../png/engineering/22.png)

修改为`-r-xr-x---`

```
r(4) + x(1) = 5
r(4) + x(1) = 5
--- = 0
=> 550
```

**执行命令**

```
chmod 550 libcm_framework_*.so
```

刷新读写权限成功
![image](../png/engineering/23.png)

#框架so #框架 #so文件 #更改so权限 #更改so文件 #Read-only file system