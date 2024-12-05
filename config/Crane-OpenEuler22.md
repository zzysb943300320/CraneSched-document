# Crane 后端环境配置 - OpenEuler22 #
#### 该教程基于 ARM64 架构。请全程以 root 用户执行命令 ####

## 1. 环境准备 ##
### 1.1 添加 EPEL 软件源 ###
~~~bash
curl -o /etc/yum.repos.d/epel-OpenEuler.repo https://down.whsir.com/downloads/epel-OpenEuler.repo
~~~

### 1.2 启用时间同步 ###
~~~bash
dnf install -y chrony
systemctl restart systemd-timedated
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp true
~~~

### 1.3 关闭防火墙 ###
~~~bash
systemctl stop firewalld
systemctl disable firewalld
~~~
#### 若集群不允许关闭防火墙，则考虑开放 10010、10011、10012、873 端口。####
~~~bash
firewall-cmd --add-port=10012/tcp --permanent --zone=public
firewall-cmd --add-port=10011/tcp --permanent --zone=public
firewall-cmd --add-port=10010/tcp --permanent --zone=public
firewall-cmd --add-port=873/tcp --permanent --zone=public
firewall-cmd --reload
~~~
#### 注意：若有多个节点，需在每个节点上执行此步骤，否则无法进行节点间的通信。####

### 1.4 关闭SELinux ###
~~~bash
#重启后失效
setenforce 0  
#重启后生效
sed -i s#SELINUX=enforcing#SELINUX=disabled# /etc/selinux/config
~~~

### 1.5 切换CGroup 版本 ###
#### mount | grep cgroup 如果显示 CgroupV2 需要切换为 V1 ####
#### CraneSched 目前只支持 CGroup v1，需按照以下步骤切换版本 ####
~~~bash
# 设置内核启动参数，更改 CGroup 版本。
grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
# 重启系统生效
reboot
# 验证版本
mount | grep cgroup
~~~
#### 注意：所有计算节点（运行 Craned 的节点）均需要切换 CGroup 版本 ####

## 2. 安装工具链 ##
### 2.1 版本要求 ###
#### 工具链版本需符合以下要求：####
- **cmake版本**>= 3.26
- **如果安装clang，版本**>= 15
- **如果安装g++，版本**>= 13
### 2.2 安装构建工具 ###
~~~bash
dnf install -y \
    patch \
    flex \
    bison \
    ninja-build
~~~
#### 下载和解压 GCC 源码 ####
~~~bash
wget https://mirror.koddos.net/gcc/releases/gcc-13.2.0/gcc-13.2.0.tar.xz
tar -xvf gcc-13.2.0.tar.xz
cd gcc-13.2.0
~~~
#### 下载依赖 ####
~~~bash
./contrib/download_prerequisites
~~~
#### 创建编译目录并进入 ####
~~~bash
mkdir -p build && cd build
~~~
#### 配置编译参数，指定安装路径为 /opt/gcc-13.2.0 ####
~~~bash
../configure --prefix=/opt/gcc-13.2.0 --enable-languages=c,c++
~~~
#### 编译 GCC 并安装到 /opt/gcc-13.2.0 ####
~~~bash
make -j$(nproc)
make install
~~~
#### 安装 cmake ####
~~~bash
wget https://github.com/Kitware/CMake/releases/download/v3.26.4/cmake-3.26.4-linux-aarch64.sh
bash cmake-3.26.4-linux-aarch64.sh --prefix=/usr/local --skip-license
~~~
#### 检查 cmake 安装是否成功 ####
~~~bash
cmake --version
~~~

### 2.3 安装常用工具 ###
~~~bash
dnf install -y tar curl unzip git
~~~
#### 此外还可以安装以下常用运维工具（可选）： ####
~~~bash
dnf install -y tig tmux fish pdsh htop vim
~~~

## 3. 安装项目依赖 ##
~~~bash
# 安装项目相关依赖包
dnf install -y \
    libstdc++-devel \
    libstdc++-static \
    openssl-devel \
    curl-devel \
    pam-devel \
    zlib-devel \
    libaio-devel \
    systemd-devel

# 下载并编译安装 libcgroup
curl -L https://github.com/libcgroup/libcgroup/releases/download/v3.1.0/libcgroup-3.1.0.tar.gz -o /tmp/libcgroup.tar.gz 
tar -C /tmp -xzf /tmp/libcgroup.tar.gz
cd /tmp/libcgroup-3.1.0
./configure --prefix=/usr/local
make -j$(nproc)
make install
rm -rf /tmp/libcgroup-3.1.0 /tmp/libcgroup.tar.gz
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH' >> /etc/profile.d/extra.sh
~~~
#### 安装后建议重新登陆集群 / 手动加载环境配置： ####
~~~bash
source /etc/profile.d/extra.sh
~~~

