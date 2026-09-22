# 找到节点

```
kubectl get nodes --show-labels | grep 310P
```

进到节点后，重启calculator容器。

![image](../../png/engineering/24.png)

# 重装驱动

```
./Ascend-hdk-310p-npu-driver_25.5.0_linux-aarch64-custom-custom.run --full --quiet --install-username=paas --install-usergroup=paas --install-path=/usr/local/mpu/driver
```

#pending #NPU #与NPU相关的pending #npu #与npu相关的pending