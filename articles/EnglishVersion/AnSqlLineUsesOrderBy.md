# A SQL statement using order by causes IO issues

## 1. Preface
Sorting operations are very common in databases. Development colleagues can select specific columns for sorting according to business scenarios, such as publishing time, quantity, distance, etc. Sorting is crucial for improving User Experience, but it is also prone to database performance issues when dealing with large-scale data. Therefore, how to optimize sorting is crucial.

## 2. Scenario case: Using Order By in a SQL statement causes IO to be full

The SQL that caused the problem this time is very simple. The total amount of data in the table is around 2 million, which is not considered large.
```SQL
select * from device where device_type='pad' order by manufacturer,status limit 2000;
```
表结构如下:
```SQL
CREATE TABLE `device` (
`id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键',
`device_id` varchar(16) NOT NULL COMMENT '设备唯一编号',
`device_type` varchar(64) NOT NULL COMMENT '设备类型',
`manufacturer` varchar(100) NOT NULL COMMENT '生产厂商',
`status` int(11) NOT NULL COMMENT '设备状态',
`active_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '激活时间',
PRIMARY KEY (`id`),
KEY `idx_type` (`device_type`),
KEY `idx_manufacturer` (`manufacturer`),
KEY `idx_status` (`status`)
) ENGINE=InnoDB AUTO_INCREMENT=2320481 DEFAULT CHARSET=utf8mb4 COMMENT='注册设备表'
```
Execution effect:

Through DBdoctor, it was observed that this SQL directly hit disk IO from 0 to 100% during execution, triggering IO exception diagnosis.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ274Y0H8Beelib5ibbsjk4IiaSZibEAM2Uvb7dkNuJ4AxWf3cxMTPfSribXRmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

At this point, experienced DBAs must have noticed that this SQL sorting uses filesort and disk sorting. Let's take a look at the execution plan.

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ278wRJ7OxsvVAISnCRqibSIv080Qmy4uzzgTjOrTSsRttxnVTaqxbmDmA/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

From the execution plan, it can be seen that although the device_type index is used for filtering, the additional information also explicitly uses Using filesort.

Some novice colleagues may have questions about when to use indexes for sorting, when to use memory sorting, and when to use disk sorting. Below is a brief introduction to the relevant knowledge of MySQL sorting.

## 3. MySQL sorting principle

### How to implement SQL sorting?
#### 1)  Sort by index

The most commonly used B + tree index in MySQL stores ordered data, so if there is a suitable index, MySQL will directly read ordered data through the index.

We know that many usages can cause index invalidation, such as using like%%; using function for query conditions; data skew, etc.
The conditions for using indexes for sorting are even more stringent. For example, the index in the following case cannot be used for sorting:

- Multi-column sort does not match index order
```SQL
#SQL:
select * from device where device_type='pad' order by manufacturer,status limit 2000;
#Indexes that cannot be used for sorting:
(device_type,status,manufacturer)
```
-  Range query was used before sorting fields in the joint index
```SQL
#SQL:
select * from device where device_type in ('pad','phone') order by manufacturer,status limit 2000;
#Indexes that cannot be used for sorting:
(device_type,manufacturer,status)
```
####  2) Use Filesort to sort
When ordered data cannot be obtained using existing indexes, MySQL needs to allocate an additional space to store and sort data that meets the query conditions.

### What resources does FileSort sort occupy?

Some colleagues may have doubts, FileSort is called file sorting, does it represent sorting in disk files?

MySQL design is controlled by the session-level parameter sort_buffer_size, simply put, when the sorted data size exceeds sort_buffer_size, write the disk file sort, and when the sorted data is less than sort_buffer_size, only use memory sort.

How do you know if the current SQL sorting is using memory or disk temporary files? You can use MySQL optimizer_trace to track and view.

