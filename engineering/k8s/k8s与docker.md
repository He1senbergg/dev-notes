# 一、k8s

## 概念

K8s 是 **Kubernetes** 的简称，是一个用来 **管理容器化应用** 的平台。

可以简单理解为：

> 有很多服务跑在 Docker 容器里，K8s 负责自动部署、启动、停止、扩容、重启、调度这些容器。

例如有一个服务 `scheduler`，它可能被打包成一个 Docker 镜像：

```bash
192.168.2.2:20202/op_svc_pom/scheduler:26.9.9.B436
```

K8s 可以做这些事情：

1. **部署服务**

你告诉 K8s：

> 我要以主备的形式，运行这个镜像。

K8s 会在集群里的机器上启动 2 个 Pod。

---

2. **自动重启**

如果某个容器崩了，K8s 会自动把它拉起来。

比如：

```text
scheduler-xxx  崩溃了
```

K8s 会尝试重新启动它。

---

3. **滚动更新**

换镜像时，不需要手动停服务。

例如：

```bash
kubectl set image deployment/scheduler scheduler=192.168.2.2:20202/op_svc_pom/scheduler:26.9.9.B436
```

K8s 会逐步替换旧 Pod，尽量保证服务不中断。

---

4. **资源调度**

K8s 会决定容器应该跑在哪台机器上。

比如某个节点资源不够，它会把 Pod 调度到其他节点。

---

常见概念可以这么记：

| 概念        | 简单理解                                     |
| ------------- | ---------------------------------------------- |
| Cluster     | K8s 集群，一组机器                           |
| Node        | 集群中的一台机器                             |
| Pod         | K8s 中最小运行单位，里面通常放一个或多个容器 |
| Deployment  | 管理一组无状态 Pod，比如后端服务             |
| StatefulSet | 管理有状态 Pod，比如数据库、主从服务         |
| Service     | 给 Pod 提供固定访问入口                      |
| ConfigMap   | 存配置文件                                   |
| Secret      | 存密码、证书等敏感配置                       |
| Namespace   | 命名空间，用来隔离不同项目或环境             |

总结：

> **Docker 负责把应用打包成容器，K8s 负责在一堆机器上管理这些容器。**

## 常用命令

**1. 替换镜像**

```
kubectl set image -n {Name Space} deploy/{Pod Name} {Docker Name}={Docker Image Url}
```

示例

```
kubectl set image deployment/scheduler scheduler=192.168.2.2:20202/op_svc_pom/scheduler:26.9.9.B436
```

**2. 编辑环境变量**

```
kubectl edit cm -n {Name Space} {Service Name}
```

示例

```
kubectl edit cm -n fst-manage llmservice
```

**3. 编辑TOSCA**

```
kubectl edit deploy -n {Name Space} {Service Name}
```

示例

```
kubectl edit deploy -n fst-manage llmservice
```

**4. 查看Pod的状态**

```
kubectl describe pod -n {Name Space} {Service Name}
```

示例

```
kubectl describe pod -n fst-manage llmservice
```

# 二、Docker

## 概念

容器可以理解为：

> **一个轻量级的“应用运行环境包”。**

它把应用程序运行所需的东西打包在一起，比如：

| 内容     | 例子                                   |
| ---------- | ---------------------------------------- |
| 程序本身 | `scheduler`                 |
| 依赖库   | `.so`动态库、Python 包、C++ 运行库 |
| 配置文件 | `config.json`、`models.json`   |
| 运行环境 | 文件目录、环境变量、启动命令           |

这样这个应用就可以比较稳定地在不同机器上运行。

---

1. 为什么需要容器？

假设写了一个服务，在本地电脑上能跑：

```bash
python app.py
```

但是换到服务器上可能报错：

```text
ModuleNotFoundError: No module named 'xxx'
```

或者：

```text
libxxx.so: cannot open shared object file
```

原因是服务器上的环境和你本地不一样。

容器的作用就是把这些环境一起打包：

```text
应用程序 + 依赖库 + 配置 + 启动命令
```

这样部署时就不用一台机器一台机器地手动配环境。

---

2. 容器和虚拟机有什么区别？

| 对比项               | 虚拟机         | 容器                 |
| ---------------------- | ---------------- | ---------------------- |
| 隔离方式             | 模拟一整台电脑 | 隔离一个应用运行环境 |
| 是否包含完整操作系统 | 通常包含       | 不包含完整系统内核   |
| 启动速度             | 较慢           | 较快                 |
| 资源占用             | 较大           | 较小                 |
| 常见用途             | 跑完整系统     | 部署应用服务         |

容器不是一台完整的虚拟电脑，它更像是：

> 在宿主机操作系统上隔离出来的一个应用运行空间。

---

3. Docker 镜像和容器的关系

| 概念           | 类比                         |
| ---------------- | ------------------------------ |
| 镜像 Image     | 安装包 / 模板                |
| 容器 Container | 用这个模板启动出来的运行实例 |

比如你有一个镜像：

```bash
nginx:latest
```

用它启动后，才会得到一个正在运行的容器：

```bash
docker run nginx:latest
```

所以：

- 镜像是静态的
- 容器是运行中的

---

4. 用业务场景理解

比如你有一个推理服务：

```text
scheduler
```

可以把它做成一个镜像：

```text
scheduler:26.9.9.B436
```

镜像里面可能包含：

- scheduler 可执行文件
- 配置文件
- 依赖库
- 启动脚本
- 运行参数

然后在服务器上启动容器：

```text
容器 = 正在运行的 scheduler 服务
```

如果容器崩了，K8s 可以根据镜像重新拉起一个新的容器。

---

总结

> **容器就是把应用和它需要的运行环境打包并隔离起来，让应用可以更方便、更稳定地运行。**
> **镜像是应用的模板，容器是镜像跑起来后的实例。**

## 常用命令

重启容器（保留当前文件）

```
docker restart {容器ID}
```

重建容器（刷新为镜像中携带的原始文件）

```
docker kill {容器ID}
```

停止容器的健康检查

```
systemctl stop kubelet
```

开启容器的健康检查

```
systemctl start kubelet
```