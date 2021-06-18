# MIT-6.S081 lab2-syscall 详解


<!--more-->

> LAB链接：[https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html](https://pdos.csail.mit.edu/6.S081/2020/labs/syscall.html)

## `System call tracing`

> 实验说明：`trace`接受一个整数`mask`，如果这个`mask`的二进制表示下某一位为1，该进程及其子进程在执行系统调用时如果使用到了对应的系统调用，就会打印进程的编号`pid`，该系统调用的名字`name`以及返回值`return value`。

其中系统调用的编号定义在`kernel/syscall.h`中，在最后添加一行`#define SYS_trace  22`方便我们后面输出`trace`，代码如下：
~~~c
// System call numbers
#define SYS_fork    1
#define SYS_exit    2
#define SYS_wait    3
#define SYS_pipe    4
#define SYS_read    5
#define SYS_kill    6
#define SYS_exec    7
#define SYS_fstat   8
#define SYS_chdir   9
#define SYS_dup    10
#define SYS_getpid 11
#define SYS_sbrk   12
#define SYS_sleep  13
#define SYS_uptime 14
#define SYS_open   15
#define SYS_write  16
#define SYS_mknod  17
#define SYS_unlink 18
#define SYS_link   19
#define SYS_mkdir  20
#define SYS_close  21
#define SYS_trace  22
~~~

当`trace`传入的参数`mask`为32时，因为`32 = 1 << 5`，只有第五位为1，从而在系统调用跟踪时只会输出`read`相应的内容，其中一个例子如下：
~~~shell
$ trace 32 grep hello README
3: syscall read -> 1023
3: syscall read -> 966
3: syscall read -> 70
3: syscall read -> 0
$
~~~

明白了题目意思后，但是代码层面还是不知道该怎么下手。别担心，跟着官网给的提示一步步来：

+ 将`$U/_trace\`写入`Makefile`中的`UPROGS`。
+ 执行`make qemu`后发现编译器无法编译`user/trace.c`中`trace()`函数。往`user/user.h`中添加`trace`系统函数的原型`prototype`，往`user/usys.pl`中添加`trace`系统调用代码生成指令调用，往`kernel/syscall.h`中添加`trace`系统调用编号。此时就可以成功执行`make qemu`，因为我们还没有具体实现该系统调用，当运行`trace 32 grep hello README`后报错。
+ 在`kernel/proc.h`的`proc`结构体中添加一个新变量`trace_mask`，在`kernel/sysproc.c`中添加一个`sys_trace()`函数用于获取`trace`系统调用传入的`mask`并且保存在该进程的`trace_mask`中。
+ 向`kernel/proc.c`中的`fork()`函数中添加一行代码`np->trace_mask = p->trace_mask;`，将父进程的`trace mask`复制到子进程中。
+ 修改`kernel/syscall.c`中`syscall()`函数打印追踪路径，同时需要添加一个包含系统调用名字的字符串数组`syscall_names`。此外，还需要将新添加的`sys_trace`加入到全局函数声明和`syscalls`数组中。

> 具体代码修改见 [GitHub Commit](https://github.com/PeiLeiScott/xv6-labs-2020/commit/186bbb886b1432ee4f30fc7d2cf87f259b91d688)

最后运行`./grade-lab-syscall trace`验证代码正确性。(其中有个别测试需要时间较长，需耐心等待一会)

## `Sysinfo`

+ 将`$U\_sysinfotest\`写入`Makefile`中的`UPROGS`。
+ 添加新的系统调用`sysinfo`的过程和前面的`trace`一致，这里就不再赘述。
+ 在`kernel/sysproc.c`中添加一个`sys_sysinfo()`函数用于将空闲内存`freemem`和进程数量`nproc`等系统信息`sysinfo`从内核拷贝回用户空间，其中获取`freemem`和`nproc`需要添加对应的函数，如何使用`copyout`函数需要参照`kernel/sysfile.c`中的`sys_fstat()`函数和`kernel/file.c`中的`filestat()`函数。
+ 在`kernel/kalloc.c`中添加一个函数`kfree_memory()`用来获取空闲内存的大小。通过分析`kalloc.c`中的代码发现内核中维护着一个空闲链表`freelist`，链表的每个节点对应`PGSIZE`(4K)大小的空闲内存，通过遍历`freelist`很容易得到剩余空闲内存的大小。
+ 在`kernel/proc.c`中添加一个函数`proc_count()`函数用来获取状态不是`UNUSED`的进程的数量。不难发现，在`proc.c`中所有进程保存在`proc[NPROC]`数组中，遍历该数组统计出状态不是`UNUSED`的进程的数量即可。
+ 添加完两个函数后，执行`make qemu`编译器指出`sysproc.c`中并没有引入两个函数原型，阅读`xv6 book`第二章的第4节发现内核中所有的函数声明都在`kernel/defs.h`头文件中。因此在`defs.h`中对应位置添加两个函数原型即可。

> 具体代码修改见 [GitHub Commit](https://github.com/PeiLeiScott/xv6-labs-2020/commit/112e7e451f5cfff5299ab14cbad14f34163248cd)

最后运行`./grade-lab-syscall sysinfo`验证代码正确性。

> 最后创建一个`time.txt`并写入一个数字表示你完成实验所花的时间，执行`make grade`，如果显示分数为`35/35`，说明实验全部做完啦~
