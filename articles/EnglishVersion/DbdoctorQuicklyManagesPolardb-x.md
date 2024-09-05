# 性能诊断工具DBdoctor如何快速纳管数据库PolarDB-X

近日，DBdoctor（V3.1.0）正式通过了阿里云PolarDB分布式版（V2.3）产品集成认证测试，并获得阿里云颁发的产品生态集成认证。

![DBdoctor快速纳管PolarDB-X](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/PolarDB.png)

本文将介绍PolarDB的特性，以及如何快速纳管数据库PolarDB-X。

## 01. PolarDB-X (集中式形态) 产品架构
PolarDB-X 是阿里自主设计的高性能云原生分布式数据库产品，采用 Shared-nothing 与存储分离计算架构，支持集中式和分布式一体化形态，具备金融级数据高可用、分布式水平扩展、混合负载、低成本存储和极致弹性等能力，为用户提供高吞吐、大存储、低延时、易扩展和超高可用的云时代数据库服务。

2023年10月份，PolarDB-X 开源正式发布V2.3.0版本，重点推出集中式形态标准版，将DN节点提供单独服务（存计未分离）。在性能场景上，采用生产级部署和参数(开启双1 + Paxos多副本强同步)，相比于开源MySQL 8.0.34，PolarDB-X在读写混合场景上有30~40%的性能提升，可以作为开源MySQL的最佳替代选择。

##### PolarDB-X架构图

![架构图](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/ArchitectureDiagram.png)

##### PolarDB-X 采用 Shared-nothing 与存储计算分离架构进行设计，系统由4个核心组件组成：

- 计算节点（CN, Compute Node）

计算节点是系统的入口，采用无状态设计，包括 SQL 解析器、优化器、执行器等模块。负责数据分布式路由、计算及动态调度，负责分布式事务 2PC 协调、全局二级索引维护等，同时提供 SQL 限流、三权分立等企业级特性。

- 存储节点（DN, Data Node）

存储节点负责数据的持久化，基于多数派 Paxos 协议提供数据高可靠、强一致保障，同时通过 MVCC 维护分布式事务可见性。

- 元数据服务（GMS, Global Meta Service）

元数据服务负责维护全局强一致的 Table/Schema, Statistics 等系统 Meta 信息，维护账号、权限等安全信息，同时提供全局授时服务（即 TSO）。

- 日志节点（CDC, Change Data Capture）

日志节点提供完全兼容 MySQL Binlog 格式和协议的增量订阅能力，提供兼容 MySQL Replication 协议的主从复制能力。

##### PolarDB-X基于Paxos的MySQL三副本工作原理：

- 在同一时刻，整个集群中至多会有一个Leader节点来承担数据写入的任务，其余节点作为follower参与多数派投票和数据同步


- Paxos的协议日志Consensus Log，全面融合了MySQL原有的binlog内容。在Leader主节点上会在binlog协议中新增Consensus相关的binlog event，同时在Follower备节点上替换传统的Relay Log，备库会通过SQL Thread进行Replay日志内容到数据文件，可以简单理解Paxos Consensus Log ≈ MySQL Binlog


- 基于Paxos多数派自动选主机制，通过heartbeat/election timeout机制会监听Leader节点的变化，在Leader节点不可用时Follower节点会自动完成切主，新的Leader节点提供服务之前需等待SQL Thread完成存量日志的Replay，确保新Leader有最新的数据。

##### PolarDB-X技术特点

- 高性能，采用单Leader的模式，可以提供类比MySQL semi-sync模式的性能

- RPO=0，Paxos协议日志全面融合MySQL原有的binlog内容，基于多数派同步机制确保数据不丢

- 自动HA，基于Paxos的选举心跳机制，自动完成节点探活和HA切换。

## 02. DBdoctor纳管PolarDB-X

### 1）准备已有集中式版本的PolarDB-X

如果只是用于测试验证可以手动进行PolarDB-X安装，可按照以下方式进行安装部署

#### a）下载
下载地址：

https://openpolardb.com/download

![RPM包](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/rpm.png)


#### b）PolarDB-X数据库安装和登录
##### 1. 依赖包安装
```Bash
rpm -ivh t-polardbx-engine-2.3.0-b959577.el7.x86_64.rpm
yum install libaio*
```
##### 2.创建并切换到 polarx 用户
```Bash
useradd -ms /bin/bash polarx
echo "polarx:polarx" | chpasswd
echo "polarx ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoerssu - polarx# 创建必要目录mkdir polardbx-enginecd polardbx-engine && mkdir log mysql run data tmp
```
##### 3. 创建my.cnf配置文件
```Bash
[mysqld]
basedir = /opt/polardbx-engine
log_error_verbosity = 2
default_authentication_plugin = mysql_native_password
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = mysql-binlog
binlog_format = row
binlog_row_image = FULL
master_info_repository = TABLE
relay_log_info_repository = TABLE

# change me if needed
datadir = /home/polarx/polardbx-engine/data
tmpdir = /home/polarx/polardbx-engine/tmp
socket = /home/polarx/polardbx-engine/tmp.mysql.sock
log_error = /home/polarx/polardbx-engine/log/alert.log
port = 4886
cluster_id = 1234
cluster_info = 127.0.0.1:14886@1

[mysqld_safe]
pid_file = /home/polarx/polardbx-engine/run/mysql.pid
```
##### 4. 初始化
```Bash
/opt/polardbx_engine/bin/mysqld --defaults-file=my.cnf --initialize-insecure
```
##### 5. 启动
```Bash
/opt/polardbx_engine/bin/mysqld_safe --defaults-file=my.cnf &
```
##### 6. 登录数据库

