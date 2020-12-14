---
title: pipe
toc: true
date: 2020-12-12 14:36:56
tags: Linux
categories: Linux
mathjax: true
---

### 1.管道的概念

管道是一种最基本的IPC机制，作用于有血缘关系的进程之间，完成数据传递，通过pipe()系统调用即可创建一个管道。

+ 其本质是一个伪文件，实为内核缓冲区
+ 两个文件描述符引用，一个表示读端，一个表示写端
+ 管道采用半双工通信方式，数据只能在一个方向流动
+ 只能在有公共祖先的进程间使用管道

### 2.pipe函数

```c++
int pipe(int (*pipefd)[2]);   
ret 0 : SUCCESS
	-1 : Failure
```

函数调用成功返回read/write两个文件描述符，无需open，需要手动close。

![img](https://img-blog.csdn.net/20161223173958916)

+ 父进程调用pipe函数创建管道，得到两个文件描述符fd[0]、fd[1]指向管道的读端和写端

+ 父进程调用fork()创建子进程，子进程继承父进程文件描述符。

+ 父进程关闭管道读端，子进程关闭管道写端，父进程可以向管道中写入数据，子进程将管道中数据读出。

### 3.管道读写行为

 使用管道需要注意以下4种特殊情况

+ 如果所有指向管道写端的文件描述符都关闭了（管道写端引用计数为0），而仍然有进程从管道的读端读数据，那么管道中剩余的数据都被读取后，再次read会返回0，就像读到文件末尾一样。
+ 如果有指向管道写端的文件描述符没关闭（管道写端引用计数大于0），而持有管道写端的进程也没有向管道中写数据，这时有进程从管道读端读数据，那么管道中剩余的数据都被读取后，再次read会阻塞，直到管道中有数据可读了才读取数据并返回。
+ 如果所有指向管道读端的文件描述符都关闭了（管道读端引用计数为0），这时有进程向管道的写端write，那么该进程会收到信号SIGPIPE，通常会导致进程异常终止。当然也可以对SIGPIPE信号实施捕捉，不终止进程。具体方法信号章节详细介绍。
+ 如果有指向管道读端的文件描述符没关闭（管道读端引用计数大于0），而持有管道读端的进程也没有从管道中读数据，这时有进程向管道写端写数据，那么在管道被写满时再次write会阻塞，直到管道中有空位置了才写入数据并返回。

### 4.示例程序 采用管道实现 ls /home | wc

​     主线程调用fork()创建两个进程一个执行ls /home，另一个执行 wc，利用管道重定向两个进程的stdout，stdin。

​	踩坑记录，在主线程wait(NULL)前，需要手动关闭主线程的所有管道，否则第二个进程会处于读阻塞的状态，因此发生了死锁。

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <ctype.h>
#include <iostream>
using namespace std;

int main() {
    int f[2];
    pipe(f);
    for (int i = 0; i < 2; ++i) {
        pid_t pid = fork();
        if (pid == 0) {
            if (i == 1) {
                close(f[1]);
                dup2(f[0], 0);
                char *name = "/usr/bin/wc";
                char **argv = new char*[1];
                argv[0] = "/usr/bin/wc";
                int exet_st = execv(name, argv);
                close(f[0]);
                _exit(EXIT_SUCCESS);
            } else {
                close(f[0]);
                char buff[] = "test";
                write(f[1], buff, strlen(buff));
                close(f[1]);
                _exit(EXIT_SUCCESS);
            }
        }
    }
    close(f[0]);
    close(f[1]);
    for (int i = 0; i < 2; ++i) {
        wait(NULL);
    }
    return 0;
}
```





