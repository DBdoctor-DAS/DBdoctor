# MySQL Using temporary cases and optimization methods
In the previous article ["Using order by in SQL, causing IO problems"] (../articles/AnSqlLineUsesOrderBy.md), we discussed MySQL's sorting strategy and optimization direction for Using Filesort. Today, we will introduce another common situation that can cause slow SQL, namely "Using temporary". In this article, we will lead you to explore in detail the reasons for such problems and how to optimize them.

## 一. Scenario cases

Before introducing the specific content, let's first take a look at a simulated case.

The device table (id, device_name, device_type, time) has 100,000 data, 50 device_type, 50,000 device_name, and neither has an index .

Group by according to device_name and device_type, and observe the same concurrency pressure on the two types of SQL.

```
select device_name,count(*) as c from device group by device_name  limit 0,5;
select device_type,count(*) as c from device group by device_type  limit 0,5;
```

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwJZ0LnGibL55sOA1a63uiaRFiaKzGO3kg6Ap0PzoFIGoR8DUcia1pJIZMxw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwzKMgljABPy0f2GzLKiakFt3hqvvibnk63cIuqTKqWNJ0mqmhl9rx465A/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

As shown in the above figure, several characteristics can be observed:

- Although the total QPS of the two types of SQL is only 10, it fills the CPU and affects the performance of the entire instance
- group by device_name Execution time reaches 5 seconds, performance is worrying

- The root cause of CPU full is attributed to group by device_name

- Group by device_name has a green converting HEAP to MySIAM event (starting from 5.7 is converting HEAP to ondisk), and the proportion is not small

- Both have the same execution plan, full table scan, same number of scanned rows, and EXTRA fields are Using temporary; Using filesort.

Some novice colleagues may have some questions when they see this:
- The number of lines scanned is the same, the execution plan is the same, why is group by device_name much slower?

- What does converting HEAP to MySIAM mean?

- What does using temporary mean?

- How to optimize such problems?

The following will answer these questions by introducing the mechanism related to using temporary internal temporary tables .

## 二、What is an internal temporary table?

To understand internal temporary tables, you need to first know about temporary tables . Temporary tables are session-level database objects that only exist during the database connection activity that created them. Unlike regular persistent tables, temporary tables disappear automatically after the connection is closed or the server is restarted. MySQL temporary tables can be divided into two ways of creation:

- **External temporary tables**：Temporary tables created by the user by explicitly executing the command create temporary table.

- **Internal temporary table**：Corresponding to the external temporary table, it is not a temporary table created by the user using the display command, but a temporary table created by the database optimizer to assist in the execution of complex SQL. Users can use the explain command to see if there is Using temporary in the Extra column. If there is, it is using an internal temporary table.

## 三、In which scenarios will internal temporary tables be used?

Most of these scenarios require aggregation operations. MySQL uses temporary tables to store aggregated data. The following scenarios may use internal temporary tables , which need to be confirmed by checking the execution plan.

- UNION

- Group BY

- Use TEMPTABLE algorithm, UNION query, and aggregated views

- The ORDER BY column in the table join is not in the driving table

- DISTINCT query and add ORDER BY

- When using the SQL_SMALL_RESULT modifier in SQL
- Complex derived tables, etc

## 四、How to store internal temporary tables?

Some colleagues may ask whether the internal temporary table of MySQL is stored in memory or disk? In fact, it is possible. There are a total of three storage methods :

### 1） Use memory
The amount of data to be stored does not exceed the values tmp_table_size and max_heap_table_size configuration items.

### 2）Use memory first, then convert to disk file

When the memory cannot meet the amount of data stored in the internal temporary table, MySQL will transfer the temporary table from memory to a disk file. If the amount of temporary data is large, it may cause abnormal occupation of disk capacity.

The following is the toptimizer_trace path for group by device_name:
- MySQL first creates a temporary table in memory with its location in memory (heap).

- Then I found that there was not enough memory, so I transferred the data to disk. The location became disk (MyISAM/InnoDB), that is, it was stored in the disk file table through a specific engine. Here, the default engine type is different between different versions

- In addition, row_limit_estimate can know the number of rows that can be stored in the current memory temporary table

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjw4rwvR7bpFtT0dQ0UwbXXNj4PjWo55IV59bSFGqCiaA9iaFkQ3vfiaQTZQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)


### 3）Directly use disk files

When using the decoration of SQL_BIG_RESULT, the disk file will be directly used; the other is that when there is a BLOB or TEXT column in the temporary table field, the memory temporary table will be directly abandoned.

**How do we usually monitor and evaluate temporary table issues?**

**Monitoring metrics**focus on two global statuses:

- created_tmp_tables: 1 accumulates for each temporary table created.

- created_tmp_disk_tables: 1 accumulates each time the memory temporary table is converted to the disk temporary table.

**Evaluation method**：The larger the created_tmp_disk_tables/created_tmp_tables ratio, the higher the proportion of the disk temporary table and the worse the performance.

