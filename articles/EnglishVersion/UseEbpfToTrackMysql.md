# Can the performance of tracking dolphins (MySQL) with bees (eBPF) catch up?

DBdoctor realizes database observability based on eBPF, which can track the database kernel without changing the code, starting parameters, or restarting the process , obtain finer grain kernel data for Data Analysis, and achieve mathematical modeling of performance observation. This is a new technical means in the field of databases, but occasionally some friends will consult some puzzles:

**Does eBPF observe MySQL to make a toy? Can it be used in production?**

**Every SQL execution process is probed, how much does it consume in terms of database performance?**

**Can a bee (eBPF) catch up with a dolphin (MySQL)?**

This article will speak with data and use torture testing to uncover the above mysteries together!

**Due to the rigorous testing, the full text is relatively long. Busy friends can directly view the summary at the bottom.**

## Which steps in the Agent collection and processing process will incur overhead?

For Agents, resource usage is fixed. How can we minimize resource usage? The following will detail how DBdoctor controls resource consumption.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv06kX43rc0XRDWFGN5sIsVWZicEe5yBT77PAE5uD22njiamLeUia8Ayr5WA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Agent uses eBPF to implement kernel detection of MySQL. From the principle and mechanism, the consumption of eBPF is mainly on CPU, which includes the following two aspects:

### 1）Kernel Mode Probing Overhead
When we add probing to a function in the MySQL kernel, it is equivalent to executing our custom eBPF program when the function enters and returns, and then executing the next step of MySQL. The added overhead here is the overhead of executing the eBPF program code when the function is called (which actually occupies the CPU resources of the MySQL process).

From the above figure, we can see that controlling single call consumption and call frequency is particularly important. How to control the performance overhead of these two indicators in actual production?

After understanding the Agent processing flow, we hope to design a set of tests, especially for OLTP databases with high performance requirements. Unlike other observable vendors' testing methods in the industry, such as eBPF detection APP applications, which generally evaluate performance loss under a certain amount of load pressure injection, we require that under extreme load pressure conditions (the original intention of database performance diagnosis is to perform critical Data Acquisition under extreme database conditions in order to analyze the root cause of performance problems), we hope that this test case can evaluate the following points:

- What impact does the operation of Agent have on business performance?

    - How much has the QPS/TPS of the business decreased?

    - How much has the RT (Response Time) of the business increased?

    - How much has the CPU/MEM consumption of the business increased?

- How is the processing performance of the Agent itself?

    - How is the CPU/MEM consumption of the Agent under specific pressure?
### 2）User Mode Data Processing Overhead

After the probe collects detailed MySQL kernel metric data, the agent needs to assemble and process the data, and finally deliver it over the network for storage.

From the figure, we can see that for a 16c16g host, we limit the agent container resources to 1c1g to ensure that the agent does not preempt host resources (stability fallback). This resource usage is very small and can be ignored.

## How to conduct testing to evaluate eBPF overhead?
Firstly, in order to evaluate the impact of Agent on business performance, we hope to design typical business scenarios for evaluation, and hope to cover all important links in the Agent processing flow. We have designed two scenarios in total, as shown in the following figure.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0TPqHDP1kwDwjISR9LmvEsI5FVX6KG8T6ECDibkS5890Sj0BO2wzTE5Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 1）Evaluate the impact of Agent on business performance through typical business scenarios
- Scene A - Benchmark torture testing read-only, write-only, read-write hybrid scenario

    Using sysbench as a stress tool, the performance (TPS, QPS) under different concurrency modes of read-write, read-only, and write-only.

    - Use 1c1g/2c1g Agents for torture testing on MySQL with 4c8g (bp: 4G)/8c16g (bp: 8G) respectively, without deploying Agent torture testing consumption data as a control.

    - Tested with 1/2/4/8/16/32/64 concurrent users.

    - Each test runs for 30 minutes, collecting TPS and QPS performance data, actual physical resource consumption data of the agent, and consumption data of MySQL CPU resources.
- Scenario B - Typical Order Trading OLTP Scenario

    The transaction scenario is a typical OLTP type in the database, and there are often performance-related issues, especially in scenarios such as big promotions. Torture testing under the TPC-C model evaluates the impact of agents on business processing capabilities.

