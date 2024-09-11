# The locked elephant (Postgres), how to race against MySQL
PG Elephant and MySQL Dolphin, who runs faster, has always been a hot topic in the industry. However, once locked, neither can escape. Do you feel familiar with the following scenario?
- The application service has not released any code changes, but the normally fast interface suddenly becomes slow and jittery?
- The relatively smooth functional page suddenly cannot be opened, and the browser keeps spinning?

The emergence of these problems is mostly due to performance issues caused by locks, just like a sword hanging over your head, which can cause disasters at any time!

## 一. PostgreSQL lock issue
#### 1.Common lock-related issues
- Lock waiting: A lot of time is spent waiting for lock resources, which seriously affects database performance.
- Deadlock: Causing business abnormalities or losses.
- Uncommitted transactions and long transactions: Holding lock resources for a long time increases the probability of lock problems occurring.

#### 2.Difficulties in analyzing lock problems
- **Complexity of lock mechanism**：PostgreSQL's lock mechanism is quite complex, involving multiple types of locks, such as row-level locks, table-level locks, page-level locks, etc. Different lock levels will affect concurrency and performance. This makes understanding and managing locks complex, especially in large or highly concurrent Database Systems.

- **Concurrent Transaction Complexity**：Locking issues often occur between multiple concurrent transactions, which increases the complexity of locating the problem. Understanding the interdependencies and lock contention between concurrent transactions is crucial to analyzing lock-related issues.

- **Complexity of dead chains**：There may be multiple dead chains, further increasing the difficulty of sorting and analysis.

- **Difficult to reproduce and debug**：Some lock issues may be sporadic or difficult to reproduce, which makes debugging and problem solving more difficult. Especially in production environments, because the problem cannot be simply reproduced, it needs to be analyzed and debugged more carefully.

- **Challenges of log analysis**：Although PostgreSQL logs deadlock events, deadlock logs are difficult to read and comb through. Logs need to be analyzed in detail to understand the context information when deadlocks occur, the transactions involved, and lock waiting conditions.

- **Limitations of monitoring and diagnostic tools**：Although PostgreSQL provides some system views and capabilities to monitor lock activity, it is difficult to quickly combine these tools to analyze results in complex production environments.
## 二.Lock analysis case
Taking the deadlock problem as an example here:

1. Lock problem identification. At this stage, deadlock events are found through business service exception logs or PostgreSQL database monitoring.

2. By checking the PostgreSQL log, you can see further deadlock information, as shown in the following figure. The deadlock log shows the two transactions that caused the deadlock, as well as the two SQL statements that are waiting for the lock. At this stage, there is no more information to infer how these two transactions formed the deadlock ring.
```Bash
2024-05-26 21:59:20.868 EDT [14761] ERROR:  deadlock detected
2024-05-26 21:59:20.868 EDT [14761] DETAIL:  Process 14761 waits for ShareLock on transaction 36986; blocked by process 14762.
        Process 14762 waits for ShareLock on transaction 36985; blocked by process 14761.
        Process 14761: update people set age=33 where id=1;
        Process 14762: update people set age=23 where id=2;
2024-05-26 21:59:20.868 EDT [14761] HINT:  See server log for query details.
2024-05-26 21:59:20.868 EDT [14761] CONTEXT:  while updating tuple (1090,20) in relation "people"
2024-05-26 21:59:20.868 EDT [14761] STATEMENT:  update people set age=33 where id=1;
2024-05-26 21:59:20.968 EDT [14766] WARNING:  there is already a transaction in progress
```
3.  If we want to continue the in-depth analysis, there are several approaches:

    **Approach 1**：If the SQL characteristics that cause deadlock are obvious, you can find the business logic that contains the SQL in the business code and sort out the complete transaction. Then try to reproduce the problem manually.

    **Approach 2**：Analyzing audit logs is also a common way used by DBAs. The premise is to enable audit logs (log_statement = 'all'). Many businesses consider performance and storage and will not enable audit logs for a long time. It is necessary to enable audit logs and wait for deadlock events.

    1. First, find the deadlock event and the process ID of the deadlock transaction in the audit log.

        ```Bash
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on tra
        nsaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28417: update people set age=23 where id=2;
        Process 28419: update people set age=33 where id=1;","See server log for query details.",,,"while updating tuple (1090,20) in relation ""people""","update people set age=23 where id=2;",,,"pgbench"
        ```
        
    2. Then sort out in the audit log which SQL was executed by these process PIDs before and after the deadlock time point.
        ```Bash
        bash-4.2$ cat postgresql-2024-05-27_025951.csv |grep 28417
        2024-05-27 03:04:13.265 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,1,"idle",2024-05-27 03:04:13 EDT,8/93,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.268 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,2,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,0,LOG,00000,"statement: update people set age=23 where id=1;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.289 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,3,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,37429,LOG,00000,"statement: SELECT pg_sleep(5);",,,,,,,,,"pgbench"
        2024-05-27 03:04:18.293 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,4,"idle in transaction",2024-05-27 03:04:13 EDT,8/93,37429,LOG,00000,"statement: update people set age=23 where id=2;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on transaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28417: update people set age=23 where id=2;
        ```
        <center>SQL executed by Pid1 before and after deadlock</center>

        ```Bash
        bash-4.2$ cat postgresql-2024-05-27_025951.csv |grep 28419
        2024-05-27 03:04:13.281 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,1,"idle",2024-05-27 03:04:13 EDT,9/173,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.282 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,2,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,0,LOG,00000,"statement: update people set age=33 where id=2;",,,,,,,,,"pgbench"
        2024-05-27 03:04:13.291 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,3,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: SELECT pg_sleep(5);",,,,,,,,,"pgbench"
        2024-05-27 03:04:18.307 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,4,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: update people set age=33 where id=1;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.296 EDT,"postgres","postgres",28417,"10.18.215.89:41572",6654306d.6f01,5,"UPDATE",2024-05-27 03:04:13 EDT,8/93,37429,ERROR,40P01,"deadlock detected","Process 28417 waits for ShareLock on transaction 37430; blocked by process 28419.
        Process 28419 waits for ShareLock on transaction 37429; blocked by process 28417.
        Process 28419: update people set age=33 where id=1;","See server log for query details.",,,"while updating tuple (1090,20) in relation ""people""","update people set age=23 where id=2;",,,"pgbench"
        2024-05-27 03:04:28.300 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,5,"idle in transaction",2024-05-27 03:04:13 EDT,9/173,37430,LOG,00000,"statement: commit;",,,,,,,,,"pgbench"
        2024-05-27 03:04:28.319 EDT,"postgres","postgres",28419,"10.18.215.89:41576",6654306d.6f03,6,"idle",2024-05-27 03:04:13 EDT,9/174,0,LOG,00000,"statement: begin;",,,,,,,,,"pgbench"
        ```
        <center>SQL executed by Pid2 before and after deadlock</center>


    3. If the deadlock ring is caused by more than two transactions, more transaction SQL needs to be sorted out, and finally the deadlock ring can be sorted out.
    4. Finally, analyze which SQL and lock resources in the transaction conflict.

