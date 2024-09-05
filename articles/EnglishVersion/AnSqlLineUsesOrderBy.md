# 一条SQL使用order by，引发IO问题

## 一.前言
排序操作在数据库中非常普遍，开发同学可以根据业务场景选择指定列进行排序，比如按照发布时间，数量、距离等等。排序对于提升用户体验很关键，但在处理大规模数据时，也极易产生数据库性能问题，如何进行排序优化至关重要。

## 二.场景案例：一条SQL使用Order By，引发IO打满

本次引发问题的SQL很简单，全表数据量在200W左右不算大。
问题SQL如下：
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
执行效果：

通过DBdoctor观测到该条SQL在执行时直接将磁盘IO从0打到了100%，触发了IO异常问题诊断。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ274Y0H8Beelib5ibbsjk4IiaSZibEAM2Uvb7dkNuJ4AxWf3cxMTPfSribXRmw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

此时有经验的DBA肯定看出来了，这条SQL的排序使用了filesort，而且还是用的磁盘排序，我们不妨来看下执行计划。

![](https://mmbiz.qpic.cn/mmbiz_jpg/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ278wRJ7OxsvVAISnCRqibSIv080Qmy4uzzgTjOrTSsRttxnVTaqxbmDmA/640?wx_fmt=jpeg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从执行计划上可以看到，虽然用了device_type的索引进行筛选，但是额外信息里也明确的使用了Using filesort。

有些新手同学可能会有疑问，针对排序哪些情况会用索引，哪些情况使用内存排序，哪些情况用磁盘，下面就来简单介绍下MySQL 排序的相关知识。

## 三.MySQL排序原理

### 如何实现SQL排序？
#### 1) 利用索引进行排序

MySQL中最常用的B+树索引保存的数据本身就是有序的，所以如果有合适的索引MySQL会直接通过索引读取有序数据。
我们知道很多用法会导致索引失效，例如：使用like %%;查询条件使用函数；数据倾斜等情况。
而排序使用索引的条件更加苛刻，比如下面的情况的索引也无法用于排序：
- 多列排序与索引顺序不匹配
```SQL
#SQL:
select * from device where device_type='pad' order by manufacturer,status limit 2000;
#无法用于排序的索引:
(device_type,status,manufacturer)
```
-  在联合索引排序字段之前使用了范围查询
```SQL
#SQL:
select * from device where device_type in ('pad','phone') order by manufacturer,status limit 2000;
#无法用于排序的索引:
(device_type,manufacturer,status)
```
####  2) 使用Filesort进行排序
当无法使用已有索引获取有序数据时，MySQL就需要额外开辟一块空间来保存符合查询条件的数据并进行排序。

### FileSort排序占用资源有哪些？

有同学可能有疑问，FileSort名字叫文件排序，是代表使用在磁盘文件里进行排序吗?

MySQL设计是通过参数sort_buffer_size这个会话级别参数来控制，简单说当排序数据大小超过sort_buffer_size时写入磁盘文件排序，当排序数据小于sort_buffer_size时只使用内存排序就足够。

那如何知道当前SQL排序用的是内存还是磁盘临时文件呢，可以使用MySQL optimizer_trace 来跟踪查看。

```SQL
#在当前会话开启optimizer_trace
SET optimizer_trace = 'enabled=on,one_line=off';
#执行查询SQL
select * from device where device_type='pad' order by manufacturer,status limit 2000;
#查看执行过程
SELECT * FROM information_schema.OPTIMIZER_TRACE\G;
```
其中结果里有一段filesort_summary记录排序的概况：
```json
"filesort_summary": {
"rows": 500000,
"examined_rows": 500000,
"number_of_tmp_files": 456,
"sort_buffer_size": 1047635,
"sort_mode": ""
}
```
- **rows:** 表示在排序过程中涉及的总行数。
- **examined_rows:** 表示实际被检查或处理的行数。
- **number_of_tmp_files:** 表示在排序过程中使用的临时文件的数量。这里是 456 个临时文件，是0时，代表未使用磁盘临时文件排序。
- **sort_buffer_size:** 表示用于排序操作的缓冲区的大小。这个值是 1,047,635 字节（约 1 MB），它决定了排序操作中用于存储数据的内存量，8.0.12之前优化器直接给会话分配sort_buffer_size大小的内存，从8.0.12开始，优化器采用按需分配，直到 sort_buffer_size限制。
- **sort_mode:** 指定排序操作的模式。在这里， 表示排序是基于排序键（sort_key）和附加字段（additional_fields）进行的。具体的排序模式取决于查询的 ORDER BY 子句。

### FileSort排序模式有哪几种？

前面提到是用排序的数据量与sort_buffer_size做比较，排序数据里又存了哪些字段呢，官方trace把sort_buffer中保存哪些字段叫做排序模式。

**1）双路排序（回表排序）< sort_key, rowid >**

解释：在sort_buffer中只保存排序字段与主键ID，在完成排序之后如果还需要查询其他字段，需要根据主键ID回表查询主键对应的数据记录。

优点：可以在sort_buffer中保存更多的数据

缺点：需要额外回标查询，造成额外的CPU与IO消耗

**2）单路排序（全字段排序）< sort_key, additional_fields >**

解释：将SQL中需要的字段都放入sort_buffer中，排序完成后可直接返回所需要的字段。

优点：无需再次回表查询

缺点：占用更多的排序空间

**3）压缩单路排序< sort_key, packed_additional_fields >**

与单路排序相似，存储的字段更紧密，比如char类型以实际长度存储，不会保留补位长度。
MySQL在8.0.20之前由配置参数max_length_for_sort_data来决定使用哪种模式，当所需字段长度之和大于max_length_for_sort_data时使用双路排序< sort_key,rowid>,从MySQL8.0.20开始弃用了max_length_for_sort_data，优先使用<sort_key, packed_additional_fields >。

## 四.排序优化建议
### 1) 整体优化方向

通过上面对MySQL排序策略的介绍，可以看出MySQL针对排序的整体优化方向：

- 最优选择：能用索引尽量用索引来排序。

- 其次选择：在内存充足的情况下，优先内存排序且不回表的方式，通过调参牺牲内存占用换取性能提升。

- 更差选择：在内存有限的情况下，可以使用内存排序+回表查询，磁盘临时表排序，在资源受限的情况下通过调参适量均衡内存、CPU、IO的占用。

- 最差情况：在内存有限且排序数据集很大的情况，可能遇到磁盘临时表排序+回表查询，数据库本身能优化的余地不多，需要更多从业务层面或资源层面优化。
### 2) 索引与SQL‍优化

- 通过联合索引实现查询与排序最优
- 避免查询非必要字段，尽量使用覆盖索引。
- 碰到类似SQL我们也可以借助DBdoctor的SQL审核功能来审查该条SQL的问题，审核结果如下：
    - 找出了更优的索引建议。
    - 审核出了select *、Using filesort、limit 大值等问题点。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmdzHOsy5QGBbMIzGmkqZ27UEWag6tSquueccm0EYOHFXU7bJwEHKgPQyp0V1DMGZ38JATzmsT3ww/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3) 参数优化

除了上面介绍到的sort_buffer_size和max_length_for_sort_data外，read_rnd_buffer_size、max_sort_length在某些场景下也会影响到排序的表现。

另外还需要综合负载SQL、CPU、MEM、IO等资源情况来调整，对于非资深DBA来说难度较大，目前DBdoctor的AI智能参数调优功能正在内部测试中，敬请期待。