```SQL
#Open optimizer_trace current session
SET optimizer_trace = 'enabled=on,one_line=off';
#Execute query SQL
select * from device where device_type='pad' order by manufacturer,status limit 2000;
#View the execution process
SELECT * FROM information_schema.OPTIMIZER_TRACE\G;
```
Among them, there is a summary of filesort_summary record sorting in the results:
```json
"filesort_summary": {
"rows": 500000,
"examined_rows": 500000,
"number_of_tmp_files": 456,
"sort_buffer_size": 1047635,
"sort_mode": ""
}
```
- **rows:** Represents the total number of rows involved in the sorting process.
- **examined_rows:** Indicates the number of lines actually checked or processed.
- **number_of_tmp_files:**  Indicates the number of temporary files used in the sorting process. Here are 456 temporary files. When it is 0, it means that the disk temporary file sorting is not used.
- **sort_buffer_size:**  Indicates the size of the buffer used for the sort operation. This value is 1,047,635 bytes (about 1 MB), which determines the amount of memory used to store data in the sort operation. 8.0.12 previously the optimizer directly allocated sort_buffer_size size of memory to the session, starting from 8.0.12, the optimizer uses on-demand allocation until sort_buffer_size limit.
- **sort_mode:** Specifies the mode of the sort operation. Here, it indicates that the sort is based on the sort key (sort_key) and the additional field (additional_fields). The specific sort mode depends on the ORDER BY clause of the query.

### What are the types of FileSort sorting modes?

As mentioned earlier, the amount of sorted data is compared with the sort_buffer_size. What fields are stored in the sorted data? The official trace calls the fields stored in the sort_buffer the sorting mode.

**1）Two-way sort (back to table sort) < sort_key, rowid >**

Explanation: Only save the sorting field and the primary key ID in the sort_buffer. If you need to query other fields after sorting, you need to query the data record corresponding to the primary key according to the primary key ID back to the table.

Advantage: More data can be stored in sort_buffer

Disadvantages: Additional backtracking queries are required, resulting in additional CPU and IO consumption

**2）Single-way sorting (full field sorting) < sort_key, additional_fields >**

Explanation: SQL required fields are put into the sort_buffer, after sorting can be directly returned to the required fields.

Advantage: No need to query the table again

Disadvantages: Take up more sorting space

**3）Compressed single-way sorting < sort_key, packed_additional_fields >**

Similar to single-way sort, fields are stored more tightly, for example, the char type is stored with the actual length and does not retain the complement length.

MySQL before the 8.0.20 by the configuration parameter max_length_for_sort_data to determine which mode to use, when the sum of the required field length is greater than max_length_for_sort_data using two-way sorting < sort_key, rowid >, since MySQL 8.0.20 deprecated max_length_for_sort_data, preferred to use < sort_key, packed_additional_fields >.

## 4. Sorting optimization suggestions
### 1) Overall optimization direction

From the above introduction to the MySQL sorting strategy, we can see the overall optimization direction of MySQL for sorting.

- Best choice: Use the index as much as possible to sort by index.

- Secondly, choose: in the case of sufficient memory, prioritize memory sorting without returning to the table, and sacrifice memory usage for performance improvement through parameter tuning.

- Worse choice: In the case of limited memory, you can use memory sorting + back table query, disk temporary table sorting, and balance the occupancy of memory, CPU, and IO by debugging parameters in the case of limited resources.

-Worst case: In the case of limited memory and large sorting datasets, disk temporary table sorting + return table query may be encountered, and the database itself has little room for optimization, requiring more optimization from the business level or resource level.
### 2) Indexing and SQL optimization

- Optimization of queries and sorting through joint indexes
- Avoid querying non-essential fields and use overlay indexes whenever possible.
- When encountering similar SQL, we can also use DBdoctor's SQL audit function to review the SQL problem. The audit results are as follows:
    - Finding better index suggestions.
    - Check out problems such as select *, Using filesort, and limit large values.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ27UEWag6tSquueccm0EYOHFXU7bJwEHKgPQyp0V1DMGZ38JATzmsT3ww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3) Parameter optimization

In addition to the sort_buffer_size and max_length_for_sort_data described above, read_rnd_buffer_size, max_sort_length can also affect the performance of sorting in some scenarios.

In addition, it is also necessary to adjust the load of SQL, CPU, MEM, IO and other resources, which is more difficult for non-senior DBAs. At present, DBdoctor's AI intelligent parameter tuning function is in dogfooding, so stay tuned.