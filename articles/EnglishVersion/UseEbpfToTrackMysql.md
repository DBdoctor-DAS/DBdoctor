# 用蜜蜂(eBPF)来追踪海豚(MySQL)，性能追的上吗

DBdoctor 基于 eBPF 实现了数据库可观测，能够在**不改代码、不改启动参数、不重启进程的前提下对数据库内核进行追踪**，拿到更细粒度的内核数据进行数据分析，做到了性能观测的数学模型化。在数据库领域这是一种全新的技术手段，但偶尔也有小伙伴会咨询一些不解：

**eBPF观测MySQL做出来是不是一个玩具？能否用于生产？**

**每一条SQL执行过程都探测，对数据库的性能消耗到底有多大？**

**让一只蜜蜂(eBPF)去追海豚（MySQL），能追得上吗？**

本文将用数据说话，带大家一起用压测对比的方法揭开以上谜团！

**因测试比较严谨，全文比较长，忙碌的小伙伴可直接查看最下方的总结。**

## Agent采集处理流程中，哪些环节会存在开销？

对于Agent来说占用资源是一定的，那如何才能做到最小化资源占用？下面将详细展开DBdoctor是怎样进行资源消耗控制的。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv06kX43rc0XRDWFGN5sIsVWZicEe5yBT77PAE5uD22njiamLeUia8Ayr5WA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Agent利用eBPF实现对MySQL的内核探测，从原理机制上看eBPF的消耗主要在CPU上，CPU的消耗包含以下两个方面：

### 1）内核态探测开销
当我们对MySQL内核的某个函数增加探测，相当于在函数的进入和返回都会执行我们自定义的eBPF程序，执行完后再进行MySQL的下一步执行。那这里增加的开销就是函数被调用的时候执行**eBPF程序代码**的开销（实际占用的是MySQL进程的CPU资源）。

从上图我们能看到，控制**单次调用消耗**和**调用频率**尤为重要。在实际生产中这两个指标的性能开销该如何控制呢？

理解Agent处理流程以后，我们希望设计一组测试，尤其是针对性能要求很高的OLTP数据库。与业界其他可观测厂商测试方法不同，比如基于eBPF探测APP应用，一般都是注入多少负载压力的情况下进行测试评估性能损耗，我们要求是在极端负载压力情况下（数据库性能诊断的初衷就是要在数据库极端情况下进行关键数据采集，才能分析出性能问题根因），希望该测试用例能评估以下几点：

- Agent 的运行对业务性能有什么样的影响？

    - 业务的 QPS/TPS 降低了多少？

    - 业务的 RT（Response Time）升高了多少？

    - 业务的 CPU/MEM 消耗升高了多少？

- Agent 自身的处理性能如何？

    - 特定压力下 Agent 的 CPU/MEM 消耗如何？
### 2）用户态数据处理开销

探针采集到详细MySQL内核指标数据后，agent需要对数据进行组装处理，最后再通过网络投递出去进行存储。

从图上我们能看到，16c16g规格的主机，我们给agent容器资源限制在1c1g，这样可以保证Agent不会抢占主机资源（稳定性兜底），这块资源占用很小，可以忽略不计。

## 如何进行测试，才能评估出eBPF开销？
首先，为了评估 Agent对业务性能的影响，我们希望设计典型的业务场景来进行评估，并希望能覆盖到 Agent 处理流程中的所有重要环节。我们一共设计了两个场景，如下图所示：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0TPqHDP1kwDwjISR9LmvEsI5FVX6KG8T6ECDibkS5890Sj0BO2wzTE5Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 1）通过典型业务场景评估 Agent 对业务性能的影响
- 场景 A - 基准压测只读、只写、读写混合场景

    使用 sysbench 作为压力工具，分别在读写、只读、只写模式下不同并发时的性能（TPS、QPS）。

    - 分别对4c8g(bp:4G)/8c16g(bp: 8G) 的MySQL 使用1c1g/2c1g的Agent进行压测，没有部署Agent压测的消耗数据作为对照。

    - 分别在并发用户数为1/2/4/8/16/32/64 的情况下进行测试。

    - 每个测试运行30分钟，收集TPS和QPS性能数据、Agent 的实际物理资源消耗数据、对MySQL CPU资源的消耗数据。
- 场景 B - 典型订单交易 OLTP 场景

    交易场景在数据库中是比较典型的OLTP类型，而且经常会有性能相关问题，特别是大促等场景下尤为突出。TPC-C 模型下的压测，评估Agent对业务处理能力的影响程度。

