# 用DBdoctor做数据库巡检：顺滑应对春节流量洪峰

## 目录

- [01.数据库巡检了，为何还要烧香拜佛](#01.数据库巡检了，为何还要烧香拜佛)
- [02.数据库性能诊断工具DBdoctor的巡检能力](#02.数据库性能诊断工具DBdoctor的巡检能力)
- [03.如何使用DBdoctor巡检数据库](#03.如何使用DBdoctor巡检数据库)

## 01.数据库巡检了，为何还要烧香拜佛
数据库巡检是一种预防性的维护措施，一般会使用脚本或者工具自动化进行巡检，涉及的巡检对象包括硬件和系统资源、数据库日志、参数配置、数据库性能、安全性、备份恢复等，是一个非常杂且细的工作，任何的一个小问题都有可能成为数据库不稳定的风险点。特别是数据库性能问题，是整个数据库行业的难题。尽管基于规则、指标等DBA的日常运维经验可以搞定大部分的基础巡检，但如果巡检出了性能问题，DBA该怎么做呢？是处理还是不处理？定位问题后没有现场该如何排查呢？这些纠结时刻令人对数据库巡检又恨又怕，有时又无能为力，只能去“拜一拜”。

![DBA](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/DBA.png)
比如我们公司曾遇到的下面这几个问题场景：

### 1）数据库SQL资源消耗量化难，性能抖动找根因SQL难
在巡检过程中，我们统计一周的数据库实例cpu和disk IO的使用情况，发现有很多实例都存在性能抖动打满问题，很难量化到是哪条SQL导致的。这让巡检同学很犹豫，不知道针对这项结果是处理还是不处理，处理的话没有现场数据不好排查，可能收益也甚微，如果不处理又担心成为随时爆炸的雷点。
![IO抖动存在打满](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/ioJitterIsFull.png)
### 2）数据库本身提供根因结果不完整，看不出数据库内核中真实结果到底是什么样子

比如死锁日志，在5.6和5.7的MySQL内核中第一个事务的Hold锁信息是不打印的，日志中展示的都是当前执行的SQL，不能展示该事务的所有SQL，因此难以准确获得这个死锁环的完整形成过程。即使把死锁日志给到业务方也是很难彻底解决。
TheHoldLockInformationIsMissing.png
![缺少HOLD锁信息](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/TheHoldLockInformationIsMissing.png)
再比如业务卡顿问题，在MySQL中基于系统表进行锁等待分析是不能直接得出事务之间的详细SQL阻塞关系，需要分析具有事务信息的审计日志才能准确分析。而目前行业内的审计日志基本都是通过网络抓包的方式，事务相关信息是缺失的，准确关联是做不到的。
### 3）强依赖经验，索引优化难评估
目前索引推荐基本强依赖DBA的经验，需要case by case分析，有些有研发能力的数据库相关厂商实现了基于规则的索引推荐能力，但在很多场景下是不准的，索引优化难。
![索引优化](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/IndexOptimization.png)
从上面这些问题可以看出，性能问题是悬在每个DBA头顶上的一把达摩克利斯之剑，全靠运气。这也就有了某些互联网公司通过烧香拜佛的行为来求数据库稳定的情况。饶是如此，DBA假期还是得时刻背着电脑，24小时待命，靠运气决定今天是否可以愉快地玩耍。那到底有没有解决的办法呢？让我们有请DBdoctor！

## 02.数据库性能诊断工具DBdoctor的巡检能力
与传统的巡检不同，除了支持基础监控、一键巡检、巡检报告的基础能力外，DBdoctor的3.1.1版本增加了针对性能问题的深度巡检项，进行深入分析并得出巡检结果，并提供现场还原和找到异常的准确根因SQL，并提供优化建议。让DBA能实现快速巡检、及时闭环解决性能问题。

### 1）基于eBPF技术的巡检深度下钻，找出异常问题准确根因

1. 性能CPU、IO等资源问题可量化，快速评估资源指标异常抖动根因。将资源抖动作为巡检项纳入巡检，可更进一步评估实例的健康状况，针对巡检出来的该问题，能准确提示根因SQL。

2. 基于eBPF技术的MySQL内核级数据采集，窥探每条SQL的完整执行过程，提供精细化指标数据，做到异常现场可回放，不用靠猜测或者类似故障review即可得出真实结论。
3. 补齐MySQL内核本身死锁不完整的根因结果，DBdoctor可以不侵入用户数据库源码，完整准确还原死锁形成过程，也不用开启死锁打印参数即可获取到用户的全量死锁日志，无需蹲守。

4. 业务卡顿问题全量记录，可视化泳道图回放数据库内核的锁等待的事务SQL形成过程，可以准确回答一条SQL变慢的真实根因是什么。比如SQL内核等锁、物理资源瓶颈、SQL本身Cost消耗大等。
![数据采集器](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/DataCollector.png)

### 2）自研基于Cost的InnoDB优化器，针对SQL问题提供最佳优化建议
与基于规则经验的方式不同，我们自研旁路Cost优化器，可以快速评估一条SQL的实际Cost消耗，用户无需在生产数据库上开启Optimizer Trace即可得到该SQL的实际Cost代价。在索引推荐这块Cost优化器可以做到无需在用户的生产库上增加索引，即可评估出这条SQL选择新增索引的Cost代价消耗，从而可以推荐最优索引。同时我们针对推荐的索引做了代价量化，让用户可以快速知道推荐的索引与源表相比，Cost代价消耗降低多少倍。也可正面回答研发同学的疑问——加该索引能带来多少性能提升。
![最优索引推荐](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/OptimalIndexRecommendation.png)

## 03.如何使用DBdoctor巡检数据库

DBdoctor巡检分为定时巡检和一键巡检两种模式，用户可以快速知道租户项目下的数据库实例有哪些问题，并可查看详细巡检报告。针对巡检报告中的性能异常项可使用性能洞察功能还原异常时刻现场，帮助用户快速找到异常根因和最佳优化建议。您可以按照以下步骤进行巡检：

### 1）找出全部问题

#### step1.选择租户项目，打开Dashbord，拖到最下面，会有巡检相关卡片

![最优索引推荐](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/Dashbord.png)

备注：
**① 数据库健康巡检得分**：只展示巡检分数小于100分的实例，按照分数低到高的严重程度实例展示，并可以查看该实例的巡检报告，同时用户可以选择主动下发立即巡检。当发现数据库监控巡检得分的卡片中没有出现实例，那么恭喜你，该租户项目下的所有实例都健康。

**② 巡检报告列表**：展示当前租户项目维度的实例巡检汇总报告，同时也可以点击展开单个实例的详细巡检报告。
#### step2.查看详细实例的巡检报告

![巡检报告](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/InspectionReport.png)

**实例的详细巡检报告主要分为以下几个部分：**
**① 巡检评分**：针对实例的健康状况进行评分，分数100分为满分，表示健康
**② 问题汇总**：汇总该实例巡检的问题总数，并按照风险等级分别统计问题数，比如严重问题数和警告问题数。
**③ 巡检项**：巡检报告涉及每一条巡检项，可针对每条巡检项进行特定分析，并得出巡检结果和修复建议。
### 2）解决致命问题
参数配置问题、资源问题、安全问题、备份问题等常见问题，通过手动或者脚本等方式都比较容易解决，DBdoctor的巡检报告也涵盖了针对常见问题的修复建议，但性能问题这把悬在DBA头顶上的剑比较棘手，需要花费大量的时间去跟踪定位分析，该怎么彻底解决呢?

#### step1.打开Dashboard
可针对性能问题直接列出当前租户项目下存在性能问题的实例，并详细统计了该实例的性能问题数量以及根因SQL。
![性能问题数量](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/NumberOfPerformanceProblems.png)
#### step2.点击实例名跳转到根因分析页面
根因分析可汇总该实例哪几类SQL有性能问题（不用去翻慢日志或者审计日志），同时针对每一类SQL都会有详细的历史执行记录，可回放整个事务SQL的完整执行过程。如果该SQL存在性能问题，DBdoctor将主动识别并基于COST优化器提供最优修复建议，比如推荐索引。

![最优修复建议](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/OptimalRepairRecommendations.png)

针对锁的问题，DBdoctor可以回放整个加锁过程，通过事务泳道图的方式快速还原现场。DBA可通过图例展示，轻松找到锁的形成过程，将其发给相关业务研发同事，在大促及假期节点等流量洪峰前，提前优化一轮。

![提前优化](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/CopeWithTheSpringFestivalTrafficPeak/AdvanceOptimization.png)

### 3）结论

DBdoctor部署起来，不但可以顺滑应对大促流量洪峰，晚上、周末、假期再也不用受告警折磨，减少跨团队故障推诿扯皮，真正实现快乐工作，认真生活！

