# Crane 项目环境配置

***以下内容为配置代码编译环境，在编译项目的节点执行***

## 1.环境准备

安装ntp ntpdate同步时钟

```shell
yum -y install ntp ntpdate -y
systemctl start ntpd
systemctl enable ntpd

timedatectl set-timezone Asia/Shanghai
```

关闭防火墙，不允许关闭防火墙则考虑开放10011、10010、873端口
```shell
systemctl stop firewalld
systemctl disable firewalld

# 或者开放端口
firewall-cmd --add-port=10011/tcp --permanent --zone=public
firewall-cmd --add-port=10010/tcp --permanent --zone=public
firewall-cmd --add-port=873/tcp --permanent --zone=public
# 重启防火墙(修改配置后要重启防火墙)
firewall-cmd --reload
```

## 2.安装依赖包

```shell
yum install -y epel-release pv openssl-devel libcgroup-devel curl-devel boost169-devel boost169-static
```

## 3.安装工具链

安装C++11

```shell
# Install CentOS SCLo RH repository:
yum install -y centos-release-scl-rh
# Install devtoolset-11 rpm package:
yum install -y devtoolset-11
# 第三步就是使新的工具集生效
scl enable devtoolset-11 bash
```
这时用gcc --version查询，可以看到版本已经是11.2系列了

```shell
$ gcc --version
gcc (GCC) 11.2.1 20220127 (Red Hat 11.2.1-9)
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

为了避免每次手动生效，可以在.bashrc中设置，此文件中修改对当前用户永久生效

```shell
vim ~/.bashrc

# 在最后一行加上
source /opt/rh/devtoolset-11/enable
or
source scl_source enable devtoolset-11

# 使环境变量生效
source ~/.bashrc 
```

安装cmake和ninja

选择一个合适的源码存放位置，从github下载源码
```shell
wget https://github.com/ninja-build/ninja/releases/download/v1.10.2/ninja-linux.zip
wget https://github.com/Kitware/CMake/releases/download/v3.21.3/cmake-3.21.3.tar.gz
```

解压编译安装，首次编译时间较长，请耐心等待
```shell
unzip ninja-linux.zip
cp ninja /usr/local/bin/

tar -zxvf cmake-3.21.3.tar.gz
cd cmake-3.21.3
./bootstrap
gmake
gmake install
```

检查安装是否成功
```shell
cmake --version
#cmake version 3.21.3
#
#CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

报错

```shell
CMake Error: Could not find CMAKE_ROOT !!!
CMake has most likely not been installed correctly.
Modules directory not found in
/usr/local/bin
Segmentation fault
```

出现这种情况一般情况下是因为我们在安装cmake之前执行过cmake命令，终端的哈希表会记录下执行过的命令的路径，相当于缓存。第一次执行命令shell解释器默认的会从PATH路径下寻找该命令的路径，当我们第二次使用该命令时，shell解释器首先会查看哈希表，没有该命令才会去PATH路径下寻找。

所以哈希表可以大大提高命令的调用速率，但是CMake Error: Could not find
CMAKE_ROOT错误的原因也出在这里，如果我们之前在这个终端执行过cmake命令，那么哈希表就会自动记录下之前版本cmake的路径，我们可以通过输入hash -l查看，如下所示：

```shell
[root@cn17 cmake-3.21.3]# hash -l
builtin hash -p /usr/bin/wget wget
builtin hash -p /usr/bin/cmake cmake
```

所以当我们更新了cmake以后，当我们输入cmake相关命令时，shell解释器便会去哈希表里面查找之前版本cmake的路径，然后便产生了错误。

此时我们可以重新开一个终端，也可以在该终端执行hash -r命令来清除哈希表的内容，然后再执行cmake --version命令。

## 4.安装mongodb

***安装数据库仅在需要存储数据的节点安装***

```shell
# 下载并解压安装包
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-5.0.9.tgz
tar -zxvf mongodb-linux-x86_64-rhel70-5.0.9.tgz
# 重命名
mv mongodb-linux-x86_64-rhel70-5.0.9  /opt/mongodb
# 添加环境变量  
vim /etc/profile
```

在配置文件中添加如下内容（路径应对应mongodb安装路径）
```shell
export MONGODB_HOME=/opt/mongodb
export PATH=$PATH:${MONGODB_HOME}/bin
```

```shell
# 使环境变量生效
source /etc/profile 
# 创建db目录和log目录
cd /opt/mongodb
mkdir -p ./data/db
mkdir -p ./logs
touch ./logs/mongodb.log
```

创建mongodb.conf配置文件，内容如下：

```shell
vim mongodb.conf

#端口号
port=27017
#db目录
dbpath=/opt/mongodb/data/db
#日志目录
logpath=/opt/mongodb/logs/mongodb.log
#后台
fork=true
#日志输出
logappend=true
#允许远程IP连接
bind_ip=0.0.0.0
#开启权限验证
#auth=true
```

启动测试
```shell
mongod --config /opt/mongodb/mongodb.conf
mongo
```

创建用户

```shell
use admin
db.createUser({
  user:'admin',  # 用户名
  pwd:'123456',  # 密码
  roles:[{ role:'root',db:'admin'}]   #root 代表超級管理员权限 admin代表给admin数据库加的超级管理员
})

db.shutdownServer() # 重启前先关闭服务器
```

修改/opt/mongodb/mongodb.conf配置文件，将权限验证的注释放开

```shell
vim /opt/mongodb/mongodb.conf

......
#开启权限验证
auth=true
```

重新启动mongodb数据库

```shell
mongod --config /opt/mongodb/mongodb.conf
```

编辑开机启动

```shell
vi /etc/rc.local
# 加入如下语句，以便启动时执行：
mongod --config /opt/mongodb/mongodb.conf
```

## 5.编译Crane程序

```shell
# 由于便于项目克隆git仓库，可以先设置好git代理
git config --global http.proxy http://<ip>:<port>
git config --global http.proxy http://<ip>:<port>

# 选择一个合适的位置克隆项目
git clone https://github.com/PKUHPC/Crane.git

cd Crane
mkdir build
cd build/

# 首次编译需要下载第三方库，耗时较长
cmake -G Ninja -DCMAKE_C_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/gcc -DCMAKE_CXX_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/g++ -DBOOST_INCLUDE_DIR=/usr/lib64/ -DBOOST_LIBRARY_DIR=/usr/include ..
ninja install
```

## 6.Pam模块(待完善)

首次编译完成后需要将pam模块动态链接库放入系统指定位置

```shell
cp Crane/build/src/Misc/Pam/pam_Crane.so /usr/lib64/security/
```

同时计算节点“/etc/security/access.conf”文件禁止非root用户登录

Required pam_access.so


## 7.运行项目

首先根据自身的集群情况在配置文件当中进行相应配置，配置文件样例保存在/etc/crane/config.yaml.example
```shell
cp /etc/crane/config.yaml.example /etc/crane/config.yaml
vim /etc/crane/config.yaml
```

启动服务
```shell
systemctl start cranectld       # 控制节点守护程序服务
systemctl start cranectld       # 计算节点守护程序服务
```

## *其他节点环境部署

其他节点部署项目，无需编译项目，仅需复制相应的执行文件和配置文件即可
```shell
pscp /usr/local/bin/cranectld all:/usr/local/bin/
pscp /usr/local/bin/craned all:/usr/local/bin/
pscp /etc/systemd/system/cranectld.service all:/etc/systemd/system/
pscp /etc/systemd/system/craned.service all:/etc/systemd/system/
pscp /etc/crane/config.yaml all:/etc/crane/
```
