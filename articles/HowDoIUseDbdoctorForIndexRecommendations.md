# DBA必备！如何使用DBdoctor进行索引推荐

近期，一些用户在安装DBdoctor并完成实例纳管后，常在DBdoctor概览页面或实例性能洞察页面看到索引推荐的相关信息，他们对这些信息的来源、索引推荐的触发场景以及实现流程等比较关注，也想了解是否存在其他能够触发索引推荐的场景。

## 索引推荐流程

索引推荐触发场景主要分为以下几类：

IO异常根因SQL索引推荐、CPU异常根因SQL索引推荐、慢查询SQL索引推荐、用户手动输入索引推荐。

DBdoctor会监听这些事件，并进行相关索引推荐，详细流程如下：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZj9AZtRXsrriaicfSc9tCBVd7sQ5SACKukUPrVmX0JdkDjOIe4TMDSKX1Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 针对各个索引推荐场景，DBdoctor将这些任务收集后创建调度任务，进而并发索引分析，可用最快的速度让用户看到结果。

2. 通过SqlParse对SQL进行解析分析，获取SQL中用到的相关表和列等信息。

3. 基于eBPF收集原实例中真实统计信息，并通过外置的Cost优化器得出各个索引组合的Cost。

4. 选择最小的Cost的索引（不存在该索引即最优推荐）。

## 哪些场景会触发索引推荐？

**场景一**： 当我们业务代码开发完成，针对一些特殊的SQL，可能会担心是否会导致SQL性能问题，当前索引在性能上是否能够HOLD住，这时我们可以通过DBdoctor SQL审核功能进行索引验证【菜单路径：实例诊断->SQL审核】。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjd4Lmv4K4JOJicBqjQA8OwWofgq9hXTlYK7bsSZ3z83icgOlKicqhKBFow/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过SQL审核功能，我们可以手动输入需要分析的SQL，点击提交审核，审核成功后，我们可以在详情页面看到数据库下，当前SQL执行时走的哪个索引，扫描多少行，是否为最优索引，以及DBdoctor推荐的最优索引。

**场景二**：当我们的服务已经上线，可以通过DBdoctor观察线上是否存在慢SQL，以及该慢SQL是否有必要进行索引优化，这时我们可以在DBdoctor 【实例诊断->性能洞察->SQL关联分析】下找到解决方案。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjQxK3hIcUclZjLwsrf2m9u80mPulicXpG5emBpZO9KMMBlgpvqmIXPcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，我们发现线上存在delete慢SQL，我们通过点击执行计划，可以查看当前执行的是全表扫描且并未添加索引，此时系统会将该慢SQL进行索引推荐，待我们下次进入该页面时，即可看到最新的索引推荐信息。DBdoctor推荐将start_time添加索引，性能可直接提升上万倍！

**场景三**：当线上服务异常告警，服务响应变慢等异常场景下，我们同样可以通过DBdoctor来判断是否因SQL未能添加最优索引而导致的服务性能问题，这时我们可以通过【实例诊断->性能洞察】来找到对应解决方案。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjb3xoOia6RT9BsrJy6Tuia9eibKFK0dLQy7AjTcoLIYu5DGmib3pmN2lsYQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过上图，我们可以看到CPU存在一个抖动点，这个抖动点通过DBdoctor算法诊断模型可直接分析出根因SQL。该问题是由一条普通的查询SQL引起的，通过查询数据库，我们看到当前查询条件使用的字段并未增加索引，导致大量消耗CPU异常，DBdoctor可以准确抓取该异常点，并成功推荐索引，通过Cost值对比，索引优化性能提升巨大。

**场景四**：作为运维或者DBA，特别是在节假日，数据库巡检必不可少，那针对索引推荐是否能通过全局视角查看所有实例，并汇总统计出哪些SQL需要进行索引优化？答案是肯定的，在【Dashboard概览】页面我们能够看到当前租户项目下的所有实例，存在哪几条SQL需要优化，并给出优化建议。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZj0xTZtbPjDtnoLFhXJMuTiavMuoVeUZZh85FZPia70xHyvNJTaTdJgWtQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

当然，我们也可以在实例【性能诊断->索引推荐】页面查看当前实例有哪几条SQL需要优化，并查看优化建议。该页面也汇总了按库表维度的DDL聚合，支持索引详情查看，方便DBA快速低消耗执行DDL变更索引。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkUbWibH3eNaWQF1miaJIrPZjamIqtE3iaHuu4wC8reMPqGDTiboxrGOibCN5Rp9eV5yDHSDeJ3LjRNhqQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

DBdoctor的索引推荐功能包括在代码上线前进行SQL审核和推荐，事中识别主要根因SQL并提供推荐，以及事后的紧急处理并给出优化建议。通过我们的介绍，您应该对索引推荐可能出现的场景以及整体的运作流程有了进一步了解。现在就使用DBdoctor，看看您的SQL是否有优化提升的空间吧！