- Scenario C - Database single SQL request consumption scenario

    MySQL's SQL execution process detection will execute the corresponding detection point eBPF program, and the cost of this program is the impact cost of business performance. This test adds printing to the program (only used for internal verification and evaluation). In all the above three scenarios, we will test the two situations of disabling Agent (base line) and running Agent respectively, and compare them to determine the impact of Agent on business performance. In addition, we will inject different TPS pressures until the business limit processing capacity is reached to evaluate whether there are differences in the impact under different pressures. On the other hand, for the evaluation of the processing performance of the Agent process itself, we will record the CPU/MEM overhead of the Agent process in scenarios A, B, and C. In addition, we also hope to design some more extreme scenarios to evaluate the resource consumption and maximum processing capacity of the Agent under resource constraints. The processing of the Agent is mainly the overhead of reading data and network delivery, without any other logical processing. All the overhead is mainly on the CPU, so the CPU overhead is related to QPS. We need to verify the impact of QPS on the Agent's resource consumption.
### 2）Evaluate the processing performance of the Agent itself
- Scene A - Collect only host performance data

    We only collect performance data related to host and instance resources, without enabling performance insights and lock analysis. At the same time, we manage multiple database instances on one machine and evaluate how much CPU/MEM resources the Agent needs to occupy.

- Scenario B - Enable full-featured collection and torture testing example

    Based on the torture testing tool, we continuously increase the amount of QPS and evaluate the relationship between the CPU/MEM resource usage of the Agent and QPS. We will provide a detailed explanation of the testing methods and results for all the above two scenarios. During the testing process, DBdoctor Agent used the default configuration without any optimization. This helps users evaluate how much Agent resource configuration is needed for business volume.

## How much impact does Agent have on business by putting pressure on the database?
By analyzing the processing flow of the Agent, we designed three major scenarios for comprehensive evaluation.

Testing environment information:
```
Database server: ECS exclusive CPU
CPU：Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz
ECS specification: 16c16g
Machine Memory: 32GB DDR4
MySQL version: 5.7.37
Operating System: CentOS Linux 8
Kernel Version: 4.18.0-348.7.1.el8_5.x86_64
```
### 1）Scene A - Benchmark torture testing read-only, write-only, read-write mixed scenario test results

The following tests are all conducted by connecting to the local database. Stress testing is performed on 1000 single tables and 1W data tables with different concurrent connections. The testing time is 30 minutes, and the torture testing data is the average of 5 times. The torture testing is performed using the MySQL 4C8G specification, with Agent resources limited to 1C1G/2C1G. The performance is compared in three modes: read-write, read-only, and write-only, before and after enabling Agent.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0PNMmjxkAKaL9QQJSORDr9PkUPJOibBWy4qw2XkR7jIlRqIN6FdnsnXQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Below we will show the performance of business QPS with or without Agent or with the addition of Agent CPU by comparing the line chart.
#### a）Read and write mode
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0K4KeQJapicfuqLpx8tIosQWtXLI0ibMsNey0RSBzqCQ6NSsv98QNLlxw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| Control group  | 9506 |  143 | 230% | 1.68G |
| Agent Group  | 9406 | 146 | 250% | 1.68G |

Result description:

- QPS decreased by 1.05%

- Latency increased by 2%

- 1W QPS CPU usage increased by 0.2c.
> This part of the resources mainly consumes a large number of perf event submissions generated under high concurrency


#### b）Read-only mode
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0cuL7mmyHxNJkq69WyXcRyjibI9JricwRpNCqFQ3ibZOq2kb7TjtkfrBcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
|  Control group  | 46499 |  82 | 310% | 1.68G |
| Agent Group   | 44195 | 86 | 290% | 1.68G |

 Result description:
