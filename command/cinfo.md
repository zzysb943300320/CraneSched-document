
# cinfo 查看节点与分区状态 #

**cinfo可查询各分区节点的队列资源信息。**

查看分区节点状态：
~~~bash
cinfo
~~~

**cinfo运行结果展示**

![cinfo](../images/cinfo.png)


#### 主要输出项 ####

- **PARTITION**：分区名
- **AVAIL**： 分区状态
  - **idel**： 空闲
  - **mix**： 节点部分核心可以使用
  - **alloc**： 节点已被占用
  - **down**： 节点不可用
- **NODES**：节点数
- **NODELIST**： 节点列表



#### 主要参数 ####

- **--help**: 显示帮助
- **-N/--Node**：以节点为中心显示信息，按节点名从小到大输出
- **-C/--config string**：配置文件路径(默认为 "/etc/crane/config.yaml")
- **-d/--dead**：只显示无响应节点
- **-o/--format string**：以指定格式输出结果表，可以指定输出指定列以及列宽。
- **-i/--iterate uint**：指定间隔秒数刷新查询结果。如 -i=3表示每隔三秒输出一次查询结果
- **-n/--nodes string**：显示指定节点信息，多个节点用逗号隔开。例：cinfo -n dby01,dby02
- **-p/--partition string**：显示指定分区信息，多个分区用逗号隔开。例：cinfo -p CPU,GPU
- **-r/--responding**：只显示有响应节点
- **-t/--states string**：仅显示状态的信息。状态可以为(不区分大小写): IDLE, MIX, ALLOC和DOWN
- **-s/--summarize**：显示摘要信息，每个分区显示一行
  - **例**
    ```shell
    cinfo -h
    ```
    ![cinfo-h](../images/cinfo_h.png)
    ```shell
    cinfo -N
    ```
    ![cinfo-N](../images/cinfo_N_big.png)
    ```shell
    cinfo -d
    ```
    ![cinfo-d](../images/cinfo_d.png)
    ```shell
    cinfo -o "%.10P %.8l %.9F %.25l %.15a %.9D"
    ```
    ![cinfo-o](../images/cinfo_o.png)
    - format中指定列的对应缩写对照（区分大小写）：
      - P：PARTITION，F：NODES（A/I/O/T），a：AVAIL，D：NODES，I：TIMELIMIT
    ```shell
    cinfo -i 3
    ```
    ![cinfo-i](../images/cinfo_i.png)
    ```shell
    cinfo -n dby01,dby02
    ```
    ![cinfo-n](../images/cinfo_n.png)
    ```shell
    cinfo -p GPU,CPU
    ```
    ![cinfo-p](../images/cinfo_p.png)
    ```shell
    cinfo -r
    ```
    ![cinfo-r](../images/cinfo_r.png)
    ```shell
    cinfo -t IDLE
    ```
    ![cinfo-t](../images/cinfo_t.png)
    ```shell
    cinfo -s
    ```
    ![cinfo-s](../images/cinfo_s.png)
    