From the manual processing process above, it can be seen that when this problem occurs, it is very troublesome to handle, and there will be a lot of interference information and audit log performance issues in the middle. Is there a better solution?

**Recently, DBdoctor 3.2.0 version has launched the PostgreSQL lock pivot function. Friends who use PostgreSQL databases no longer need to worry about business system lag and deadlock problems caused by database locks. DBdoctor can help you quickly find the source of the lag!**

Now let's take a look at how DBdoctor performs lock pivot for PostgreSQL.

## 三. DBdoctor restores the formation process of PG deadlock problem
DBdoctor uses eBPF technology to collect the execution process data of PostgreSQL transaction SQL, including the lock data of fine grain, and visualizes the detailed formation process of locks in multi-transaction concurrent execution through swimlane diagrams.

PostgreSQL's lock pivot includes four scenarios: lock waiting, deadlock, uncommitted transactions, and long transactions. Let's take a look at the deadlock case above and how DBdoctor accurately finds and restores the scene.
1. First, we open the lock perspective function, and we can see that there is a deadlock problem in the lock exception event summary.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticK9nXEOckzLwwhn4kZWiaydNT2O4Xlmlib2KNyiaPUzSV1jZQPic7diaVlSjw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. Click Deadlock to view the deadlock visualization analysis.
In the deadlock visualization analysis, as shown in the figure below, the deadlock loop formed by > = 2 transactions will be drawn, and the specific lock resources held and waited for by each transaction will be marked, such as a certain page or tuple. At the same time, the rolled-back transactions will be marked in red.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKDSdm3jIj9eqaTMX1Tx0GfGqJvRWAvADYqcqNetnyH5fJ7goftiawKmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKDSdm3jIj9eqaTMX1Tx0GfGqJvRWAvADYqcqNetnyH5fJ7goftiawKmg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Deadlock visualization analysis is divided into two parts.

- The top part shows how many transactions are involved in the formation of the deadlock, and which rows of records are locked, resulting in a deadlock loop formed by mutual waiting.

- The following section shows the transactions involved in the deadlock. According to the timeline, which SQL statements were executed for each transaction, which SQL statement generated a lock wait, and which transaction was ultimately rolled back. The complete process of deadlock formation is replayed.

In this case, we can directly see that transaction A and transaction B caused a deadlock during concurrent execution. The deadlock ring diagram at the top can directly demonstrate how transaction AB forms a loop.

## 四. How to quickly locate DBdoctor in other lock scenarios
- Lock wait visualization analysis

In the lock waiting lane diagram, the waiting relationship and detailed transaction SQL execution process of two transactions will be displayed. Among them, the waiting lock event will be marked in the waiting lock transaction lane diagram, making it easy to trace the stuck problem.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKXVYhK3rib7LVPcJ0Xy3gU8iaEZMbJDYtFDyGtx6IDK0paKOvJlO4Z1fg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- Uncommitted transaction visualization

The occurrence of uncommitted transactions is often due to business code logic issues. In some extreme scenarios, transactions are not closed normally (defined as uncommitted transactions with a sleep state greater than 10 seconds). Failure to close transactions normally can cause lock occupation and failure to release, which may lead to problems such as normal business transactions not being able to obtain lock timeouts. From the figure below, we can see that after the UPDATE execution is completed, the commit is not immediately executed, but rather remains in a non-operational state for a long time, often due to unreasonable transaction scope design.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKrvicO3ECeHSvY0qDnWvCvsJ4IqC4sLDVqhjvBkqc2ZHImArlzBEvia0g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- Long transaction visualization

The appearance of long transactions is often due to the existence of slow SQL in the transaction, which is always executing but takes a long time (defined as a long transaction if the execution time exceeds 10s). Long transactions take too long during the SQL execution process, indicating low SQL execution efficiency, and often need to optimize slow SQL.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZknEq30vu5BeLK0y3BoxticKLAsKUo2GRpAiaLDqDGEWTBhCa2Lo6vxIyOJHoTuUB1e7icUN2G7yhb8A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

DBdoctor's lock pivot function is very powerful, which can quickly diagnose and locate lock problems in the database.