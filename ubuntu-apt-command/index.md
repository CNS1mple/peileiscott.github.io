# Ubuntu下包管理器apt常见命令


<!--more-->

> 本文旨在介绍`Ubuntu`下`apt`包管理工具的常用命令，在进行`Linux`开发时作为词典参考。

## `apt update`
~~~shell
sudo apt update
~~~
每次下载或更新本地软件时最好先执行`apt update`指令从服务器更新本地包仓库，从而获取最新版本的软件。

## `apt upgrade`
将本地软件更新到仓库中最新版本，执行以下指令
~~~shell
# 更新所有软件
sudo apt upgrade
~~~
如果只想更新指定软件而非所有软件，执行以下指令
~~~shell
# 找出可更新的软件
apt list --upgradeable
# 更新指定的软件
sudo apt upgrade <package_1> <package_2> ...
~~~
有些软件更新时需要输入`y/n`确认是否安装，如果不想输入的话可以在`apt upgrade`后面加`-y` Flag
~~~shell
sudo apt upgrade -y 
~~~
通常为了方便，我们将上面提到的`apt update`和`apt upgrade`写在一行
~~~shell 
sudo apt update && sudo apt upgrade -y
~~~

## `apt install`
安装指令软件，执行以下指令
~~~shell
sudo apt install <package_1> <package_2> ...
~~~
有些软件安装时需要输入`y/n`确认是否安装，如果不想输入的话可以在`apt install`后面加`-y` Flag
~~~shell
sudo apt install -y <package_1> <package_2> ...
~~~
默认情况下`apt install`安装的软件都是最新版本的，如果想安装老版本的软件可以在软件名后面加`=version`
~~~shell
# 安装 mysql-5.7
sudo apt install mysql-server=5.7
~~~

## `apt remove/purge`
删除指定软件，执行以下指令
~~~shell
# apt remove只会删除软件本身，相关配置文件依旧保留
sudo apt remove <package_1> <package_2> ...
# 如果执行了上面的指令后，想要删除相关配置文件，只需要执行下面这条指令即可
sudo apt autoremove

# apt purge会将软件和配置信息一起删除
sudo apt purge <package_1> <package_2> ...
~~~

## `apt list`
~~~shell
sudo apt list
~~~
`apt list`命令会列出所有的软件，包括仓库中的、已安装的和可更新的软件等，但是**不建议直接执行该指令**，最好搭配`grep`命令一起使用
~~~shell
sudo apt list | grep package_name
~~~
如果只想查看已下载的软件，执行以下指令
~~~shell 
sudo apt list --installed
~~~
查看可更新的软件，执行以下指令
~~~shell
sudo apt list --upgradeable
~~~

## `apt show`
~~~shell
sudo apt show package_name
~~~
`apt show`命令会列出软件的包依赖、版本、大小等等信息
