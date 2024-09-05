# eBPF实战教程六｜USDT的预埋与性能测评

## 前言

各位小伙伴们，非常感谢您对我们eBPF专题系列文章的持续关注和热情支持。在先前的文章[《eBPF实战教程五｜如何使用USDT探针定位MySQL异常访问(含源码)》](https://github.com/DBdoctor-DAS/DBdoctor/blob/main/articles/EBPF05.md)中，我们探讨了MySQL中DTrace的应用，该方法需要修改数据库内核代码（嵌入静态钩子），然后利用eBPF进行探测。而通过uprobe(User-space Probes)方式无需修改数据库内核源码即可探测。那么uprobe(User-space Probes)方式和USDT(User Statically Defined Tracing）相比，哪种方式更好呢？

本文我们将与您深入探讨USDT的预埋，并从性能和扩展上进行对比分析。

## 一. 如何定USDT（DTrace）的探针？

那我们该如何在代码中定义这样的静态探针呢？

对于C++中的DTrace探针的定义也是非常简单的，只需要包含头文件,剩下的一行代码搞定啦！

```C
#include <iostream>
#include <sys/sdt.h>
#include <unistd.h>
// 模拟执行数据库查询的函数
void executeQuery(const std::string &query) {
    int status = 0;

    // 触发查询开始的探针
    DTRACE_PROBE1(myprovider, query_start, query.c_str());

    std::cout << "Executing query: " << query << std::endl;

    // 模拟查询执行（此处可以放置实际的数据库操作逻辑）
    // ...

    // 假设查询执行成功，设置status为1
    status = 1;

    // 触发查询结束的探针
    DTRACE_PROBE2(myprovider, query_end, query.c_str(), status);
}

int main() {
    std::string query = "SELECT * FROM users;";
    executeQuery(query);

    return 0;
}
```

DTRACE_PROBEn 便是用来定义DTrace探针的宏，n是定义有几个参数。
- myprovider 是探针的提供者（provider）名称。提供者通常是定义一组探针的模块或应用程序。它的命名一般反映了该探针组的来源，比如模块名或应用程序名。
- query_start 这是探针的名称，用于标识触发的事件。探针名通常描述该探针记录的特定事件。在这里，`query_start` 可能表示某个查询操作的开始。
- query.c_str() 这是传递给探针的参数。在这个例子中，`query.c_str()` 返回一个 `const char*` 类型的字符串，这可能是某个 SQL 查询的内容。这个参数会被传递给探针并记录下来。

在上面的示例中第一个探针  DTRACE_PROBE1(myprovider, query_start, query.c_str());  定义了一个参数，query -- SQL语句

第二个探针DTRACE_PROBE2(myprovider, query_end, query.c_str(), status); 定义了两个参数，分别是query,以及status -- SQL执行状态。

## 二. 如何使用BCC探测DTrace

我们上面写完了带有DTrace探针的demo,接下来我们对该demo使用eBPF进行探测。
```Python
#!/usr/bin/python

from bcc import BPF, USDT

# 创建USDT探针
u = USDT(path="./main")
u.enable_probe(probe="query_start", fn_name="trace_query_start")
u.enable_probe(probe="query_end", fn_name="trace_query_end")

# 定义eBPF程序
bpf_text = """
#include

//处理 query_start 探针事件
int trace_query_start(struct pt_regs *ctx) {
char query[256];
// 从探针中读取参数
bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
bpf_trace_printk("Query start: %s\\n", query);
return 0;
}

// 处理 query_end 探针事件
int trace_query_end(struct pt_regs *ctx) {
char query[256];
int status = 0;

// 从探针中读取参数
bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
bpf_usdt_readarg(2, ctx, &status);

bpf_trace_printk("Query end: %s, Status: %d\\n", query, status);
return 0;
}
"""

# 加载eBPF程序
b = BPF(text=bpf_text, usdt_contexts=[u])

# 输出探针捕获的信息
print("Tracing USDT probes... Hit Ctrl-C to end.")
b.trace_print()
```
在trace_query_start及trace_query_end函数中分别对query_start和query_end两个探针进行探测，并打印了探针中的值。

### Demo 示例

编译C++的demo：
```Bash
g++ -o main main.cpp
```
执行探测脚本：
```Bash
python trace.py
```
执行C++的demo:

```Bash
./main
```

此时trace脚本的打印如下：

```Bash
[root@localhost C]# python trace.py
Tracing USDT probes... Hit Ctrl-C to end.
b'            main-1176932 [012] d... 1028814.624214: bpf_trace_printk: Query start: SELECT * FROM users;'
b''
b'            main-1176932 [012] d... 1028814.624283: bpf_trace_printk: Query end: SELECT * FROM users;, Status: 1'
```

这样我们就可以看到通过该探测脚本获取到了demo中的SQL以及执行后的status。

## 三. DTrace性能测评

使用DTrace对程序性能损耗有多少呢？
使用DTrace这种静态探针和uprobe这种动态探针，对程序性能损耗有什么差异呢?
我们接下来分为两组进行测试：

### 1. 程序单条DTrace耗时

```C
#include <iostream>
#include <sys/sdt.h>
#include <unistd.h>
#include <ctime>

void executeQuery(const std::string &query) {
    int status = 0;

    // 触发查询开始的探针
    //DTRACE_PROBE1(myprovider, query_start, query.c_str());

    // 模拟查询执行（此处可以放置实际的数据库操作逻辑）
    // ...

    // 假设查询执行成功，设置status为1
    status = 1;
}

int main() {
    std::string query = "SELECT * FROM users;";

    // 获取开始时间戳
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    // 执行二十亿次查询
    for (int i = 0; i < 2000000000; ++i) {
        executeQuery(query);
    }

    // 获取结束时间戳
    clock_gettime(CLOCK_MONOTONIC, &end);

    // 计算执行时间（纳秒）
    long long duration = (end.tv_sec - start.tv_sec) * 1000000000LL + (end.tv_nsec - start.tv_nsec);

    std::cout << "Total execution time for queries: " << duration << " ns" << std::endl;

    return 0;
}
```
循环调用20亿次该函数，并打印纳秒级时间戳。

次数     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
时间(ns) | 5303264202|5227036785| 5301668029| 5336016048| 5335845259| 5300766064.6| 

将DTRACE_PROBE1(myprovider, query_start, query.c_str());注释删除，再次编译执行。

次数     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
时间(ns) | 8692785702|8717224846| 8667429411| 8585855930| 8687992775| 8670257732.8| 

**小结**：使用DTtrace后所需时间增幅较大，由于executeQuery函数没有任何逻辑处理，直接对比两组时间意义不大。可通两组数据的差值得出DTrace单次执行的时间消耗，20亿次DTrace的时间消耗：8670257732.8 - 5300766064.6 = 3369491668.2, 根据该数据可估计出单条DTrace时间消耗在1.68ns，该值放应用链路探测中**消耗非常低**可忽略不计。

### 2. USDT与uprobe对比

仍使用上面的demo代码，将循环执行次数改为一千万次 ，为了尽可能减少变量，我们在eBPF的逻辑中不做任何处理，用来直接对比USDT和uprobe陷入的性能。使用USDT探测：

```python
#!/usr/bin/python

from bcc import BPF, USDT

# 创建USDT探针
u = USDT(path="./main")
u.enable_probe(probe="query_start", fn_name="trace_query_start")

# 定义eBPF程序
bpf_text = """
#include <uapi/linux/ptrace.h>

// 处理 query_start 探针事件
int trace_query_start(struct pt_regs *ctx) {
    //char query[256];
    // 从探针中读取参数
    //bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
    return 0;
}

"""

# 加载eBPF程序
b = BPF(text=bpf_text, usdt_contexts=[u])

# 输出探针捕获的信息
print("Tracing USDT probes... Hit Ctrl-C to end.")
b.trace_print()
```

运行 python trace.py后，执行./main 统计执行时间

次数     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
时间(ns) | 10561827908|10702234936| 10307730644| 10934509995| 10393669721| 10579994640.8| 

使用uprobe进行探测：
```python
from bcc import BPF

# 定义eBPF程序
bpf_text = """
#include <uapi/linux/ptrace.h>

// 处理 uprobe 事件
int trace_execute_query(struct pt_regs *ctx) {
    return 0;
}

"""

# 加载eBPF程序
b = BPF(text=bpf_text)

# 设置 uprobe
b.attach_uprobe(name="./main", sym="_Z12executeQueryRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE", fn_name="trace_execute_query")

# 输出探针捕获的信息
print("Tracing uprobe... Hit Ctrl-C to end.")
b.trace_print()
```
运行 python trace.py后，执行./main 统计执行时间

次数     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
时间(ns) | 11745072927|10933262397| 10971148300| 11217268481| 11511004602| 11275551341.4| 

通过对比可知，在数字上我们能看到USDT耗时比uprobe少69.5ns，但该值放应用链路探测中**消耗也非常低**可忽略不计。

实际在使用eBPF进行工程化探测时，由于程序没有预埋静态探针，很多时候还会用到uretprobe, 使用同样的demo对uretprobe 进行一组测试
```python
from bcc import BPF

# 定义eBPF程序
bpf_text = """
#include <uapi/linux/ptrace.h>

// 处理 uprobe 事件
int trace_execute_query(struct pt_regs *ctx) {
    return 0;
}

"""

# 加载eBPF程序
b = BPF(text=bpf_text)

# 设置 uprobe
b.attach_uretprobe(name="./main", sym="_Z12executeQueryRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE", fn_name="trace_execute_query")

# 输出探针捕获的信息
print("Tracing uprobe... Hit Ctrl-C to end.")
b.trace_print()
```
运行 python trace.py后，执行./main 统计执行时间

次数     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
时间(ns) | 22906574920|22257599559| 21733037699| 21800112726| 21897448725| 22118954725.8| 

由该组数据可知uretprobe触发耗时是USDT和uprobe的两倍，但该值放应用链路探测中**消耗也非常低**可忽略不计。

## 四. 总结

在使用eBPF探测用户态应用程序时，从上面探针的触发耗时我们能看到USDT<uprobe<uretprobe, USDT比uprobe优6%，比uretprobe快将近一倍，但从耗时的值上看都非常低，在应用链路探测这块消耗可忽略不计。USDT静态探针的定义，单条的绝对时间消耗在1.68ns左右，在应用开发时（可修改应用源码），追求极致性能可以选择添加DTrace探针进行链路监测！当然如果你想用USDT这种方式探测数据库成本太高（定义USDT需要修改数据库源码，不具备通用性，对源码有侵入），uprobe用来探测数据库才是最佳选择（非常低的性能损耗和无需更改数据库源码）。