# Database index recommendation big PK, the ultimate competition between DBdoctor and senior DBAs

## Preface
In the previous article ["Dragon Boat Festival Special Edition: Do You Really Understand Database Indexes?"](../../articles/DoYouReallyKnowAnythingAboutDatabaseIndexing.md) In this article, Ji Kuan raised a question about a business SQL recommended index optimization problem. He found that the index combination (status, purchase_date, device_name, device_id) recommended by DBdoctor seemed to be very different from the index scheme recommended by him as a DBA with many years of experience. Who is right or wrong between DBdoctor and senior DBA? In this article, we will decipher the PK results of DBdoctor and senior DBA in detail, and I will use actual verification and analysis to beat him hard!

```SQL
SELECT *
FROM 
    device
WHERE 
    purchase_date >= '2023-05-31'
    AND status = 'active' and device_id>=0
  AND device_name like '%162b%'
```
## SQL analysis

In order to verify the accuracy of the above recommended indexes, we added the indexes recommended by DBA experience and DBdoctor, and finally handed them over to MySQL itself to see which index it would choose.

- Recommended by DBA:

```SQL
KEY `dbdoctor_idx_status_purchase_date` (`status`,`purchase_date`）
```
- Recommended by DBdoctor:
```SQL
KEY `dbdoctor_idx_status_purchase_date_device_name_device_id` (`status`,`purchase_date`,`device_name`,`device_id）`
```

**DBA's conclusion**：that DBdoctor's recommendation is not accurate, completely inconsistent with the principles and rules of MySQL, such as the leftmost principle and fuzzy match, what is the real situation?

## Manually verify who is the optimal index

### 1. Add the above three indexes to the device table, and the table structure is as follows:
```SQL
CREATE TABLE `device` (
  `id` int NOT NULL AUTO_INCREMENT,
  `device_id` int DEFAULT NULL,
  `device_name` varchar(255) COLLATE utf8mb4_general_ci NOT NULL,
  `device_type` varchar(100) COLLATE utf8mb4_general_ci NOT NULL,
  `purchase_date` date DEFAULT NULL,
  `status` varchar(50) COLLATE utf8mb4_general_ci DEFAULT 'active',
  `created_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `dbdoctor_idx_status_purchase_date_device_name_device_id` (`status`,`purchase_date`,`device_name`,`device_id`),
  KEY `dbdoctor_idx_purchase_date_status` (`purchase_date`,`status`),
  KEY `dbdoctor_idx_status_purchase_date` (`status`,`purchase_date`)
) ENGINE=InnoDB AUTO_INCREMENT=20446488 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci
```
Number of index sampling pages and sampling records:

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOFonK4pz559AwczhhNRMMGpNH4QkS4xm4znhX2EuNhiaThU3gXQNRmFw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2. Open trace to check the actual cost consumption of three SQL indexes, and find that DBdoctor recommends the best.

```SQL
set session optimizer_trace="enabled=on";
```

After obtaining the following trace cost consumption information, we found that the index recommended by DBdoctor dbdoctor_idx_status_purchase_date_device_name_device_id is the optimal one with the lowest cost consumption.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOs3RHliaUzVLXAOichePGxjmoOicDeCp8sX3FkXy4utaA5Cbv5r3vgLia6g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Are DBAs starting to doubt their lives? Why is that? Let's further analyze it below.

## Further inquire into the root cause

Returning to these two indexes:

```
DBdoctor recommendation:
KEY `dbdoctor_idx_status_purchase_date_device_name_device_id` (`status`,`purchase_date`,`device_name`,`device_id`)

