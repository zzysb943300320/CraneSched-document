# Crane 后端环境配置 - Ubuntu #

## 1. 环境准备 ##
### 1.1 更新镜像源 ###
#### 最好将镜像源更换到国内 #### 
~~~bash
apt update && apt upgrade -y
~~~

### 1.2 安装证书 ###
~~~bash
apt install ca-certificates -y
~~~

### 1.3 启用时间同步 ###
~~~bash
apt install -y chrony
systemctl restart systemd-timedated
timedatectl set-timezone Asia/Shanghai
timedatectl set-ntp true
~~~

### 1.4 关闭防火墙 ###
~~~bash
systemctl stop ufw
systemctl disable ufw
~~~
#### 若集群不允许关闭防火墙，则考虑开放 10010、10011、10012、873 端口。####
~~~bash
ufw allow 10012
ufw allow 10011
ufw allow 10010
ufw allow 873
~~~
#### 注意：若有多个节点，需在每个节点上执行此步骤，否则无法进行节点间的通信。####

## 2.安装项目依赖 ##
~~~bash
apt install -y \
    libcgroup-dev \
    libssl-dev \
    libcurl-dev \
    libpam-dev \
    zlib1g-dev \
    libaio-dev \
    pkg-config \
    ninja \
    libelf-dev \
    bcc \
    linux-headers-$(uname -r)
# 安装libbpf
wget https://github.com/libbpf/libbpf/archive/refs/tags/v1.4.6.zip
unzip v1.4.6.zip
cd libbpf-1.4.6/src

# 编译并安装
make
make install
~~~

## 3. 安装工具链 ##
#### 工具链版本需符合以下要求：####
- **cmake版本**>= 3.26
- **libstdc++版本**>= 11
- **如果安装clang，版本**>= 15
- **如果安装g++，版本**>= 13
### 1. 安装常用工具 ###
~~~bash
apt install -y wget tar unzip linux-tools-generic
~~~

### 2. 安装cmake ###
#### Ubuntu 20.04 / 22.04: 从 GitHub 下载安装脚本 ####
~~~bash
wget https://github.com/Kitware/CMake/releases/download/v3.26.4/cmake-3.26.4-linux-x86_64.sh
bash cmake-3.26.4-linux-x86_64.sh --prefix=/usr/local --skip-license
~~~
#### Ubuntu 24.04: 使用包管理器安装 ####
~~~bash
apt install -y cmake
~~~
#### 检查 cmake 安装情况 #### 
~~~bash
cmake --version
# cmake version 3.26.4
~~~
#### 3.安装新版 Clang ####
~~~bash
wget https://apt.llvm.org/llvm.sh
bash ./llvm.sh 19
~~~


## 4. 安装和配置 MongoDB ##
### 此步骤仅在存储节点（控制节点）进行，其他节点不需要安装数据库。###
### 4.1 安装 MongoDB ###
#### 安装 MongoDB 并添加开机启动 ####
~~~bash
apt install -y mongodb-org
# 添加开机启动
systemctl enable mongod
systemctl start mongod
~~~
#### 安装完mongod用户的home目录为/var/lib/mongo ####
#### 利用openssl在/var/lib/mongo生成密钥文件 ####
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
### 编译Crane程序 ###
~~~bash
#代理
git config --global http.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890
git config --global https.proxy http://crane:hf2lH9UUC3E0@192.168.1.1:7890

# 选择一个合适的位置克隆项目
git clone https://github.com/PKUHPC/Crane.git

cd Crane
mkdir build
cd build

# 首次编译需要下载第三方库，耗时较长
cmake -G Ninja -DCMAKE_CXX_COMPILER=/usr/bin/clang++-17 -DCMAKE_C_COMPILER=/usr/bin/clang-17 ..
cmake --build . --target cranectld craned pam_crane

# 仅安装到本机，craned节点需手动scp
ninja install
~~~
#### 注意：如果拉取仓库或依赖时出错，请使用代理 ####

## 6. 运行项目 ##
#### 首先根据自身的集群情况在配置文件当中进行相应配置，配置文件样例保存在etc/crane/config.yaml.example ####
~~~bash
mkdir -p /etc/crane

cp CraneSched安装路径/etc/config.yaml.example /etc/crane/config.yaml
# 上述命令报错旧按照如下输入
cp CraneSched安装路径/etc/config.yaml /etc/crane/

cp CraneSched安装路径/build/src/Misc/BPF/cgroup_dev_bpf.o /etc/crane/cgroup_dev_bpf.o
sudo cp etc/database.yaml /etc/crane/
~~~

#### 在 /etc/crane/config.yaml 中配置节点信息、调度偏好等选项（需所有节点保持一致）请根据集群实际情况填写。例如，一个四节点的集群中，控制节点的主机名为 crane01，计算节点的主机名为 crane01、crane02、crane03、crane04，则如下填写：####
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

## 7. 可能遇到的问题 ##
### 7.1 libbpf  ### 
~~~bash
 error while loading shared libraries: libbpf.so.1: cannot open shared object file: No such file or directory
~~~
~~~bash
 # 验证安装是否正确
 find /usr/lib -name "libbpf*"
 find /usr -name "libbpf*"
 # ubuntu 安装后.so文件可能会在/usr/lib64目录下，我们需要简历软链接
 ln -s /usr/lib64/libbpf* /usr/lib/
~~~

### 7.2 BPF 文件系统挂载 ###
~~~bash
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






# Crane 前端环境配置-Ubuntu #
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