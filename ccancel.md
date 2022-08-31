scancel
取消队列中已提交的作业；

介绍几个使用使用案例，分别是，取消指定作业、取消自己上机账号上所有作业、取消自己上机账号上所有状态为PENDING的作业，最后介绍scancel常见的参数。

取消指定作业；

# 取消作业ID为123的作业
scancel 123
取消自己上机上号上所有作业；

# 注意whoami前后不是单引号
scancel -u `whoami`
取消自己上机账号上所有状态为PENDING的作业；

scancel -t PENDING -u `whoami`
scancel常见参数;

--help                # 显示scancel命令的使用帮助信息；
-A <account>        # 取消指定账户的作业，如果没有指定job_id,将取消所有；
-n <job_name>        # 取消指定作业名的作业；
-p <partition_name> # 取消指定分区的作业；
-q <qos>            # 取消指定qos的作业；
-t <job_state_name> # 取消指定作态的作业，"PENDING", "RUNNING" 或 "SUSPENDED"；
-u <user_name>        # 取消指定用户下的作业；