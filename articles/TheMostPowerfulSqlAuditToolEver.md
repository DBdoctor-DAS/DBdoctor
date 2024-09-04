# 史上最强的SQL审核工具，求挑战

## 业界SQL审核现状

SQL审核是指对未上线的SQL进行检测，提前识别是否符合规范，以保证上线质量。Yearning、Archery 、Bytebase、goInception、SQLE 是当前国内主流的开源 SQL 审核工具，均是基于**规则**来进行**SQL审核**，可以检查规范性，但无法**基于线上数据情况**进行**SQL性能评估**，所以**审核后的SQL**上线后依然会存在**性能隐患**。

## 基于规则是否能做SQL性能审核？

每个公司都有自己的SQL规范标准，即不同的规则。规则越多，说明管理越严格。如以下常见的SQL审核规则：

- 建表需要有自增主键
- 禁止使用触发器
- 禁止使用存储过程
- 禁止使用索引套函数
- 查询条件中使用in（...）关键字元素个数不要超过5000

通过这些规则可以将SQL语法层面的问题提前识别出来。大家也许有疑问，基于DBA经验沉淀的审核规则是否可靠？以上面第五条规则为例，以5000为个数限制该规则本身就有问题，实际还是从SQL涉及的表行记录长度、数据量、离散层度、IO页扫描次数等多因素综合考虑，不应该是一个具体经验值。

SQL规则审核能否覆盖性能的审核？我们手写一条规则来验证SQL是否有性能问题：
```SQL
#待审核的SQL如下，pur_jit_item表有自增主键id，隔离级别RR，没有其他索引
update pur_jit_item set qualified_qty=5316 where fatory_code=7 and mat_sap_code=3
```
![性能问题](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/TheMostPowerfulSqlAuditToolEver/PerformanceProblem.png)

当字段少的时候，我们从上图中可以看到SQL规则审核**命中**的**字段未加索引**是存在有**性能问题**的，但如果字段过多、部分字段存在索引、字段区分度不好等场景时，规则该如何写？很多有经验的DBA，试图通过列举所有场景的规则来实现性能识别是不可取的。从实际生产来看，要考虑的因素太多，同样的SQL在不同的数据集上的结果都可能有N种，无法准确命中。

综上所述，我们得出结论，以传统经验规则的方式做SQL性能审核并不可取，亟须一种低消耗快速的方式。

## SQL性能审核灵感
对数据库内核熟悉的应该了解过MySQL、PgSQL的SQL最优执行路径，通过Cost-based Optimization进行评估，然后进行最优路径选择。我们来看下MySQL Cost优化器评估过程。

![审核灵感](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/TheMostPowerfulSqlAuditToolEver/AuditInspiration.png)

##### 1）建立评估标准

MySQL系统表内置默认的代价计算标准，分为Server层CPU代价和存储引擎层IO代价，比如物理临时表disk_temptable_create_cost默认一次代价为20。通过这种详细标准定义就可以将SQL在每个操作上的代价量化出来，得出具体的Cost值，进而为后续的路径选择提供数据支撑。

##### 2）SQL最优路径选择

Session开启OPTIMIZER_TRACE，SQL执行后，Trace日志中会记录该SQL详细的各条索引路径Cost消耗的数值，最终会从所有的路径中选择Cost消耗最低的路径作为best access path（命中的索引）。

综上所述，实现旁路Cost优化器需要解决两个问题，首先仿照MySQL定义一套一样的Cost评估标准，然后梳理出每条索引路径的Cost依赖哪些统计信息并获取。

## 数据库Cost优化器源码解读

数据库最终的路径选择是基于Cost标准计算出来的，外置Cost计算标准和公式即可旁路来计算Cost消耗。下来我们来从MySQL Access Type维度一起拆解Cost优化器的内核计算过程。

![Cost优化器源码解读](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/TheMostPowerfulSqlAuditToolEver/CostOptimizerSourceCodeInterpretation.png)

##### 1）源码解析Cost计算公式
Cost的计算公式可以直接从代码中解读出来，比如以Table scan全表扫描为例，Table scan的Cost分为IO Cost跟CPU Cost两个部分之和，大致的公式为：

```C
IO-Cost:#Pages_In_Table * IO_BLOCK_READ_COST + CPU-Cost:#Records * ROW_EVALUATE_COST
```

