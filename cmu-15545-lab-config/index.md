# CMU-15545 实验环境配置


<!--more-->

> **写在前面**：
> + 此实验可以直接在Mac(Intel chip)环境下运行，但是笔者今年年初入手的是Mac with M1 chip，经过测试后发现此实验并不能直接在M1芯片上运行(同样的还有[MIT6.S081](https://pdos.csail.mit.edu/6.S081/2020/)课程，最后只能转战Linux，所以购买Mac M1需谨慎。。)
> + 由于国内网络的某些特殊原因，以下实验环境的搭建过程中需要用到**科学上网**，具体如何操作请自行研究。

## 课程主页：[CMU-15545 HOMEPAGE](https://15445.courses.cs.cmu.edu/fall2020/)

## 实验环境：Ubuntu20.04(其余环境的搭建过程请参考Github中的README)

1. 从[https://github.com/cmu-db/bustub](https://github.com/cmu-db/bustub)将实验代码`fork`到个人仓库，然后通过`git clone`将代码拷贝到本地。
2. 在`bustub`目录下执行`sudo build_support/packages.sh`。

> 可能出现的错误：`sudo: build_support/packages.sh command not found`
>
> 解决办法：执行`chmod +x build_support/packages.sh`后再执行上面代码即可。

3. 在`bustub`目录下依次执行以下命令：

~~~bash
mkdir build
cd build
cmake ..
make
make check-tests
~~~

如果上述步骤没有错误，那么恭喜你，实验环境搭建成功，接下来就可以去完成实验了！
