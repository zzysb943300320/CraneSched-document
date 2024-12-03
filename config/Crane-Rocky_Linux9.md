# Crane 后端环境配置 - Rocky Linux 9 #
## 该教程基于 Rocky Linux 9，但也基本适用于 Rocky Linux 8 / Fedora。该教程涉及的软件基于 x86-64。如果使用 ARM64 等架构，请调整软件下载链接。请全程以 root 用户执行命令。 ##

## 1. 环境准备 ##
### 1.1 添加 EPEL 软件源 ###
~~~bash
dnf install -y yum-utils epel-release
dnf config-manager --set-enabled crb
dnf update -y

# 建议重启以应用更新后的内核
reboot
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
#### Rocky 9 默认使用 CGroup v2。CraneSched CGroup v2支持需要额外配置 ####
#### 1. CgroupV1切换和配置使用CgroupV1需按照以下步骤切换版本： ####
~~~bash
# 设置内核启动参数，更改 CGroup 版本。
grubby --update-kernel=/boot/vmlinuz-$(uname -r) --args="systemd.unified_cgroup_hierarchy=0 systemd.legacy_systemd_cgroup_controller"
# 重启系统生效
reboot
# 验证版本
mount | grep cgroup
~~~
#### 2.  CGroup V2配置 ####
#### 需要注意 root cgroup 下的子 cgroup 有无对相关资源的操作权限 #### 
~~~bash
# 检查子cgroup是否有相关资源的权限，如输出 cpu io memory 等
cat /sys/fs/cgroup/cgroup.subtree_control
# 设置子组的权限
echo '+cpuset +cpu +io +memory +pids' > /sys/fs/cgroup/cgroup.subtree_control
~~~
#### 注意：CGroup V2 对 device 的控制与 V1 使用的机制不同，V2 使用 eBPF 控制设备访问，在 CGroup V2 eBPF对 bpf 程序进行配置 ####

## CGroup V2 eBPF ##
#### eBpf需要使用clang编译，请确保系统中有clang17及以上 ####
### 1. clang19安装教程： ###
~~~bash
dnf install \
    bpftool \
    bcc \
    bcc-tools \
    elfutils-libelf-devel \
    zlib-devel

# 源码编译安装 clang19
git clone --depth=1 --branch llvmorg-19.1.0 https://github.com/llvm/llvm-project.git \
    llvm-project-19.1.0
cd llvm-project-19.1.0/

dnf install -y libedit-devel ncurses-devel libxml2-devel python3-devel swig

mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX='/usr/local' \
    -DCMAKE_BUILD_TYPE='Release' -G Ninja \
    -DLLVM_ENABLE_PROJECTS='clang;clang-tools-extra;lld;lldb' -DLLVM_ENABLE_RUNTIMES=all \
    -DLLVM_TARGETS_TO_BUILD='X86;BPF' ../llvm
ninja && ninja install

cd ../
mkdir build-libcxx && cd build-libcxx
cmake -G Ninja -DCMAKE_INSTALL_PREFIX='/usr/local' -DCMAKE_C_COMPILER=clang \
    -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_BUILD_TYPE=Release -S ../runtimes \
    -DLLVM_ENABLE_RUNTIMES="libcxx;libcxxabi;libunwind"
ninja cxx cxxabi unwind
#ninja check-cxx check-cxxabi check-unwind
ninja install-cxx install-cxxabi install-unwind

# Install asan and tsan header and libs for develop build
cd ../
mkdir build-compiler-rt && cd build-compiler-rt
cmake ../compiler-rt -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
    -DCMAKE_INSTALL_PREFIX='/usr/local' -DCMAKE_BUILD_TYPE='Release' -G Ninja \
    -DLLVM_CMAKE_DIR=../cmake/modules
ninja install

# 下载并编译安装 libbpf
wget https://github.com/libbpf/libbpf/archive/refs/tags/v1.4.6.zip
unzip v1.4.6.zip
cd libbpf-1.4.6/src

# 编译并安装
make
make install
~~~

### 2. ebpf 系统配置 ###
~~~bash
# 在项目build目录下
cp ./src/Misc/BPF/cgroup_dev_bpf.o /etc/crane/cgroup_dev_bpf.o

# 检查子cgroup是否有相关资源的权限，如输出 cpu io memory 等
cat /sys/fs/cgroup/cgroup.subtree_control
# 设置子组的权限
echo '+cpuset +cpu +io +memory +pids' > /sys/fs/cgroup/cgroup.subtree_control

# 出现 bpf load失败解决方式：
# 挂载 bpf 文件系统
mount -t bpf bpf /sys/fs/bpf
#挂载 bpf 调试文件
mount -t debugfs none /sys/kernel/debug
# 使用'cat /sys/kernel/debug/tracing/trace_pipe'查看设备访问日志
~~~

### 3.  BPF 文件系统挂载 ###
~~~bash
# 出现下面错误后：
libbpf: specified path /sys/fs/bpf/dev_map is not on BPF FS
libbpf: map 'dev_map': failed to auto-pin at '/sys/fs/bpf/dev_map': -22
libbpf: map 'dev_map': failed to create: Invalid argument(-22)
libbpf: failed to load object 'cgroup_dev_bpf.o'
Failed to load BPF object
~~~
~~~bash
# 检查 bpf 文件系统是否挂载
mount | grep bpf
# 挂载 BPF 文件系统
mkdir -p /sys/fs/bpf 
mount -t bpf bpf /sys/fs/bpf
~~~



