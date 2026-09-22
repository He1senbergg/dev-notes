# 1. 下载tmux

[下载链接](https://github.com/tmux/tmux-builds/releases)

看下要arm还是x86，下一个。

传到机器上

```
scp -P {port} {file_path} {username}@{IP}:{path}
```

执行

```
mkdir -p "$HOME/.local/bin"

tar -xzf "$HOME/tmux-3.7b-linux-x86_64.tar.gz" \
      -C "$HOME/.local/bin"

chmod +x "$HOME/.local/bin/tmux"
```

测试一下tmux好用了没

```
"$HOME/.local/bin/tmux" -V
```

正常应输出

```
tmux {version}
```

永久激活

```
echo 'export PATH="$HOME/.local/bin:$PATH"' >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```


# 2. 使用tmux新建窗口

```
tmux new -s {session-name}
```

## 2.1 窗口挂起

### 2.1.1 直接快捷键

窗口挂起，需同时按`ctrl`、`b`，松开后再按`d`。

### 2.1.2 命令式

键入

```
tmux detach
```

# 3. 恢复连接窗口

```
tmux attach -t {session-name}
```

# 4. 关闭窗口

## 4.1 开启状态

直接按下`ctrl`、`b`。

## 4.2 未连接窗口的状态

```
# 使用会话编号
tmux kill-session -t {session-id}

# 使用会话名称
tmux kill-session -t {session-name}
```