# CraneSched 配置文件 # 
~~~bash
vim /etc/crane/config.yaml

# 以上省略...
# 控制节点（主节点）
ControlMachine: crane01
# ...

# Nodes and partitions settings
# 计算节点
Nodes:
  - name: "crane[01-04]"
    #每个节点的资源
    cpu: 2
    memory: 2G
    #设备资源
    gres:
            #资源的name
          - name: gpu
            #资源的type
            type: a100
            # 设备资源对应的/dev设备文件，每个文件解析为一个设备
            DeviceFileRegex: /dev/nvidia[0-3]
            # 设备资源对应的/dev设备文件，/dev/dri/renderer[0-3]解析为一个设备
            DeviceFileList:
              - /dev/dri/renderer[0-3]
              - /dev/dri/renderer[4-7]
            #设备对应的环境变量 如CUDA_VISIABLE_DEVICES等
            #可选：nvidia,hip,ascend
            EnvInjector: nvidia

# partition information list
# 计算节点的分区
Partitions:
  - name: CPU              # 分区的名称（可自定义）
    nodes: "crane[01-02]"  # 分区中的节点，需要和 Nodes 部分对应
    priority: 5
  - name: GPU
    nodes: "crane[03-04]"
    priority: 3
    # Optional default memory per cpu in MB, 0 let scheduler decide
    DefaultMemPerCpu: 0     # 建议设置为0
    # Optional maximum memory per cpu in MB, 0 indicates no limit
    MaxMemPerCpu: 0         # 建议设置为0

# 默认分区，未指定分区的作业将被提交到默认分区，需要和 Partitions 部分对应
DefaultPartition: CPU
~~~

## Gres配置 ##
#### 设备资源相关配置 ####
- **name**：一般是资源类型如：GPU，NPU等
- **type**：一般是资源型号如：A100，3090等
- **DeviceFileRegex**: 资源对应的 /dev 目录下的设备文件，适用于一个物理设备对应一个设备文件的资源，每个文件对应系统内的一个 Gres 资源，支持 Regex 格式。常见设备对应设备文件。如 Nvidia、AMD、海光 DCU、昇腾等。
- **DeviceFileList**：适用于一个物理设备对应多个 /dev 目录下设备文件的 Gres 资源，每一组文件对应系统内的一个 Gres 资源，支持 Regex 格式。
#### DeviceFileRegex与DeviceFileList二选一，以上设备文件必须存在，否则 Craned 启动时将报错退出 ####
- **EnvInjector**: 设备需要注入的环境变量
  #### 可选值：对应环境变量 ####
  - **nvidia**：CUDA_VISIABLE_DEVICES
  - **hip**：HIP_VISIABLE_DEVICES
  - **ascend**：ASCEND_RT_VISIBLE_DEVICES
####          常见厂商设备文件路径及相关配置 ####

| 厂商                   |    设备文件路径            |          EnvInjector    |
| ---------------------- | ------------------------ |-------------------------|
| **Nvidia**            | **dev/nvidia0 ...**         |  nvidia               |
| **AMD/海光DCU**       | **/dev/dri/renderer128...**  |hip                    |
| **昇腾**              |**/dev/davinci0 ...**        |ascend                 |