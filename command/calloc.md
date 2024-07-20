# calloc 提交交互式任务 #

**calloc 使用命令行指定的参数申请资源，任务启动时，会进入新的用户终端，用户需要自行登陆到计算节点并启动任务。calloc需要在有cfored运行的节点上启动。**

calloc 只支持通过命令行指定请求参数，支持的命令行选项：

- **--help/-h**：显示帮助
- **-A, --account string**：提交作业的账户
- **--chdir string**：任务工作路径
- **-C, --config string**：配置文件路径(默认 "/etc/crane/config.yaml")
- **-c, --cpus-per-task float**：每个节点申请的CPU核心数
- **-J, --job-name string**：作业名
- **--mem string**：每个节点申请的内存大小
- **-N, --nodes uint32**：申请的节点数量
- **--ntasks-per-node uint32**：每个节点上运行的任务数量
- **-p, --partition string**：作业使用的分区/队列
- **-Q, --qos string**：指定作业使用的qos名称-
- **-t, --time string**：作业的最长运行时间
- **-w, --nodelist**：提交作业到指定节点运行
- **-x, --exclude**：提交的作业排除某些指定节点运行
- **-D, --debug-level**：日志输出等级

退出calloc新启动的终端将结束任务。

在CPU分区，申请两个个节点，一个CPU核心，200M内存

~~~bash
calloc -c 1 --mem 200M -p CPU -N 2
~~~

运行结果：
![calloc](../images/calloc.png)
