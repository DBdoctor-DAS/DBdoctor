# eBPF Practical Tutorial 6 | USDT Embedding and Performance Evaluation

## Preface

Dear friends, thank you very much for your continued attention and enthusiastic support to our eBPF series of articles. In the previous article [eBPF Practical Tutorial 5 | How to Use USDT Probes to Locate MySQL Exceptional Access (Including Source Code) ] (../../articles/EBPF05.md), we discussed the application of DTrace in MySQL. This method requires modifying the database kernel code (embedding static hooks) and then using eBPF for detection. By using uprobe (User-space Probes), the database kernel source code can be detected without modification. So, which method is better, uprobe (User-space Probes) or USDT (User Statically Defined Tracing)?

In this article, we will delve into the pre-embedding of USDT with you and conduct a comparative analysis from the perspectives of performance and scalability.

## 一. How to determine the probe of USDT (DTrace)?

So how do we define such static probes in the code?

The definition of the DTrace probe in the C++ is also very simple, just need to include the Header File, and the rest of the code is done!

```C
#include <iostream>
#include <sys/sdt.h>
#include <unistd.h>
// Simulate the function of executing database queries
void executeQuery(const std::string &query) {
    int status = 0;

    // Trigger the probe that starts the query
    DTRACE_PROBE1(myprovider, query_start, query.c_str());

    std::cout << "Executing query: " << query << std::endl;

    // Simulate query execution (where actual database operation logic can be placed).
    // ...

    // Assuming the query execution is successful, set the status to 1.
    status = 1;

    // Probe that triggers the end of the query
    DTRACE_PROBE2(myprovider, query_end, query.c_str(), status);
}

int main() {
    std::string query = "SELECT * FROM users;";
    executeQuery(query);

    return 0;
}
```

DTRACE_PROBEn is used to define the macro DTrace probe, n is defined with several parameters.
- Myprovider is the provider name of the probe. A provider is usually a module or application that defines a group of probes. Its name generally reflects the source of the probe group, such as the module name or application name.
- query_start This is the name of the probe used to identify the event that triggered it. The probe name usually describes the specific event that the probe logged. Here, 'query_start' may indicate the start of a query operation.
- query.c_str () This is the parameter passed to the probe. In this example, 'query.c_str () ' returns a string of type'const char * ', which may be the content of an SQL query. This parameter will be passed to the probe and recorded.

In the example above, the first probe DTRACE_PROBE1 (myprovider, query_start, query.c_str ()); defines a parameter, query -- SQL statement

The second probe DTRACE_PROBE2 (myprovider, query_end, query.c_str (), status); defines two parameters, query, and status -- SQL execution status.

## 二. How to use BCC to detect DTrace

We have finished writing the demo with DTrace probe above. Next, we will use eBPF to probe the demo.
```Python
#!/usr/bin/python

from bcc import BPF, USDT

# Create USDT probe
u = USDT(path="./main")
u.enable_probe(probe="query_start", fn_name="trace_query_start")
u.enable_probe(probe="query_end", fn_name="trace_query_end")

# Define eBPF program
bpf_text = """
#include

//Handling query_start probe events
int trace_query_start(struct pt_regs *ctx) {
char query[256];
// Read parameters from the probe
bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
bpf_trace_printk("Query start: %s\\n", query);
return 0;
}

// Handling query_end probe events
int trace_query_end(struct pt_regs *ctx) {
char query[256];
int status = 0;

// Read parameters from the probe
bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
bpf_usdt_readarg(2, ctx, &status);

bpf_trace_printk("Query end: %s, Status: %d\\n", query, status);
return 0;
}
"""

# Load eBPF program
b = BPF(text=bpf_text, usdt_contexts=[u])

# Output the information captured by the probe
print("Tracing USDT probes... Hit Ctrl-C to end.")
b.trace_print()
```
在trace_query_start及trace_query_end函数中分别对query_start和query_end两个探针进行探测，并打印了探针中的值。

### Demo example

Compile C++ demo:
```Bash
g++ -o main main.cpp
```
Execute probe script:
```Bash
python trace.py
```
Execute C++ demo:

```Bash
./main
```

At this time, the printing of the trace script is as follows:

```Bash
[root@localhost C]# python trace.py
Tracing USDT probes... Hit Ctrl-C to end.
b'            main-1176932 [012] d... 1028814.624214: bpf_trace_printk: Query start: SELECT * FROM users;'
b''
b'            main-1176932 [012] d... 1028814.624283: bpf_trace_printk: Query end: SELECT * FROM users;, Status: 1'
```

This way we can see the SQL in the demo and the status after execution obtained through the probe script.

## 三. DTrace Performance Evaluation

How much performance loss does using DTrace cause to the program?

What is the difference in program performance loss between using static probes like DTrace and dynamic probes like uprobe?

We will now divide the test into two groups:

### 1. Program single DTrace time consumption

