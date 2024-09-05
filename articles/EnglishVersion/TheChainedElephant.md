# 被锁住的大象(Postgres)，如何跟MySQL赛跑
PG大象和MySQL海豚到底谁跑的更快，一直都是业界热点，但是一旦被锁住，谁都跑不了，以下场景你是否似曾相识？
- 应用服务没有发布任何代码变更，但平时很快的接口突然响应变慢又抖动？
- 比较顺滑的功能页面突然打不开了，浏览器一直在转圈圈？

这些问题的出现大多是由于锁引发的性能问题，就如同悬在头顶的利剑，随时可能引发灾难！

## 一. PostgreSQL锁问题
#### 1.常见的锁相关问题
- 锁等待：大量时间用于等待锁资源，严重影响数据库性能。
- 死锁：造成业务异常或损失。
- 未提交事务与长事务：会长时间持有锁资源，增加锁问题发生的概率。

#### 2.锁问题分析的难点
- **锁机制的复杂性**：PostgreSQL的锁机制是相当复杂的，涉及多种类型的锁，如行级锁、表级锁、页级锁等，不同的锁级别又会影响并发性和性能。这使得理解和管理锁变得复杂，尤其是在大型或高并发的数据库系统中。

- **并发事务复杂性**：锁问题通常发生在多个并发事务之间，这增加了定位问题的复杂性。理解并发事务之间的相互依赖关系和锁竞争情况对于分析锁相关问题至关重要。

- **死锁链的复杂性**：可能存在多个死锁链，进一步增加梳理与分析的难度。

- **难以重现和调试**：有些锁问题可能是偶发性的或难以重现的，这使得调试和解决问题变得更加困难。特别是在生产环境中，由于无法简单地重现问题，需要更加谨慎地分析和调试。

- **日志分析的挑战**：虽然PostgreSQL会记录死锁事件，但死锁日志难以阅读和梳理。需要详细分析日志以了解死锁发生时的上下文信息、涉及的事务和锁等待情况。

- **监控和诊断工具的局限**：尽管PostgreSQL提供了一些系统视图和功能来监控锁活动，但在复杂的生产环境中，很难快速的组合利用好这些工具分析出结果。
## 二.锁分析案例
此处以死锁问题为例：

1. 锁问题识别，在该阶段多通过业务服务异常日志或者通过PostgreSQL数据库监控发现存在死锁事件。

2. 查看PostgreSQL日志可以查看到进一步的死锁信息，如下图所示，死锁日志展示造成死锁的两个事务，以及等锁的两条SQL，到此环节无更多信息推断出这两个事务是如何形成的死锁环。
```Bash
2024-05-26 21:59:20.868 EDT [14761] ERROR:  deadlock detected
2024-05-26 21:59:20.868 EDT [14761] DETAIL:  Process 14761 waits for ShareLock on transaction 36986; blocked by process 14762.
        Process 14762 waits for ShareLock on transaction 36985; blocked by process 14761.
        Process 14761: update people set age=33 where id=1;
        Process 14762: update people set age=23 where id=2;
2024-05-26 21:59:20.868 EDT [14761] HINT:  See server log for query details.
2024-05-26 21:59:20.868 EDT [14761] CONTEXT:  while updating tuple (1090,20) in relation "people"
2024-05-26 21:59:20.868 EDT [14761] STATEMENT:  update people set age=33 where id=1;
2024-05-26 21:59:20.968 EDT [14766] WARNING:  there is already a transaction in progress
```
3. 如果要继续深入分析，有以下几种思路：

    **思路一**：如果造成死锁的sql特征比较明显，可以从业务代码中找出包含该sql的业务逻辑，梳理出完整事务。然后再通过手动方式尝试复现问题。

    **思路二**：分析审计日志，也是DBA用的比较多的一种方式。前提需要开启审计日志（log_statement = 'all'），很多业务考虑性能与存储不会长期开启审计日志，需要开启审计日志后蹲守死锁事件。

    1. 首先在审计日志里面找到死锁事件，以及死锁事务所在的进程ID。

        ```Bash
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on tra
        nsaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28417: update people set age=23 where id=2;
        Process 28419: update people set age=33 where id=1;","See server log for query details.",,,"while updating tuple (1090,20) in relation ""people""","update people set age=23 where id=2;",,,"pgbench"
        ```
        
    2. 然后再在审计日志里梳理进程这几个PID在死锁时间点前后都执行了哪些SQL。
        ```Bash
        bash-4.2$ cat postgresql-2024-05-27_025951.csv |grep 28417
        2024-05-27 03:04:13.265 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,1,"idle",2024-05-27 03:04:13 EDT,8/93,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.268 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,2,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,0,LOG,00000,"statement: update people set age=23 where id=1;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.289 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,3,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,37429,LOG,00000,"statement: SELECT pg_sleep(5);",,,,,,,,,"pgbench"
        2024-05-27 03:04:18.293 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,4,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,37429,LOG,00000,"statement: update people set age=23 where id=2;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on transaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28417: update people set age=23 where id=2;
        ```
        <center>死锁前后Pid1执行过的SQL</center>

        ```Bash
        bash-4.2$ cat postgresql-2024-05-27_025951.csv |grep 28419
        2024-05-27 03:04:13.281 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,1,"idle",2024-05-27 03:04:13 EDT,9/173,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.282 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,2,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,0,LOG,00000,"statement: update people set age=33 where id=2;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.291 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,3,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: SELECT pg_sleep(5);",,,,,,,,,"pgbench"
        2024-05-27 03:04:18.307 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,4,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: update people set age=33 where id=1;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on transaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28419: update people set age=33 where id=1;","See server log for query details.",,,"while updating tuple (1090,20) in relation ""people""","update people set age=23 where id=2;",,,"pgbench"
        2024-05-27 03:04:28.300 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,5,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: commit;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.319 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,6,"idle",2024-05-27 03:04:13 EDT,9/174,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        ```
        <center>死锁前后Pid2执行过的SQL</center>


    3. 如果是超过两个事务造成的死锁环，则需要梳理更多的事务SQL，最终可以梳理出死锁环。
    4. 最终分析出事务里哪些SQL、哪些锁资源间有冲突。

