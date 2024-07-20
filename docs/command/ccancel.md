# ccancel 取消作业 #

**ccancel可以终止正在运行或者在排队中的作业。**

主要参数

- **--help/-h**: 显示帮助
- **-A, --account string**：取消账户下的任务
- **-C, --config string**：配置文件路径(默认 "/etc/crane/config.yaml")
- **-n, --name string**：仅使用任务名称取消任务
- **-w, --nodes strings**：取消节点上运行的任务
- **-p, --partition string**：取消分区上运行的任务
- **-t, --state string**：取消某状态的任务。有效的任务状态是 PENDING(PD)、RUNNING(R)。任务状态不区分大小写
- **-u, --user string**：取消特定用户提交的任务
  - **例**
    ![ccancel-cqueue](../images/ccancel_cqueue.png)
    ~~~bash
    ccancel -w crane02
    ~~~
    ![ccancel-w](../images/ccancel_w.png)
    ~~~bash
    ccancel -t Pending
    ~~~
    ![ccancel-t](../images/ccancel_t.png)

取消作业号为280的作业：

~~~bash
ccancel 280
~~~

**ccancel运行结果展示**

![ccancel](../images/ccancel.png)

取消作业之后，如果被分配节点上没有用户的其他作业，作业调度系统会终止用户在所分配节点的所有进程，并取消用户在所分配节点上的ssh权限。
