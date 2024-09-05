# MySQL加个索引都可能丢数据，这个坑你知道吗？

## 前言

近期，我们收到一位数据库运维小伙伴的咨询，他们有一个MySQL 5.6的数据库，需要**对核心支付表做DDL加索引**，咨询我们如何加索引更优雅。基于DBA经验，给表添加索引主要有以下几种方式：

- 用MySQL原生的DDL语句（包括OnlineDDL）

- 用pt-osc，新建临时表+触发器来实现

- 用gh-ost，新建临时表+基于binlog来实现

Session级别关闭Binlog，备库先加索引，然后做主备切换
具体选择那种方式需要综合考虑表结构、主备延时、业务写入负载、磁盘IO、是否影响业务、使用习惯等多方面因素。经过我们综合评估该支付场景使用原生的OnlineDDL更佳，但该小伙伴反馈他们的运维流程要求DDL变更必须使用pt-osc，迫于流程最终还是使用pt-osc添加索引。然而接下来运维小伙伴的操作差点让公司发生巨大损失，幸好使用了DBdoctor性能洞察功能提前发现问题并及时止损。

## pt-osc原理，有坑吗?
### 1）pt-osc原理

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34Isya01ic73LhmcElpGzptQr8dAytp6lXgqaq1Xib4QuYEj4Je0utTsw7lA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

pt-osc的大致方式是通过创建一个临时表，然后按照主键chunk分批方式，拷贝源表中数据，同时通过三个写相关触发器来控制增量数据实时写入到临时表中，达到与源表最终的数据一致，最终rename交换。从原理上来看，实现逻辑比较简单。那么该原理对业务有损吗？

### 2）运维评估pt-osc是否符合要求
在从pt-osc的实现，我们能直接看出有以下问题：
- 创建临时表，会导致空间翻倍，需要预留足够空间
- 触发器的存在，会导致写入翻倍，需要确保磁盘IO能支撑
- 写入增加会导致主备存在延时
- 存在死锁导致业务事务回滚
经过运维同学人员的详细评估，基于当前业务量，前三点不会有问题，第四点当前MySQL的innodb_autoinc_lock_mode参数为2，不会产生自增锁导致的死锁问题。所以运维同学评估是可以直接在线上通过pt-osc来添加索引。

### 3）线上变更，踩坑死锁了

在线上进行变更过程中，发现业务发生了死锁，下面是业务的详细死锁日志：

```Bash
LATEST DETECTED DEADLOCK
-----------------------
2024-06-26 02:25:20 7fe147aa4700
*** (1) TRANSACTION:
TRANSACTION 2653761553, ACTIVE 0 sec inserting
mysql tables in use 2, locked 2
LOCK WAIT 7 lock struct(s), heap size 1184, 5 row lock(s), undo log entries 3
MySQL thread id 143920619, OS thread handle 0x7fe146cad700, query id 7056859581 update
REPLACE INTO `pay`.`_pay_info_new` (`pay_info_id`, `at_date`, `at_progress`, `ag_no`, `bs_status`, `buyer_id`, `contract_code`, `contract_id`...
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 169 page no 507288 n bits 288 index `
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** (2) TRANSACTION:
TRANSACTION 2653761554, ACTIVE 0 sec inserting
mysql tables in use 2, locked 2
6 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 3
MySQL thread id 144763335, OS thread handle 0x7fe147aa4700, query id 7056859583 update
REPLACE INTO `pay`.`_pay_info_new` (`pay_info_id`, `at_date`, `at_progress`, `ag_no`, `bs_status`, `buyer_id`, `contract_code`, `contract_id`...
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 169 page no 507288 n bits 288 index `pay_info_id` of table `pay`.`_pay_info_new` trx id 2653761554 lock_mode X locks rec but not gap
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 169 page no 507288 n bits 288 index `pay_info_id` of table `pay`.`_pay_info_new` trx id 2653761554 lock_mode X waiting
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** WE ROLL BACK TRANSACTION (2)
------------
```

从上面的日志中我们能看到trigger对应的临时表操作发生了回滚，相当于正常业务针对源表的操作也回滚了，影响到业务。

## pt-osc紧急回滚，会丢数据？

### 1）紧急回滚

运维紧急进行回滚，按下面步骤进行操作：

- 首先kill掉pt-osc的脚本进程

- 删除线上pt-osc的三个触发器

最终，运维确认已经回滚完成，待第二天分析完原因再变更。

### 2）发现大坑，pt-osc没处理干净

业务开发同学使用DBdoctor的性能洞察进行问题分析时，发现业务当前还有临时表的insert语句，认为是运维又开始进行加索引变更了，找运维确认。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34IsC81fAhVWOiaN9ibsx8PJ5HYJa7nxfPiad9duibNfm8H4RicQK2YYcZCejgQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

运维反馈变更已经停止，经过执行检查发现kill pt-osc脚本的时候只kill了shell脚本的进程，子进程没有kill，运维同学再次进行了kill，经过无触发器状态跑了8个多小时，pt-osc进程最终停止了。

### 3）如果pt-osc在无触发器状态，进程最终完成了，会丢数据吗？

从上面pt-osc的原理上来看，全量chunk拷贝都是按照主键递增的方式去做chunk切分并处理，不会对已经处理过的chunk再拷贝。所以会有以下两种问题：

- 如果该变更的表只有按照主键自增id进行写入数据，那么最终全量拷贝的最后一个chunk就是最新的数据，能保证数据一致，**不会丢数据**。

- 对已经全量拷贝后的chunk再发生数据变更，由于触发器没有了，相当于增量丢失，insert/delete/update变更的**数据存在丢失**。

这么大的坑，庆幸pt-osc的进程没有执行完。经过检查，8个小时内业务发生了**9千多条支付数据更新**，如果不是DBdoctor发现，将有9千多条支付数据丢失，对公司来说简直要命。

pt-osc不是万能药，大家做变更一定要仔细，稍微不慎，可能引发很严重的故障。

#### 4）最终原生的OnlineDDL变更完成

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34IsGIzMBnZicUmZyq4GonMVxDE2uNSbT5eKusybsv7DPFYiaPqdXfKxyPWw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最终，该运维小伙伴接受了我们的建议，使用原生OnlineDDL进行**核心支付表OnlineDDL变更**，最终变更成功。从上图中可以看到实际变更的详细进展情况，对变更可回溯可追踪。

## 总结

各位研发或者DBA小伙伴们，你们是否也会经常遇到类似数据库DDL变更导致的故障吗？可以用DBdoctor进行数据库性能评估是否能做DDL，执行DDL的过程全程可见，大家可以试试~

