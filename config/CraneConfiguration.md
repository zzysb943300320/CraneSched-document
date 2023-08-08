# CraneSched 项目环境配置

***以下内容为配置代码编译环境，在编译项目的节点执行***
环境准备：
关闭seLinux
安装节点：crane01, crane02, crane03
cranectld节点：crane01
craned节点：crane02， crane03


## 1.环境准备

安装ca-certificates
```shell
yum -y install ca-certificates
```

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
yum install -y openssl-devel libcgroup-devel \
    curl-devel boost169-devel boost169-static pam-devel \
    zlib-devel zlib-static
```

## 3.安装工具链

工具链版本：
* `cmake`版本 >= `3.24`
* `libstdc++`版本 >= `11`
* 如果安装`clang`，版本 >= `15`
* 如果安装`g++`，版本 >= `11`

### 3.1.所有系统都适用

安装cmake，选择一个合适的源码存放位置，从github下载源码

```
wget https://github.com/Kitware/CMake/releases/download/v3.24.2/cmake-3.24.2-linux-x86_64.sh
```

执行cmake安装脚本
```
bash cmake-3.24.2-linux-x86_64.sh --prefix=/usr/local --skip-license
```

检查cmake安装是否成功
```
cmake --version
#cmake version 3.24.2
```

### 3.2. CentOS7

安装其他工具链包
```
yum install -y ninja-build patch devtoolset-11 rh-git218
```

为了避免每次手动生效，可以在`~/.bash_profile`中设置
```
vim ~/.bash_profile
```
```
source scl_source enable devtoolset-11
source scl_source enable rh-git218
```

然后重启终端，这时用gcc --version命令查询，可以看到版本已经是11.2系列了
```
gcc (GCC) 11.2.1 20220127 (Red Hat 11.2.1-9)
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### 3.3. Ubuntu 20.04或包管理无`libstdc++-11`的系统

#### 3.3.1. 编译安装`gcc`

由于Ubuntu 20.04官方仓库中`libstdc++`最大为`10`，通过手动编译安装`g++-11`来安装`libstdc++-11`
```
wget http://ftp.gnu.org/gnu/gcc/gcc-11.3.0/gcc-11.3.0.tar.gz
tar -zxvf gcc-11.3.0.tar.gz

sudo apt-get install bzip2
cd gcc-11.3.0
./contrib/download_prerequisites

mkdir build
cd build/
../configure -enable-checking=release -enable-languages=c,c++ -disable-multilib

make -j 12
sudo make install
```

#### 3.3.2. 指定`clang`选择手动安装的`libstdc++`

由于没有指定`--prefix`，默认安装到`/usr/local`。如果希望`clang`和`clang++`能选择安装到这个目录的`gcc-toolchain`，需要通过`clang`和`clang++`的`--gcc-toolchain`的参数指定目录。

在`cmake`的命令行参数指定`-DCMAKE_C_FLAGS_INIT="--gcc-toolchain=/usr/local" -DCMAKE_CXX_FLAGS_INIT="--gcc-toolchain=/usr/local"`

## 4. 编译`Crane`程序
```shell
git config --global http.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890
git config --global https.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890

# 选择一个合适的位置克隆项目
git clone https://github.com/PKUHPC/Crane.git

cd Crane
mkdir build
cd build

# 首次编译需要下载第三方库，耗时较长
cmake -G Ninja -DCMAKE_C_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/gcc -DCMAKE_CXX_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/g++ -DBoost_INCLUDE_DIR=/usr/include/boost169/ -DBoost_LIBRARY_DIR=/usr/lib64/boost169/ ..
ninja cranectld craned pam_crane

# 仅安装到本机，craned节点需手动scp
ninja install
```

## 5. Pam模块
不同系统不一样

### 5.1. CentOS 7
首次编译完成后需要将pam模块动态链接库放入系统指定位置
```
cp Crane/build/src/Misc/Pam/pam_Crane.so /usr/lib64/security/
```
在 /etc/pam.d/sshd 中添加红色行：
```
#%PAM-1.0
auth       required     pam_sepermit.so
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
-auth      optional     pam_reauthorize.so prepare
account    required     pam_crane.so
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    optional     pam_crane.so
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
```

注意：`session optional pam_crane.so`必须位于 `session include password-auth`之后！因为`password-auth`中有`pam_systemd.so`这个模块，会导致sshd session被移入`systemd:/user.slice`这个cgroups中！

目前不清楚systemd是否会定期轮询相应的进程是否被steal，待测试。

## 6.安装mongodb

***安装数据库仅在需要存储数据的节点安装***

修改mongodb的yum源:
```shell
cat >> /etc/yum.repos.d/mongodb-6.0.2.repo << 'EOF'
[mongodb-org-6.0.2]
name=MongoDB 6.0.2 Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=0
enabled=1
EOF

yum makecache
```
安装并添加mongodb开机启动
```
yum install mongodb-org -y
# 添加开机启动
systemctl enable mongod
systemctl start mongod
```
安装完`mongod`用户的home目录为`/var/lib/mongo`

利用openssl在`/var/lib/mongo`生成密钥文件
```shell
openssl rand -base64 756 | sudo -u mongod tee /var/lib/mongo/mongo.key
sudo -u mongod chmod 400 /var/lib/mongo/mongo.key
```
创建用户
```shell
# 从mongodb6.0开始，mongo命令被mongosh命令取代
mongosh

# 进入mongodb之后进行下列操作
use admin
# user: 用户名 pwd：密码 roles：root 代表超級管理员权限 admin代表给admin数据库加的超级管理员
db.createUser({
  user:'admin', pwd:'123456', roles:[{ role:'root',db:'admin'}]   
})

# 重启前先关闭服务器
db.shutdownServer() 
quit
```
修改/etc/mongod.conf配置文件，开启权限验证，并配置副本集配置
```shell
vim /etc/mongod.conf

......
#开启权限验证
security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongo.key
replication:
  #副本集名称,crane的配置文件要与此一致
  replSetName: crane_rs 
```
重新启动mongodb数据库
```shell
systemctl restart mongod
```
进入mongosh，初始化副本集
```shell
mongosh
db.auth（"admin","123456"）
rs.initiate()
```

## 7.运行项目

首先根据自身的集群情况在配置文件当中进行相应配置，配置文件样例保存在/`etc/crane/config.yaml.example`
```shell
mkdir -p /etc/crane
cp etc/config.yaml.example /etc/crane/config.yaml
vim /etc/crane/config.yaml
```
直接执行可执行文件启动 此时目录应该在项目根目录
```shell
cd build/src

# Cranectld启动命令
CraneCtld/cranectld

# Craned启动命令
Craned/craned
```
Systemctl 启动服务 (现在这个用不了！)
```shell
systemctl start cranectld       # 控制节点守护程序服务
systemctl start craned       # 计算节点守护程序服务
```

## 8. 其他节点环境部署
其他节点部署项目，无需编译项目，仅需复制相应的执行文件和配置文件即可
```shell
# 比如配置计算节点crane02
ssh crane02 "mkdir -p /etc/crane"
scp /usr/local/bin/craned crane02:/usr/local/bin/
scp /etc/systemd/system/craned.service crane02:/etc/systemd/system/
scp /etc/crane/config.yaml crane02:/etc/crane/
```

## 9. 其他配置
1. Fish shell
    ```shell
    cd /etc/yum.repos.d/
    wget https://download.opensuse.org/repositories/shells:/fish:/release:/3/CentOS_7/shells:fish:release:3.repo
    yum makecache
    yum install -y fish
    ```
2. Nix
   
    见 https://mirrors.tuna.tsinghua.edu.cn/help/nix/