- The peak QPS of pure read mode can reach 4.5w. Continuously increasing the pressure until the CPU is full will cause the QPS to drop to 3.5w (without an agent). Enabling the agent to collect the peak situation will consume more CPU (equivalent to occupying MySQL's own CPU, which will consume 0.2 * 4.5 = 0.9c for executing eBPF programs. The 4c8g MySQL specification can actually be used for MySQL processes, but the 3c8g specification can be used. When the CPU is full, the QPS can reach 3.2w). If an additional 0.9c CPU is added to this instance, the performance throughput can reach the same level as the current one without an agent.
- No change in memory usage
#### c）Write pattern only

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlN8BsI6woicUKwusEJonQv0qchmEBwWm7WNoP3k6q1efLFfkDR1POGzIk5DywzjndPYbgjrjBOlvQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| Control group  | 5587 |  112 | 210% | 1.68G |
| Agent Group  | 5656 | 113 | 220% | 1.68G |

Result description:

- No drop in QPS

- No significant change in delay

- 5k QPS CPU usage increased by 0.1c.

### 2）Scene B - TPC-C test results

|     |QPS  | Latency(ms) | CPU(mysqld) | Memory(mysqld) |
|  ----  | ----  | ---- | ----  | ---- |
| Control group  | 9120 |  143 | 230% | 1.68G |
| Agent Group  | 9003 | 146 | 250% | 1.68G |

Result description:

- QPS decreased by 1.28%

- Latency increased by 2%

- 1W QPS CPU usage increased by 0.2c.

- The data of Scenario B is basically consistent with that of Scenario A, indicating that the main consumption of Agent comes from QPS. 1w QPS occupies 0.2c of CPU resources

### 3）Scenario C - Database single SQL request consumption scenario
```
connection-28688 [014] d... 20941.295869: bpf_trace_printk: mysql_sql_req_entry start:20942742697886
connection-28688 [014] d... 20941.295956: bpf_trace_printk: mysql_sql_req_entry end:20942742817108
connection-28688 [014] d... 20941.304001: bpf_trace_printk: mysql_sql_return start:20942750859976
connection-28688 [014] d... 20941.304019: bpf_trace_printk: mysql_sql_return end:20942750881134
````

|     |Program consumption (ns) | uprobe probe hit consumption (ns) | consumption summary (ns)) |
|  ----  | ----  | ---- | ----  | 
| SQL Request | 119222 | 3000 | 0.122 |
| SQL return | 21158 | 4500 | 0.025 |

Result description:

- After adding the eBPF probe, the single SQL Latency (ms) increased by 0.147ms

The test results show that **Agent consumption is only related to the QPS of the business database**, and **has almost no impact in the scenarios of read-write mode and write-only mode**. In the extreme read-only scenario torture testing, the impact of lightweight collection on production business requires a CPU resource consumption of 0.22c.

## How does the Agent process of 1c1g consume its own resources?

During the testing process of scenarios A, B, and C, we also recorded the resource consumption of the Agent process on the host under high load, as shown in the table below.
|     |CPU  | Memory（G）|
|  ----  | ----  | ---- |
| torture testing status （25000QPS） | 61% |  0.168 | 
| Static (monitored without pressure instance) | 5% | 0.168 |
| Do not manage instances (do not enable eBPF) | 1% | 0.028 |

Agent's own resource usage:

- When only collecting host performance data, the CPU usage is 0.01c and the memory usage is 0.028G

- When an instance is added and there is no pressure, CPU usage increases by 0.04c and memory increases by 0.14G.
- For every additional 1w QPS of collected instances, the CPU consumption of the agent increases by 0.22c, with no memory growth
## Summary

Through a comprehensive evaluation of six scenarios, we have a complete understanding of the performance of Agent, DBdoctor for the collection of consumption accurate to the QPS level, 1w QPS consumption of 0.2c of resources, the overall performance loss is normally controlled within 2% (very low range), but also through DBdoctor's performance diagnosis function to optimize the database, reduce SQL resource consumption, can greatly improve performance.

Performance loss and resource comparison of extreme torture testing:

- Impact on MySQL performance:

    - In read-only and read-write scenarios, the impact on MySQL performance is not significant, and QPS and latency fluctuate around 2%, within the error fluctuation range

- Impact on MySQL resources:

    - Every 10K QPS will increase the CPU usage of MySQL by 0.2c.

    - No impact on MySQL memory usage

- Agent process's own resource usage:

    - Without collecting Mysql/data (this scenario only collects host resource data), use 0.04c CPU and 0.24G memory

    - In the acquisition state (MySQL QPS 2w), using a 0.85c CPU and 0.24G memory.

The above performance loss results do not know what everyone feels? Compared with the performance loss after MySQL itself opens general_log, it is simply not worth mentioning.

Such low performance loss can collect the full amount of logs and perform precise diagnosis at the kernel level , it is indeed technology that changes the world.

Finally, all the test data in this article was measured under the default configuration of the Agent. We also look forward to more reviews from our friends, and even more so, we look forward to sharing the review report with us. You can download DBdoctor from the official website address below. The software has zero dependencies and can be pulled up with one click to produce your own torture testing data.