## 4. 安装和配置 MongoDB ##
### 此步骤仅在存储节点（控制节点）进行，其他节点不需要安装数据库。###
### 4.1 安装 MongoDB ###
#### 1. 添加 MongoDB 的 YUM 源： ####
~~~bash
cat > /etc/yum.repos.d/mongodb-org-7.0.repo << 'EOF'
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/7.0/aarch64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
EOF

dnf makecache
~~~

#### 2. 安装 MongoDB 并添加开机启动 ####
~~~bash
wget https://repo.mongodb.org/yum/redhat/8/mongodb-org/7.0/aarch64/RPMS/mongodb-org-database-tools-extra-7.0.9-1.el8.aarch64.rpm
rpm -ivh --force ./mongodb-org-database-tools-extra-7.0.9-1.el8.aarch64.rpm --nodeps
dnf install -y mongodb-org
# 添加 MongoDB 开机启动
systemctl enable mongod
systemctl start mongod
~~~

#### 3. 使用 OpenSSL 生成密钥文件 ####
~~~bash
openssl rand -base64 756 | sudo -u mongod tee /var/lib/mongo/mongo.key
sudo -u mongod chmod 400 /var/lib/mongo/mongo.key
~~~

### 4.2 配置 MongoDB ###
#### 1. 在数据库中创建用户 ####
~~~bash
# 使用 mongosh 进入 MongoDB 命令行
mongosh
# 进入 mongosh 后执行下列操作
use admin
# user: 用户名
# pwd: 密码
# roles: root 代表超级管理员权限
# db: admin 代表给 admin 数据库添加超级管理员
db.createUser({
  user: 'admin', pwd: '123456', roles: [{ role: 'root', db: 'admin' }]   
})

# 重启前先关闭服务器
db.shutdownServer() 
quit
~~~

#### 2. 开启权限验证，并配置副本集 ####
#### 打开 /etc/mongod.conf 配置文件，找到 security 和 replication 部分，进行如下修改： ####
~~~bash
vim /etc/mongod.conf

# 以上省略...
# 开启权限验证
security:
  authorization: enabled
  keyFile: /var/lib/mongo/mongo.key
replication:
  # 副本集名称，稍后填写 crane 配置文件时需一致
  replSetName: crane_rs
~~~
####  重启 MongoDB 使配置生效 ####
~~~bash
systemctl restart mongod
~~~
#### 3. 初始化副本集 ####
~~~bash
mongosh
use admin

