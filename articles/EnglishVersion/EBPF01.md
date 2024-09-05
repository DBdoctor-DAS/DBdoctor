# eBPF专题一 | 手把手教你用eBPF诊断MySQL(含源码)

## 背景
被称之为“革命性”内核技术的eBPF一直以来都备受关注，而DBdoctor作为一款数据库性能诊断工具，首次将eBPF技术深入应用在了数据库领域，目前已涵盖SQL性能审核、问题SQL一分钟定位、锁分析、审计日志、根因诊断、索引推荐、智能巡检等诸多功能。很多小伙伴对如何利用eBPF技术观测数据库内核有浓厚兴趣，因此我们在【DBdoctor】公众号开设了eBPF技术专栏，将定期与您分享eBPF相关的技术文章，手把手教您使用eBPF探测数据库，欢迎关注我们！

本文是eBPF专题的首篇，将用一个具体的例子介绍如何采用eBPF在MySQL连接校验处加探针，并打出hello world，快来一起体验eBPF的强大吧！
## eBPF 概述

eBPF (extended Berkeley Packet Filter)是一种在Linux内核中执行代码的技术，它允许开发人员在不修改内核代码的情况下运行特定的功能。传统的BPF 只能用于网络过滤，而 eBPF则更加强大和灵活，有更多的应用场景，包括网络监控、安全过滤和性能分析等。uprobe是eBPF的一种使用方式，它允许我们在用户空间的应用程序中插入代码来监控和分析内核中的函数调用。
具体来说，eBPF uprobe可以在用户空间的应用程序中选择一个目标函数，并在该函数执行之前和之后插入自定义的代码逻辑。这样可以实现对目标函数的监控、性能分析、错误检测等功能。通过eBPF uprobe，我们可以在不修改内核源代码的情况下，对内核中的函数进行动态追踪和分析。
## 01. 工作原理
![uprobe](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/EBPF01/uprobe.png)
uprobe是一种用户探针，uprobe探针允许在用户程序中动态插桩，插桩位置包括：函数入口、特定偏移处，以及函数返回处。当我们定义uprobe时，内核会在附加的指令上创建快速断点指令，当程序执行到该指令时，内核将触发事件，程序陷入到内核态，并以回调函数的方式调用探针函数，执行完探针函数再返回到用户态继续执行后序的指令。

uprobe基于文件，当一个二进制文件中的一个函数被跟踪时，所有使用到这个文件的进程都会被插桩，这样就可以在全系统范围内跟踪系统调用。uprobe适用于在用户态去解析一些内核态探针无法解析的流量，例如http2流量（报文header被编码，内核无法解码）、https流量（加密流量，内核无法解密）等。

## 02. eBPF uprobe如何探测MySQL?
### 1）环境准备
> 准备一台 Linux 机器，安装好 Python 和内核开发包。（注：内核开发包版本必须和内核版本一致）
安装带有符号表的MySQL

### 2）基于BCC工具实现探测MySQL

![uprobe](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/EBPF01/bcc.png)
BCC程序使用 Python 编写，它会嵌入一段 c 代码，执行时将 c 代码编译成BPF字节码加载到内核运行。而 Python 代码可以通过 perf event 从内核将数据拷贝到用户空间读取到数据然后展示出来。

接下来我们将基于BCC的uprobe，写一个eBPF程序，观测MySQL上是否存在大量短连接。

