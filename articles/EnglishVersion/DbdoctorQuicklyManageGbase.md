# DBdoctor快速纳管GBase 8a数据库

GBase 8a是一款由南大通用开发的分析型数据库产品，它主要面向数据仓库、大数据分析等场景，提供高性能的数据存储、处理和管理能力。GBase 8a 以其高可用性、高可靠性和优秀的扩展性在国内市场占据重要地位，尤其适用于需要处理大规模数据并进行复杂查询分析的业务环境。

目前DBdoctor已实现对Gbase 8a分布式数据库的快速纳管，确保数据库的稳定运行，可及时发现并处理其潜在性能问题。

## 如何快速纳管GBase 8a？

下面将详细介绍如何使用DBdoctor纳管GBase 8a并对其进行性能诊断。

### 1. GBase 8a分析型数据库纳管部署架构

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp95z9zic8MIfrCy3nQn2G0DpN38xQDEia9KMwAVU4MHTAmEaPFtgzdPgw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图中GBase 8a分布式数据库有2个虚拟集群，分别是VC1 和VC2，并且每个虚拟集群都有2个Gnode和4个VC分片。DBdoctor可按照虚拟集群来进行纳管，自动发现所有的Gnode信息。一个Gnode上只需要部署一个Agent(自动部署)，即可实现对虚拟集群的纳管。

### 2. 一分钟零依赖DBdoctor Server安装

**环境要求**：4c8g（建议独立的资源部署，可以添加选项--unlimited忽略4c8g的限制）

**下载安装包**：https://www.dbdoctor.cn/col.jsp?id=126

```
#解压安装包并执行一条命令即可部署完成./dbd -I
```

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlf8zDWbymcPT9ZRo5eDQ8I4Ch754YlkeKvuPtn4A4XuHDCaAIlxAZ14VjVsZ6iccwzSMLSWb4U70g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**服务访问地址**：http://<部署服务的主机ip>:13000
**登录账号**：tester/Root2023!
**详细文档**：https://demo.dbdoctor.cn/modules/dbDoctor/mdPreview/index.html?readme=help#/

### 3. 快速纳管GBase 8a

#### a) 创建访问账号
```SQL
create user 'test'@'%' IDENTIFIED BY 'Root2023!';
grant select,process, show view on *.*.* TO 'test'@'%';
```
#### b）页面纳管GBase 8a实例
1. 点击“实例纳管”按钮后，在类型下拉框中选中【gbase-8a】引擎类型；

2. 填写数据库的Gcluster访问地址、虚拟集群VC名称、Gnode端口、账号以及密码等基本信息;

3. 点击【check】按钮，检查实例数据库是否连接正常，检查通过则会在纳管界面展示所有的Gnode信息；

4. 录入Gnode所在主机的账号信息，默认自动安装Agent。注意：开启拓扑自适应后，DBdoctor Server可自动感知集群节点的拓扑并进行节点纳管或者下线。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEphbJFbgLhxAMC2oe3XKAY9ESLzXQNWkXrwmfLYPpe5GeQVH3WNpgPfw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

连通性检测通过后，点击提交后即可成功纳管GBase 8a数据库。在实例列表界面，可以看到已纳管的Gcluster集群实例及node节点实例信息。性能洞察开启完成，开始体验DBdoctor的强大功能吧！

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpNdCNKJ2ic6NrmxyXsR3XhiccG7dk2mQ2thHKrf1RVzn1pwguIXl0MYeg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  重点说明：

#### a）GBase集群资源整体消耗

采集方式选择“部署Agent”方式，DBdoctor会自动对该Gcluster集群下的的所有Gnode节点进行Agent安装，并展示该Gcluster集群的整体资源消耗和数据库负载情况，同时针对每个Gnode也可查看详细的节点资源消耗和数据库负载。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpoR5fvkZK7tdF8wq81utm5tDABmAnMu3wQ9ctTltibKZtTdO1ibhQ9EFg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### b）实时感知Gnode节点拓扑变化并进行管理

当开启“拓扑自适应”后，一旦数据库集群动态扩增Gnode，DBdoctor可以自动感知并将扩增的Gnode节点自动纳管。并且针对已删除的Gnode，会自动解绑该节点。DBdoctor可对GBase集群节点进行自动管理，减少人工干预，降低管理成本。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpcA80G1tkpayAMB3tO9QCr6Tjjibpet2Jjuq1hQyah5ZLZ0CLnKtJG9A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 针对GBase 8a，DBdoctor提供哪些功能服务？

