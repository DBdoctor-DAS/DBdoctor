# DBDoctor How to Manage Public Cloud RDS

Recently, many friends have asked if DBdoctor can manage RDS on Public Cloud, and the answer is of course yes. Taking Alibaba Cloud Ali Cloud Aliyun as an example, let's introduce DBdoctor's custom extension RDS module and management.

## 1）DBdoctor collection supports three types of data source access
DBdoctor supports three data sources for collection to meet different user access scenarios.

- DBdoctor agent Data Acquisition: It needs to be deployed on the host of the database node, supports automatic installation of page configuration, and has more in-depth data indicator collection.

- Prometheus data access: If the user already has the Data Acquisition tool and provides monitoring data in Promethus format, DBdoctor can seamlessly integrate with Promethus without the need to redeploy the acquisition tool.

- Custom third-party extensions: DBdoctor supports custom third-party service interfaces, which can be customized and adapted using Python scripts, such as connecting to the RDS OpenAPI of Public Cloud.
## 2）DBdoctor quick deployment in one minute, zero dependencies
```
#Environmental requirements: Recommended 4c8g
#Download the installation package：https://www.hisensecloud.com/h-col-133.html
#Official installation documentation：https://dbdoctor.hisensecloud.com/modules/dbDoctor/mdPreview/index.html?readme=help#/mdManageDocument/6.1.1-Quick-Install
```
## 3）DBdoctor Quick Custom Extension Compatible with Third-Party Monitoring Services
After setting up the DBdoctor server-side service, you need to install the Python SDK module of Alibaba Cloud Ali Cloud Aliyun's OpenAPI.

#### a）Install Alibaba Cloud Ali Cloud Aliyun SDK

##### 1、Cd to the DBdoctor installation directory and execute
```Bash
chroot dbdoctor-3.1.2
```
##### 2、Execute the following command to install Alibaba Cloud Ali Cloud Aliyun SDK (it is recommended to use the domestic mirroring source).
```Bash
pip3 install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple
pip3 install alibabacloud_rds20140815==5.0.0 -i https://mirrors.aliyun.com/pypi/simple
pip3 install flask -i https://mirrors.aliyun.com/pypi/simple
```
#### b）Create and connect Alibaba Cloud Ali Cloud Aliyun RDS OpenAPI script template

1. Use admin user (default password: 123456) to log in to DBdoctor, click Data Acquisition Management - > Template Configuration - > Create Template


![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk6voTsqLmkFxnX1TgHjiaEKuIZRNqeNUGlccRKvmErBY2OV3rQicvTngQedSt8dZTPNKcVuxz1DEjA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. DBdoctor provides a third-party monitoring extension module, which can be customized to write scripts. After clicking Create Template, select Custom Template Type , and fill in the template name, template description, copy the following RDS OpenAPI docking script to the Python code text box, and click Submit.

Below is the Python code for expanding the Alibaba Cloud Ali Cloud Aliyun RDS template.

```
# Alibaba Cloud Ali Cloud Aliyun RDS Template Python Code Download Address:
https://jhktob.oss-cn-beijing.aliyuncs.com/aliyun.py
# Note: access_key_id and access_key_secret in template code should be replaced with real aksk.
```
Alibaba Cloud Ali Cloud Aliyun's aksk requires Console permissions for the following two interfaces:
```
# OpenAPI interface that requires Alibaba Cloud Ali Cloud Aliyun Console authorization
# Interface 1：
DescribeDBInstanceAttributeRequest
# Interface 2
DescribeDBInstancePerformanceRequest
```
### 4）Alibaba Cloud Ali Cloud Aliyun RDS
1. Log out of the admin account and log in with the tester account (default password: Root2023!).

2.  Click Instance Management, where the database collection method is selected from the newly created template above. Third-party parameter configuration, the key value is instanceId, the value is the instance ID of the instance to be managed in Alibaba Cloud Ali Cloud Aliyun, and finally click Submit.

![Instance tube](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwg1qLH9NVTmbDSttZLficM1qtT2jDicdo2ONeArGTJogzk34NlDYmUoqNw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. Finally, just open the performance analysis.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgD1PDRFQ9bnUha3l9bVEnEiaUKwRm0g28ordMqsL7WyicRNPEGiaiakcFEg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5）Feature introduction

#### a）Performance Insights
1.  Find the RDS database that has been managed in the example list, and click the Performance Insights switch button to start the analysis.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgD1PDRFQ9bnUha3l9bVEnEiaUKwRm0g28ordMqsL7WyicRNPEGiaiakcFEg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
2. Click on the instance to see through the performance of the RDS database
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgQJ0X4kONy5Bp0Bd53zjiaJEHnJ9KvlVsUeJKq5GptPFsAh8A11vuicmQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

The performance insight function can associate hardware resources with the performance data of the database through the timeline and perform multi-dimensional analysis. For example, the above CPU exception case we only need 1 step to find the root cause of the problem SQL, and can also restore the detailed scene of the exception time interval.

- Automatic root cause recognition: When the CPU resource indicator experiences jitter, the exception interval is automatically selected and the root cause SQL is prompted.

- Detailed on-site restoration:
    - CPU exception time interval box selection
    - In the AAS module, we can see that the number of active sessions in the database far exceeds the Max vCPU watermark during this time interval, indicating a performance bottleneck.
    - The event that shows the performance bottleneck is ACTIVE (i.e. blue-colored event), which occupies the largest area in the AAS trend chart.
    - Based on the event with the largest area and color, we can find the green SQL as the first one, which is the root cause of the CPU surge. Clicking to expand this SQL can display the worst sample of this SQL. Clicking on the execution plan to find and scan the entire table can add an index to the SQL.
#### b）SQL Audit
DBdoctor not only supports the mainstream rule-based audit in the industry, but also innovatively implements SQL audit covering performance diagnosis. Based on online data models, it can quickly and accurately evaluate SQL performance issues and provide optimization suggestions before release, avoiding online failures in advance.
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwg0lBAOpaRhG4p2QjWDyQKCzUjjsv8mq5pxWdesic768mhTTaIsRMgExg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Exclusive industry pre-launch evaluation of SQL performance audit, principle introduction: The strongest SQL audit tool in history!

OMG, managing Alibaba Cloud Ali Cloud Aliyun RDS and conducting database performance diagnosis and SQL audit is surprisingly simple! Moreover, for RDS, DBdoctor also supports functions such as instance inspection, basic monitoring, root cause diagnosis, index recommendation, etc. Such a practical and free tool, not using it is really a waste!

