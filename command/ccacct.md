# cacct 查看作业信息 #
**cacct可以查看队列中的作业信息。**

查看集群中所有队列的作业信息（包括所有状态），默认输出100条信息。

```shell
cacct
```
**cacct运行结果展示**
![cacct](../images/cacct.png)

### 主要输出项 ###
- **TaskId**：作业号
- **TaskName**: 作业名
- **Partition**：作业所在分区
- **Account**：作业所属账户
- **AllocCPUs**：作业分配的CPU数量
- **State**：作业状态
### 主要参数 ###
- **--help/-h**: 显示帮助
- **--noHeader/-N**：输出隐藏表头
- **--starttime/-S**：指定查询该时间之后开始的作业，例：cacct -S=2023-03-14T10:00:00
- **--endtime/-E**：指定查询该时间之前结束的作业，例：cacct -E=2023-03-14T10:00:00
- **--job/-j**：指定查询作业号，指定多个作业号时用逗号隔开。如 -j=2,3,4
- **--name/-n**：指定查询作业名，指定多个作业名时用逗号隔开。
- **--user/-u**：指定查询作业的所属用户，指定多个用户时用逗号隔开。
- **--Account/-a**：指定查询作业的所属账户，指定多个账户时用逗号隔开。
- **--MaxVisibleLines/-m**：指定输出结果的最大条数。如-m=500表示最多输出500行查询结果
- **--format/-o**：以指定格式输出结果表，可以指定输出指定列以及列宽。
  - 例
    ```shell
    cacct -h
    ```
    ![cacct_h](../images/cacct_h.png)
    ```shell
    cacct -N
    ```
    ![cacct_N](../images/cacct_n.png)
    ```shell
    cacct -S=2023-03-14T10:00:00
    ```
    ![cacct_S](../images/cacct_s.png)
    ```shell
    cacct -E=2023-05-07T10:00:00
    ```
    ![cacct_E](../images/cacct_e.png)
    ```shell
    cacct -j=25,24,23
    ```
    ![cacct_j](../images/cacct_j.png)
    ```shell
    cacct -u=root
    ```
    ![cacct_u](../images/cacct_u.png)
    ```shell
    cacct -A=ROOT
    ```
    ![cacct_A](../images/cacct_a.png)
    ```shell
    cacct -m=10
    ```
    ![cacct_m](../images/cacct_m.png)
    ```shell
    cacct -o="TaskId,TaskName%30,Partition,Account,State"
    ```
    ![cacct_o](../images/cacct_o.png)
