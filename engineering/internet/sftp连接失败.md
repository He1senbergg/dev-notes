ssh进到节点

修改sshd_config

```
vi /etc/ssh/sshd_config
```

找到`sftp`，会发现是注释了的状态，把注释解开。

![image](../png/engineering/1.png)

重启sshd

```
systemctl restart sshd
```

---

#sftp连接失败 #SFTP连接失败 #SFTP #sftp