软件下载，传到机器上、容器里。[gdb](../installer/gdb/gdb_package.zip)

----

# 首先

以特权进容器。

```
docker exec -itu 0 --privileged {dockerID} bash
```

------

# 操作

## 最简版

```
set height 0
thread apply all bt
```

## 实操（需在进程对应的容器中使用）

```
{路径}/gdb_r12_arm attach -p {pid}
set height 0
handle SIG35 SIG36 SIG33 SIGUSR2 nostop noprint
command
thread apply all bt
c
end
c
```

---

# ChatGPT说明

这段代码看起来是与 **GDB（GNU 调试器）** 相关的调试脚本。GDB 是用于调试程序的工具，允许开发人员在程序运行时暂停并检查程序的状态。让我们逐行解释这段代码。

## 1. `/tmp/gdb_r12_arm attach {pid}}`

```bash
/tmp/gdb_r12_arm attach {pid}}
```

* `/tmp/gdb_r12_arm` 是一个执行文件（可能是 GDB 的一个自定义脚本或者 GDB 本身的副本）。
* `attach {pid}`：`attach` 命令用于将 GDB 附加到已经在运行的进程上。这里 `{pid}` 是一个占位符，表示进程 ID（PID），它是你想调试的目标进程的标识符。`{pid}` 应该被实际的进程 ID 替换。
* `}}`：这个结尾似乎多了一个右括号 `}`，看起来像是一个打字错误或无关的符号。如果没有这个符号，`attach` 命令将会正常执行。

## 2. `set height 0`

```bash
set height 0
```

* `set height 0`：这条命令用于设置 GDB 中输出的行数限制。`height` 控制 GDB 输出的行数，`0` 表示没有限制，所有输出将完整显示，而不需要分页。这通常用于避免每次输出超过屏幕时要求用户按键来继续查看的情况。

## 3. `handle SIG35 SIG36 SIG33 SIGUSR2 nostop noprint`

```bash
handle SIG35 SIG36 SIG33 SIGUSR2 nostop noprint
```

* `handle` 命令用于配置 GDB 如何处理特定的信号。信号是操作系统传递给进程的异步通知，通常用于进程间的通信。
* `SIG35`, `SIG36`, `SIG33`, `SIGUSR2` 是信号名称。GDB 的默认行为可能会停止进程并打印出信号信息。这里的 `nostop` 和 `noprint` 表示：
  
  * `nostop`：当接收到这些信号时，进程不会暂停（即不停止执行）。
  * `noprint`：当接收到这些信号时，GDB 不会打印出任何信息。
  
  这意味着当进程收到这些信号时，GDB 会继续运行进程并且不会显示任何额外的信息。

## 4. `command`

```bash
command
```

* `command` 命令表示 GDB 中的一个命令块的开始。通常它会在一个条件下执行一系列的 GDB 命令，比如在某个断点被命中时执行的一组操作。
* 然而，在这段代码中，`command` 之后没有具体的内容，表示它是一个空命令块。这可能是一个占位符，或者该命令块的实际内容在其他部分定义。

## 5. `thread apply all bt`

```bash
thread apply all bt
```

* `thread apply all`：这个命令让 GDB 对所有线程执行指定的命令。
* `bt`：是 `backtrace` 的缩写，用于打印当前线程的堆栈跟踪信息（即函数调用栈）。`bt` 会显示程序执行到当前位置时的函数调用链，帮助开发人员了解程序的执行路径，尤其是在调试程序崩溃时非常有用。
  
  结合起来，这条命令表示 **对所有线程执行堆栈跟踪**。

## 6. `c`

```bash
c
```

* `c` 是 GDB 中的 `continue` 命令的缩写。`continue` 命令用于让程序继续执行，直到下一个断点或信号中断为止。
* 在这里，`c` 表示继续执行当前进程的代码。

## 7. `end`

```bash
end
```

* `end` 表示 GDB 命令块的结束。它标志着一个命令块（例如在 `command` 命令中的一系列命令）的结束。

## 8. `c`

```bash
c
```

* 再次执行 `c`，表示让程序继续执行。

## 总结：

这段代码的目的是调试一个正在运行的进程，具体操作如下：

1. **附加到进程**：使用 `attach {pid}` 附加到目标进程。
2. **设置输出行为**：通过 `set height 0` 禁止分页输出，使输出不被限制。
3. **处理信号**：配置 GDB 如何处理特定的信号（如 `SIG35`, `SIG36`, `SIG33`, `SIGUSR2`），设置不暂停进程并且不打印信号信息。
4. **命令块**：启动一个命令块，但没有具体操作（`command` 和 `end` 没有定义操作）。
5. **打印堆栈跟踪**：对所有线程执行 `bt`（堆栈跟踪）。
6. **继续执行**：通过 `c` 命令继续执行程序。

这段脚本的目的是在调试过程中，不仅附加到一个正在运行的进程，还设置了处理信号的行为，打印出所有线程的堆栈跟踪，并确保程序继续运行。