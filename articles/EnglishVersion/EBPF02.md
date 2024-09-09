# eBPF Practical Tutorial 2 | The most accurate quantification method for database network traffic (including source code)

## Preface
Since DBdoctor took the lead in deeply applying eBPF technology to the database field, it has quickly attracted widespread attention and discussion in the industry. Many leading companies such as Alibaba, Meituan, JD.com, ByteDance, etc. have actively engaged in in-depth technical exchanges with us. At the same time, we have also gained a large number of eBPF enthusiasts' attention. After the first article about uprobe was published, many friends have successfully run the code according to the tutorial and conducted in-depth self-learning and application.

In response to the enthusiastic urging of readers, we are now launching the second pure technical sharing article on eBPF - How to manually code a Kprobe function to analyze the network traffic of MySQL database. The aim is to provide more in-depth analysis and practical guidance on eBPF. We hope this article can be helpful to everyone. We will continue to publish more technical articles in this topic in the future. Welcome to follow our official account!

## What is Kprobe?
Kprobe is a dynamic tracing technology provided by the Linux kernel. It can dynamically insert probe points at the beginning, return point, or instruction address of a function at runtime. Using kprobe technology, probe points can be dynamically inserted into kernel functions to collect detailed information about kernel execution flow, register status, global data structure, etc., without the need to recompile or modify kernel code, achieving monitoring and analysis of functions. Kprobe greatly enhances the flexibility of kernel debugging and performance analysis, and can be applied to scenarios such as network optimization, security control, performance monitoring, and fault diagnosis, enabling developers to have a deeper understanding of kernel behavior. Especially in the field of database performance diagnosis, Probe redefines database observability, which can quickly and accurately identify potential performance issues and optimize them.
## Selection of Kprobe function

### 1）Network protocol stack parsing, get the function returned by MySQL SQL execution to the client
Based on Kprobe traffic detection, it is necessary to analyze the network protocol stack. The following figure shows the sending process of network data packets.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyearsEZVSyMtCgZib6VFIwEQsiaevbHLFsoTU4pKeibjVqgekKKOl7M6faA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the function call of the protocol stack above, we can see that tcp_sendmsg function is the entry function for sending packets.

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

From the source code of the function, we can know that the return value of the function is the result we need (i.e. the size of the data packet sent).

Therefore, we can select the tcp_sendmsg function in the transport layer as the probe point to Statistical Data database to send the total amount of data packets per second.

### 2）Network protocol stack parsing, find the function of MySQL receiving data packets from clients
Based on Kprobe traffic detection, it is necessary to analyze the network protocol stack. The following figure shows the process of receiving network data packets.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyeouGrIO8kwDLWgtkRZ8NXffIpyb10jaNoOCHGGTPvibCxWtwCXBeGfoQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

For counting the network traffic received by TCP, tcp_cleanup_rbuf function should be selected instead of tcp_recvmsg. Choosing tcp_recvmsg function will result in duplicate and omitted statistics.

- tcp_recvmsg () is a higher-level function on the TCP receive path that is responsible for copying data from the TCP layer to user space. When an application calls a function such as recv () or read () to read data from the TCP buffer, tcp_recvmsg () is triggered. If the data is not fully read by the application in the TCP receive buffer (for example, if the application calls recv () twice to read different parts of the same data segment), each call to tcp_recvmsg () will be triggered. This may result in the same data being evaluated multiple times during statistics.

- TCP data may not reach the tcp_recvmsg () layer due to kernel optimization processing (such as emergency data processing, data discard due to certain security checks), resulting in statistical omissions.

- Some direct input/output operations (such as splice system calls) can bypass the regular recvmsg path and transfer data directly from the kernel buffer to user space or other file descriptors without triggering tcp_recvmsg ().
tcp_cleanup_rbuf this function is called after TCP data has been received (that is, the data has been transferred from the kernel to user space and processed), it can more reliably count the amount of data actually consumed by the application without duplication or omission.

The function prototype is as follows:
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

The second parameter copied in the function is the size of the received data packet.

Therefore, we can select the tcp_cleanup_rbuf function in the transport layer as the probe point to Statistical Data database receives the total amount of data packets per second from the application side.

## How to detect MySQL's sending and receiving data packets per second with eBPF kprobe?
### 1）Environment preparation
> Prepare a Linux machine and install g ++ and bcc

### 2）Implement MySQL detection based on BCC tool
To achieve packet statistics, we first define a storage structure to store the total size of the released version of the process's received code packets. Based on Kprobe, the received and sent packets are accumulated and stored in the structure. Then, the accumulated data packets in the current storage structure can be read and printed every second to achieve the collection of received and sent data packets per second.

