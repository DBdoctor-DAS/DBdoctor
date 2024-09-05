# eBPF实战教程五｜如何使用USDT探针定位MySQL异常访问(含源码)

## 前言

各位小伙伴们，非常感谢你们对我们eBPF专题系列文章的持续关注和热情支持！在之前的文章中，我们深入探讨了如何手写一个uprobe探测用户态程序。许多热心的小伙伴给我们发私信表达了他们对eBPF技术在数据库领域应用的浓厚兴趣，并希望我们能分享更多的相关案例。为了满足大家的期待，本文将带您深入了解用户态探测的另一种强大工具——USDT探针，以及它在数据库优化和监控中的潜在应用。

本文是我们的eBPF专题系列第五篇纯技术分享文章——如何手码eBPF程序探测MySQL5.6 USDT，来实时识别数据库可疑的连接访问来源（user/host）。

## USDT原理介绍

**USDT**（User Statically-Defined Tracing）是动态追踪系统 DTrace 的一部分，允许在用户态应用程序中定义静态探针。这些探针提供了一个强大的调试和性能分析工具，可以在运行时捕获和分析应用程序的行为，而无需修改应用程序代码或重新编译。

- 探针定义

开发人员在代码中插入静态探针点。这些探针点通常是一些宏定义，用于标记需要追踪的代码位置。例如，在 MySQL 中可以看到类似于 #define MYSQL_COMMAND_START(arg0, arg1, arg2, arg3) 这样的探针定义。

- 探针注册

编译应用程序时，这些探针会被注册到探针表中，生成相应的探针元数据。探针在正常运行时是无操作的（noop），不会影响应用程序性能。

- 探针启用

当需要调试或分析时，调试器或追踪工具（如 DTrace、SystemTap 或 BPF）可以附加到这些探针上，并启用它们。当探针被启用时，它们会执行指定的动作，例如记录日志、捕获堆栈跟踪或收集性能数据。

- 捕获数据

当应用程序运行并触发探针时，探针会调用附加到它们的追踪程序，执行指定的调试或分析任务。这些任务可以包括打印变量值、收集性能指标等。

- 数据分析

通过收集的数据，开发人员可以分析应用程序的行为，找出性能瓶颈、调试问题或优化代码。

## MySQL5.6 DTrace探针
MySQL5.6源码probes_mysql_nodtrace.h中定义了大量的DTrace探针，我们从中选取了以下两个探测：

```C
#define  MYSQL_COMMAND_START(arg0, arg1, arg2, arg3)
#define  MYSQL_COMMAND_DONE(arg0)
```

