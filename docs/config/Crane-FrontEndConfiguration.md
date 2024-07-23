# CraneSched-FrontEnd 项目环境配置 #

## 1.安装go语言 ##

```shell
cd download/
wget https://golang.google.cn/dl/go1.21.12.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.21.12.linux-amd64.tar.gz

# 在 /etc/profile中设置环境变量
export GOROOT=/usr/local/go
export GOPATH=/usr/local/gopath
export PATH=$PATH:/$GOROOT/bin:$GOPATH/bin

source /etc/profile     # 加载环境变量

go version

#设置代理
go env -w GOPROXY=https://goproxy.cn,direct
#开启go mod管理
go env -w GO111MODULE=on

# 安装插件
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
```

## 2.安装protoc ##

```shell
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.4/protobuf-all-3.19.4.tar.gz
tar -xzf protobuf-all-3.19.4.tar.gz
cd protobuf-3.19.4
./configure -prefix=/usr/local/
make && make install
protoc --version
# libprotoc 3.19.4
```

## 3.拉取项目 ##

```shell
git clone https://github.com/PKUHPC/CraneSched-FrontEnd.git # 克隆项目代码
```

## 4.编译项目 ##

生成proto文件

```shell
cd CraneSched-FrontEnd/
make all
```

## 5.部署前端命令 ##

### 5.1. 本地更新 ###

```shell
cp ./bin/* /usr/local/bin/
```

### 5.2. 用pdsh远端更新 ###

以下用demo集群为例：

```bash
pdcp -w login -w crane0[1-4] -w cranectl ./bin/* /usr/local/bin/
```

### 5.3. 部署cfored ###

在登录节点和计算节点部署

```bash
systemctl stop firewalld
systemctl disable firewalld

# 上述两条命令不成功需要执行下面命令
#或者开放端口
firewall-cmd --add-port=10012/tcp --permanent --zone=public
# 重启防火墙(修改配置后要重启防火墙)
firewall-cmd --reload

# 在Crane-FrontEnd/目录下
# 本地节点
cp ./etc/cfored.service /etc/systemd/system/
# 远端节点
pdcp -w login -w crane0[1-4] -w cranectl ./etc/cfored.service /etc/systemd/system/
# 启动cfored
systemctl start cfored
```

### 5.4. 部署cwrapper ###

在登录节点和计算节点上，用cwrapper命令将鹤思命令与slurm常用命令关联：

```bash
cat > /etc/profile.d/cwrapper.sh << 'EOF'
alias sbatch='cwrapper sbatch'
alias sacct='cwrapper sacct'
alias sacctmgr='cwrapper sacctmgr'
alias scancel='cwrapper scancel'
alias scontrol='cwrapper scontrol'
alias sinfo='cwrapper sinfo'
alias squeue='cwrapper squeue'
EOF
```