Next, we will write an eBPF program based on BCC using Kprobe to observe the data packets received and sent by MySQL (i.e. MySQL's NetIO statistics).
#### a）Analyze the kernel network protocol stack source code related network data packet processing function

```C
//Send packet function of kernel network protocol stack (return value)
int tcp_sendmsg(struct sock *sk, struct msghdr *msg, size_t size){
  ...
}
//receive packet function of kernel network protocol stack (imported parameter)
void tcp_cleanup_rbuf(struct sock *sk, int copied){
  ...
}
```
#### b）Import BPF objects from BCC

```C
//This object can embed our observation code into the observation point for execution
#include <bcc/BPF.h>

#include <string>
#include <iostream>
#include <thread>
#include <time.h>
```
#### c）Write eBPF code in C
```C

std::string strBPF = R"(
#include <linux/ptrace.h>
#include <net/sock.h>
#include <bcc/proto.h>

#include <linux/in6.h>
#include <linux/net.h>
#include <linux/socket.h>
#include <net/inet_sock.h>

//Define the index storage structure key for collection.
struct key_t {
    u32 pid;
    u16 type;
};

//Define the index storage structure value for collection
BPF_HASH(net_map, struct key_t,u64);

//Get the data packet returned by MySQL when executing SQL, and process the return value with the hook
int kretprobe__tcp_sendmsg(struct pt_regs *ctx)
{
    /*Get the pid of the current process.*/
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


//Get the data packet sent to MySQL, and handle the function imported parameters with the hook
int kprobe__tcp_cleanup_rbuf(struct pt_regs *ctx, struct sock *sk, int copied)
{
    /*Get the pid of the current process.*/
    u32 pid = bpf_get_current_pid_tgid() >> 32;

    if(FILTER_PID) return 0;

    /* Error detection*/
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
#### d）Observe the function that needs to be observed in the associated network protocol stack
```C

//Used for pid replacement in ebpf code programs
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

//Specify process pid for kprobe package statistics
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
    /*Detection tcp_sendmsg*/
    auto attachRes = bpf.attach_kprobe("tcp_sendmsg", "kretprobe__tcp_sendmsg",0,BPF_PROBE_RETURN);
    if(!attachRes.ok()) {
        std::cerr << "attach tcp_sendmsg error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
     /*Detection tcp_cleanup_rbuf*/
    attachRes = bpf.attach_kprobe("tcp_cleanup_rbuf", "kprobe__tcp_cleanup_rbuf");
    if(!attachRes.ok()) {
        std::cerr << "attach tcp_cleanup_rbuf error,msg: "<< attachRes.msg() << std::endl;
        return 1;
    }
    /*Read and print once per second*/
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
#### e）Effect demonstration
Compile and execute the eBPF program
```Bash
#compile command
g++ -std=c++17 -o static_netio static_netio.cpp -lbcc -pthread
```
Specify the mysqld process pid 2004756 for netio collection.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyeh689a6ZZ6NpjYGdXV4IFvQAickuwb4A6iaM7OoWMdh8QhwGkBAbG28Eg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Execute commands to connect to MySQL remotely and execute SQL.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyed44kYTgAUf1Nwy380kyJAWib3PcxfRibjCVCbSt2I3AoX6CIBu4nYEJg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Print the results of observations

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnPxt8yeZvBP4svKcohnEyezUqZpyYVicItXze24ZdTnkaPshyd2OF041xmebicN5AkMbkcAib7aBiaEw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the above demonstration, we can see that when the Client establishes a connection with MySQL, a log is printed every second, showing the time it takes to read and accumulate send and recv data packets, the process pid of mysqld, the size of the data packets accumulated by send and recv. Then we can analyze the collected data.
- If there is a large send data packet, it indicates that there is a large amount of traffic on the database or a large amount of data will be returned after executing a single large SQL, such as a full table query, which will cause the application to occupy a large amount of memory and even trigger OOM.
- If the recv data packet is too large, it indicates that the SQL text sent by the user application to the database is too large, and the business needs to further pay attention to whether the business logic is normal.
## Summary

Using eBPF technology to probe MySQL has the advantages of being more efficient, scalable, and secure, and can observe database performance without modifying the kernel. Through the above example, have you found that using eBPF to track databases is not difficult? The main threshold is to be proficient in database kernel and Linux programming, and to have a sense of excellence in the code.