#### a）分析MySQL源码相关连接处理的函数
```C
//从MySQL源码中分析函数选用了mysql-server层的连接校验处理函数check_connection
static int check_connection(THD *thd){
  ...
}
```
#### b）导入BCC的BPF对象
```python
#!/usr/bin/python
//这个对象可以将我们的观测代码嵌入到观测点中执行
from bcc import BPF
```
#### c）用c编写观测代码
```Python

bpf_text="""
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>

//定义结构体 data_t 保存我们每次观测到的结果
struct data_t {
    u32 pid;
    u32 tgid;
    u64 ts;
    char info[40];
};

//PF_PERF_OUTPUT 定义了一个叫 events 的表，观测代码可以将观测数据写入到 events 表中
BPF_PERF_OUTPUT(events);

//该自定义函数用来和MySQL观察的内核函数进行绑定关联
int do_check_connection(struct pt_regs *ctx) {
    //获取了 MySQL 进程对应的结构体 task_struct，然后从获取了其中的 线程pid 和 用户空间进程tgid
    struct task_struct *t = (struct task_struct *)bpf_get_current_task();
    
    //初始化data用来保存观测结果
    struct data_t data = {};
    
    // create a new connection
    char a[] = "hello world,create a new connection";
    bpf_probe_read_kernel_str(&data.info,sizeof(data.info),a);
    
    // process id
    bpf_probe_read(&data.pid,sizeof(data.pid),&t->pid);
    
    // thread id
    bpf_probe_read(&data.tgid,sizeof(data.pid),&t->tgid);
    
    // bpf_ktime_get_ns returns u64 number of nanoseconds. Starts at system boot time but stops during suspend.
    data.ts = bpf_ktime_get_ns();
    
    //将观测的数据提交到表中
    events.perf_submit(ctx, &data, sizeof(data));

    return 0;
}"""
```
#### d）观测代码关联MySQL 中需要观测的函数
```Python
# initialize BPF
//自定义的ebpf程序
b = BPF(text=bpf_text)
//将ebpf观察代码中的自定义函数和MySQL的内核函数进行绑定（name为mysqld进程路径，sym为编译后的探测函数名，fn_name为绑定的ebpf自定义函数）
b.attach_uprobe(name="/home/mysqld", sym="_ZL16check_connectionP3THD",fn_name='do_check_connection')
```
#### e） 打印观测结果
```Python

# output trace result.
print("Starting to Trace MySQL server do_check_connection function")
print("------------------------------------------------------------")

//定义打印的回调函数
def print_event(cpu, data, size):
    event = b["events"].event(data)
    decoded_string = event.info.decode('utf-8')
    print("%-35s, process id %-6s, thread id %-6s, sytem uptime(s) %-14s" % (decoded_string,event.tgid, event.pid,event.ts/1000000000))

//open_perf_buffer 将打印的回调函数注册到 events 表中
b["events"].open_perf_buffer(print_event)
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```
#### f）效果演示
执行该eBPF程序

![eBPF程序](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/EBPF01/EbpfProgram.png)

分别开两个窗口执行连接MySQL的命令

![MySQL命令](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/EBPF01/MysqlCommand.png)

打印观测的结果

![观测的结果](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/images/EBPF01/ObservedResult.png)

从上面的演示中我们能看到，客户端和MySQL建立连接的时候会打印日志，显示这个连接的校验时间、线程id、进程id。如果存在大量日志输出，说明数据库上一直在创建新连接（即短连接），再进行下一步分析是哪个应用程序导致的。
## 03. eBPF在程序开发过程中有哪些限制？

eBPF在应用和开发过程中会出现多种问题，下面就跟大家分享一下eBPF在程序开发过程中可能会遇到的限制：

#### 1）栈大小512字节

eBPF单个探测函数对栈大小限制为512字节，使用BPF_PERF_OUTPUT将数据从内核态输出到用户态时，会直接限制单个数据结构的大小定义不能超过512字节。

#### 2）多参数获取

BCC中的宏定义从PT_REGS_PARM1(x)~PT_REGS_PARM5(x)，栈传递的参数，是从右边向左压栈，想要获取探测函数的入参可以通过PT_REGS_PARM来获取，但X86寄存器的个数是有限的，BCC的定义是能取到5个参数，对于超过6个参数的不能直接获取。

#### 3）循环遍历

Kernel 内核版本低于4.15，不支持任何循环。

#### 4）复杂数据结构的解析

对于参数中复杂数据结构进行解析，由于eBPF程序无法直接引用用户态的头文件，且无法直接在eBPF代码中定义C++的类，对于复杂的数据结构获取其成员变量不能直接获取。

例：
```C

class THD: public MDL_context_owner,
           public Query_area,
           public Open_tables_state
 {
    public:
    MDL_context mdl_context;
    enum enum_mark_columns mark_used_columns;
    unlong wat_privilege;
    LEX *lex;
    bool gtid_executed_warning_issued;
    ...
    private:
    ...
    LEX_CSTRING m_catalog;
 }
 ```
 该数据结构较为复杂，在使用eBPF获取该数据结构的m_catalog时，显然是不能在内核中定义相同的数据结构来进行转换。
 #### 5）uretprobe获取入参值
 在uretprobe触发时，寄存器中只能确保rax有效保留返回值，无法直接获取函数入参的值。

## 总结

利用eBPF技术探测MySQL ,具有更高效，更扩展，更安全等优势，不用修改内核就可观测数据库性能。通过上面例子您是否发现采用eBPF跟踪数据库其实并不难，主要的门槛在于精通数据库内核和Linux编程，而且要对代码有精益求精的意识。您的hello world出现了吗？欢迎进群跟我们探讨！