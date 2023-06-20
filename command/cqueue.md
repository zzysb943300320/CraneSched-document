# cqueue 查看作业队列#

**cqueue可以查看队列中的作业信息。**

查看集群中所有队列的作业信息。

~~~bash
cqueue
~~~

**cqueue运行结果展示**

![cqueue](../images/cqueue.png)

#### 主要输出项 ####

- **TaskId**：作业名
- **Name**: 作业名
- **Status**：作业状态
- **Partition**：作业所在分区
- **User**：作业所属用户
- **Account**：作业所属账户
- **Type**： 作业类型
- **NodeIndex**： 作业运行的节点名
- **Nodes**：作业所分配节点数
- **TimeLimit**：作业时间限制

#### 主要参数 ####

- **--help**: 显示帮助
- **--noHeader/-N**：输出隐藏表头
- **--start/-S**：显示作业的开始时间（pending作业显示预期开始时间）
- **--job/-j**：指定查询作业号，指定多个作业号时用逗号隔开。如 -j=2,3,4
- **--name/-n**：指定查询作业名，指定多个作业名时用逗号隔开。
- **--qos/-q**：指定查询作业的QoS，指定多个QoS时用逗号隔开。
- **--state/-t**：指定查询作业状态，指定多个状态时用逗号隔开。
- **--user/-u**：指定查询作业所属用户，指定多个用户时用逗号隔开。
- **--Account/-A**：指定查询作业所属账户，指定多个账户时用逗号隔开。
- **--iterate/-i**：指定间隔秒数刷新查询结果。如 -i=3表示每隔三秒输出一次查询结果
- **--partition/-p**：指定查询作业所在分区，指定多个分区时用逗号隔开。
- **--MaxVisibleLines/-m**：指定输出结果的最大条数。如-m=500表示最多输出500行查询结果
- **--format/-o**：以指定格式输出结果表，可以指定输出指定列以及列宽。
  - **例**
    ```shell
    cqueue -h
    ```
    ![cqueue-h](../images/cqueue_h.png)
    ```shell
    cqueue -N
    ```
    ![cqueue-N](../images/cqueue_N.png)
    ```shell
    cqueue -S
    ```
    ![cqueue-S](../images/cqueue_S.png)
    ```shell
    cqueue -j 65,61,63
    ```
    ![cqueue-j](../images/cqueue_j.png)
    ```shell
    cqueue -q normal
    ```
    ![cqueue-q](../images/cqueue_q.png)
    ```shell
    cqueue -t Pending
    ```
    ![cqueue-t](../images/cqueue_t.png)
    ```shell
    cqueue -u root
    ```
    ![cqueue-u](../images/cqueue_u.png)
    ```shell
    cqueue -A ROOT
    ```
    ![cqueue-A](../images/cqueue_A.png)
    ```shell
    cqueue -i 3
    ```
    ![cqueue-i](../images/cqueue_i.png)
    ```shell
    cqueue -p CPU
    ```
    ![cqueue-p](../images/cqueue_p.png)
    ```shell
    cqueue -m 3
    ```
    ![cqueue-m](../images/cqueue_m.png)
    ```shell
    cqueue -o="%n %u %.5j %.5t %.3T %.5T"
    ```
    ![cqueue-o](../images/cqueue_o.png)
  - format中的指定列的对应缩写对照：
      - j-TaskId；n-Name；t-State；p-Partition；u-User；a-Account；T-Type；I-NodeIndex；l-TimeLimit；N-Nodes