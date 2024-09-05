# eBPF实战教程二｜数据库网络流量最精准的量化方法(含源码)

## 前言
自从DBdoctor率先将eBPF技术深度应用于数据库领域后，便迅速在业界引起了广泛的关注和讨论。阿里、美团、京东、字节跳动等众多头部企业纷纷主动与我们展开了深入的技术交流。与此同时，我们也收获了大量eBPF爱好者的关注，在第一篇关于uprobe的文章发布后，许多小伙伴都已按教程成功跑通代码并进行深入自学应用。

应广大读者的热情催促，现推出第二篇eBPF的纯技术分享文章——如何手码一个Kprobe函数来分析MySQL数据库的网络流量。旨在为大家提供更多关于eBPF的深入分析和实用指南，希望本文能对大家有所帮助，后续我们也将持续在此专题内发布更多技术文章，欢迎关注公众号！

## 什么是Kprobe
Kprobe是Linux内核提供的一种动态跟踪技术，它可以在运行时动态地在函数的开头、返回点或指令地址处插入探测点。利用kprobe技术，可以在内核函数中动态插入探测点，收集有关内核执行流程、寄存器状态、全局数据结构等详细信息，无需重新编译或修改内核代码，实现对函数的监控和分析。Kprobe极大地增强了内核调试和性能分析的灵活性，可应用于网络优化、安全控制、性能监控、故障诊断等场景，使得开发者能够更深入地理解内核的行为。特别是数据库性能诊断这块，Probe重新定义数据库可观测，可以快速准确找出潜在性能问题并优化。
## Kprobe函数的选取

### 1）网络协议栈解析，获取MySQL SQL执行返回给客户端的函数
基于Kprobe的流量探测，需要对网络协议栈进行分析，下图是网络数据包的发送过程：
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyearsEZVSyMtCgZib6VFIwEQsiaevbHLFsoTU4pKeibjVqgekKKOl7M6faA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的协议栈的函数调用可以看到，tcp_sendmsg函数是发送包的入口函数：

```C
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size)
{
  int ret;
  
  lock_sock(sk);
  ret = tcp_sendmsg_locked(sk, msg, size);
  release_sock(sk);

  return ret;
}
```

从函数的源码我们可以得知，该函数返回值即为我们需要的结果（即发送数据包的大小）。

因此，我们可以选取传输层中的tcp_sendmsg函数作为探测点，来统计数据库发送应用端每秒的数据包总量。

### 2）网络协议栈解析，找到MySQL从客户端接收数据包的函数
基于Kprobe的流量探测，需要对网络协议栈进行分析，下图是网络数据包的接收过程：
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyeouGrIO8kwDLWgtkRZ8NXffIpyb10jaNoOCHGGTPvibCxWtwCXBeGfoQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

对于统计TCP接收的网络流量，应该选择tcp_cleanup_rbuf函数，而不是选择tcp_recvmsg。选用tcp_recvmsg函数会存在统计的重复和遗漏：

- tcp_recvmsg()是一个在TCP接收路径上较高层的函数，它负责从TCP层向用户空间复制数据。当应用程序调用如recv()或read()这类函数来从TCP缓冲区读取数据时，tcp_recvmsg()会被触发。如果数据在TCP接收缓冲区中未被应用程序完全读取（例如，应用程序两次调用recv()读取同一数据段的不同部分），每次调用tcp_recvmsg()都会被触发。这可能导致在统计时同一数据被计算多次。

- TCP数据可能由于内核的优化处理（如紧急数据处理、某些安全检查导致的数据丢弃）而未达到tcp_recvmsg()层会导致统计的遗漏。

- 使用某些直接输入输出操作（如splice系统调用）可以绕过常规的recvmsg路径，直接从内核缓冲区向用户空间或其他文件描述符传输数据，这些操作不会触发tcp_recvmsg()。
tcp_cleanup_rbuf 这个函数在TCP数据确认已被接收（即数据已经从内核传输到了用户空间，并得到了处理）后调用，因此可以更可靠地统计到实际被应用消费的数据量，而不会重复也不会遗漏。

该函数原型如下：
```C
void tcp_cleanup_rbuf(struct sock *sk, int copied)
{
  struct sk_buff *skb = skb_peek(&sk->sk_receive_queue);
  struct tcp_sock *tp = tcp_sk(sk);

  WARN(skb && !before(tp->copied_seq, TCP_SKB_CB(skb)->end_seq),
       "cleanup rbuf bug: copied %X seq %X rcvnxt %X\n",
       tp->copied_seq, TCP_SKB_CB(skb)->end_seq, tp->rcv_nxt);
  __tcp_cleanup_rbuf(sk, copied);
}
```

函数中第二个形参copied即为接收的数据包大小。

因此，我们可以选取传输层中的tcp_cleanup_rbuf函数作为探测点，来统计数据库每秒从应用端接收的数据包总量。

## eBPF kprobe如何探测MySQL的每秒发送和接收数据包?
### 1）环境准备
> 准备一台 Linux 机器，安装好g++和bcc

### 2）基于BCC工具实现探测MySQL
要实现包量的统计，我们首先定义一个存储结构用来存放进程的收发包的总Size，基于Kprobe分别对接收包和发送包进行累加并存储到该结构中，然后每秒去读并打印当前存储结构中累加的数据包量，即可实现每秒的接收和发送数据包的采集。

接下来我们将基于BCC，利用Kprobe写一个eBPF程序，观测MySQL的接收和发送的数据包（即MySQL的NetIO统计）。
#### a）分析内核网络协议栈源码相关网络数据包处理的函数

