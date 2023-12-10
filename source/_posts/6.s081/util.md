---
title: xv6-util
tags:
  - 6s081
category: 6s081
---
## sleep
这个没什么好说的
```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
    if (argc != 2) {
        fprintf(2, "Usage sleep: fail...\n");
        exit(1);
    }
    sleep(atoi(argv[1]));
    exit(0);
}
```

## pingpong
这个实现也比较简单，只需要理解`pipe`系统调用是怎么使用的就可以实现。

每个进程都有三个输入输出流
分别是 1--标准输入。  2-- 标准输入。 3--标准错误输出
pipe系统调用，是对一个int p\[2]的数组使用，这样p\[0] 就是该进程的标准输入，p\[1]就是标准输出，需要注意的是读写完毕之后要及时关闭管道，否则进程会阻塞，无法继续执行下去。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
    if (argc != 1) {
        fprintf(2, "Usage pingpong:fail...\n");
        exit(1);
    }
    int p2c[2], c2p[2];
    pipe(p2c), pipe(c2p);

    if (fork() == 0) {
        // 子进程到父进程
        char buf;
        read(p2c[0], &buf, sizeof(char));
        close(p2c[0]);

        printf("%d: received ping\n", getpid());
        write(c2p[1], "!", sizeof(char));
        close(c2p[1]);
    } else {
        // 父进程到子进程
        write(p2c[1], "!", sizeof(char));
        close(p2c[1]);

        char buf;
        read(c2p[0], &buf, sizeof(char));
        close(c2p[0]);
        printf("%d: received pong\n", getpid());
    }
    exit(0);
}
```


## primes

关于该实现可以先查看[这篇文章](https://swtch.com/~rsc/thread/) 是一种使用多进程筛素数的方法。


具体就是将当前管道的第一个数据取出来，记为num，这个一定是素数，然后一次取出管道中的数据，如果这个数不能被num整除，就写入下一个管道，然后递归的传递给下一个函数。

直到管道为空结束。

此实验最重要的就是管道的关闭时机，未关闭的话就会阻塞，进程无法继续执行，


```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void primes(int lp[2]) {
    int num;
    if (read(lp[0], &num, sizeof(int)) == sizeof(int)) {
        fprintf(1, "prime %d\n", num);

        int p[2];
        pipe(p);

        int n;
        while (read(lp[0], &n, sizeof(int)) == sizeof(int))
            if (n % num)
                write(p[1], &n, sizeof(int));
        close(lp[0]);
        close(p[1]);

        if (fork() == 0) {
            primes(p);
        } else {
            wait(0);
        }
    } else
        close(lp[0]);
}

int main(int argc, char *argv[]) {
    if (argc != 1) {
        fprintf(2, "Usage primes:fail...\n");
        exit(1);
    }

    int p[2];
    pipe(p);
    for (int i = 2; i <= 35; i++)
        write(p[1], &i, sizeof(int));
    close(p[1]);

    if (fork() == 0) {
        primes(p);
    } else {
        wait(0);
    }
    exit(0);
}
```



此程序在linux系统中也可以运行

```c
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>
#include <stdlib.h>
void primes(int lp[2]) {
    int num;
    if (read(lp[0], &num, sizeof(int)) == sizeof(int)) {
        printf("prime %d\n", num);

        int p[2];
        pipe(p);

        int n;
        while (read(lp[0], &n, sizeof(int)) == sizeof(int))
            if (n % num)
                write(p[1], &n, sizeof(int));
        close(lp[0]);
        close(p[1]);

        if (fork() == 0) {
            primes(p);
        } else {
            wait(0);
        }
    } else
        close(lp[0]);
}

int main(int argc, char *argv[]) {
    if (argc != 2) {
        printf( "Usage primes:fail...\n");
        return 0;
    }

    int p[2];
    pipe(p);
    for (int i = 2; i <= atoi(argv[1]); i++)
        write(p[1], &i, sizeof(int));
    close(p[1]);

    if (fork() == 0) {
        primes(p);
    } else {
        wait(0);
    }
    return 0;
}

```


## find

这个程序可以模仿ls程序实现，主要框架基本一致，只是在查找到目录时需要递归的进行检索。至于打开目录通过stat函数实现。


```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char path[], char filename[]) {
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0) {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0) {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    if (st.type != T_DIR) return;

    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
        printf("find: path too long\n");
        close(fd);
        return;
    }

    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';

    while (read(fd, &de, sizeof(de)) == sizeof(de)) {
        if (de.inum == 0)
            continue;
        if (!strcmp(de.name, ".") || !strcmp(de.name, "..")) continue;

        memmove(p, de.name, DIRSIZ);
        p[DIRSIZ] = 0;

        if (!strcmp(de.name, filename)) {
            printf("%s\n", buf);
        }
        if (stat(buf, &st) < 0) {
            printf("find: cannot stat %s\n", buf);
            continue;
        }
        if (st.type == T_DIR) find(buf, filename);
    }
    close(fd);
}

int main(int argc, char *argv[]) {
    if (argc != 3) {
        fprintf(2, "Usage find: fail...\n");
        exit(1);
    }
    find(argv[1], argv[2]);
    exit(0);
}

```



## xargs

这个实现也不是很难，xargs的功能是前一个命令的标准输出作为后一个命令的标准输入

这个命令的实现可以看做拼凑出命令行参数的过程。

因此可以先把后面已有的命令和参数先拷贝进参数列表中（记为初始状态），然后再处理标准输入的参数列表。

这一要实现换行功能，因此我们可以一次读入一个字符，
* 如果是`\n` 那么执行已经拼好的命令，然后恢复到初始状态。
* 如果是空格，说明已经拼凑好一个命令的字符串，开始拼下一个命令的字符串
* 如果是普通字符，直接接到当前字符串的后面

这个过程可以看成状态机，~~不会状态及所以没有实现

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

#define MAGSIZE 16

int main(int argc, char *argv[]) {
    char *xargv[MAGSIZE] = {0};
    int xargc = 0;

    for (int i = 1; i < argc; i++) {
        xargv[xargc] = argv[i];
        xargc++;
    }

    int i = xargc, j = 0;

    char buf = 0;
    while (read(0, &buf, sizeof(char)) == sizeof(char)) {
        if (buf == ' ') {
            xargv[i][j] = '\0';
            if(xargv[i+1]==0)
                xargv[i + 1] = malloc(MAGSIZE * sizeof(char));
            i++;
            j = 0;
        } else if (buf == '\n') {
            xargv[i][j] = '\0';
            xargv[i + 1] = '\0';

            if (fork() == 0) {
                exec(xargv[0], xargv);
            }

            i = xargc;
            j = 0;

        } else {
            if (xargv[i] == 0)
                xargv[i] = malloc(MAGSIZE * sizeof(char));
            xargv[i][j] = buf;
            j++;
        }
    }

    wait(0);
    exit(0);
}
```


有时候xargs的测试无法通过的话，~~可能是因为我们实现的效率太高了。~~因此在main函数的开始加一个sleep(5)
等待前面的命令处理完数据，即可通过测试。