DBA recommendation:
KEY `dbdoctor_idx_status_purchase_date` (`status`，`purchase_date`)
```

If you are careful, you can notice that there are two suspicious points in the two index traces above.

- DBA recommended index has enabled MRR algorithm

- The index rows recommended by DBdoctor and DBA have different results

Now let's further analyze these two doubts:

#### 1.Let's take a closer look at the trace log above and find that the MRR optimization algorithm is also included in the index recommended by the DBA. This is equivalent to doing an extra sorting and then checking other fields in the table, which can replace discrete IO with sequential IO and improve performance. Could it be that there is a problem with this MRR, and the recommended algorithm is not optimal, resulting in an increase in overall cost overhead?

Close the MRR algorithm and verify whether MRR is the cause of the increase in COST consumption.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOW3S6NFI4cSvZU1uPW9gstSvMqT5JPlrfxL2VonaqPVSwkhhxvCI2jw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

We found that dbdoctor_idx_status_purchase_date enable mrr, the cost overhead is reduced a lot, bringing optimization, cost reduction 533216-509474 = 23742

**Conclusion:**The MRR algorithm brings optimization, and the algorithm recommended by MySQL is not a problem. Why is the remaining doubt Rows different?
#### 2.From the conclusion of the above investigation, the problem lies in the estimated number of scanned rows. We check the execution plan separately through forced indexing to see the scanned rows.

- DBdoctor recommended index scan behavior 439628

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOxX0L42mUwv8D1ZVdd02fflZiabYK73eniboALp4L3lHeYUR0SXchplcw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- DBdoctor recommended index scan behavior 439628

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOmKiclzt0qgFLHbzNEnGO6JqX4KexfcUfDMADc9AEAdjphopbtojTT3Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

The scanned rows of the index recommended by DBA are too large, and more than 484744-439628 = 45116 rows of records are scanned. From the MySQL source code Cost calculation formula, we know that the cost of each row is the Server layer row_evalute_cost (default is 0.1), so part of the reason why the index is too large comes from this scanned row. So why is there such a big difference in rows?

**Let's take a look at the MySQL source code. How did Rows come about?**

Through GDB, we quickly found the function for estimating rows. The recursion function btr_estimate_n_rows_in_range_on_level used to estimate the number of index rows between two slots at a certain level in the B-tree. The general principle is as follows:

```C
/* 4867 */static int64_t btr_estimate_n_rows_in_range_on_level(
/* ...  */
/* 4876 */    bool *is_n_rows_exact)        /*!< out: true if the returned
/* 4877 */                                   value is exact i.e. not an
/* 4878 */                                   estimation */
/* 4879 */{
/* 4880 */  int64_t n_rows;
/* 4881 */  ulint n_pages_read;
/* 4882 */  ulint level;
/* 4883 */
/* 4884 */  n_rows = 0;
/* 4885 */  n_pages_read = 0;
/* 4886 */
/* 4887 */  /* Assume by default that we will scan all pages between
/* 4888 */  slot1->page_no and slot2->page_no. */
/* 4889 */  *is_n_rows_exact = true;
/* ...  */
/* 4905 */  /* Count the records in the pages between slot1->page_no and
/* 4906 */  slot2->page_no (non inclusive), if any. */
/* 4907 */
/* 4908 */  /* Do not read more than this number of pages in order not to hurt
/* 4909 */  performance with this code which is just an estimation. If we read
/* 4910 */  this many pages before reaching slot2->page_no then we estimate the
/* 4911 */  average from the pages scanned so far. */
/* 4912 */
/* 4913 */  constexpr uint32_t N_PAGES_READ_LIMIT = 10;
/* ...  */
/* 4940 */    /* It is possible that the tree has been reorganized in the
/* 4941 */    meantime and this is a different page. If this happens the
/* 4942 */    calculated estimate will be bogus, which is not fatal as
/* 4943 */    this is only an estimate. We are sure that a page with
/* 4944 */    page_no exists because InnoDB never frees pages, only
/* 4945 */    reuses them. */
/* 4946 */    if (!fil_page_index_page_check(page) ||
/* 4947 */        btr_page_get_index_id(page) != index->id ||
/* 4948 */        btr_page_get_level(page) != level) {
/* 4949 */      /* The page got reused for something else */
/* 4950 */      mtr_commit(&mtr);
/* 4951 */      goto inexact;
/* 4952 */    }
/* ...  */
/* 4985 */inexact:
/* 4986 */
/* 4987 */  *is_n_rows_exact = false;
/* 4988 */
/* 4989 */  /* We did interrupt before reaching slot2->page */
/* 4990 */
/* 4991 */  if (n_pages_read > 0) {
/* 4992 */    /* The number of pages on this level is
/* 4993 */    n_rows_on_prev_level, multiply it by the
/* 4994 */    average number of recs per page so far */
/* 4995 */    n_rows = n_rows_on_prev_level * n_rows / n_pages_read;
/* 4996 */  } else {
/* 4997 */    /* The tree changed before we could even
/* 4998 */    start with slot1->page_no */
/* 4999 */    n_rows = 10;
/* 5000 */  }
/* 5001 */
/* 5002 */  return (n_rows);
/* 5003 */}
```
- Starting from the start page:

    Starting from the page where the starting slot slot1 is located, scan several pages to the right.

- Count the number of rows:

    Read each page and count the number of records it contains.

- Determine if the target page has been reached.

If the page where the target slot slot2 is located is reached quickly during the scanning process, the number of records between slot1 and slot2 can be accurately calculated, and the is_n_rows_exact flag is set to true.

- Estimate the number of rows on the remaining pages.

    - If the page where slot2 is located is not reached quickly, calculate the average number of records in the scanned pages.

    - Based on this average, estimate the number of records in unscanned pages. Assuming the number of records in these pages is the same as the number of records in scanned pages.

    - Multiply by the number of pages between slot1 and slot2 to estimate the total number of records.
- Return result:

    Return the estimated number of rows (excluding boundary records), and set the is_n_rows_exact flag according to whether it is accurately calculated. This recursion function attempts to accurately calculate the number of rows by quickly scanning the initial few pages. If it cannot be accurately calculated, it is estimated based on the average value to achieve a balance between performance and accuracy. From the trace, we can see that the SQL of the two index paths has the same range, but the index height is different, the number of recursions is different, and the average number of page records is also different, resulting in different estimated rows.

From the above data, we can see that the additional cost = the index recommended by DBA Cost-DBdoctor recommended index Cost=533216-483589=49627, the main reasons for the increase in the cost of DBA recommended index are as follows:

- Estimated Sever CPU Cost of Bias Rows = 45116 * 0.1 = 4511.6

- Estimated Deviation Rows Page IO Cost (default is 1 per io_read_block_cost)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZnvgiaHYkP7H9bhQcAN1ozjOwqmw666bdgWOr7t4UvG5BHWxLWsASaUDwic0tyMRl8pYIhS5TTDA2bw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**In summary**：Accurate estimation of the number of rows and the estimated number of rows from the average value have data biases, which will also affect the final selection of the index. This is determined by the solidified code logic of the inner core. DBA cannot perceive this layer when making index recommendations, and cannot cover this scene through regular methods. However, through DBdoctor's eBPF technology, the data at this level can be perceived, and the optimal path consistent with MySQL selection can be made. As can be seen from the above execution results **The index recommended by DBdoctor takes twice as much time to execute as the index recommended by DBA.**

**Summary**

DBdoctor's optimal path analysis based on actual cost is the most accurate. The experience rules of DBAs can indeed bring optimization in certain scenarios, but it is difficult to handle in some scenarios, such as the above scenario. Here, I have briefly listed the following points:

- Multiple algorithm paths will result in different Costs

- The path behavior is different in different versions, resulting in different costs

- Estimated index rows are different, Cost is different

- Additional variable factors can also cause changes

**In summary, DBdoctor's bypass optimizer is the ultimate killer! Regardless of the scenario, using SQL audit function to copy and paste SQL can quickly reach the optimal index conclusion in one minute.**

