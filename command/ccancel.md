# ccancel 取消作业 #

**ccancel可以终止正在运行或者在排队中的作业。**

主要参数

- **-h/--help**: 显示帮助
- **-A/--account string**：取消账户下的任务
- **-C/--config string**：配置文件路径(默认 "/etc/crane/config.yaml")
- **-n/--name string**：仅使用任务名称取消任务
- **-w/--nodes strings**：取消节点上运行的任务
- **-p/--partition string**：取消分区上运行的任务
- **-t/--state string**：取消某状态的任务。有效的任务状态是 PENDING(PD)、RUNNING(R)。任务状态不区分大小写
- **-u/--user string**：取消特定用户提交的任务
- **--json**：json格式输出命令执行结果
- **-v/--version**：查询版本号
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

取消作业号为30686的作业：

~~~bash
ccancel 30686
~~~

**ccancel运行结果展示**

![ccancel-cqueue](../images/ccancel_cqueue_30686.png)
![ccancel_30686](../images/ccancel_30686.png)
![ccancel-cqueue](../images/ccancel_cqueue_2.png)

取消作业号为test1的作业：

~~~bash
ccancel -n test
~~~

**ccancel运行结果展示**

![ccancel-cqueue](../images/ccancel_cqueue_test1.png)
![ccancel-n](../images/ccancel_n.png)
![ccancel-cqueue](../images/ccancel_cqueue_test1_2.png)

取消CPU分区上的作业

~~~bash
ccancel -p GPU
~~~

**ccancel运行结果展示**

![ccancel-cqueue](../images/ccancel_cqueue_p.png)
![ccancel-p](../images/ccancel_p.png)
![ccancel-cqueue](../images/ccancel_cqueue_p_2.png)

~~~bash
ccancel -A PKU
~~~
![ccancel-A](../images/ccancel_A.png)

~~~bash
ccancel -u ROOT
~~~
![ccancel-u](../images/ccancel_u.png)

取消作业之后，如果被分配节点上没有用户的其他作业，作业调度系统会终止用户在所分配节点的所有进程，并取消用户在所分配节点上的ssh权限。
