# 一个接口未做超时处理，引发数据库hang了

## 前言
在代码开发过程中，你是否会经常遇到以下问题？
- 数据库连接被瞬间占用，出现性能瓶颈

- 系统资源被大量占用，出现锁等待或性能下降
- 事务日志大量增长

上述这些状况的出现可能是**未提交事务**引发的。该类事务开启后，**长时间未向数据库发出SQL执行请求或事务处理(COMMIT/ROLLBACK)请求**，会对数据库的稳定性和性能产生重大影响。本文我们将详细分析未提交事务的原因，并提供有效的排查和解决方案。

## 一个接口未做超时处理，引发数据库hang了

近期，一位开发小伙伴给我们反馈说，他在进行事务处理时，尝试调用了一个第三方接口，且未对这一调用过程增加超时限制。在某天第三方服务突然发生异常，导致接口响应变慢，当前事务迟迟未提交，直接导致产生了大量未提交事务，数据库直接hang住了!

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZzv7Tew2JeR83NTDltaTlHaA3I6B1FupwedIm8UDfeVFv6icTAu525mQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面简化后的代码，我们能看到通过注解@Transactional开启事务，执行update完成后，开始调用第三方接口，这个时候相当于卡住了，后面的代码迟迟得不到执行。而业务针对该接口会频繁操作，相当于数据库会有大量未提交事务，导致大量资源占用、长时间锁占用产生大量锁等待、连接数暴涨、系统性能急剧下降等问题。

对于运维DBA来说，从数据库层面只能看到很多处于Sleep状态的连接且看不到具体SQL，问题排查起来比较困难。

最终DBA通过终极大招**重启数据库**解决了该问题。

## 哪些场景会触发未提交事务？
造成未提交事务的原因有多种可能，以下是几种常见的触发场景：

**1. 存在复杂查询或计算**：有些事务可能需要执行复杂的查询、计算或批量操作。这些操作通常需要较长时间才能完成，在整个过程中，事务保持未提交状态，直到所有操作完成并且事务提交。

**2. 存在长时间运行的批处理任务**：在一些批处理任务中，可能会有大量数据需要处理。为了确保数据处理的原子性和一致性，这些操作通常会在一个事务中进行。在整个批处理任务完成之前，事务会保持未提交状态。

**3. 事务锁等待**：如果事务在执行过程中需要等待其他事务释放锁，它会保持未提交状态，直到锁被释放并且操作可以继续。

**4. 事务中需访问第三方服务**：事务开启后，与第三方交互，严重依赖第三方业务执行的速度，大大增加事务时长 。

## 如何快速排查未提交事务？
当运维DBA遇到此类问题时，会通过SHOW PROCESSLIST或者INNODB_TRX来进行未提交事务分析。下面我们手动模拟创造了一个未提交事务：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZ8FoFJHuics4Wnweq4uWVl4NVhia1uicWtK6XNttibMckKhUTkIouAgwaSQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZl2qH5f8dRjGNdVGiblhsmic1Hco5qj35ZMzkphoXZ2Mq9Z4lnlGGJxoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的图中，通过SHOW PROCESSLIST可查看会话状态，但是看不到会话90323的具体SQL，只能看到一条处于Sleep状态的连接会话，没有详细SQL很难定位是哪个业务逻辑异常导致的。

### 利用DBdoctor进行未提交事务的快速诊断

使用DBdoctor纳管实例后，会对该实例实时主动诊断（包含未提交事务），我们可以在实例诊断->锁透视->未提交事务tab页中进行列表查看，点击指定未提交事务『查看事务详情』，即可通过泳道图的形式慢动作回放事务SQL的完整执行过程。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnruxGibYb4DU619v2PpTGJv99es0bFzMDWT0080Zp8DAWFianCZqP0x9diajTQx7FfX4TGD6JPx2TXA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**未提交事务列表支持锁下砖**，点击上图中会话ID小箭头可以展示该**未提交事务**引发哪些SQL发生了**锁等待**，如果存在锁等待SQL，那说明业务开发同学要对该未提交事务做紧急优化了，对于DBA同学只需要将该未提交事务的会话ID进行kill即可完成紧急救火。

## 总结

偶尔的一条未提交事务在线上可能不会造成业务异常，但如果哪天需要对涉及该事物的表做DDL变更，那么可能引发故障（未提交事务占用了元数据锁，会导致涉及该表的所有SQL被阻塞）。利用DBdoctor锁透视功能，可帮助开发及运维DBA快速‌识别数据库是否存在未提交事务，DBA可以紧急救火，业务开发同学可基于泳道图快速异常代码定位并优化，彻底解决该问题，赶快下载试用吧！