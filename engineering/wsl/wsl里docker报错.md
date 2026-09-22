# 错误现象

wsl里的docker run报错

```
Input/output error
```

# 解决方案

- 退出wsl，`ctrl d`
- 在 Windows 右下角系统托盘：
    ```
    右键 Docker Desktop 图标
     → Quit Docker Desktop
    ```
- 在 Windows CMD 或 PowerShell 中执行
    
    docker 强制重启

    ```
    docker desktop status
    docker desktop stop
    ```

    ```
    wsl --shutdown
    ```

    看下全停了没

    ```
    wsl -l -v
    ```

    应该是类似如下

    ```
    NAME              STATE     VERSION
    Ubuntu            Stopped   2
    docker-desktop    Stopped   2
    ```
    
- 重新进入wsl
- 检查挂载文件
    ```
    /mnt/wsl/docker-desktop/cli-tools/usr/bin/docker version
    ```

#wsl #WSL #Docker #DOCKER #容器