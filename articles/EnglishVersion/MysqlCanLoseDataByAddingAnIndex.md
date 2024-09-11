# Adding an index to MySQL may result in data loss. Do you know this pitfall?

## Preface

Recently, we received a consultation from a database operation and maintenance partner. They have a MySQL 5.6 database and need to add DDL indexes to the core payment table . They asked us how to add indexes more elegantly. Based on DBA experience, there are mainly the following ways to add indexes to tables:

- Use MySQL native DDL statements (including OnlineDDL)

- Use pt-osc, create a temporary table + trigger to achieve

- Use gh-ost to create a new temporary table and implement it based on binlog

Close Binlog at the Session level, add indexes to the standby database first, and then switch between primary and secondary

The specific choice of which method to use requires comprehensive consideration of various factors such as table structure, primary and backup latency, business write load, disk IO, whether it affects business, and usage habits. After our comprehensive evaluation, it is better to use the native OnlineDDL for this payment scenario. However, the colleague feedback that their DevOps flow requires the use of pt-osc for DDL changes, and ultimately uses pt-osc to add indexes due to the process. However, the operation of the operation and maintenance colleague almost caused huge losses to the company. Fortunately, the DBdoctor performance insight function was used to detect problems in advance and stop losses in time.

## Principle of pt-osc, any pitfalls?
### 1）Principle of pt-osc

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34Isya01ic73LhmcElpGzptQr8dAytp6lXgqaq1Xib4QuYEj4Je0utTsw7lA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

The general method of pt-osc is to create a temporary table, then copy the data in the source table in batches according to the primary key chunk, and control the incremental data to be written to the temporary table in real time through three write-related triggers, so as to achieve consistency with the final data of the source table and finally rename exchange. From a principle perspective, the implementation logic is relatively simple. So, does this principle harm the business?

### 2）Operations and maintenance evaluate whether pt-osc meets the requirements
From the implementation of pt-osc, we can directly see the following problems:
- Creating a temporary table will cause the space to double, and sufficient space needs to be reserved
- The existence of the trigger will cause the write to double, and it is necessary to ensure that the disk IO can support it
- Increased writing will cause delay between primary and secondary
- Deadlock causes business transaction rollback

After detailed evaluation by operation and maintenance colleagues, based on the current business volume, the first three points will not be a problem, and the fourth point is that the current MySQL innodb_autoinc_lock_mode parameter is 2, which will not cause deadlock problems caused by auto-increment locks. Therefore, the evaluation of operation and maintenance colleagues can directly add indexes online through pt-osc.

### 3）Change online, step on the pit deadlock

During the online change process, a deadlock was found in the business. Below is the detailed deadlock log of the business.

```Bash
LATEST DETECTED DEADLOCK
-----------------------
2024-06-26 02:25:20 7fe147aa4700
*** (1) TRANSACTION:
TRANSACTION 2653761553, ACTIVE 0 sec inserting
mysql tables in use 2, locked 2
LOCK WAIT 7 lock struct(s), heap size 1184, 5 row lock(s), undo log entries 3
MySQL thread id 143920619, OS thread handle 0x7fe146cad700, query id 7056859581 update
REPLACE INTO `pay`.`_pay_info_new` (`pay_info_id`, `at_date`, `at_progress`, `ag_no`, `bs_status`, `buyer_id`, `contract_code`, `contract_id`...
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 169 page no 507288 n bits 288 index `
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** (2) TRANSACTION:
TRANSACTION 2653761554, ACTIVE 0 sec inserting
mysql tables in use 2, locked 2
6 lock struct(s), heap size 1184, 4 row lock(s), undo log entries 3
MySQL thread id 144763335, OS thread handle 0x7fe147aa4700, query id 7056859583 update
REPLACE INTO `pay`.`_pay_info_new` (`pay_info_id`, `at_date`, `at_progress`, `ag_no`, `bs_status`, `buyer_id`, `contract_code`, `contract_id`...
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 169 page no 507288 n bits 288 index `pay_info_id` of table `pay`.`_pay_info_new` trx id 2653761554 lock_mode X locks rec but not gap
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 169 page no 507288 n bits 288 index `pay_info_id` of table `pay`.`_pay_info_new` trx id 2653761554 lock_mode X waiting
Record lock, heap no 214 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 8; hex 80000000096dd335; asc      m 5;;
 1: len 8; hex 800000000009edf2; asc         ;;

*** WE ROLL BACK TRANSACTION (2)
------------
```

From the above log, we can see that the temporary table operation corresponding to the trigger has been rolled back, which is equivalent to the normal business operation on the source table being rolled back, affecting the business.

## Pt-osc emergency rollback, will data be lost?

### 1）Emergency rollback

Emergency rollback for operation and maintenance, follow the steps below:

- First kill the script process of pt-osc

- Remove three triggers for online pt-osc

Finally, the operation and maintenance confirmation has been rolled back, and changes will be made after analyzing the reasons the next day.

### 2）Found a big pit, pt-osc did not clean it up

When a business development colleague used DBdoctor's performance insight for problem analysis, they found that there were still insert statements for temporary tables in the business. They thought that the operation and maintenance department had started to add index changes again and asked the operation and maintenance department to confirm.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34IsC81fAhVWOiaN9ibsx8PJ5HYJa7nxfPiad9duibNfm8H4RicQK2YYcZCejgQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

The operation and maintenance feedback change has stopped. After checking the execution, it was found that only the process of the shell script was killed when killing the pt-osc script, and the child process was not killed. The operation and maintenance colleague performed the kill again. After running for more than 8 hours without triggers, the pt-osc process finally stopped.

### 3）If pt-osc is in a trigger-free state and the process eventually completes, will data be lost?

From the principle of pt-osc above, full chunk copying is done by chunk splitting and processing in a primary key increment manner, and chunks that have already been processed will not be copied again. Therefore, there will be the following two problems:

- If the changed table only writes data according to the primary key auto-increment id, then the last chunk of the final full copy is the latest data, which can ensure data consistency and will not lose data .
- For the chunk that has been fully copied, if the data changes again, because the trigger is gone, it is equivalent to incremental loss, and the data is lost in insert/delete/update changes.

Such a big pit, fortunately, the process of pt-osc has not been completed. After inspection, more than 9,000 payment data updates occurred in the business within 8 hours. If it were not discovered by DBdoctor, more than 9,000 payment data would be lost, which is simply fatal for the company.

PT-OSC is not a panacea. Everyone must be careful when making changes. A slight mistake may cause serious malfunctions.

#### 4）Final native OnlineDDL changes completed

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnQLibXQKiciaBeelc6FXb34IsGIzMBnZicUmZyq4GonMVxDE2uNSbT5eKusybsv7DPFYiaPqdXfKxyPWw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

In the end, the operation and maintenance team accepted our suggestion and used the native OnlineDDL for the core payment table OnlineDDL change , and the change was successful. The detailed progress of the actual change can be seen from the above figure, and the change can be traced back.
## Summary

Dear R & D or DBA colleagues, do you often encounter failures caused by database DDL changes? You can use DBdoctor to evaluate database performance and see if DDL can be done. The entire process of executing DDL is visible. You can give it a try~