## 2. 安装工具链 ##
### 2.1 版本要求 ###
#### 工具链版本需符合以下要求：####
- **cmake版本**>= 3.26
- **如果安装clang，版本**>= 15
- **如果安装g++，版本**>= 13
### 2.2 安装构建工具 ###
~~~bash
dnf install -y \
    gcc-toolset-13 \
    cmake \
    patch \
    flex \
    bison \
    ninja-build
 
echo 'source /opt/rh/gcc-toolset-13/enable' >> /etc/profile.d/extra.sh
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
    automake
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
cmake -G Ninja ..
cmake --build . --target cranectld craned pam_crane
# CgroupV2
cmake --build . --target cgroup_dev_bpf_object

# 将可执行文件和 Systemd 服务安装到本机，其他节点需要手动拷贝
ninja install
popd
~~~
#### 注意：如果拉取仓库或依赖时出错，请使用代理 ####

### 5.2 配置 PAM 模块 ###
#### 请在整个集群部署成功并正常运行后再进行此操作，否则会导致 SSH 认证失败无法连接！ ####
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

# 如果使用 CGroup V2
cp build/src/Misc/BPF/cgroup_dev_bpf.o /etc/crane/cgroup_dev_bpf.o
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

## 附 3：便捷安装脚本 ##
#### 该脚本适用于 Rocky 9，包括工具链安装、项目依赖安装和 CraneSched 的编译 ####
#### 注意：#### 
#### 执行该脚本前需要先完成 环境准备 部分 ####
#### 执行该脚本后仍然需要自行 安装 MongoDB 和 配置 CraneSched #### 
#### 为保证依赖下载顺利，该脚本设置了代理，请在 setp 函数中修改代理配置 ####
~~~bash
#!/bin/bash

# Tested on Rocky Linux 9.3

set -eo pipefail

# Function to set and unset proxy
setp() {
    export https_proxy=http://crane:hf2lH9UUC3E0@192.168.1.1:7890
    export http_proxy=http://crane:hf2lH9UUC3E0@192.168.1.1:7890
    git config --global http.proxy $http_proxy
    git config --global https.proxy $https_proxy
}

unsetp() {
    unset http_proxy
    unset https_proxy
    git config --global --unset http.proxy
    git config --global --unset https.proxy
}

# Tools 
dnf install -y tar unzip git wget curl || {
    echo "Error installing tools" && exit 1
}

# Dependency for libcgroup
dnf install -y bison flex systemd-devel || {
    echo "Error installing dependency" && exit 1
}

# Ensure the installation can be found
export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH

# Check if libcgroup is already installed
if pkg-config --exists libcgroup; then
    echo "libcgroup is already installed."
else
    if [ ! -f "libcgroup-3.1.0.tar.gz" ]; then
        setp
        wget https://github.com/libcgroup/libcgroup/releases/download/v3.1.0/libcgroup-3.1.0.tar.gz || {
            echo "Error downloading libcgroup" && exit 1
        }
        unsetp
    fi

    tar -xzf libcgroup-3.1.0.tar.gz && pushd libcgroup-3.1.0
    (./configure --prefix=/usr/local && make -j && make install) || {
        echo "Error compiling libcgroup" && exit 1
    }
    popd
fi

# Install dependencies and toolchain for Crane
dnf install -y \
    patch \
    ninja-build \
    openssl-devel \
    pam-devel \
    zlib-devel \
    libatomic \
    libstdc++-static \
    libtsan \
    libasan \
    libaio \
    libaio-devel || {
    echo "Error installing toolchain and dependency for craned" && exit 1        
}
# libstdc++-static libatomic for debug
# libtsan for CRANE_THREAD_SANITIZER

# Check if cmake version is higher than 3.24
required_version="3.24"
install_version="3.28.1"
download_url="https://github.com/Kitware/CMake/releases/download/v${install_version}/cmake-${install_version}-linux-x86_64.sh"

current_version=$(cmake --version 2>/dev/null | awk 'NR==1{print $3}')

if [[ -z "$current_version" ]] || [[ "$(printf '%s\n' "$current_version" "$required_version" | sort -V | head -n1)" != "$required_version" ]]; then
    echo "Installing cmake ${install_version}..."
    setp
    wget -O cmake-install.sh "$download_url" || { echo "Error downloading cmake"; exit 1; }
    bash cmake-install.sh --skip-license --prefix=/usr/local || { echo "Error installing cmake"; exit 1; }
    rm cmake-install.sh
    unsetp
else
    echo "Current cmake version ($current_version) meets the requirement."
fi

# Clone the repository
setp
if [ ! -d "CraneSched" ]; then
    git clone https://github.com/PKUHPC/CraneSched.git || {
        echo "Error cloning CraneSched" && exit 1
    }
fi

pushd CraneSched
# git checkout master
git fetch && git pull
unsetp

BUILD_DIR=cmake-build-release
mkdir -p $BUILD_DIR && pushd $BUILD_DIR

if [ -f "/opt/rh/gcc-toolset-13/enable" ]; then
    echo "Enable gcc-toolset-13"
    source /opt/rh/gcc-toolset-13/enable
fi

setp
cmake --fresh -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DENABLE_UNQLITE=ON \
    -DENABLE_BERKELEY_DB=OFF .. || {
    echo "Error configuring with cmake" && exit 1
}
unsetp

cmake --build . --clean-first || {
    echo "Error building" && exit 1
}

popd
popd
~~~









# Crane 前端环境配置-Rocky Linux 9 #
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