# git clone 执行失败

如果发现`git clone ssh`拉不下来CodeHub的文件，改成使用`git clone http`，即在复制的时候，选择http的链接，就可以把文件拉下来。

# git clone 证书失败

临时单次关闭证书校验的git clone

```
GIT_SSL_NO_VERIFY=true git clone <http地址>
```

全局长期关闭证书校验

```
git config --global http.sslVerify false
```

关了以后验一下关成功没

```
git config --global --get http.sslVerify
```

输出`false`就是关成功了。

想恢复证书校验的话，用

```
git config --global http.sslVerify true
```

或者

```
git config --global --unset http.sslVerify
```