```C
#include <iostream>
#include <sys/sdt.h>
#include <unistd.h>
#include <ctime>

void executeQuery(const std::string &query) {
    int status = 0;

    // Trigger the probe that starts the query
    //DTRACE_PROBE1(myprovider, query_start, query.c_str());

    // Simulate query execution (where actual database operation logic can be placed).
    // ...

    // Assuming the query execution is successful, set the status to 1.
    status = 1;
}

int main() {
    std::string query = "SELECT * FROM users;";

    // Get the start timestamp
    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    // Execute 2 billion queries
    for (int i = 0; i < 2000000000; ++i) {
        executeQuery(query);
    }

    // Get the end timestamp.
    clock_gettime(CLOCK_MONOTONIC, &end);

    // Calculate execution time (nanoseconds).
    long long duration = (end.tv_sec - start.tv_sec) * 1000000000LL + (end.tv_nsec - start.tv_nsec);

    std::cout << "Total execution time for queries: " << duration << " ns" << std::endl;

    return 0;
}
```
The function is called 2 billion times and a nanosecond timestamp is printed.。

Number     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
Time (ns) | 5303264202|5227036785| 5301668029| 5336016048| 5335845259| 5300766064.6| 

将DTRACE_PROBE1(myprovider, query_start, query.c_str());注释删除，再次编译执行。

Number     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
Time (ns) | 8692785702|8717224846| 8667429411| 8585855930| 8687992775| 8670257732.8| 

**Summary**：After using DTtrace, the time required increases significantly. Since the executeQuery function has no logical processing, it is not meaningful to directly compare the two sets of times. The time consumption of a single DTrace execution can be calculated by the difference between the two sets of data. The time consumption of 2 billion DTraces: 8670257732.8 - 5300766064.6 = 3369491668.2. Based on this data, the time consumption of a single DTrace can be estimated to be 1.68ns. This value is very low consumption in application link detection and can be ignored.

### 2. Comparison between USDT and uprobe

Still using the demo code above, change the number of loop executions to 10 million times. In order to minimize variables, we do not do any processing in the logic of eBPF, which is used to directly compare the performance of USDT and uprobe. Use USDT probing.

```python
#!/usr/bin/python

from bcc import BPF, USDT

# Create USDT probe
u = USDT(path="./main")
u.enable_probe(probe="query_start", fn_name="trace_query_start")

# Define eBPF program
bpf_text = """
#include <uapi/linux/ptrace.h>

// Handling query_start probe events
int trace_query_start(struct pt_regs *ctx) {
    //char query[256];
    // Read parameters from the probe
    //bpf_usdt_readarg_p(1, ctx, &query, sizeof(query));
    return 0;
}

"""

# Load eBPF program
b = BPF(text=bpf_text, usdt_contexts=[u])

# Output the information captured by the probe
print("Tracing USDT probes... Hit Ctrl-C to end.")
b.trace_print()
```

After running python trace.py, execute./main to count the execution time

Number     |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
Time(ns) | 10561827908|10702234936| 10307730644| 10934509995| 10393669721| 10579994640.8| 

Use uprobe for detection.
```python
from bcc import BPF

# Define eBPF program
bpf_text = """
#include <uapi/linux/ptrace.h>

// Handle uprobe events
int trace_execute_query(struct pt_regs *ctx) {
    return 0;
}

"""

# Load eBPF program
b = BPF(text=bpf_text)

# Set up uprobe
b.attach_uprobe(name="./main", sym="_Z12executeQueryRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE", fn_name="trace_execute_query")

# Output the information captured by the probe
print("Tracing uprobe... Hit Ctrl-C to end.")
b.trace_print()
```
After running python trace.py, execute./main to count the execution time

Number    |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
Time(ns) | 11745072927|10933262397| 10971148300| 11217268481| 11511004602| 11275551341.4| 

By comparison, we can see that USDT takes 69.5ns less than uprobe in terms of numbers, but this value is negligible in the application link detection consumption is also very low .

In fact, when using eBPF for engineering probing, because the program does not pre-embed static probes, uretprobe is often used, and the same demo is used to perform a set of tests on uretprobe
```python
from bcc import BPF

# Define eBPF program
bpf_text = """
#include <uapi/linux/ptrace.h>

// Handle uprobe events
int trace_execute_query(struct pt_regs *ctx) {
    return 0;
}

"""

# Load eBPF program
b = BPF(text=bpf_text)

# Set up uprobe
b.attach_uretprobe(name="./main", sym="_Z12executeQueryRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE", fn_name="trace_execute_query")

# Output the information captured by the probe
print("Tracing uprobe... Hit Ctrl-C to end.")
b.trace_print()
```
After running python trace.py, execute./main to count the execution time

Number      |  1 | 2 | 3 | 4 | 5 | avg |
-------- | ----- | ----- | ----- | ----- | -----| -----|
Time (ns) | 22906574920|22257599559| 21733037699| 21800112726| 21897448725| 22118954725.8| 

From this set of data, it can be seen that the triggering time of uretprobe is twice that of USDT and uprobe, but the consumption is also very low and can be ignored.

## 四. Summary

When using eBPF to probe User Mode applications, we can see from the trigger time of the probe above that USDT < uprobe < uretprobe. USDT is 6% better than uprobe and nearly twice as fast as uretprobe, but the time consumption is very low. The consumption of application link detection can be ignored. The definition of USDT static probe consumes an absolute time of about 1.68ns. During application development (which can modify the application source code), Aim for the Highest performance can choose to add DTrace probes for link monitoring! Of course, if you want to use USDT to probe databases, the cost is too high (defining USDT requires modifying the database source code, which is not universal and has intrusion into the source code). Uprobe is the best choice for probing databases (with very low performance loss and no need to change the database source code).