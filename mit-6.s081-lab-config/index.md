# MIT-6.S081实验环境搭建


<!--more-->

>  本人实验环境：Ubuntu20.04(其余环境请参考课程主页的配置教程)

更新`apt`源：

~~~shell
sudo apt update && sudo apt upgrade
~~~

安装必要软件：

~~~shell
sudo apt install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 
~~~

下载并解压`qemu`：

~~~shell
wget https://download.qemu.org/qemu-5.1.0.tar.xz
tar xf qemu-5.1.0.tar.xz
~~~

编译`qemu`：

~~~shell
cd qemu-5.1.0
./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"

# 此时可能出现下方错误
ERROR: pixman >= 0.21.8 not present.
       Please install the pixman devel package.
# 解决方法
sudo apt install libpixman-1-dev

# 解决后继续执行下方命令
./configure --disable-kvm --disable-werror --prefix=/usr/local --target-list="riscv64-softmmu"
make
sudo make install
cd ..
~~~

安装`riscv64-unknown-elf-gcc`：

~~~shell
sudo apt install gcc-riscv64-unknown-elf
~~~

下载课程代码：

~~~shell
git clone git://g.csail.mit.edu/xv6-labs-2020
~~~

验证实验环境是否搭建完成：

~~~shell
$ riscv64-unknown-elf-gcc --version
riscv64-unknown-elf-gcc () 9.3.0
...

$ qemu-system-riscv64 --version
QEMU emulator version 4.2.1
...

$ cd xv6-labs-2020
$ git checkout util
$ make qemu
... 
init: starting sh
$
~~~
如果上面步骤都没有问题，说明你的实验环境已经成功搭建好了，最后按Ctrl-a再加x退出qemu
