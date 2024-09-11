# DBA essential! How to use DBdoctor for index recommendation

Recently, some users, after installing DBdoctor and completing instance management, often see relevant information about index recommendation on the DBdoctor overview page or instance performance insight page. They are concerned about the source of this information, the triggering scenarios of index recommendation, and the implementation process. They also want to know if there are other scenarios that can trigger index recommendation.

## Index Recommendation Process

Index recommendation triggering scenarios are mainly divided into the following categories:

IO exception root cause SQL index recommendation, CPU exception root cause SQL index recommendation, slow query SQL index recommendation, user manual input index recommendation.

DBdoctor will monitor these events and make relevant index recommendations. The detailed process is as follows:

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZj9AZtRXsrriaicfSc9tCBVd7sQ5SACKukUPrVmX0JdkDjOIe4TMDSKX1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. For each index recommendation scenario, DBdoctor collects these tasks and creates scheduling tasks, and then performs concurrent index analysis, allowing users to see the results at the fastest speed.

2. Use SqlParse to parse and analyze SQL, and obtain relevant tables and columns used in SQL.

3. Collect the real statistical information in the original instance based on eBPF, and obtain the Cost of each index combination through the external Cost optimizer.

4. Choose the index with the smallest cost (the index that does not exist is the best recommendation).

## Which scenarios will trigger index recommendation?

**Scenario 1**： When our business code development is completed, for some special SQL, we may worry about whether it will cause SQL performance problems, whether the current index can HOLD in performance, then we can verify the index through DBdoctor SQL audit function [menu path: Instance Diagnostics - > SQL Audit].

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjd4Lmv4K4JOJicBqjQA8OwWofgq9hXTlYK7bsSZ3z83icgOlKicqhKBFow/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

By using the SQL audit function, we can manually input the SQL that needs to be analyzed, click submit for audit, and after Good to go, we can see on the details page which index the current SQL is running at, how many rows are scanned, whether it is the optimal index, and the optimal index recommended by DBdoctor under the database.

**Scenario 2**：When our service is already online, we can observe whether there is slow SQL online through DBdoctor, and whether it is necessary to optimize the index of the slow SQL. At this time, we can find the solution under DBdoctor [Instance Diagnosis - > Performance Insights - > SQL Association Analysis].

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjQxK3hIcUclZjLwsrf2m9u80mPulicXpG5emBpZO9KMMBlgpvqmIXPcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

As shown in the above figure, we found that there is a delete slow SQL online. By clicking on the execution plan, we can check that the current execution is a full table scan and no index has been added. At this time, the system will recommend the slow SQL index. When we enter the page next time, we can see the latest index recommendation information. DBdoctor recommends adding an index to the start_time, and the performance can be directly improved by tens of thousands of times!

**Scenario 3**：When the online service abnormal alarm, slow service response and other abnormal scenarios, we can also use DBdoctor to determine whether the service performance problem caused by SQL failure to add the optimal index, then we can find the corresponding solution through [Instance Diagnostics - > Performance Insights].

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjb3xoOia6RT9BsrJy6Tuia9eibKFK0dLQy7AjTcoLIYu5DGmib3pmN2lsYQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Through the above figure, we can see that there is a jitter point in the CPU. This jitter point can be directly analyzed by the DBdoctor algorithm diagnostic model to determine the root cause of the SQL. This problem is caused by a common query SQL. By querying the database, we can see that the fields used by the current query conditions do not have an index added, resulting in a large amount of CPU consumption exception. DBdoctor can accurately capture this exception point and successfully recommend the index. Through cost value comparison, the index optimization performance is greatly improved.

**Scenario 4**：As an operation and maintenance or DBA, especially during holidays, database inspections are essential. Can we view all instances through a global perspective for index recommendations and summarize which SQL needs to be optimized? The answer is yes. On the [Dashboard Overview] page, we can see all instances under the current tenant project, which SQL needs to be optimized, and give optimization suggestions.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZj0xTZtbPjDtnoLFhXJMuTiavMuoVeUZZh85FZPia70xHyvNJTaTdJgWtQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Of course, we can also check which SQL statements in the current instance need to be optimized and see optimization suggestions on the "Performance Diagnosis - > Index Recommendation" page. This page also summarizes DDL aggregation by database & table dimension, supports index details viewing, and facilitates DBAs to quickly and low-cost execute DDL changes to indexes.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjamIqtE3iaHuu4wC8reMPqGDTiboxrGOibCN5Rp9eV5yDHSDeJ3LjRNhqQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## Summary

DBdoctor's index recommendation function includes SQL review and recommendation before code release, identifying the main root cause SQL during the process and providing recommendations, as well as emergency handling afterwards and providing optimization suggestions. Through our introduction, you should have a further understanding of the possible scenarios and overall operation process of index recommendation. Now use DBdoctor to see if there is room for optimization and improvement in your SQL!