# 使用刚刚设置的账号密码登录
db.auth("admin","123456")
# 如果不需要配置外部连接，并且副本集只有该主机一个节点，config 可不配置
config = {
  "_id": "crane_rs",  # 注意名称一致
  "members": [
    {
      "_id": 0,
      "host": "<hostname>:27017" # 替换为部署副本集的主机名，默认为127.0.0.1
    }
    # ... 其他节点（若有）
  ]


# 初始化并启动副本集
rs.initiate()
~~~

## 5. 安装和配置 CraneSched ##
### 5.1 编译二进制文件 ###
~~~bash
# 选择一个合适的位置 Clone 项目
git clone https://github.com/PKUHPC/CraneSched.git

# 进入项目根目录
cd CraneSched

# 创建并进入编译目录
mkdir -p build
pushd build

# 在编译目录下进行编译，首次编译需下载第三方库，耗时较长
cmake -G Ninja \
    -DCMAKE_CXX_COMPILER=/opt/gcc-13.2.0/bin/g++ \
    -DCMAKE_C_COMPILER=/opt/gcc-13.2.0/bin/gcc ..
cmake --build . --target cranectld craned pam_crane

# 将可执行文件和 Systemd 服务安装到本机，其他节点需要手动拷贝
ninja install
popd
~~~
#### 注意：如果拉取仓库或依赖时出错，请使用代理 ####

### 5.2 配置 PAM 模块 ###
#### 请在整个集群部署成功并正常运行后再进行此操作，否则会导致 SSH 认证失败无法连接 ####
#### 1. 将 PAM 模块拷贝到系统指定位置 ####
~~~bash
# 在项目根目录下操作
cp build/src/Misc/Pam/pam_crane.so /usr/lib64/security/
~~~
#### 2. 修改 /etc/pam.d/sshd 配置文件 ####
#### 仿照以下样例配置文件（不同系统略有不同）， 找到加粗的行，在对应位置添加标红的行：####
#### 找到 account include password-auth，在之前添加 account required pam_crane.so ####
#### 找到 session include password-auth，在之后添加 session required pam_crane.so ####
~~~bash
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
account    required     pam_sepermit.so
account    required     pam_nologin.so
account    required     pam_crane.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    optional     pam_motd.so
session    include      password-auth
session    required     pam_crane.so
session    include      postlogin
~~~
#### session optional pam_crane.so必须位于 session include password-auth之后。因为password-auth中有pam_systemd.so模块，会导致 sshd session 被移入systemd:/user.slice这个 cgroups 中。目前不清楚 systemd 是否会定期轮询相应的进程是否被 steal，待测试。####

### 5.3 配置 CraneSched ###
#### 1. 根据样例创建配置文件。样例配置文件在项目目录下etc/crane/ 中，将其拷贝到 /etc/crane 中：####
~~~bash
# 创建配置文件目录
mkdir -p /etc/crane

# 假设当前在项目根目录
# 拷贝配置文件样例
cp etc/config.yaml /etc/crane/config.yaml
cp etc/database.yaml /etc/crane/database.yaml
~~~

#### 2. 在 /etc/crane/config.yaml 中配置节点信息、调度偏好等选项（需所有节点保持一致）请根据集群实际情况填写。例如，一个四节点的集群中，控制节点的主机名为 crane01，计算节点的主机名为 crane01、crane02、crane03、crane04，则如下填写：####
~~~bash
vim /etc/crane/config.yaml

# 以上省略...
# 控制节点（主节点）
ControlMachine: crane01
# ...

# Nodes and partitions settings
# 计算节点
Nodes:
  - name: "crane[01-04]"
    cpu: 2
    memory: 2G

# partition information list
# 计算节点的分区
Partitions:
  - name: CPU              # 分区的名称（可自定义）
    nodes: "crane[01-02]"  # 分区中的节点，需要和 Nodes 部分对应
    priority: 5
  - name: GPU
    nodes: "crane[03-04]"
    priority: 3
    # Optional default memory per cpu in MB
    DefaultMemPerCpu: 0     # 建议设置为0
    # Optional maximum memory per cpu in MB, 0 indicates no limit
    MaxMemPerCpu: 0         # 建议设置为0

# 默认分区，未指定分区的作业将被提交到默认分区
DefaultPartition: CPU
~~~

#### 3. 在 /etc/crane/database.yaml 中配置数据库信息（只需在控制节点配置）####
~~~bash
# EmbeddedDb settings
# BerkeleyDB or Unqlite(default)
CraneEmbeddedDbBackend: Unqlite
# File path of CraneCtld embeded DB (Relative to CraneBaseDir)
CraneCtldDbPath: cranectld/embedded.db

# MongoDB 信息需要与数据库的配置相一致
DbUser: admin
DbPassword: "123456"
DbHost: localhost
DbPort: 27017
DbReplSetName: crane_rs
DbName: crane_db
~~~

## 6. 启动 CraneSched ##
#### 可直接在前台启动 CraneSched（控制节点启动 CraneCtld，其他节点按需要启动 Craned） ####
~~~bash
# 假设当前在项目根目录
cd build/src

# 启动 Cranectld
CraneCtld/cranectld
# 启动 Craned
Craned/craned
~~~

#### 可通过 Systemd 在后台运行 CraneSched，并设置开机启动 ####
~~~bash
# CraneCtld
systemctl enable cranectld 
systemctl start cranectld

# Craned
systemctl enable craned
systemctl start craned
~~~

## 附 1：常见问题 ##
#### 1. 如果运行 CMake 查找不到 libcgroup 包或版本不匹配，请参考“安装依赖”部分，安装 Release 版本的 libcgroup。####
#### 2. 运行 craned 时，系统无法找到 libcgroup.so.0 这个共享库。通常是因为该库不在系统的默认库搜索路径中。可使用 pkg-config 工具排查。####
#### 3. Craned 和 CraneCtld 都成功启动，但无法 cinfo 查询发现 Craned 没有上线。通常是因为没有关闭防火墙。####

## 附 2：多节点环境部署说明 ##
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

## PDSH ##
#### 1. 更新 CraneCtld ####
~~~bash
# 位于编译路径下执行
pdsh -w cranectl systemctl stop cranectld
pdcp -w cranectl src/CraneCtld/cranectld /usr/local/bin
pdsh -w cranectl systemctl start cranectld
~~~

#### 2. 更新 Craned #### 
~~~bash
pdsh -w crane[01-04] systemctl stop craned
pdcp -w crane[01-04] Craned/craned /usr/local/bin
pdsh -w crane[01-04] systemctl start craned
~~~







# Crane 前端环境配置-OpenEuler22 #
#### 理论上在任何使用 systemd 的系统上都能使用（例如 Debian/Ubuntu/AlmaLinux/Fedora 等）。该教程涉及的软件基于 ARM64。如果使用 x86-64 等架构，请调整软件下载链接。请全程以 root 用户执行命令。建议先完成后端环境的安装。####
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