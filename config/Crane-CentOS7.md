# Crane后端环境配置-CentOS 7 #

## 以下内容为配置代码编译环境，在编译项目的节点执行，假设用户为root ##
## 1.换源（仅针对北大校内机器） ##
~~~bash
# 清除所有源_c
rm -rf /etc/yum.repos.d/*

cat > /etc/yum.repos.d/CentOS-Base.repo << 'EOF'
# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#
[base]
name=CentOS-$releasever - Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
baseurl=http://mirrors.pku.edu.cn/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
baseurl=http://mirrors.pku.edu.cn/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
baseurl=http://mirrors.pku.edu.cn/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
baseurl=http://mirrors.pku.edu.cn/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
EOF

~~~

### 重建缓存 ###
~~~bash
yum makecache
~~~

### 安装其他扩展仓库，如果已经装了这两个要先卸载！！ ###
~~~bash
yum install -y epel-release centos-release-scl-rh
~~~

### 更换epel仓库为北大镜像源，修改repo文件/etc/yum.repos.d/epel.repo ###
~~~bash
cat > /etc/yum.repos.d/epel.repo << 'EOF'
[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
baseurl=https://mirrors.pku.edu.cn/epel/7/$basearch
#metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
baseurl=https://mirrors.pku.edu.cn/pub/epel/7/$basearch/debug
#metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-debug-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
baseurl=http://https://mirrors.pku.edu.cn/pub/epel/7/SRPMS
#metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-source-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1
EOF
~~~

###  更改scl仓库为北大镜像源，修改repo文件 ###
~~~bash
cat > /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo << 'EOF'
[centos-sclo-rh]
name=CentOS-7 - SCLo rh
baseurl=http://mirrors.pku.edu.cn/centos/7/sclo/$basearch/rh/
mirrorlist=http://mirrorlist.centos.org?arch=$basearch&release=7&repo=sclo-rh
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
EOF
~~~

###  重建缓存 ###
~~~bash
yum makecache
~~~

## 2.环境准备  ##
#### 关闭SeLinux  ####
~~~bash
#重启后失效
setenforce 0  
#重启后生效
sed -i s#SELINUX=enforcing#SELINUX=disabled# /etc/selinux/config
~~~

###  2.1 安装ca-certificates，确保系统能够安全地与外部服务器进行通信 ###
~~~bash
yum -y install ca-certificates
~~~

###  2.2 同步时钟 ###
~~~bash
yum -y install ntp ntpdate
systemctl start ntpd
systemctl enable ntpd

timedatectl set-timezone Asia/Shanghai
~~~

### 2.3 关闭防火墙，不允许关闭防火墙则考虑开放10011、10010、873端口  ###
~~~bash
systemctl stop firewalld
systemctl disable firewalld

# 上述两条命令不成功需要执行下面命令
#或者开放端口
firewall-cmd --add-port=10011/tcp --permanent --zone=public
firewall-cmd --add-port=10010/tcp --permanent --zone=public
firewall-cmd --add-port=873/tcp --permanent --zone=public
# 重启防火墙(修改配置后要重启防火墙)
firewall-cmd --reload
~~~

## 3. 安装运维工具链  ##
~~~bash
yum install -y tig tmux fish pdsh htop
~~~

## 4.安装依赖包 ##
~~~bash
yum install -y openssl-devel libcgroup-devel curl-devel pam-devel zlib-devel zlib-static libaio-devel

# 补充：libcgroup-devel的安装需要重新编译一下libcgroup文件
wet https://github.com/libcgroup/libcgroup/releases/download/v3.1.0/libcgroup-3.1.0.tar.gz
tar -zxvf libcgroup-3.1.0.tar.gz
cd libcgroup-3.1.0
dnf install tar bison flex systemd-devel -y
sudo ./configure
make -j 12
sudo make install
~~~

## 5.安装工具链 ##
#### 工具链版本 ####
- **cmake版本**>=3.24
- **libstdc++版本**>= 11
- **如果安装clang，版本**>= 19
- **如果安装g++,版本**>= 13
### 5.1 所有系统适用工具链 ###
#### 安装wget，tar ####
~~~bash
sudo dnf install -y wget
sudo dnf install -y tar
~~~

#### 安装cmake，选择一个合适的源码存放位置，从github下载源码 ####
~~~bash
wget https://github.com/Kitware/CMake/releases/download/v3.26.4/cmake-3.26.4-linux-x86_64.sh
~~~

#### 执行cmake安装脚本 ####
~~~bash
bash cmake-3.26.4-linux-x86_64.sh --prefix=/usr/local --skip-license
~~~

#### 检查cmake安装是否成功 ####
~~~bash
cmake --version
#cmake version 3.26.4
~~~

### 5.2 CentOS7 其他工具链包 ###
#### 5.2.1 命令行安装 gcc 13 ####
~~~bash
wget https://ftp.gnu.org/gnu/gcc/gcc-13.2.0/gcc-13.2.0.tar.gz
tar -zxvf gcc-13.2.0.tar.gz

yum install -y bzip2
cd gcc-13.2.0
./contrib/download_prerequisites

mkdir build
cd build/
../configure -enable-checking=release -enable-languages=c,c++ -disable-multilib

make -j 12
sudo make install
~~~

#### 5.2.2 yum安装 ####
#### 安装其他工具链包 ####
~~~bash
yum install -y ninja-build patch devtoolset-11 rh-git218
yum install -y devtoolset-11-libasan-devel devtoolset-11-libtsan-devel
~~~
#### 为了避免每次手动生效，可以在~/.bash_profile中设置 ####
~~~bash
vim ~/.bash_profile

source scl_source enable devtoolset-11
source scl_source enable rh-git218
~~~
#### 然后重启终端，这时用gcc --version命令查询，可以看到版本已经是11.2系列了 ####
~~~bash
gcc (GCC) 11.2.1 20220127 (Red Hat 11.2.1-9)
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
~~~

## 6. 编译Crane程序 ##
~~~bash
#不必要配置代理
git config --global http.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890
git config --global https.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890

# 选择一个合适的位置克隆项目
git clone https://github.com/PKUHPC/Crane.git

cd Crane
mkdir build
cd build

# 首次编译需要下载第三方库，耗时较长
cmake -G Ninja -DCMAKE_C_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/gcc -DCMAKE_CXX_COMPILER=/opt/rh/devtoolset-11/root/usr/bin/g++ ..
cmake --build . --target cranectld craned pam_crane

# 仅安装到本机，craned节点需手动scp
ninja install
~~~

## 7. Pam模块（最后节点部署成功再操作，否则会出现ssh不上远程主机的情况） ##
#### 首次编译完成后需要将pam模块动态链接库放入系统指定位置 ####
~~~bash
cp Crane/build/src/Misc/Pam/pam_Crane.so /usr/lib64/security/
#如果不行就在Pam目录下查找有可能生成so名字大小写不一致
cp Crane/build/src/Misc/Pam/pam_crane.so /usr/lib64/security/
~~~

#### 在 /etc/pam.d/sshd 中添加红色行： ####
~~~bash
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
session    required     pam_crane.so
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
-session   optional     pam_reauthorize.so prepare
~~~
### 注意：session optional pam_crane.so必须位于 session include password-auth之后！因为password-auth中有pam_systemd.so这个模块，会导致sshd session被移入systemd:/user.slice这个cgroups中！ ###

## 8. 安装mongodb ##
### 安装数据库仅在需要存储数据的节点安装 ###
#### 修改mongodb的yum源： ####
~~~bash
cat > /etc/yum.repos.d/mongodb-6.0.2.repo << 'EOF'
[mongodb-org-6.0.2]
name=MongoDB 6.0.2 Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=0
enabled=1
EOF

yum makecache
~~~

#### 补充：若爆出-bash: /etc/yum.repos.d/mongodb-6.0.2.repo: Permission denied可使用下方命令： ####
~~~bash
sudo tee /etc/yum.repos.d/mongodb-6.0.2.repo << 'EOF'
[mongodb-org-6.0.2]
name=MongoDB 6.0.2 Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/6.0/x86_64/
gpgcheck=0
enabled=1
EOF

yum makecache
~~~

#### 安装并添加mongodb开机启动 ####
~~~bash
yum install mongodb-org -y
# 添加开机启动
systemctl enable mongod
systemctl start mongod
~~~

#### 安装完mongod用户的home目录为/var/lib/mongo  ####
#### 利用openssl在/var/lib/mongo生成密钥文件 ####
~~~bash
openssl rand -base64 756 | sudo -u mongod tee /var/lib/mongo/mongo.key
sudo -u mongod chmod 400 /var/lib/mongo/mongo.key
~~~

#### 创建用户 ####
~~~bash
# 从mongodb6.0开始，mongo
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
~~~

#### 修改/etc/mongod.conf配置文件，开启权限验证，并配置副本集配置 ####
~~~bash
vim /etc/mongod.conf

......
#开启权限验证
security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongo.key
replication:
  #副本集名称,crane的配置文件要与此一致
  replSetName: crane_rs 
~~~

#### 重新启动mongodb数据库 ####
~~~bash
systemctl restart mongod
~~~

#### 进入mongosh，初始化副本集 #### 
~~~bash
mongosh
use admin
db.auth("admin","123456")

// 如果不需要配置外部连接，并且副本集只有该主机一个节点，config可不配置
config = {
  "_id": "crane_rs",      // 注意名称一致
  "members": [
    {
      "_id": 0,
      "host": "<hostname>:27017" // 建议这里填写部署数据集的节点主机名，默认为127.0.0.1
    }
    // ... 其他节点（如果有的话）
  ]
}

rs.initiate()
~~~

## 9. 运行项目 ##
#### 首先根据自身的集群情况在配置文件当中进行相应配置，配置文件样例保存在etc/crane/config.yaml.example ####
~~~bash
mkdir -p /etc/crane
cp etc/config.yaml.example /etc/crane/config.yaml
#上述命令报错就按照如下输入
cp etc/config.yaml /etc/crane/

vim /etc/crane/config.yaml
~~~
#### 直接执行可执行文件启动 此时目录应该在项目根目录 #### 
~~~bash
cd build/src

# Cranectld启动命令
CraneCtld/cranectld

# Craned启动命令
Craned/craned
~~~
#### Systemctl 启动服务 (现在这个用不了！) ####
~~~bash
systemctl start cranectld       # 控制节点守护程序服务
systemctl start craned       # 计算节点守护程序服务
~~~

## 10. 其他节点环境部署 ##
### 10.1 SCP命令版本 ###
#### 计算节点部署项目，无需编译项目，仅需复制相应的执行文件和配置文件即可 ####
~~~bash
# 比如配置计算节点crane02
ssh crane02 "mkdir -p /etc/crane"
scp /usr/local/bin/craned crane02:/usr/local/bin/
scp /etc/systemd/system/craned.service crane02:/etc/systemd/system/
scp /etc/crane/config.yaml crane02:/etc/crane/
~~~

### 10.2 PDSH版本 ###
#### 如果没安装pdsh，安装pdsh ####
~~~bash
yum install -y pdsh
~~~

#### 更新cranectld ####
~~~bash
# 注意需位于编译路径下执行
pdsh -w cranectl systemctl stop cranectld
pdcp -w cranectl src/CraneCtld/cranectld /usr/local/bin
pdsh -w cranectl systemctl start cranectld
~~~

#### 更新craned ####
~~~bash
pdsh -w crane0[1-4] systemctl stop craned
pdcp -w crane0[1-4] Craned/craned /usr/local/bin
pdsh -w crane0[1-4] systemctl start craned
~~~

## 11. 其他配置 ##
### 11.1 Fish shell ###
~~~bash
cd /etc/yum.repos.d/
wget https://download.opensuse.org/repositories/shells:/fish:/release:/3/CentOS_7/shells:fish:release:3.repo
yum makecache
yum install -y fish
~~~

### 11.2 Nix ###
~~~bash
见 https://mirrors.tuna.tsinghua.edu.cn/help/nix/
~~~

## 附：多节点环境部署说明 ##
## scp ##
#### 在计算节点部署 Craned 时无需完整编译项目，仅需复制相应的可执行文件和配置文件： ####
~~~bash
# 例如配置计算节点 crane02
ssh crane02 "mkdir -p /etc/crane"
scp /usr/local/bin/craned crane02:/usr/local/bin/
scp /etc/systemd/system/craned.service crane02:/etc/systemd/system/
scp /etc/crane/config.yaml crane02:/etc/crane/
~~~
#### 注意：计算节点仍需完成“安装项目依赖”中的 libcgroup 安装部分。 ####





# Crane 前端环境配置-CentOS 7 #
## 理论上在任何使用 systemd 的系统上都能使用（例如 Debian/Ubuntu/AlmaLinux/Fedora 等）。该教程涉及的软件基于 ARM64。如果使用 x86-64 等架构，请调整软件下载链接。请全程以 root 用户执行命令。建议先完成后端环境的安装。##
## 1. 安装 Golang ##
~~~bash
GOLANG_TARBALL=go1.22.0.linux-amd64.tar.gz
curl -L https://go.dev/dl/${GOLANG_TARBALL} -o /tmp/go.tar.gz

# 移除旧版本的 Golang 环境
rm -rf /usr/local/go

tar -C /usr/local -xzf /tmp/go.tar.gz && rm /tmp/go.tar.gz
echo 'export GOPATH=/root/go' >> /etc/profile.d/go.sh
echo 'export PATH=$GOPATH/bin:/usr/local/go/bin:$PATH' >> /etc/profile.d/go.sh
echo 'go env -w GO111MODULE=on' >> /etc/profile.d/go.sh
echo 'go env -w GOPROXY=https://goproxy.cn,direct' >> /etc/profile.d/go.sh

source /etc/profile.d/go.sh
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
~~~

## 2. 安装 Protoc ##
~~~bash
PROTOC_ZIP=protoc-23.2-linux-x86_64.zip \
curl -L https://github.com/protocolbuffers/protobuf/releases/download/v23.2/${PROTOC_ZIP} -o /tmp/protoc.zip
unzip /tmp/protoc.zip -d /usr/local
rm /tmp/protoc.zip /usr/local/readme.txt
~~~

## 3. 拉取项目 ##
~~~bash
git clone https://github.com/PKUHPC/CraneSched-FrontEnd.git
~~~

## 4. 编译项目并部署前端 ##
#### 工作目录为CraneSched-FrontEnd，在该目录下编译所有 Golang 组件并安装。 ####
~~~bash
cd CraneSched-FrontEnd
make
make install
~~~

## 5. 启动 Cfored 和 Cplugind（可选）##
#### 如果需要提交交互式任务（crun, calloc），则需要启用 Cfored： ####
~~~bash
# 设置开机启动
systemctl enable cfored
systemctl start cfored
~~~
#### 如果配置文件中启用了插件系统，则需要启用 Cplugind： ####
~~~bash
# 设置开机启动
systemctl enable cplugind
systemctl start cplugind
~~~

## 6. 安装 Cwrapper 别名（可选） ##
#### 可以通过下列命令安装 Crane 的 Slurm 别名，从而支持使用 Slurm 的命令形式使用 Crane： ####
~~~bash
cat > /etc/profile.d/cwrapper.sh << 'EOF'
alias sbatch='cwrapper sbatch'
alias sacct='cwrapper sacct'
alias sacctmgr='cwrapper sacctmgr'
alias scancel='cwrapper scancel'
alias scontrol='cwrapper scontrol'
alias sinfo='cwrapper sinfo'
alias squeue='cwrapper squeue'
EOF

# 文件权限修改为644
~~~