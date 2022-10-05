# Crane-FrontEnd 项目环境配置

## 1.安装go语言

```shell
cd download/
wget https://golang.google.cn/dl/go1.17.3.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.17.3.linux-amd64.tar.gz

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

## 2.安装protoc

```shell
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.19.4/protobuf-all-3.19.4.tar.gz
tar -xzf protobuf-all-3.19.4.tar.gz
cd protobuf-3.19.4
./configure -prefix=/usr/local/
make && make install
protoc --version
# libprotoc 3.11.2
```

## 3.拉取项目

```shell
git clone https://github.com/RileyWen/Crane-FrontEnd.git # 克隆项目代码

mkdir Crane-FrontEnd/out
mkdir Crane-FrontEnd/generated/protos
```

## 4.编译项目

```shell
# 在Crane-FrontEnd/protos目录下
protoc --go_out=../generated --go-grpc_out=../generated ./*

# 在Crane-FrontEnd/out目录下编译所有命令
go build ../cmd/cacctmgr/cacctmgr.go
go build ../cmd/cbatch/cbatch.go 
go build ../cmd/ccancel/ccancel.go 
go build ../cmd/ccontrol/ccontrol.go 
go build ../cmd/cinfo/cinfo.go 
go build ../cmd/cqueue/cqueue.go 
```

## 5.部署前端命令

```shell
cp ./* /usr/local/bin/
```