# DBdoctor快速纳管OceanBase

## OceanBase纳管架构

![纳管架构](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesOceanbase/ManagementArchitecture.png)

DBdoctor按照节点级别进行分别纳管，只需要找个ecs服务器进行一键安装部署，然后全流程web界面操作即可完成纳管OceanBase的实例。agent以主机维度进行部署，一个主机只需要部署一个agent即可完成对主机上所有的数据库节点进行指标采集。

1. OceanBase按照OBServer节点进行纳管，这里我们需要分别对OBserver1、OBserver2、OBserver3、OBserver4、OBserver5、OBserver6进行纳管。

2. OceanBase的节点是可以分布式扩展的，如果新增节点，还需要按照上面方式对新增节点进行纳管。

### 1）1分钟零依赖DBdoctor Server安装

**环境要求**：4c8g（建议独立的资源部署）
**下载安装包**：https://www.hisensecloud.com/h-col-126.html

```Bash
#解压安装包并执行一条命令即可部署完成
./dbd -I
```
![安装](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesOceanbase/Install.png)

**服务访问地址**：http://<部署服务的主机ip>:13000
**登录账号**：tester/Root2023!
**详细文档**：https://www.hisensecloud.com/h-col-144.html

### 2）快速纳管OceanBase

#### a）首先在OceanBase上创建数据库账号并授予权限
```Bash
create user zx identified by 'Root2023!';
GRANT SELECT, PROCESS, SHOW VIEW ON *.* TO 'zx'@'%';
```
#### b）在DBdoctor的实例列表页面进行OceanBase实例纳管

- 点击“实例纳管”按钮录入实例的数据库访问地址、账号密码等基本信息，然点击连通性检测提示Sucess表示校验正确。

- 默认采用agent方式进行数据采集，录入主机账号信息，支持自动和手动两种方式进行agent安装。
![实例纳管](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesOceanbase/InstanceManagement.png)

**备注**：agent系统支持说明:支持 X86_64 和ARM系统，不支持x86_32服务器。两个连通性检测都通过，恭喜你实例纳管成功，即可开启性能诊断。

### 3）如何使用性能洞察功能
#### a） 开启性能分析

实例列表中找到已纳管的OceanBase节点，点击性能洞察开关按钮即可开启分析。

#### b）查看性能洞察

点击性能洞察透视OceanBase的节点性能。

![节点性能](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesOceanbase/NodePerformance.png)

性能洞察功能可将硬资源和数据库等多维度的指标性能数据通过时间轴关联起来，并进行分析。比如上面的**CPU异常的Case**我们只需要**1步走即可找到问题根因SQL**，同时可以**还原**异常时间区间**详细现场**。

- **自动根因识别**：CPU资源指标发生抖动，自动框选异常区间并提示根因SQL。

- **详细现场还原**：

    - CPU异常时间区间框选

    - 异常时间区间在AAS模块中我们能看到数据库的活跃会话数在这一时间区间内**超Max vCPU水位线**，说明存在**性能瓶颈**

    - 可以看到性能瓶颈的事件是ACTIVE（即蓝颜色事件），而这一**颜色**在AAS趋势图中占的**面积最大**。

    - 基于这个面积最大颜色的事件，我们能找到绿色颜色的SQL为第一条，即导致CPU飙高的根因SQL。点击展开这个SQL可以展示这个SQL的**最差样本**，点击执行计划发现扫描全表扫描行，对SQL进行添加索引即可。


