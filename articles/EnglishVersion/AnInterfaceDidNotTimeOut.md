# An interface did not perform timeout processing, causing the database to hang

## Preface
Do you often encounter the following problems during the code development process?
- The database connection was instantly occupied, causing a performance bottleneck
- System resources are heavily occupied, resulting in lock waiting or performance degradation
- The transaction log has grown significantly

The occurrence of the above situations may be caused by **uncommitted transactions**. After such transactions are started, **not sending SQL execution requests or transaction processing (COMMIT/ROLLBACK) requests to the database for a long time** will have a significant impact on the stability and performance of the database. In this article, we will analyze in detail the reasons for uncommitted transactions and provide effective troubleshooting and solutions.

## An interface did not perform timeout processing, causing the database to hang

Recently, a development colleague gave us feedback that he tried to call a third-party interface while performing transaction processing without adding a timeout limit to the call process. One day, the third-party service suddenly encountered an exception, causing the interface response to slow down. The current transaction was not committed for a long time, which directly led to a large number of uncommitted transactions, and the database hung up directly!

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZzv7Tew2JeR83NTDltaTlHaA3I6B1FupwedIm8UDfeVFv6icTAu525mQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

In the simplified code above, we can see that by using the annotation @Transactional to start a transaction, after executing the update, the third-party interface is called, which is equivalent to getting stuck and the subsequent code cannot be executed for a long time. The business will frequently operate on this interface, which is equivalent to a large number of uncommitted transactions in the database, resulting in a large amount of resource occupation, long-term lock occupation, a large number of lock waits, a surge in connection numbers, and a sharp decline in system performance.

For operation and maintenance DBAs, they can only see many connections in a sleeping state from the database level and cannot see specific SQL, which makes troubleshooting difficult.

Finally, the DBA solved the problem by **restarting the database** with the ultimate move.

## Which scenarios trigger uncommitted transactions?
There are various possible reasons for uncommitted transactions, and the following are several common triggering scenarios:

**1. There are complex queries or calculations**: Some transactions may need to perform complex queries, calculations, or batch operations. These operations usually take a long time to complete, and the transaction remains uncommitted throughout the process until all operations are completed and the transaction is committed.

**2. There are long-running batch tasks** : In some batch tasks, there may be a large amount of data that needs to be processed. To ensure atomicity and consistency in data processing, these operations are usually performed in a transaction. The transaction will remain uncommitted until the entire batch task is completed.

**3. Transaction lock waiting** : If a transaction needs to wait for another transaction to release the lock during execution, it will remain uncommitted until the lock is released and the operation can continue.

**4. The transaction needs to access third-party services** : After the transaction is opened, the interaction with the third party depends heavily on the speed of the third-party business execution, which greatly increases the transaction duration.

## How to quickly troubleshoot uncommitted transactions?
When the operation and maintenance DBA encounters such problems, it will perform uncommitted transaction analysis through SHOW PROCESSLIST or INNODB_TRX. Below, we manually simulate and create an uncommitted transaction.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZ8FoFJHuics4Wnweq4uWVl4NVhia1uicWtK6XNttibMckKhUTkIouAgwaSQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlIEiaaRBiaMXRBez6mF3ZcVZl2qH5f8dRjGNdVGiblhsmic1Hco5qj35ZMzkphoXZ2Mq9Z4lnlGGJxoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the above figure, the session status can be viewed through SHOW PROCESSLIST, but the specific SQL of session 90323 cannot be seen. Only a connection session in Sleep state can be seen. Without detailed SQL, it is difficult to locate which business logic exception is causing it.

### Use DBdoctor for quick diagnosis of uncommitted transactions

After using DBdoctor to manage an instance, it will actively diagnose the instance in real time (including uncommitted transactions). We can view the list in the Instance Diagnosis - > Lock Pivot - > Uncommitted Transactions tab page, click "View Transaction Details" to specify uncommitted transactions, and then slowly replay the complete execution process of the transaction SQL in the form of a swimlane diagram.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnruxGibYb4DU619v2PpTGJv99es0bFzMDWT0080Zp8DAWFianCZqP0x9diajTQx7FfX4TGD6JPx2TXA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Uncommitted transaction list supports locking brick** , click the session ID small arrow in the above figure can show the uncommitted transaction which SQL occurs lock wait , if there is a lock wait SQL, it means that the business development colleagues to do emergency optimization of the uncommitted transaction, for DBA colleagues only need to kill the session ID of the uncommitted transaction to complete the emergency firefighting.
## Summary

Occasional uncommitted transactions online may not cause business exceptions, but if DDL changes need to be made to tables related to the transaction one day, it may cause a failure (uncommitted transactions occupy metadata locks, which will block all SQL related to the table). By using the DBdoctor lock pivot function, it can help development and operation DBAs quickly identify whether there are uncommitted transactions in the database. DBAs can urgently put out fires, and business development colleagues can quickly locate and optimize abnormal code based on swimlane diagrams, completely solving this problem. Download and try it now!