- 场景 C - 数据库单个 SQL 请求消耗场景

    MySQL的SQL执行过程探测，都会执行对应的探测点 eBPF 程序，该程序的开销即业务性能的影响开销。该测试通过给程序增加打印（仅内部验证评估使用）。所有上述三大场景，我们均会分别测试停用 Agent（基线）、运行Agent 两种情况，通过对比得出 Agent 对业务性能的影响。另外，我们也会注入不同 TPS 的压力，直至达到业务极限处理能力，以评估不同压力下的影响是否存在差异。另一方面，对于Agent进程自身处理性能的评估，我们会记录场景 A、B、C 中Agent 进程的 CPU/MEM开销。除此之外，我们也希望设计一些更极端的场景，用来评估Agent在资源受限情况下的资源消耗和极限处理能力。Agent的处理主要是读取数据并网络投递的开销，没有做其他逻辑的处理，所有开销主要在CPU上，那么CPU的开销和QPS有关，我们需要验证QPS对 Agent 资源消耗的影响。
### 2）评估 Agent 自身的处理性能
- 场景 A - 只采集主机性能数据

    我们只采集主机和实例资源相关的性能数据，不开启性能洞察和锁分析，同时纳管一个机器上多个数据库实例，并评估 Agent 需要占用多少 CPU/MEM 资源。

- 场景 B - 开启全功能采集并压测实例

    基于压测工具不断增加 QPS 的量，评估 Agent 的 CPU/MEM 资源占用与QPS的关系，我们将会对所有上述 2 个场景的测试方法和结果进行详细的阐述。测试过程中 DBdoctor Agent全部使用默认配置，没有进行任何调优。帮助用户评估业务量需要多少 Agent资源配置。

## 给数据库施压，Agent对业务的影响到底有多大？
通过分析Agent的处理流程，我们设计了3大场景进行全方位评测。

测试环境信息：
```
数据库服务器：ECS 独享cpu
CPU：Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz
ECS规格：16c16g
机器内存：32GB DDR4
mysql版本：5.7.37
操作系统：CentOS Linux 8
内核版本：4.18.0-348.7.1.el8_5.x86_64
```
### 1）场景 A - 基准压测只读、只写、读写混合场景测试结果

以下测试均是本地连接数据库进行，在不同并发连接对 1000 张单表 1W 数据表进行压力测试，测试时间 30 分钟，压测数据为 5 次数据均值。下面以 MySQL 4C8G 的规格进行压测，Agent 资源限制在 1C1G/2C1G，对比开启 Agent 前后的读写、只读、只写三种模式下的性能表现。
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0PNMmjxkAKaL9QQJSORDr9PkUPJOibBWy4qw2XkR7jIlRqIN6FdnsnXQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

下面我们通过对比折线图来展示，有无Agent或者增加Agent CPU情况下业务QPS的表现：
#### a）读写模式
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0K4KeQJapicfuqLpx8tIosQWtXLI0ibMsNey0RSBzqCQ6NSsv98QNLlxw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| 对照组  | 9506 |  143 | 230% | 1.68G |
| Agent组  | 9406 | 146 | 250% | 1.68G |

结果说明：

- QPS 下降 1.05%

- 时延提高 2%

- 1W 的 QPS CPU 占用提高 0.2c
> 该部分资源主要消耗在高并发下产生的大量perf事件提交


#### b）只读模式
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0cuL7mmyHxNJkq69WyXcRyjibI9JricwRpNCqFQ3ibZOq2kb7TjtkfrBcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| 对照组  | 46499 |  82 | 310% | 1.68G |
| Agent组  | 44195 | 86 | 290% | 1.68G |

 结果说明：
- 纯读模式的 QPS 峰值可达到 4.5w，该压力下面持续增压直到CPU打满也会导致QPS下降至3.5w（无 Agent 的情况）。而开启 agent 采集峰值情况会消耗更多的 CPU(相当于占用了 MySQL 本身的CPU,会消耗 0.2*4.5=0.9c 用于执行eBPF 程序，4c8g 的 MySQL 规格实际能用于 MySQL 进程使用的是 3c8g 的规格，CPU打满情况QPS 可达到 3.2w)，如果给该实例额外增加0.9c的CPU，该性能吞吐可以达到和当前无Agent一致。
- 内存使用无变化

