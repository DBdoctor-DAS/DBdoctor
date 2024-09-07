# DBdoctor quickly manages GBase 8a database

GBase 8a is an analytical database product developed by Nanjing University General. It is mainly aimed at scenarios such as data warehousing and big data analysis, providing high-performance data storage, processing, and management capabilities. GBase 8a occupies an important position in the domestic market with its high availability, high reliability, and excellent scalability, especially suitable for business environments that require processing large-scale data and conducting complex query analysis.

At present, DBdoctor has realized the fast management of Gbase 8a distributed database, ensuring the stable operation of the database and timely discovering and dealing with its potential performance problems.

## How to quickly administer GBase 8a?

The following describes in detail how to use the DBdoctor tube GBase 8a and perform performance diagnostics.

### 1. GBase 8a analytical database management deployment architecture

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp95z9zic8MIfrCy3nQn2G0DpN38xQDEia9KMwAVU4MHTAmEaPFtgzdPgw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

In the above figure, GBase 8a distributed database has two virtual clusters, namely VC1 and VC2, and each virtual cluster has two Gnodes and four VC sharding. DBdoctor can manage according to the virtual cluster and automatically discover all Gnode information. Only one Agent (automatic deployment) needs to be deployed on one Gnode to manage the virtual cluster.

### 2. One minute zero dependency DBdoctor server installation

**Environment requirements**：4c8g (recommended independent resource deployment, you can add the option --unlimited to ignore the restrictions of 4c8g)

**Download installation package**：https://www.dbdoctor.cn/col.jsp?id=126

```
#Unzip the installation package and execute a command to complete the deployment ./dbd-I
```

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlf8zDWbymcPT9ZRo5eDQ8I4Ch754YlkeKvuPtn4A4XuHDCaAIlxAZ14VjVsZ6iccwzSMLSWb4U70g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Service access address**：http:// host ip where the service is deployed:13000

**Login account**：tester/Root2023!

**Detailed documentation**：https://demo.dbdoctor.cn/modules/dbDoctor/mdPreview/index.html?readme=help#/

### 3. Quick administration of GBase 8a

#### a) Create an access account
```SQL
create user 'test'@'%' IDENTIFIED BY 'Root2023!';
grant select,process, show view on *.*.* TO 'test'@'%';
```
#### b）Page management GBase 8a instance
1. After clicking the "Instance Management" button, select the "gbase-8a" engine type in the type drop-down box;

2.  Fill in the basic information such as Gcluster access address, virtual cluster VC name, Gnode port, account number and password of the database;

3. Click the "check" button to check whether the instance database is connected normally. If the check is passed, all the Gnode information will be displayed in the management interface;

4. Enter the account information of the host where the Gnode is located, and the Agent is automatically installed by default. Note: After enabling Topology Self-Adaptation, the DBdoctor Server can automatically perceive the topology of the cluster nodes and perform node management or offline.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEphbJFbgLhxAMC2oe3XKAY9ESLzXQNWkXrwmfLYPpe5GeQVH3WNpgPfw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

After passing the connectivity test, click Submit to successfully manage the GBase 8a database. In the instance list interface, you can see the managed Gcluster instance and node instance information. The performance insight is now enabled, let's start experiencing the powerful features of DBdoctor!

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpNdCNKJ2ic6NrmxyXsR3XhiccG7dk2mQ2thHKrf1RVzn1pwguIXl0MYeg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

###  Key points：

#### a） GBase cluster resource overall consumption

Select the "Deployment Agent" method for collection. DBdoctor will automatically install agents on all Gnode nodes under the Gcluster cluster, and display the overall resource consumption and database load of the Gcluster cluster. At the same time, detailed node resource consumption and database load can also be viewed for each Gnode.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpoR5fvkZK7tdF8wq81utm5tDABmAnMu3wQ9ctTltibKZtTdO1ibhQ9EFg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### b）Real-time perception of Gnode topology changes and management

When "Topology Self-Adaptation" is enabled, once the database cluster dynamically expands Gnodes, DBdoctor can automatically perceive and manage the expanded Gnode nodes. And for deleted Gnodes, it will automatically unbind the nodes. DBdoctor can automatically manage GBase cluster nodes, reducing manual intervention and management costs.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpcA80G1tkpayAMB3tO9QCr6Tjjibpet2Jjuq1hQyah5ZLZ0CLnKtJG9A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## What functional services does DBdoctor provide for GBase 8a?