从上面手动处理的流程来看，当这个问题发生时，处理起来非常麻烦，中间会有很多的干扰信息和审计日志性能问题，那有没有更好的方案呢？

**近期DBdoctor3.2.0版本新上线了PostgreSQL锁透视功能，使用PostgreSQL数据库的朋友再也不用担心因数据库锁而导致的业务系统卡顿和死锁问题了，DBdoctor可助力您快速找到卡顿源头！**

下面我们来看看DBdoctor针对PostgreSQL是如何进行锁透视的。

## 三. DBdoctor还原PG死锁问题形成过程
针对锁问题分析的难点，DBdoctor利用eBPF技术采集PostgreSQL事务SQL的执行过程数据，其中包括细粒度的锁数据，并通过泳道图可视化呈现锁在多事务并发执行中的详细形成过程。

PostgreSQL的锁透视包括锁等待、死锁、未提交事务、长事务四大场景，下面我们来看看上面的死锁Case，DBdoctor是如何精准找到并还原现场的 。
1. 首先我们打开锁透视功能，能看到锁异常事件汇总中是存在死锁问题。
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticK9nXEOckzLwwhn4kZWiaydNT2O4Xlmlib2KNyiaPUzSV1jZQPic7diaVlSjw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. 点击死锁，查看死锁可视化分析
在死锁可视化分析中，如下图，会将>=2个事务所形成的死锁环绘制出来，会标注各事务持有和等待的具体锁资源，如某个page或tuple，同时会用红色标注出被回滚的事务。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKDSdm3jIj9eqaTMX1Tx0GfGqJvRWAvADYqcqNetnyH5fJ7goftiawKmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKDSdm3jIj9eqaTMX1Tx0GfGqJvRWAvADYqcqNetnyH5fJ7goftiawKmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

死锁可视化分析分为两个部分：

- 最上面部分显示的是死锁的形成有多少个事务参与，是由于锁了哪些行记录导致的互相等待形成的死锁环。

- 下面部分展示的是参与死锁的事务，按照时间轴各个事务分别执行了哪些SQL，在执行到哪个SQL产生了锁等待，最终哪个事务被回滚了，完整回放死锁的形成过程。

这个Case中，我们能直接看到事务A和事务B在并发执行产生了死锁，最上面的死锁环形图可以直接展示了AB事务是如何形成环的。

## 四. 其他锁场景DBdoctor如何快速定位
- 锁等待可视化分析

锁等待泳道图中，会展示两个事务的等待关系和详细事务SQL执行过程，其中在等锁事务泳道图中会标注出等锁事件，卡顿问题轻松追溯。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKXVYhK3rib7LVPcJ0Xy3gU8iaEZMbJDYtFDyGtx6IDK0paKOvJlO4Z1fg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 未提交事务可视化

未提交事务的出现往往是由于业务代码逻辑问题在一些极端场景等事务未正常进行关闭(Sleep状态大于10秒的定义为未提交事务)。未正常关闭事务会导致锁占用不释放，可能会导致业务正常事务获取不到锁超时等问题。从下面的图中我们能看到UPDATE执行结束后并没有立即执行commit，而是长时间处于无操作状态，往往由于事务范围设计不合理导致的。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKrvicO3ECeHSvY0qDnWvCvsJ4IqC4sLDVqhjvBkqc2ZHImArlzBEvia0g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 长事务可视化

长事务的出现往往是由于事务中存在慢SQL导致，该SQL一直处于执行中但执行时间比较长（执行耗时超过10s定义为长事务）。长事务是在SQL执行过程中耗时过长，代表SQL执行效率低，往往需要对慢SQL进行优化。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKLAsKUo2GRpAiaLDqDGEWTBhCa2Lo6vxIyOJHoTuUB1e7icUN2G7yhb8A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

DBdoctor的锁透视功能非常强大，能够快速诊断和定位数据库中的锁问题。