#### c）只写模式

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0qchmEBwWm7WNoP3k6q1efLFfkDR1POGzIk5DywzjndPYbgjrjBOlvQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| 对照组  | 5587 |  112 | 210% | 1.68G |
| Agent组  | 5656 | 113 | 220% | 1.68G |

结果说明：

- QPS 无下降

- 时延无明显变化

- 5k 的 QPS CPU 占用提高 0.1c

### 2）场景 B - TPC-C 测试结果

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| 对照组  | 9120 |  143 | 230% | 1.68G |
| Agent组  | 9003 | 146 | 250% | 1.68G |

结果说明：

- QPS 下降 1.28%

- 时延提高 2%

- 1w 的 QPS CPU 占用提高 0.2c

- 场景 B 的数据和场景 A 的基本吻合，说明 Agent 的主要消耗来自于 QPS，1w 的 QPS 占用 0.2c 的 CPU 资源

### 3）场景 C - 数据库单个 SQL 请求消耗场景
```
connection-28688 [014] d... 20941.295869: bpf_trace_printk: mysql_sql_req_entry start:20942742697886
connection-28688 [014] d... 20941.295956: bpf_trace_printk: mysql_sql_req_entry end:20942742817108
connection-28688 [014] d... 20941.304001: bpf_trace_printk: mysql_sql_return start:20942750859976
connection-28688 [014] d... 20941.304019: bpf_trace_printk: mysql_sql_return end:20942750881134
````

|     |程序消耗（ns）  | uprobe探针命中消耗（ns） | 消耗汇总（ns）） |
|  ----  | ----  | ---- | ----  | 
| SQL 请求 | 119222 |  3000 | 0.122 | 
| SQL 返回 | 21158 | 4500 | 0.025 | 

结果说明：

- 添加 eBPF 探针后，单条 SQL Latency(ms) 增加 0.147ms

测试结果表明  **Agent消耗只与业务数据库QPS有关**，**在读写模式和只写模式的场景下几乎无任何影响**。在极端只读场景压测下，轻量级的采集对生产业务的影响 1w QPS 需要消耗 0.22c 的 CPU 资源。

## 1c1g的Agent进程自身资源消耗如何？

上面 A、B、C三个场景测试过程中，我们同时也记录了主机上Agent进程自身在高负载下的资源消耗，见下表：
|     |CPU  | Memory（G）|
|  ----  | ----  | ---- |
| 压测状态（25000QPS） | 61% |  0.168 | 
| 静态（监测无压力实例） | 5% | 0.168 |
| 不纳管实例（不开启eBPF） | 1% | 0.028 |

Agent 自身资源占用情况:

- 只采集主机性能数据时，使 CPU 占用为 0.01c,内存占用 0.028G

- 增加一个实例，且该实例无压力时，CPU 占用增长 0.04c,内存增长 0.14G

- 采集的实例每增加 1w QPS，agent 的 CPU 消耗增加 0.22c, 内存无增长

## 总结

通过六个场景的全方位评测，我们对 Agent 的性能情况有了完整的了解，DBdoctor针对采集带来的消耗精确到QPS量级上，1w的QPS消耗0.2c的资源，整体性能损耗正常控制在2%以内(非常低的范围)，同时还可以通过DBdoctor的**性能诊断功能优化数据库、降低SQL资源消耗，能大幅度提升性能。**

极端压测的性能损耗和资源对照：

- 对MySQL性能影响：

    - 在只读和读写场景对MySQL性能影响均不明显，QPS和时延均在2%左右波动，在误差波动范围内

- 对MySQL资源影响：

    - 每10K QPS会使Mysql占用的CPU增加0.2c

    - 对MySQL内存占用无影响

- Agent进程自身资源占用情况:

    - 在不采集Mysql/数据的情况下(该场景只采集主机资源数据)，使用0.04c CPU，0.24G内存

    - 在采集状态下(MySQL QPS 2w)，使用0.85c CPU，0.24G内存。

以上性能损耗结果不知大家看了什么感受？**与MySQL自身开启general_log后的性能损耗相比简直不值一提。**

**如此低的性能损耗就可以采集到全量日志，并进行内核级精确的诊断**，果然是技术改变世界。

最后，本文所有测试数据均是在 Agent 默认配置下测得的。我们也期待小伙伴的更多评测，更期待把评测报告与我们共享。大家可以在下方官网地址下载DBdoctor，软件零依赖、一键拉起，亲手产出自己的压测数据。