The current DBdoctor is compatible with GBase 8a 9.5.3 and above, and provides **SQL audit, instance inspection, performance insight, root cause diagnosis, basic monitoring, storage analysis** and other functional services.

### 1. SQL auditing

Support **manual audit, slow SQL audit, full SQL audit and OpenAPI audit** methods, can realize the full life cycle closed-loop quality management of incremental SQL and online SQL. Support batch upload SQL files, SQL audit can be completed in the development stage, identify SQL problems in advance, and real-time capture and audit for online stock SQL.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpwwX6LMmFiafDice8XcGw3uiafeZNaLxtHLlU2yrCAxXdLBHVSbKFU4KTQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.  In-depth inspections and reports

Support **automatic inspection, manual inspection** two inspection methods, can timely discover the database in the configuration, performance, resources and other aspects of the problem, to ensure the stability of the database service.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp5s57VmpWN7ibaklW2QRryibBNibMsXeGOJia7XibbjibM2VgNA5U2ywleDKg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3. Performance Insights

The performance insight interface displays the resource usage rate, business traffic, and the average active session situation of the database. Once there is an abnormal interval in the resource usage rate or business traffic, the root cause SQL of the anomaly can be quickly and efficiently found through the average active session trend chart and SQL correlation analysis, so as to solve the problem in the first time.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp9ow1icPJN6uNTk4gicCcKUTbY9nHs0swu2kATuk2ILeyF1X712iaWwwAg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


From the figure, it can be seen that there is a CPU exception in the abnormal event, and the blue area is mostly in the AAS graph corresponding to the time. The corresponding color in SQL correlation analysis is the SELECT statement, so the SELECT statement is the root cause of the CPU exception SQL.
#### 1）User-executed SQL (Gcluster)

In SQL correlation analysis, we showcase the SQL statements executed by the database at a specific time and provide a query function for the execution plan of SELECT statements. The Gcluster cluster has the ability to execute scheduling, decompose the received SQL statements and send them to different Gnode nodes, and finally summarize the results of each node and return them to the caller. The following are examples of SQL statements received in a Gcluster instance.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEppRDeSnyf6M3oNlbDOQWOg9kbubXWxaBUwEo3hZHiaJPibUFccwkvxoFA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 2）SQL flows to the real SQL executed on each Gnode node

Gnode nodes are the basic storage and computing units in the GBase 8a MPP Cluster Database System. Each Gnode is responsible for actually storing cluster data on the local node and receiving the decomposed SQL execution plan from the Gcluster. After executing these plans, the Gnode returns the results to the Gcluster. The following two diagrams show the SQL statements executed on Gnode1 and Gnode2. It can be clearly seen from the diagrams that the SQL executed in Gnode is the decomposed SQL and is specific to the specified vcName.

##### a）Gnode Node 1

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpgIJKqbRXvhACcZic6m8nKBybXia6Ime6ibXicJ5AZWsu8GwyGpH0ibTMF1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### b）Gnode Node 2

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpUaB25bTuDicAT1ViasZgbvq7SYlBYyvRCz2FbhSu6wWibSepj35LMtSpQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以快速的知道该SQL的集群维度整体的消耗，同时针对拆分到每个分片节点上的SQL也是能够直观的看到，是否存在热点或者数据倾斜等导致的集群数据库性能问题。

### 4. Root cause diagnosis

The root cause diagnosis provides a detailed description of the problem caused by the SQL and counts every exception caused by the SQL fingerprint in the database.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpuKxn9Kp6V0ibpfvf9GPLBozRFQzrJDtiaYGtgMDXmQ64Jb4LVqyP5qUQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5. Basic monitoring

Through basic monitoring, real-time monitoring metrics related to database and host resources, memory, table files, different types of SQL, and connection threads can be viewed in the database.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEposZObiar2bRibqTEIibGkxrRajboDlgv9Sveic49WuesZdh1csGIlePDsw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 6. Storage analysis

In the storage analysis interface, you can see the actual usage of the disk for this instance over a period of time, and display predictions for usage trends in the future.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpk3R8RRxVgeqg4N1SO5LN0cKze0KjzKUMjkZRFbv3xa4GwN0QbguSmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)