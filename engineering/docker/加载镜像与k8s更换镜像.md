# 1. 草图

![image](../../png/engineering/25.png)

![image](../../png/engineering/26.png)

# 2. 过程代码

```
docker load -i 镜像名称
```

```
docker images | grep 镜像名称
```

```
docker tag 镜像ID:版本号 镜像地址(docker images查询  地址+服务+版本号)
```

```
docker push 镜像地址
```

# 3. n1951中所用的代码

```
docker load -i n1951-driver_26.9.9.B030.tar.gz
```

```
docker images | grep n1951
```

```
docker tag n1951-driver:26.9.9.B030 192.168.2.2:20202/op_svc_pom/n1951-driver:26.9.9.B030
```

```
docker push 192.168.2.2:20202/op_svc_pom/n1951-driver:26.9.9.B030
```

```
kubectl edit ds -n fst-manage n1951-driver
```

最后更换镜像地址：

```
192.168.2.2:20202/op_svc_pom/n1951-driver:26.9.9.B030
```

#手动换镜像 #换镜像 #更换镜像 #镜像 #docker命令