## 五、How to optimize internal temporary tables?
### General optimization direction for internal temporary tables:
- Reduce unnecessary fields for queries: Reduce the size of temporary table rows, thereby increasing the number of data rows that can be stored in the memory temporary table.

- Adjust the temporary table memory parameters: Modify the values of system variables tmp_table_size and max_heap_table_size, so that the temporary table can use more memory. You can't adjust it blindly here. You need to evaluate the actual memory size (row_length * the number of rows) of the temporary table used by the root SQL. If you slightly adjust the parameters to meet the data volume requirements of the temporary table, you can choose to adjust the parameter verification. If the required memory is significantly different from the existing configuration, you need to evaluate the overall memory usage of the server to avoid OOM.

- Force direct use of disk temporary tables: If the data in the temporary table is inevitably large, you can consider directly using the internal temporary table of the disk to save the process of converting the memory temporary table to the disk temporary table.

### Union optimization direction:
Consider whether deduplicate is necessary. If not, you can use Union ALL instead of Union to avoid using internal temporary tables.

### Group By Optimization Direction:
- Before MySQL 8.0, GroupBy would come with sorting. If there is no sorting requirement, you can add order by null.

- Using indexes for Group by aggregation can avoid using internal temporary tables.
### Let's go back to the original case to see how to optimize

#### 1. Optimizing groups by index

Case SQL is a simple Group BY statement. The most direct optimization method is to add indexes, allowing MySQL to use the orderliness of B + tree indexes to accelerate aggregation calculations.

```SQL
alter table device add index idx_name(device_name);
```

![Execution plan after adding index](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwia7g1o3JxunialI5YEiarDGLmPGS7uicichtMZK9WF0Z6a3Opo2WiaycSNrQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

<center> Execution plan after adding index </center>

It can be seen that instead of a full table scan, indexes are used, and the EXTRA field is no longer Using temporary; Using filesort. The SQL execution time has also returned to the ms level.

![Performance after adding index](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwEZVDWoxGmjqWgAYtC2Reu9Ot2VTRpxLQ8M4oanaCkEiclk0bxW3CdAQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

<center> Performance after adding index </center>

CPU usage has dropped from 99% to below 10%, SQL execution time has become less than 1 second, and there is no longer a converting HEAP to MySIAM/desk event.


#### 2. Optimize temporary table memory configuration parameters

Not all scenarios can use indexes well. Some colleagues may pay attention to how to optimize parameters without using indexes. We will still use the initial case to analyze parameter tuning.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjw4rwvR7bpFtT0dQ0UwbXXNj4PjWo55IV59bSFGqCiaA9iaFkQ3vfiaQTZQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

We review the above slow SQL optimizer_trace, MySQL default tmp_table_size and max_heap_table_size is 16MB, from the print trace can be seen that the temporary table 411 bytes per row can store 40820 rows of data; and mentioned earlier in the device table device_name the type is 50,000, if all memory is needed 411 * 50000 ≈ 19.6MB.

Then we will adjust the temporary table memory configuration parameters to 20M and see the effect:

```Bash
mysql> set global tmp_table_size=20*1024*1024;set global max_heap_table_size=20*1024*1024;
```

![ optimizer_trace after increasing the temporary table parameters](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwOV9fA2pNQWoM7xvcglcul4Cw7C3VSjYM9icd5wfTiar0FuAs59CdAxYQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

<center>  optimizer_trace after increasing the temporary table parameters </center>

It can be found that the adjusted statement for group by device_name can store 51025 data, and no longer needs the disk temporary table.

Let's observe the effect after adjusting the parameters with the same pressure: CPU usage has decreased from 99% to below 25%, SQL execution time has become less than 1 second, there is no longer a converting HEAP to MySIAM/desk event, and the performance improvement effect is more than 5 times .

![Performance after increasing the temporary table parameters](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmpofWlBEvYG6GDNkcSWGjwOnhwL1SwBTV9vAHTL4yyJFYcdfJqibPLdYiaEP9NYiaYupCIptfNdcBeg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

<center> Performance after increasing the temporary table parameters </center>

## 六、Summary
Internal temporary tables are used by MySQL to assist complex SQL aggregation calculations, and will preferentially occupy memory. The available memory size is limited by the session-level parameters tmp_table_size and max_heap_table_size at the same time. When the memory is not enough, the memory temporary table will be converted into a disk temporary table. You can also force only the disk temporary table to be used by SQL_SMALL_RESULT decoration.

Common optimization methods include adjusting memory parameters, forcing only disk temporary tables to be used when dealing with large amounts of data, aggregating SQL using indexes such as Group By, and using Union ALL instead of UNion. You can check whether the business SQL temporary tables are reasonable to speed up SQL.

## Reference

[1]: 10.4.4 Internal Temporary Table Use in MySQL:https://dev.mysql.com/doc/refman/8.0/en/internal-temporary-tables.html