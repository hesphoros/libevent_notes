在 Linux 中，**信号（Signal）** 是一种非常重要的进程间异步通信机制，可以理解为：

> **内核或其他进程向某个进程发送的一种“软件中断”。**

当进程收到信号时，会暂停当前执行流程，转而执行对应的信号处理逻辑。

---

# 信号是什么

例如：

```bash
kill -9 1234
```

实际上并不是“杀死进程”这个动作。

而是：

```text
向 PID 1234 发送 SIGKILL 信号
```

内核收到后：

```text
SIGKILL
    ↓
目标进程
    ↓
立即终止
```

---

# 常见信号

查看所有信号：

```bash
kill -l
```

例如：

|编号|名称|含义|
|---|---|---|
|1|SIGHUP|终端断开|
|2|SIGINT|Ctrl+C|
|3|SIGQUIT|Ctrl+\|
|9|SIGKILL|强制终止|
|11|SIGSEGV|段错误|
|13|SIGPIPE|管道断开|
|14|SIGALRM|定时器超时|
|15|SIGTERM|请求退出|
|17|SIGCHLD|子进程结束|
|18|SIGCONT|继续执行|
|19|SIGSTOP|暂停|

最常见的是：

```text
SIGINT
SIGTERM
SIGKILL
SIGSEGV
SIGCHLD
```

---

#  Ctrl+C 为什么能结束程序

终端驱动检测到：

```text
Ctrl + C
```

然后发送：

```text
SIGINT
```

给前台进程组。

例如：

```cpp
while (true)
{
}
```

运行：

```bash
./a.out
```

按：

```text
Ctrl+C
```

实际上：

```text
SIGINT
↓
程序退出
```

---

#  kill 命令

发送信号：

```bash
kill PID
```

默认发送：

```text
SIGTERM
```

即：

```bash
kill -15 PID
```

---

强制结束：

```bash
kill -9 PID
```

即：

```text
SIGKILL
```

这个信号：

```text
不能捕获
不能忽略
不能处理
```

内核直接终止进程。

---

#  信号处理函数

注册处理器：

```cpp
#include <signal.h>
#include <stdio.h>

void handler(int sig)
{
    printf("receive signal %d\n", sig);
}

int main()
{
    signal(SIGINT, handler);

    while (1);

    return 0;
}
```

运行：

```bash
./a.out
```

按：

```text
Ctrl+C
```

输出：

```text
receive signal 2
```

而不会退出。

---

# 更规范的 sigaction

现代 Linux 推荐：

```cpp
sigaction()
```

而不是：

```cpp
signal()
```

例如：

```cpp
#include <signal.h>

void handler(int sig)
{
}

int main()
{
    struct sigaction sa;

    sa.sa_handler = handler;

    sigemptyset(&sa.sa_mask);

    sa.sa_flags = 0;

    sigaction(SIGINT, &sa, NULL);
}
```

因为：

```text
signal()
历史遗留

sigaction()
POSIX标准
```

---

#  SIGSEGV

经典：

```cpp
int* p = nullptr;

*p = 1;
```

结果：

```text
Segmentation fault
```

实际上：

```text
SIGSEGV
```

内核发现：

```text
访问非法地址
```

发送：

```text
SIGSEGV
```

给进程。

---

# SIGCHLD

子进程退出时：

```text
内核
 ↓
父进程
 ↓
SIGCHLD
```

例如：

```cpp
fork();
```

后：

```cpp
waitpid();
```

往往配合：

```cpp
SIGCHLD
```

使用。

这也是很多服务器回收子进程的方式。

---


