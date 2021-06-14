# MIT-6.S081 lab1-util 详解


<!--more-->

LAB链接：[https://pdos.csail.mit.edu/6.S081/2020/labs/util.html](https://pdos.csail.mit.edu/6.S081/2020/labs/util.html)

## `sleep`

作为第一个lab的第一个练习，主要用来给我们熟悉下代码框架和c语言的库函数等。
> 如果对进程调度和系统调用不熟悉的话请先看[xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf)第一章的第一节。
+ 根据提示，查看`user/echo.c`, `user/grep.c`, `user/rm.c`源码了解如何获取并使用命令行参数(如果你熟悉c语言的话应该早已了解)，即`main`函数中传入了两个参数`argc`和`argv`，其中`agrc`表示命令行参数的个数(argument count)，`argv`表示命令行向量(argument vector)。例如有个可执行程序`argstest`，当你执行`./argstest 1 2 3`时，`argc`为`4`，`argv`为`{"argstest", "1", "2", "3", NULL}`，注意c语言中最后需要NULL作为终结符。
+ 知道了命令行参数相关知识之后，根据`sleep`的参数个数`argc`应该为2(一个为`sleep`，另一个为用户指定的睡眠时间)，如果不为2，输出错误信息并退出。
+ 通过`atoi`库函数将字符串转化为整数。其中`atoi`代表`Ascii to Integer`。
+ 调用系统调用`sleep`。至于内核中如何实现系统调用`sleep`，可以查看`kernel/sysproc.c`中的`sys_sleep`函数；系统调用`sleep`从用户态跳到内核态的过程，可以查看`user/usys.S`中的相关汇编代码(本人太菜看不懂...)
+ 最后将`sleep`写入`Makefile`中的`UPROGS`。

最终代码如下：
~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int 
main(int argc, char *argv[]) 
{
  if (argc != 2) {
    fprintf(2, "Usage: sleep <number>\n");
    exit(1);
  }

  int i = atoi(argv[1]);
  sleep(i);
  exit(0);
}
~~~

运行`sleep`指令：
~~~shell
$ make qemu
...
init: starting sh
$ sleep 10
(大概睡眠1秒)
$
~~~

执行`./grade-lab-util sleep`验证代码正确性。


## `pingpong`
> 如果对I/O，文件描述符以及管道知识不熟悉的话请先看[xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf)第一章的第二、三节。

+ 首先根据`pingpong`的使用，如果参数的个数大于1，直接输出错误信息并退出。
+ 使用`pipe()`系统调用创建两个管道，将描述符存放在`p1[2]`和`p2[2]`，父进程向`p1`写，从`p2`读；子进程从`p1`读，向`p2`写。
+ **父进程和子进程的读写顺序**：不使用`wait()`的情况下父进程应该先写后读，子进程应该先读后写。
+ 父进程不需要用到`p1`的读端和`p2`的写端，因此直接调用`close()`关闭，然后向`p1`写入一个字节内容，写完后关闭`p1`的写端，最后从`p2`读取内容，如果读取的字节数正好为1，则输出响应信息并关闭`p2`读端。子进程执行类似操作。
+ 最后将`pingpong`写入`Makefile`中的`UPROGS`。

最终代码如下：
~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// Read and Write end of a pipe
#define READ_END 0
#define WRITE_END 1

int 
main(int argc, char *argv[]) 
{
  if (argc > 1) {
    fprintf(2, "Usage: pingpong\n");
    exit(1);
  }

  // Create two pipes: p1 and p2
  // Child reads from p1, writes to p2
  // Parent reads from p2, writes to p1
  int p1[2], p2[2];

  pipe(p1);
  pipe(p2);

  char buf[1];

  if (fork() == 0) { // Child should read first
    close(p1[WRITE_END]);
    close(p2[READ_END]);
    if (read(p1[READ_END], buf, 1) == 1) {
      close(p1[READ_END]);
      printf("%d: received ping\n", getpid());
    }
    // Child sends a byte to parent
    write(p2[WRITE_END], buf, 1);
    close(p2[WRITE_END]);
  } else { // Parent should write first
    close(p1[READ_END]);
    close(p2[WRITE_END]);
    // Parent sends a byte to child
    write(p1[WRITE_END], buf, 1);
    close(p1[WRITE_END]);
    if (read(p2[READ_END], buf, 1) == 1) {
      close(p2[READ_END]);
      printf("%d: received pong\n", getpid());
    }
  }

  exit(0);
}
~~~

运行`pingpong`指令：
~~~shell
$ make qemu
...
init: starting sh
$ pingpong
4: received ping
3: received pong
$
~~~

运行`./grade-lab-util pingpong`验证代码正确性。

## `primes`

+ 首先通过阅读[this page](https://swtch.com/~rsc/thread/)了解`primes`的工作原理：创建一个管道，父进程向管道写入从2到35的整数，子进程从管道读取数据后输出2，并且再次创建一个管道，向新创建的管道写入不是2的倍数的所有数字，依次递归进行下去，直到最后管道里没有任何数据。具体如下图：
![primes1](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/primes1.393njen654e0.png)
![primes2](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/primes2.1ozv0tktsoow.png)
+ 根据`primes`的使用，如果参数的个数大于1，直接输出错误信息并退出。
+ `main`函数中，创建一个管道，父进程中先关闭管道的读端，然后向管道写入从2到35的整数，最后关闭管道的写端，**注意此时需要执行`wait()`系统调用等待子进程执行完毕，否则父进程会在向管道写完数据后直接退出**；子进程调用`child()`函数执行响应的操作。
+ `child()`函数中，首先关闭父进程中创建的管道`pp`的写端，从`pp`中读入一个`int`大小的数据，如果`read()`返回0，说明管道中没有数据，关闭`pp`的读端后直接返回；反之，输出从`pp`中读取到的第一个数`i`，随后创建当前进程的管道`cp`，在当前进程中关闭`cp`的读端后循环向`cp`中写入所有不是`i`的倍数，最后关闭`pp`的读端和`cp`的写端，调用`wait()`等待子进程；当前进程的子进程中，关闭`pp`的读端和`cp`的写端后，递归调用`child()`函数。
+ 最后将`primes`写入`Makefile`中的`UPROGS`。

最终代码如下：
~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

// Read and Write end of a pipe
#define READ_END 0
#define WRITE_END 1

// pp, cp stands for parent pipe and child pipe, respectively
void
child(int pp[])
{
  close(pp[WRITE_END]);
  int i;
  if (read(pp[READ_END], &i, sizeof(i)) == 0) {
    close(pp[READ_END]);
    exit(0);
  }
  printf("prime %d\n", i);
  int num, cp[2];
  pipe(cp);
  if (fork() == 0) {
    close(pp[READ_END]);
    close(cp[WRITE_END]);
    child(cp);
  } else {
    close(cp[READ_END]);
    while (read(pp[READ_END], &num, sizeof(num)) != 0) {
      if (num % i != 0) {
        write(cp[WRITE_END], &num, sizeof(num));
      }
    }
    close(pp[READ_END]);
    close(cp[WRITE_END]);
    wait((int *) 0);
  }
  
  exit(0);
}

int 
main(int argc, char *argv[]) 
{
  if (argc > 1) {
    fprintf(2, "Usage: primes\n");
    exit(1);
  }

  int p[2];
  pipe(p);

  if (fork() == 0) {
    child(p);
  } else {
    close(p[READ_END]);
    // Feed the numbers 2 through 35 into the pipe
    for (int i = 2; i <= 35; i++) {
      write(p[WRITE_END], &i, sizeof(i));
    }
    close(p[WRITE_END]);
    wait((int *) 0);
  }

  exit(0);
}
~~~

运行`primes`指令：
~~~shell
$ make qemu
...
init: starting sh
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
$
~~~

运行`./grade-lab-util primes`验证代码正确性。


## `find`

> 如果对文件系统相关知识不熟悉的话请先看[xv6 book](https://pdos.csail.mit.edu/6.S081/2020/xv6/book-riscv-rev1.pdf)第一章的第四节。

+ 首先阅读`user/ls.c`源码了解如何读取文件目录信息以及判断文件类型，对`ls.c`的分析见下方代码中注释。
~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char*
fmtname(char *path)
{
  // DIRSIZ 在`kernel/fs.h`中定义
  // #define DIRSIZ 14
  static char buf[DIRSIZ+1];
  char *p;

  // Find first character after last slash.
  for(p=path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++; // 此时 p 为最后一个 "/" 后的字符串，即文件名。例如当路径为"a/b/cd"时，此时p为"cd"

  // Return blank-padded name.
  // 如果 p 的长度大于 DIRSIZ，直接返回 p
  if(strlen(p) >= DIRSIZ)
    return p;
  // 如果 p 的长度小于 DIRSIZ，则用空格补齐
  memmove(buf, p, strlen(p));
  memset(buf+strlen(p), ' ', DIRSIZ-strlen(p));
  return buf;
}

void
ls(char *path) 
{
  char buf[512], *p;
  int fd; // 文件描述符 file descriptor
  // 定义位于 kernel/fs.h
  struct dirent de; // 目录项 directory entry
  // 定义位于 kernel/stat.h
  struct stat st; // 文件信息 file status

  // 调用 open() 打开路径 path 并返回相应的文件描述符 fd
  if((fd = open(path, 0)) < 0){
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  // 调用 fstat 将 fd 对应的文件信息记录在 st 中
  if(fstat(fd, &st) < 0){
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch(st.type){
  // st.type 为文件时直接返回即可
  case T_FILE:
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    break;

  // 当 st.type 为目录时
  case T_DIR:
    // 此处只是限制了路径 path 的长度防止缓存溢出，不是很重要可以忽略
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf("ls: path too long\n");
      break;
    }
    // 将 path 的内容拷贝到 buf
    strcpy(buf, path);
    // 指针 p 定位到 buf 的最后
    p = buf+strlen(buf);
    // 在 buf 的最后位置添加 "/"
    *p++ = '/';
    // 每次读取一个目录项
    while(read(fd, &de, sizeof(de)) == sizeof(de)){
      // inum == 0 说明无效文件或目录
      if(de.inum == 0)
        continue;
      // 将 de.name 拷贝到 p 中
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  // 最后别忘了关闭文件描述符，否则浪费资源
  close(fd);
}

int
main(int argc, char *argv[])
{
  int i;

  // 如果 ls 后面没有参数，默认对当前目录 "." 执行 ls 命令
  if(argc < 2){
    ls(".");
    exit(0);
  }
  // 对 ls 后面的目录分别执行 ls 命令
  for(i=1; i<argc; i++)
    ls(argv[i]);
  exit(0);
}

~~~

+ 根据`find`的使用，参数的个数应该为3，分别是`find`、指定目录`path`、指定文件名`filename`。
+ 递归调用`find`进入子目录寻找，注意不要进入"."和".."目录。
+ 使用`strcmp()`函数判断两个字符串是否相等。
+ 将`find`写入`Makefile`中的`UPROGS`。

最终代码如下：
~~~c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char*
fmtname(char *path) 
{
  char *p;

  // Find first character after last slash
  for (p = path+strlen(path); p >= path && *p != '/'; p--)
    ;
  p++;

  return p;
}

void
find(char *path, char *filename) 
{
  char buf[512], *p;
  int fd;           // file descriptor
  struct dirent de; // directory entry
  struct stat st;   // file status

  if ((fd = open(path, 0)) < 0) {
    fprintf(2, "find: cannot open %s\n", path);
    return;
  }

  if (fstat(fd, &st) < 0) {
    fprintf(2, "find: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch (st.type) {
  case T_FILE:
    if (strcmp(fmtname(path), filename) == 0) {
      printf("%s\n", path);
    }
    break;

  case T_DIR:
    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf) {
      printf("find: path too long\n");
      break;
    }
    
    // Add '/'
    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';

    while (read(fd, &de, sizeof(de)) == sizeof(de)) {
      // Invalid directory
      if (de.inum == 0) { 
        continue;
      }

      // Should not recurse into "." or ".."
      if (strcmp(de.name, ".") == 0 || strcmp(de.name, "..") == 0) {
        continue;
      }

      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      // Call find recursively
      find(buf, filename);
    }
    break;
  }
  close(fd);
}

int 
main(int argc, char *argv[]) 
{
  if (argc != 3) {
    fprintf(2, "Usage: find <path> <filename>\n");
    exit(1);
  }

  find(argv[1], argv[2]);
  exit(0);
}
~~~

运行`find`指令：
~~~shell
$ make qemu
...
init: starting sh
$ echo > b
$ mkdir a
$ echo > a/b
$ find . b
./b
./a/b
$ 
~~~

运行`./grade-lab-util find`验证代码正确性。

## `xargs`

> 如果不熟悉`xargs`指令可以查看[xargs命令](https://www.linuxcool.com/xargs)。

+ 首先要知道，我们要实现的`xargs`每行只从标准输入流获取**一个**额外的命令参数。功能类似下图：
![xargs](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/xargs.1w9kuh0cngcg.png)
+ 根据`xargs`的使用，参数的个数至少为2个，`xargs`、指定的命令`command`。
+ `kernel/param.h`中定义了最大参数个数`MAXARG`，创建一个大小为`MAXARG`的数组`params`用于存放命令参数，将`argv`中相应的参数拷贝到到`params`，注意留一个位置用于存放从标准输入读取的那个额外的命令参数。
+ 从标准输入循环读取命令参数，当遇到`\n`说明当前行的参数已经读取完毕，将其存放入`params`后创建一个子进程调用`exec()`执行对应的指令；如果标准输入中的参数读取完毕，则退出循环。注意父进程中调用`wait()`等待子进程结束。
+ 最后将`xargs`写入`Makefile`中的`UPROGS`。

运行`xargs`指令：
~~~shell
$ make qemu
...
init: starting sh
$ sh < xargstest.sh
$ $ $ $ $ $ hello
hello
hello
$ $   
~~~

运行`./grade-lab-util xargs`验证代码正确性。

> 最后创建一个`time.txt`并写入一个数字表示你完成实验所花的时间，执行`make grade`，如果显示分数为`100/100`，说明实验全部做完啦~