备注：MySQL源码编译指定编译选项DENABLE_DTRACE=1才能开启Dtrace探针。关于使用DTrace跟踪mysqld更多信息可查看该链接文档(https://mysql.net.cn/doc/refman/5.6/en/dba-dtrace-server.html)

MySQL源码调用上述两个Dtrace的位置：

```

bool dispatch_command(enum enum_server_command command, THD *thd,
          char* packet, uint packet_length)
{
  NET *net= &thd->net;
  bool error= 0;
  DBUG_ENTER("dispatch_command");
  DBUG_PRINT("info",("packet: '%*.s'; command: %d", packet_length, packet, command));

  /* SHOW PROFILE instrumentation, begin */
#if defined(ENABLED_PROFILING)
  thd->profiling.start_new_query();
#endif

  /* DTRACE instrumentation, begin，start探针 */
  MYSQL_COMMAND_START(thd->thread_id, command, &thd->security_ctx->priv_user[0], (char *) thd->security_ctx->host_or_ip);

...  
...
  /* DTRACE instrumentation, end，End探针 */
  if (MYSQL_QUERY_DONE_ENABLED() || MYSQL_COMMAND_DONE_ENABLED())
  {
    int res MY_ATTRIBUTE((unused));
    res= (int) thd->is_error();
    if (command == COM_QUERY)
    {
      MYSQL_QUERY_DONE(res);
    }
    MYSQL_COMMAND_DONE(res);  
  }

  /* SHOW PROFILE instrumentation, end */
#if defined(ENABLED_PROFILING)
  thd->profiling.finish_current_query();
#endif

  DBUG_RETURN(error);
}
```
- start探针中四个参数分别为：

- thread_id : MySQL内部分配的线程ID，即MySQL的Connection ID

- command: SQL命令的枚举类型

- priv_user: 连接的用户名

- host_or_ip: 连接的客户端IP

end探针中只存在一个res参数，该参数为会话执行SQL的返回状态，非0表示SQL执行异常。
## eBPF USDT如何实时识别MySQL异常访问来源?

### 1）环境准备

> 准备一台 Linux 机器，安装好g++和bcc

### 2）基于BCC工具实现探测MySQL5.6的Dtrace探针
接下来我们将基于BCC，利用USDT写一个eBPF程序，实时全量采集MySQL的访问来源（User/Host）。

#### a）使用BCC对探针进行探测

```C
#!/usr/bin/python

from bcc import BPF,USDT

# BPF program to attach to the command__start USDT probe
bpf_text = """
#include <uapi/linux/ptrace.h>

int trace_command__start(void *ctx) {
    struct {
        unsigned long conn_id;
        int command;
        char user[48];
        char host[48];
    } args;

    bpf_usdt_readarg(1, ctx, &args.conn_id);
    bpf_usdt_readarg(2, ctx, &args.command);
    bpf_usdt_readarg_p(3, ctx, args.user, sizeof(args.user));
    bpf_usdt_readarg_p(4, ctx, args.host, sizeof(args.host));

    bpf_trace_printk("Command start:");
    bpf_trace_printk("Timestamp: %llu",bpf_ktime_get_ns());
    bpf_trace_printk("ConnectionId= %llu",args.conn_id);
    bpf_trace_printk("Command=%ld",args.command);
    bpf_trace_printk("User= %s",args.user);
    bpf_trace_printk("Host= %s",args.host);
    return 0;
}

int trace_command__done(void *ctx){
    bpf_trace_printk("Command done:");
    bpf_trace_printk("Timestamp: %llu",bpf_ktime_get_ns());
    int res = 0;
    bpf_usdt_readarg(1, ctx, &res);
    bpf_trace_printk("Res= %d",res);
    return 0;
}
"""

parser = argparse.ArgumentParser(description="Attach USDT probes to a running process")
parser.add_argument("pid", type=int, help="The PID of the target process")
args = parser.parse_args()
pid = args.pid
usdt = USDT(pid=pid)
usdt.enable_probe(probe = "command__start", fn_name = "trace_command__start")
usdt.enable_probe(probe = "command__done", fn_name = "trace_command__done")
bpf = BPF(text = bpf_text, usdt_contexts = [usdt])
bpf.trace_print()
```
#### b）效果演示
pid即为需要探测的mysqld的进程号，指定pid执行python invoke_static.py即可开启探测，当该mysqld有SQL执行时，该探针将触发并得到日志打印。如下示例：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqn75BnfySs9jr9Pb8S5Yp9ibJKxxJyBiaadapaTJXWtLvPXjUjQMA3pa0g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

远程执行连接MySQL的命令

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqnECGaYEWVP1mcfZhzfYiafPIJLBWoic2IMS1eEUGofdkd4qyTmCwBmRGA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

打印观测的结果

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqnDSglTMZ0EROYMU7FAfItTxfj1MogkFyGJmBzNiaIfbibD7KVnhpqOPJQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的演示中我们能看到，客户端和MySQL建立连接，显示当前连接的会话id、访问来源（user/host）、SQL Command、SQL执行开始时间、SQL结束时间、SQL是否正常执行完成（Res）等信息。然后我们针对采集上来的数据就可以做分析了：
- 如果存在user和host为非业务网段或者非业务账号，说明存在异常来源访问。
- 如果存SQL执行耗时过长，可能存在可疑账号在抽取数据。

## 总结

借助eBPF技术的强大能力，我们可以利用MySQL的USDT探针来捕获和深入分析与数据库SQL连接相关的操作活动。通过本文的详细阐述，您对否对eBPF技术在数据库性能监控和优化方面的应用有了更深层次的理解和认识？