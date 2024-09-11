# eBPF Practical Tutorial 5 | How to use USDT probes to locate MySQL abnormal access (including source code)

## Preface

Dear friends, thank you very much for your continued attention and enthusiastic support for our eBPF series of articles! In previous articles, we delved into how to handwrite a uprobe detection User Mode program. Many enthusiastic friends have sent us private messages expressing their strong interest in the application of eBPF technology in the database field and hoping that we can share more related cases. In order to meet everyone's expectations, this article will take you through another powerful tool for User Mode detection - the USDT probe, and its potential applications in database optimization and monitoring.

This article is the fifth pure technical sharing article in our eBPF series - how to manually code eBPF programs to detect MySQL 5.6 USDT and identify suspicious connection referrers (user/host) in the database in real time.

## Introduction to USDT Principle

**USDT**USDT (User Statistics-Defined Tracing) is a part of the dynamic tracing system DTrace that allows static probes to be defined in User Mode applications. These probes provide a powerful debugging and performance analysis tool that captures and analyzes application behavior at runtime without modifying application code or recompiling.

- Probe definition

Developers insert static probe points into the code. These probe points are usually macro definitions used to mark the location of the code that needs to be traced. For example, in MySQL, you can see probe definitions like #define MYSQL_COMMAND_START (arg0, arg1, arg2, arg3).

- Probe registration

When compiling an application, these probes are registered in the probe table and corresponding probe metadata is generated. Probes are noop during normal operation and do not affect application performance.

- Probe enabled

When debugging or analysis is needed, debuggers or tracing tools (such as DTrace, SystemTap, or BPF) can be attached to these probes and enabled. When the probes are enabled, they will perform specified actions, such as logging, capturing stack traces, or collecting performance data.

- Capture data

When the application runs and triggers the probe, the probe will call the tracer attached to them to perform the specified debugging or analysis tasks. These tasks can include printing variable values, collecting performance metrics, etc.

- Data Analysis

By collecting data, developers can analyze the behavior of the application, identify performance bottlenecks, debug issues, or optimize code.

## MySQL 5.6 DTrace Probe
MySQL 5.6 source code probes_mysql_nodtrace defines a large number of DTrace probes, we have selected the following two probes:

```C
#define  MYSQL_COMMAND_START(arg0, arg1, arg2, arg3)
#define  MYSQL_COMMAND_DONE(arg0)
```

Note: MySQL source code compile specifies the compile option DENABLE_DTRACE = 1 to enable Dtrace probes. More information about using DTrace to trace mysqld can be found in the linked document(https://mysql.net.cn/doc/refman/5.6/en/dba-dtrace-server.html)

MySQL source code calls the above two Dtrace locations:

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

  /* DTRACE instrumentation, begin, start probe */
  MYSQL_COMMAND_START(thd->thread_id, command, &thd->security_ctx->priv_user[0], (char *) thd->security_ctx->host_or_ip);

...  
...
  /* DTRACE instrumentation, end，End probe */
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
- The four parameters in the start probe are:

- thread_id: MySQL internally assigned thread ID, i.e. MySQL Connection ID

- command: Enumeration type of SQL command

- priv_user: Connected username

- host_or_ip: Connected Client IP

There is only one res parameter in the end probe, which is the return status of the session execution SQL. Non-zero indicates SQL execution exception.
## How to identify MySQL exception referrers in real time with eBPF USDT?

### 1）Environment preparation

> Prepare a Linux machine and install g ++ and bcc

### 2） Implement Dtrace probe for MySQL 5.6 based on BCC tool
Next, we will write an eBPF program based on BCC and use USDT to collect all MySQL referrers (User/Host) in real time.
#### a）Probe the probe with BCC

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
#### b） effect demonstration
Pid is the process number of the mysqld that needs to be probed. Specify pid to execute python invoke_static.py to start probing. When the mysqld has SQL execution, the probe will trigger and get log printing. The following example:

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqn75BnfySs9jr9Pb8S5Yp9ibJKxxJyBiaadapaTJXWtLvPXjUjQMA3pa0g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

远程执行连接MySQL的命令

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqnECGaYEWVP1mcfZhzfYiafPIJLBWoic2IMS1eEUGofdkd4qyTmCwBmRGA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

打印观测的结果

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZkkeooXBBmnISdzWejIwWqnDSglTMZ0EROYMU7FAfItTxfj1MogkFyGJmBzNiaIfbibD7KVnhpqOPJQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the above demonstration, we can see that the Client and MySQL establish a connection, displaying the session id, referrer (user/host), SQL Command, SQL execution start time, SQL end time, and whether the SQL is completed normally (Res). Then we can analyze the collected data:

- If the user and host are non-business network segments or non-business accounts, it means that there is abnormal source access.
- If the SQL execution takes too long, there may be suspicious accounts extracting data.

## summary

With the power of eBPF technology, we can take advantage of MySQL's USDT probe to capture and deeply analyze operational activities related to database SQL connections. Through the detailed explanation of this article, have you gained a deeper understanding and awareness of the application of eBPF technology in database performance monitoring and optimization?