当前DBdoctor 适配GBase 8a 9.5.3及以上版本，并提供**SQL审核、实例巡检、性能洞察、根因诊断、基础监控、存储分析**等功能服务。

### 1. SQL审核

支持**人工审核、慢SQL审核、全量SQL审核以及OpenAPI审核**方式，可实现对增量SQL以及线上SQL的全生命周期闭环质量管理。支持批量上传SQL文件，在开发阶段即可完成SQL审核，提前识别SQL问题，同时针对线上存量SQL也可以实时抓取进行审核。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpwwX6LMmFiafDice8XcGw3uiafeZNaLxtHLlU2yrCAxXdLBHVSbKFU4KTQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2. 深度巡检与报表

支持**自动巡检、手动巡检**两种巡检方式，可以及时发现数据库在配置、性能、资源等方面的问题，保障数据库服务的稳定。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp5s57VmpWN7ibaklW2QRryibBNibMsXeGOJia7XibbjibM2VgNA5U2ywleDKg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3. 性能洞察

性能洞察界面中展示各**资源使用率、业务流量**以及数据库的**平均活跃会话**情况。一旦资源使用率或者业务流量存在异常区间，可以快速高效的通过平均活跃会话趋势图及SQL关联分析找到导致出现异常的根因SQL，从而第一时间解决问题。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp9ow1icPJN6uNTk4gicCcKUTbY9nHs0swu2kATuk2ILeyF1X712iaWwwAg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


从图中可以得知，在异常事件存在CPU异常，且对应时间的AAS图中蓝色片区居多。SQL关联分析中对应颜色的是SELECT语句，因此SELECT语句就是导致CPU异常的根因SQL。

#### 1）用户执行的SQL（Gcluster）

SQL关联分析中，我们展出了数据库在特定时间内执行的SQL语句，并提供了对SELECT语句执行计划的查询功能。Gcluster集群具备执行调度的能力，能够将收到的SQL语句分解后发送至不同Gnode节点,并最终将各node的结果汇总返回给调用方。以下是Gcluster实例中收到的SQL语句示例：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEppRDeSnyf6M3oNlbDOQWOg9kbubXWxaBUwEo3hZHiaJPibUFccwkvxoFA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2）SQL流转到每个Gnode节点上执行的真实SQL

Gnode节点是GBase 8a MPP Cluster数据库系统中的基础存储和计算单元。每个Gnode负责在本地节点上实际存储集群数据，并接收来自Gcluster的分解后的SQL执行计划。Gnode执行这些计划后，将结果返回给Gcluster。在下面两个图中，展示了在Gnode1和Gnode2上执行的SQL语句，从图中可以清晰地看到Gnode中执行的SQL是已经被分解后的SQL，并具体到指定的vcName。

##### a）Gnode节点1

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpgIJKqbRXvhACcZic6m8nKBybXia6Ime6ibXicJ5AZWsu8GwyGpH0ibTMF1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### b）Gnode节点2

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpUaB25bTuDicAT1ViasZgbvq7SYlBYyvRCz2FbhSu6wWibSepj35LMtSpQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以快速的知道该SQL的集群维度整体的消耗，同时针对拆分到每个分片节点上的SQL也是能够直观的看到，是否存在热点或者数据倾斜等导致的集群数据库性能问题。

### 4. 根因诊断

根因诊断中详细的描述出该SQL导致的问题现象，并统计出SQL指纹在数据库中造成的每一次异常。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpuKxn9Kp6V0ibpfvf9GPLBozRFQzrJDtiaYGtgMDXmQ64Jb4LVqyP5qUQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5. 基础监控

通过基础监控，可以实时查看该数据库中关于数据库与主机资源、内存、表文件、不同类型SQL和连接线程相关的监控指标。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEposZObiar2bRibqTEIibGkxrRajboDlgv9Sveic49WuesZdh1csGIlePDsw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 6. 存储分析

在存储分析界面，可以看到该实例的在一段时间内磁盘实际使用的情况，并展示对未来一段时间内的使用趋势的预测。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpk3R8RRxVgeqg4N1SO5LN0cKze0KjzKUMjkZRFbv3xa4GwN0QbguSmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)