其中IO-Cost是通过Table_Scan_Cost来进行计算。这里有两个关键的变量Records跟Pages_In_Table，分别表示这个表的行记录数和页数，在INNODB中这两个变量的值可通过ha_innobase::info_low(ha_innobase::info)和ha_innobase::scan_time()来进行获取，知道了这**两个变量的值**，就知道具体的**Cost值**。

##### 2）如何获取Cost公式中的具体变量值
针对**公式中的变量**，这里**以已存在的索引为例**，MySQL **InnoDB通过分析随机采样的索引叶子页来估算索引的统计信息**（采样20个页，该统计信息可以从系统表中直接获取），通过采样来估算并进行索引路径的选择。这里旁路实现的话可以完全参照MySQL已有的逻辑进行变量值提取。

##### 3）对所有路径进行该SQL的Cost计算
通过计算所有路径的Cost，我们能得出最终Cost消耗最小的路径即最优路径。

综上所述，我们发现外置Cost优化器是可行的，外置采集该SQL表的统计信息，然后仿照MySQL原有的分析评估行为，即可得出该SQL在生产环境实际的Cost值。但这里存在一个问题，**针对不存在的索引如何获取统计信息呢？**

## 基于eBPF的旁路Cost优化器原理

要实现对生产库的SQL性能评估，需要找出这条SQL的所有可能的索引执行路径。基于系统表**对已存在的索引统计信息获取比较简单，但针对生产库上不存在的索引（避免在源库上直接进行DDL等风险较高的重操作）无法直接获取到统计信息**，需要基于该SQL表在线上实际数据进行统计信息提取。这里我们可以考虑通过**eBPF技术去获取关键数据指标。**

在数据倾斜的场景中，每条SQL的where条件字段的范围值不一样，索引执行路径选择的真实Cost会不一样，**获取采样页尤为重要。下面详细介绍SQL性能评估的流程：**

![优化器原理](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/TheMostPowerfulSqlAuditToolEver/OptimizerPrinciple.png)

1. SQL Parse解析SQL表的条件字段、查询字段。

2. 找出涉及减少Cost消耗的所有路径。针对第一步得到的字段，然后进行排列组合，建立出A(n,m)个**候选索引路径**。

3. 模拟MySQL的页采样逻辑，进行详细页分析并进行统计信息提取，还原MySQL生产库上的真实采样统计信息。针对上面Case中的数据倾斜等场景，就可以避免统计信息不准导致生产执行路径和推荐路径不一致的情况。

4. 利用**贪心算法**可计算出每个候选索引的Cost，最后**得出Cost代价最小**的即为当前的最优索引的执行路径。

5. 若该最优索引与之前原有路径Cost消耗相差较大，则说明SQL存在性能问题。

综上所述，**单条SQL是否存在性能问题**可以工程化实现，但在实际生产中还需要关注到**表维度**的全局最优，需要建更少的索引来满足各业务SQL的性能吞吐，减少维护索引导致的开销。

## 表维度全局最优如何实现？
单条SQL的性能评估可以按照上面的方式实现，但在实际生产中由于索引本身的维护是需要消耗资源的，因此索引不是越多越好，需要建立全局最优的索引，按照DBA的经验可采取“去冗余”和“去未使用索引”两种方式来进行优化，但在实际中往往由于担心对业务有性能影响而不敢进行索引的操作，只做加法不做减法。**下面将详细展开如何从表维度求全局最优解：**

![全局最优如何实现](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/TheMostPowerfulSqlAuditToolEver/HowIsGlobalOptimizationImplemented.png)


1. SQLParse解析SQL对应的库表、字段，并进行最优索引的推荐。
2. 结合全量的审计日志分析，统计数据库实例涉及该表的所有的指纹SQL。
3. 分析所有指纹SQL并提取公共高频字段、指纹SQL特有字段等进行分类。
4. 按照字段分类进行索引的排列组合，对每条指纹SQL进行Cost计算并与原Cost进行对比，在不衰减性能的情况下，得出多条最优路径。
5. 基于上一步的多条最优路径，得出全局Cost最优索引路径。
6. 若该索引与之前原有路径Cost消耗相差较大，则说明SQL存在性能问题。
7. 该推荐的索引路径为最佳方案。

综上所述，具有基于外置Cost优化器的全局最优索引推荐方案是可行的，基于该方案在SQL上线前即可实现SQL性能审核，并评估出基于生产数据模型的全局最优索引推荐。