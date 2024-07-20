# CraneSched  架构 #

针对算力中心的调度场景，CraneSched采用中心化管理模式，**Cranectld**是部署在主控节点的守护进程，也是调度系统的“大脑”。**Craned**是部署在计算节点上的守护进程，也是调度系统下达给计算节点的指令的执行者。

Cranectld负责算力中心节点生命周期的管理、作业队列的调度及管理、节点资源调度及管理，处理来自用户指令的作业提交、修改、查询等请求。Craned主要用来监控节点资源及作业状态，接受用户的各种指令，并将其发送给Cranectld，向用户传送Cranectld对指令处理的返回结果。

![architecture](./images/architecture.png)

面向算力网络的国家战略，鹤思智能调度系统计划采用两级调度架构，上层调度为XCraneSched，下层调度为CraneSched，两者结合至上而下解决算力网络中算力资源调度问题。CraneSched针对单个算力中心资源调度，主要运行高性能计算和智能计算作业，通过适配器与XCraneSched连接，并承接来自XCraneSched分派的作业。XCraneSched通过各种适配器去连接超算、智算、云计算等各类算力中心，将其汇聚成一张算力网络，接受用户提交的作业，并将作业分发到最“合适”的算力中心。

![scenario](./images/scenario.png)
