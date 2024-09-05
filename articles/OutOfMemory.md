## 前言
OOM，在业内当之无愧被称为是Java程序员的噩梦之一。无论是初出茅庐的新手还是经验丰富的专家，都难以避免在开发过程中遭遇这一难题。尤其在业务高峰时段，应用程序因频繁的OOM错误导致服务中断和重启，实在令人头疼。如何快速定位及解决此问题，本文我们将与您详细探讨。

## 什么是OOM？
**OOM**，全称OutOfMemoryError，顾名思义，就是内存不够用了。Java程序运行过程中，JVM（Java虚拟机）为程序分配内存，但有时程序占用的内存超过了JVM分配的最大内存，就会出现OOM错误。

### OOM的常见场景包括：
1. **堆内存溢出**：通常发生在应用程序创建了大量对象，并且这些对象长时间存活 。

2. **方法区内存溢出**：与类的加载和元数据的存储有关，可能由于类加载过多或类加载器泄露造成。

3. **直接内存溢出**：通常与线程的执行和递归调用有关，例如递归调用过深或线程创建过多。

**堆内存溢出最常见**，比如你写了个循环，不小心生成了无数对象，内存爆了，JVM就“罢工”了。到这里有些童鞋可能会问，不是有full GC吗，为什么还会发生OOM？这里简单解释一下，full gc收集的是“垃圾”，即不可达的对象，如果对内存中的对象大部分是可达的，此时又有新的对象需要分配内存空间，恰恰此时可用空间不够就会出现OOM。

## 如何定位OOM问题根因?

定位OOM问题的根因无非就是找到产生大量对象的那段代码，如何找呢？目前最主流的方式就是通过MAT（Memory Analyzer Tool）分析JVM堆转储文件，获取大对象及线程堆栈信息。具体流程如下：

#### 1. 获取堆转储文件
要使用MAT分析OOM问题，首先需要获取JVM在OOM时生成的堆转储文件。可以通过以下几种方式生成：

- JVM启动参数：通过在启动JVM时添加-XX:+HeapDumpOnOutOfMemoryError参数，确保在OOM发生时自动生成堆转储文件。

- 手动触发：在出现OOM问题的系统上，通过jmap命令手动生成堆转储文件。例如：jmap -dump:live,format=b,file=heapdump.hprof 。

#### 2. 使用MAT加载并分析堆转储文件

下载并安装MAT工具后，启动MAT并打开堆转储文件（通常为.hprof格式）。MAT会自动解析文件，并生成内存使用情况的报告。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicLiaZ5HKXI2YLBGGDqufXfnjibCG9n196FY9ftWwuzzyIUGYtZN7S2Fiag/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 3. 分析大对象

点击查看dominator_tree视图，通过这个视图我们可以看到http-nio-8080-exec-6线程的ArrayList几乎占去了96.32%的堆内存，而ArrayList里是大量的mysql查询结果集对象，到这里我们基本可以确认是由于查询数据库时返回的结果集过大导致的OOM。但是具体是哪块查询逻辑的哪个SQL呢？

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicSmXELYRq0iaGMicsOibFUO1lAjx5QiaMasTsGgUntkiaKhEqI5viaUx1nH8g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

可以先看下如下图所示的mysql查询结果集对象的具体内容：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicZ4f4y1K8ESdiakQfolibWFh0kNbGzUsPdPDOccQ3bmtht4Cia476VtFcQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如果通过结果集内容能直接定位到业务逻辑的话，到这里就结束了。如果没有可以继续往下看。

#### 4. 分析线程堆栈

点击查看thread_overview，通过这个视图我们可以看到http-nio-8080-exec-6线程在OOM的时候确实调用了业务代码中的查询逻辑，但这只能说是压死骆驼的最后一根稻草，并不能直接说明它就是根因。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZm2sicsmumryVDksSdCEKuxiczfEwrEDsyJldaRyMbofZcDpZ45Az6eic5AaBLfclqQRMiaSuiaq537sIg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

那么，它的根因到底是什么呢，我们需再结合DBdoctor进行分析。

### 结合DBdoctor快速分析及修复
利用DBdoctor性能洞察、审计日志以及SQL审核功能，我们可更高效的定位根因SQL并解决OOM问题。

在DBdoctor性能洞察页面我们可以看到，在OOM的时间段里有这么一条SQL，这条SQL的执行时间超过10s且网络回包特别大，通过审计日志我们也可以看到，这个SQL的返回行数高达32818160行与Arraylist的size一致，同时这条SQL只会在上一步的业务逻辑中调用到，那么到这里我们就可以确认是业务逻辑中的如下SQL导致了OOM问题。

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicOickxR2mnlzAj3PkLsYthibmhktKgqE89SCwoMu9HaG0VrjnEP76HjkQ/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxic9hC99jyDIiakkdwvbDdEIoaPCicSIqfcxYPoxJ6D6ZlzPfmlY9klf9Cg/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

定位问题根因后，接下来就是对其进行修复。使用DBdoctor的SQL审核功能，无需复杂操作，一键点击，即可快速识别SQL问题并给出修复建议:

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZm2sicsmumryVDksSdCEKuxicTgjicrkyws3NJgOFoiagfgibVzfLSx1tic6lUWbhsrf1cVeezxYzIOlehw/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结
OOM是一个常见的Java应用程序问题，但通过开发阶段的SQL审核及优化，我们就可有效避免及解决此类问题。同时对于慢SQL甚至是全量SQL，DBdoctor可以自动将其抓取出并进行SQL审核，让您的存量SQL不再有后顾之忧！此功能在下周发布的新版本中即将上线，敬请期待！