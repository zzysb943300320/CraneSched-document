
# ccontrol 查看/修改分区和节点状态 #

**ccontrol可以查看/修改分区和节点的状态。**

#### 主要命令 ####

- **help**：显示帮助
- **add**：添加一个分区或者节点
- **delete**：删除一个分区或者节点
- **show**：显示实体的状态，默认为所有记录
- **update**：修改作业/分区/节点信息
- **hold**：暂停作业调度
- **release**：继续作业调度

### 1. 查看分区状态 ###

~~~bash
ccontrol show partition
~~~

**ccontrol show partition运行结果展示**

![ccontrol](../images/ccontrol_partition.png)

#### 主要输出项 ####

- **PartitionName**：分区名
- **State**：分区状态
- **TotalNodes**：分区节点数目
- **AliveNodes**：分区中可运行的节点数目
- **TotalCpus**：分区中所有节点总CPU数目
- **AvailCpus**：分区中所有可以使用的CPU数目
- **AllocCpus**：分区中已经被分配的CPU数目
- **FreeCpus**：分区中空闲的CPU数目
- **TotalMem**：分区节点的总内存
- **AvailMem**：分区中当前可以使用的内存大小
- **AllocMem**：分区中已分配的内存大小
- **FreeMem**：分区中空闲的内存大小
- **HostList**：分区中所有节点的节点名列表

### 2. 查看节点状态 ###

~~~bash
ccontrol show node
~~~

**ccontrol show node运行结果展示**

![ccontrol](../images/ccontrol_node.png)

#### 主要输出项 ####

- **NodeName**：节点名
- **State**：节点状态
  - **IDLE**：节点空闲，可使用
  - **DOWN**：节点宕机，不可用
- **CPUs**：节点CPU数目
- **AllocCpus**：节点已分配的CPU数目
- **FreeCpus**：节点空闲的CPU数目
- **RealMemory**：节点的实际内存大小
- **AllocMem**：节点已经分配的内存大小
- **FreeMem**：节点空闲的内存大小
- **Patition**：节点所属分区
- **RunningTask**：节点上正在运行的作业数量

### 3. 查看作业状态 ###

~~~bash
ccontrol show job
~~~

**ccontrol show job 运行结果展示**
![ccontrol](../images/ccontrol_job.png)

#### 主要输出项 ####

- **JobId**：作业号
- **JobName**：作业名
- **UserId**：作业所属用户
- **GroupId**：分组id
- **Account**：作业所属账户
- **JobState**：作业状态
- **RunTime**：作业运行时间
- **TimeLimit**：作业运行时间限制
- **SubmitTime**：作业提交时间
- **StartTime**：作业开始时间
- **EndTime**：作业结束时间
- **Partition**：作业所属分区
- **Nodelist**：作业运行的节点
- **NumNodes**：节点数量

### 4. 修改作业信息 ###

~~~bash
ccontrol update job
~~~

#### 主要参数 ####

- **--job/-J**：指定修改的作业号
- **--time-limit/-T**：修改时间限制
- **--priority/-P**：修改时间限制

### 5. 暂停作业调度 ###

#### 主要参数 ####

- **--time-limit/-T**：修改时间限制

~~~bash
ccontrol hold 1               #暂停调度编号为1的任务
ccontrol hold 1,2,3 -t 0:0:5  #暂停调度编号为1,2,3的任务5秒钟（随后解除暂停）
~~~

- hold 接受 job_id 的方式与 ccancel 相同，要求为逗号分隔的任务编号。
- 只能 hold pending 任务。
- 如果此前有设置解除暂停的定时器，该操作会取消原有的定时器。
- 使用 cqueue 查询时，如果任务被 hold，Node(Reason) 一列会显示 "Held"。

### 6. 继续作业调度 ###

~~~bash
ccontrol release 1,2,3
~~~

- 如果此前有设置解除暂停的定时器，该操作会取消原有的定时器。
- 只能 release pending 任务。

### 7. 添加一个节点 ###

~~~bash
ccontrol add node
~~~

#### 主要参数 ####

- **-c, --cpu**：节点的核心数
- **-M, --memory**：节点的内存大小，默认是MB
- **-N, --name**：节点名称
- **-P, --partition**：节点所属的分区，可以不指定，那么就不属于任何一个分区

### 8. 添加一个分区 ###

~~~bash
ccontrol add partition
~~~

#### 主要参数 ####

- **-A, --allowlist**：分区允许的账户列表
- **-D, --denylist**：分区禁止的账户列表，和上面只能设置其中一个，每次设置后以最后设置的为准
- **-N, --name**：分区的名称
- **--nodes**：分区的节点列表，可以用聚合形式cn[01-15]，可以用独立节点名cn01,cn02，也可以混合写，注意参数用""包成一个字符串
- **-P, --priority**：分区的优先级，目前不支持优先级为0

### 9. 删除一个节点 ###

~~~bash
ccontrol delete node [name]
~~~

将节点从所有分区删除，如果节点上面还有作业正在运行，则询问用户是否需要杀死作业来强制删除。

### 10.  删除一个分区 ###

~~~bash
ccontrol delete partition [name]
~~~

删除一个分区，该分区上的节点如果还有正在运行的作业，依然可以继续运行不受影响，但是由于分区被删除，不能再往该分区调度作业。

### 11.  修改节点信息 ###

~~~bash
ccontrol update node
~~~

#### 主要参数 ####

- **-c, --cpu**：节点的核心数
- **-M, --memory**：节点的内存大小，默认是MB
- **-N, --name**：节点名称
- **-P, --partition**：节点所属的分区
以下参数和上面参数不能一起设置，下面参数用于修改节点状态
- **-R, --reason**：Set the reason of this state change
- **-S, --state**：Set the node state

### 12.  修改分区信息 ###

~~~bash
ccontrol update partition
~~~

#### 主要参数 ####

- **-A, --allowlist**：分区允许的账户列表
- **-D, --denylist**：分区禁止的账户列表，和上面只能设置其中一个，每次设置后以最后设置的为准
- **-N, --name**：分区的名称
- **--nodes**：分区的节点列表，可以用聚合形式cn[01-15]，可以用独立节点名cn01,cn02，也可以混合写，注意参数用""包成一个字符串
- **-P, --priority**：分区的优先级，目前不支持优先级为0