```C
//内核网络协议栈的发送包函数（返回值）
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size){
  ...
}
//内核网络协议栈的接收包函数（入参）
void tcp_cleanup_rbuf(struct sock *sk, int copied){
  ...
}
```
#### b）导入BCC的BPF对象

```C
//这个对象可以将我们的观测代码嵌入到观测点中执行
#include <bcc/BPF.h>

#include <string>
#include <iostream>
#include <thread>
#include <time.h>
```
#### c）用c编写eBPF代码
```C

std::string strBPF = R"(
#include <linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>

#include <linux/in6.h>
#include <linux/net.h>
#include <linux/socket.h>
#include <net/inet_sock.h>

//定义采集的指标存储结构key
struct key_t {
    u32 pid;
    u16 type;
};

//定义采集的指标存储结构value
BPF_HASH(net_map, struct key_t,u64);

//获取mysql执行sql返回的数据包，hook对返回值进行处理
int kretprobe__tcp_sendmsg(struct pt_regs *ctx)
{
    /*获取当前进程的pid*/
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    if(FILTER_PID) return 0;
    int size = PT_REGS_RC(ctx);

    if(size <= 0)
        return 0;

    struct key_t key= {};
    key.pid = pid;
    key.type = 1;
    u64 zero = 0;
    u64 *val = net_map.lookup_or_init(&key, &zero);
    zero = *val + size;

    net_map.update(&key, &zero);
    return 0;
}


//获取发送给mysql的数据包，hook对函数入参进行处理
int kprobe__tcp_cleanup_rbuf(struct pt_regs *ctx, struct sock *sk, int copied)
{
    /*获取当前进程的pid*/
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    if(FILTER_PID) return 0;

    /*检错*/
    if (copied <= 0) return 0;
    struct key_t key = {};
    key.pid = pid;
    key.type = 2;
    u64 zero = 0;
    u64 *val = net_map.lookup_or_init(&key, &zero);
    zero = *val + copied;

    net_map.update(&key, &zero);
    return 0;
}

)";
```
#### d）观测代码关联网络协议栈中需要观测的函数
```C

//用于ebpf代码程序中的pid替换
static std::string str_replace(std::string r, const std::string& s, const std::string& n)
{
        std::string y = std::move(r);
        std::string::size_type pos = 0;
        while((pos = y.find(s)) != std::string::npos)  
            y.replace(pos, s.length(), n);
        return y;
}

struct net_key_t {
    uint32_t pid;
    uint16_t type;
};

//指定进程pid进行kprobe包统计
int main(int argc, char* argv[]) {
    int pid = std::stoull(argv[1]);
    ebpf::BPF bpf;
    
    std::string strFilerPid = "pid != " + std::to_string(pid);
    std::string code = str_replace(strBPF, "FILTER_PID", strFilerPid);
    auto initRes = bpf.init(code);
    if (!initRes.ok()) {
        std::cerr << "bpf init error,msg: " << initRes.msg() << std::endl;
        return 1;
    }
    /*探测tcp_sendmsg*/
    auto attachRes = bpf.attach_kprobe("tcp_sendmsg", "kretprobe__tcp_sendmsg",0,BPF_PROBE_RETURN);
    if(!attachRes.ok()) {
        std::cerr << "attach tcp_sendmsg error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
     /*探测tcp_cleanup_rbuf*/
    attachRes = bpf.attach_kprobe("tcp_cleanup_rbuf", "kprobe__tcp_cleanup_rbuf");
    if(!attachRes.ok()) {
        std::cerr << "attach tcp_cleanup_rbuf error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    /*每秒完成一次读取并打印*/
    while (true){
            std::this_thread::sleep_for(std::chrono::seconds(1));
            auto net_map = bpf.get_hash_table<net_key_t, uint64_t>("net_map");
            auto table = net_map.get_table_offline();
            for (auto &item : table) {
                std::cout << "time: " << std::time(0) << "pid: " << item.first.pid << " type: " << (item.first.type == 1 ? "sendMsg" : "recvMsg") << " size: " << item.second << std::endl;
            }
        }
    return 0;
}
```
#### e）效果演示
编译并执行该eBPF程序
```Bash
#编译命令
g++ -std=c++17 -o static_netio static_netio.cpp -lbcc -pthread
```
指定mysqld进程pid 2004756进行netio采集：

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyeh689a6ZZ6NpjYGdXV4IFvQAickuwb4A6iaM7OoWMdh8QhwGkBAbG28Eg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

远程执行连接MySQL的命令并执行SQL

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyed44kYTgAUf1Nwy380kyJAWib3PcxfRibjCVCbSt2I3AoX6CIBu4nYEJg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

打印观测的结果

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyezUqZpyYVicItXze24ZdTnkaPshyd2OF041xmebicN5AkMbkcAib7aBiaEw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面的演示中我们能看到，客户端和MySQL建立连接，每秒会打印日志，显示这个读取累加send和recv数据包的时间、mysqld的进程pid、send累加的数据包和recv累加的数据包大小。然后我们针对采集上来的数据就可以做分析了：
- 如果存在send数据包过大，说明数据库上存在较大的流量或者单条大SQL执行完会有大量的数据返回，比如全表查询返回这种，会导致应用出现内存大量占用问题，甚至引发OOM。
- 如果存在recv数据包过大，说明用户应用端发送给数据库的SQL文本存在过大问题，需要业务进一步关注业务逻辑是否正常。
## 总结

利用eBPF技术探测MySQL ,具有更高效，更扩展，更安全等优势，不用修改内核就可观测数据库性能。通过上面例子您是否发现采用eBPF跟踪数据库其实并不难，主要门槛在于需精通数据库内核和Linux编程，而且要对代码有精益求精的意识。