```Bash
mysql -h127.0.0.1 -P4886 -uroot
```

### 2）DBdoctor一键安装部署

环境要求：4c8g
下载安装包：
https://www.hisensecloud.com/col.jsp?id=126
解压安装包并执行
```Bash
./dbd -I
```
![RPM包](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/zip.png)


安装成功后登录http://<部署服务的主机ip>:13000
登录账号：tester/Root2023!
详细安装使用文档：
https://www.hisensecloud.com/h-col-144.html

### 3）DBdoctor纳管PolarDB-X

在部署的PolarDB-X进程层面我们能看到几乎和MySQL的集群一致，DBdoctor可以完全按照MySQL的方式进行纳管。DBdoctor的PolarDB-X纳管是按照集群的DN节点维度进行纳管的(DN在进程层面相当于mysqld)，上面三个节点相当于对每个节点进行依次纳管。

![纳管](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/Nanotube.png)

#### a）首先在PolarDB-X上创建数据库账号并授予权限
```Bash
create user zx identified by 'Root2023!';
GRANT SELECT, PROCESS, SHOW VIEW ON *.* TO 'zx'@'%';
```
#### b）在DBdoctor的实例列表页面进行集中式PolarDB-X纳管

切换到‍实例列表的tab，点击实例纳管按钮，进行实例录入。

##### ① 录入实例的主数据库基本信息，并进行连通性检测：
点击“实例纳管”按钮可进行用户已有存量的数据库实例录入 。
- 可配置该实例的数据库基本信息，包括实例备注、数据库访问地址

- 可设置数据库实例的账号信息，包括实例的账号和密码

- 可进行数据库实例的连通性检测

![录入实例](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/InputInstance.png)

##### ② 配置数据洞察
> 设置洞察日志存储的天数（默认为10天）
##### ③ 选择采集方式
针对PolarDB-X数据采集方式选择部署Agent，用来采集主机和实例性能数据，锁和审计日志相关当前版本还不支持，后续会考虑对该引擎的适配支持。

![数据采集](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/DataAcquisition.png)

## 03. 如何使用DBdoctor对PolarDB-X进行性能诊断

### 1）开启性能分析

查看实例列表找到PolarDB-X的纳管数据库节点，点击该实例节点的性能分析开关就可以对该实例进行性能洞察功能接管。PolarDB-X的锁分析功能暂未开放（审计日志和4大锁场景诊断），需要后续版本GA后才能支持该功能。
![性能分析](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/PerformanceAnalysis.png)

### 2）查看性能洞察
点击性能洞察按钮进行该实例的性能分析。

![性能洞察](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/PerformanceInsight.png)

点击后可以查看该实例的性能洞察分析结果。

![分析结果](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/DbdoctorQuicklyManagesPolardb-x/AnalysisResult.png)

#### 性能洞察的详细功能主要分为下面7个部分：

**1. 实例搜索**：用户可以选择已接管的实例进行性能洞察的功能展示，实例指定时间区间或者近5m、1h、5h、24h、2d的快速时间选择搜索，并可指定刷新频率。

**2. 关键资源**：展示实例关键资源的使用情况来评估是否存在资源瓶颈，显示该实例的DB维度/主机维度的关键资源使用率指标的趋势图，指标包括CPU使用率、MEM使用率、IOPS使用率、DISK Space使用率、连接数使用率。同时展示当前时间区间内的各指标的最大值。

**3. 业务流量**：展示业务的流量指标，包括网络包的进出流量，数据库层面的业务QPS情况。

**4. AAS**：通过业务的数据库平均活跃会话来展示数据库的负载，同时展示负载中的TOP等待事件。

**5. 关联SQL**：展示AAS负载相关联的根因SQL,同时可以查看该SQL的最差样本，并可以进行执行计划展示。

**6. 业务负载流量分布**：按照用户/来源服务IP维度展示AAS负载，评估数据库负载问题的源头来自哪个业务系统。

**7. 专家经验库**：详细展示性能问题事件的专家经验文档，从原理上解释这一事件是如何产生，应该怎样进行优化。
#### 性能洞察实践：

性能洞察最大特色是可以将上面所有维度的指标性能数据通过时间轴关联起来，并进行分析。比如上面的CPU异常的Case我们只需要4步走即可找到问题SQL根因。
**step1**：cpu资源指标在15:45~16:20发生抖动，CPU存在打满的情况
**step2**：在AAS模块中我们能看到数据库的活跃会话数在这一时间区间内远超Max vCPU水位线，说明存在性能瓶颈
**step3**：可以看到性能瓶颈的事件是Sending data（即绿色颜色事件），而这一颜色在AAS趋势图中占的面积最大，点击事件也可以看到最右边的专家经验文档。
**step4**：基于这个面积最大颜色的事件，我们能找到绿色颜色的SQL为第一条，即导致CPU飙高的根因SQL。点击展开这个SQL可以展示这个SQL的最差样本，点击执行计划发现扫描全表扫描4w多行，对SQL进行添加索引即可。