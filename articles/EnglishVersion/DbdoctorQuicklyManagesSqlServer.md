# DBdoctor Quickly manage SQL Server

Recently, we have often received many inquiries from users, asking when can we manage SQLServer? Can't stand the fierce urging and eagerness of the friends, the R & D team that didn't want to manage SQLServer has also put this requirement on the agenda. And in DBdoctor v3.2.2 version, the support for SQLServer management has been successfully implemented.

## 一、What functional services does DBdoctor provide for SQL Server?
### 1.Powerful SQL auditing
SQLServer has enterprise-level SQL audit capabilities, supports batch audit of SQL file uploads, and can complete SQL audits during the development stage to identify SQL problems in advance. At the same time, it can also be reviewed in real time for online existing SQL.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicbz7V6SdbNZrRy9icibiaMeY5FeXFVxocVm8txe7n6zUYUume0KImibX5AA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.Comprehensive inspection and reporting

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDic9WFAW3fG8Rue7UoYXmqZFsbnfjyiciaDiacSKicBldDDaYBicZL4zG4ahoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Support timing and manual inspection modes , engine inspection items reach more than 50 items, support single instance or batch inspection, and support multiple dimensions of report display . Inspection items can be customized, or you can choose inspection items at will to form inspection templates.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZl4TbSDAQg9tnvxOz1UP5IG9KJcvZjaTyYJI07GDVuYbjVPDz98sQKbBYvzG54lIGEy8kpOlylZPQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

### 3.Accurate active discovery

It can automatically analyze the monitoring trend. When the resource jitter exceeds the water level, it will automatically mark the problem interval and perform SQL root cause analysis to match the root cause with the resource, so that slow SQL can be analyzed based on color . Further, according to the correlation and recommendation algorithm, the root cause SQL is given.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicuO6xElqro0jQia0EZTCjno0fOBRIpJibPvfprEq5exxABzdWNUxXpoYg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4. Accurate storage predictions
Support storage analysis based on AI prediction , based on the trained prediction model, real-time display of disk trends in the next week, and storage analysis based on storage events such as data, logs, temporary table files, etc., to obtain storage future trends.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicWNF0EmiaL8UibIuvq5YDl282aydIWuhhJoxuquV7dsahGZ86PDzaHG5w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 5.Rich monitoring metrics

SQLServer engine contains nine types of monitoring indicators, including: host, instance and other dimensions, the monitoring indicators of the engine reach more than 30 items, automatically classify the indicators of the database according to the type of resources, and can flexibly view the indicator data at any time point.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicp91YK99M3C6TREIzVyqBI7wQUFjgeqt01ibgID62FibpmVnoiczHzxz2Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 6.Fine user permissions

The enterprise monitoring platform must have user and authority management, the ability of DBdoctor free version to fully open platform management, and the two-level management and control system based on tenant and project can be flexibly adapted to the company's organizational structure or project structure.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZl4TbSDAQg9tnvxOz1UP5IGHxvI7wWJXb5uKcDu5JLmSv8YMLxvlcT95ppY1d5M6icbhcmww7wMZuQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

Based on user groups, users with the same privileges can be quickly organized together, eliminating the need to configure cumbersome instance and function permissions for new users.

## 二、How to manage SQLServer quickly?

The following describes in detail how to use DBdoctor to manage SQL Server and perform performance diagnostics on it.

### 1.SQL Server database management architecture

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDic900UWDPjp2FF51628C6whHs1WxchfVJtmwFEicJic8BQ9YHaeTgqyFpA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

The SQLServer database is hosted by two nodes, Primary and Standby. DBdoctor Server recommends choosing an independent ECS/host deployment installation. After successful deployment, open the browser access instance list page to manage the Primary and Standby nodes respectively.

By default, Agent is automatically deployed for Data Acquisition. Agent is deployed according to the host dimension, and a single Agent can take over all instance nodes on the host.


### 2.One minute zero dependency DBdoctor Server installation

**Environmental requirements**：4c8g (independent resource deployment is recommended, you can add the option --unlimited ignore the 4c8g limit)

**Download the installation package**：https://www.hisensecloud.com/h-col-126.html

```
#Unzip the installation package and execute a command to complete the deployment
./dbd -I
```
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicibicOkV21UwWqeQtgZBa4m5pESUvfMrS36P54ZD0VFxesrCWZpricsJNA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Service access address**：http:// host IP where the service is deployed:13000

**Login account**：tester/Root2023!

**Detailed documentation**：https://dbdoctor.hisensecloud.com/h-col-144.html
### 3.Quick Admin SQLServer

#### a）Create an access account
```SQL
CREATE LOGIN test WITH PASSWORD ='Root2023!';
GRANT VIEW SERVER STATE TO test;
```
#### b） page to manage instance nodes
- Click "Instance Nanotube" to enter the SQL Server instance node access connection string information and detect connectivity.
- Enter the host account information, select Manual installation, and execute the specified command using PowerShell on the Windows server where the instance is located.
- Click "Submit".

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicoyUO5p23nh2lCccEqJU9abkJRjicXKqvJyIrsn3skpkjux8bZicy0ppw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**Remarks**：When the connectivity test is passed, congratulations on your successful instance management, performance insights have been turned on, start to experience the power of DBdoctor!

## 三、How to use the performance insights feature?

### 1.Turn on performance analysis

Find the SQL Server node that has been managed in the instance list and click Instance Diagnostics.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicqKRuia5C1quicLZPzHmz2FbRImgheNia4wGzicQAicoFdNoTnl8M7ZoJNug/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.View performance insights

Click Performance Insights to gain insight into SQL Server node performance.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicuO6xElqro0jQia0EZTCjno0fOBRIpJibPvfprEq5exxABzdWNUxXpoYg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- Go to the Performance Insights page, you can see that DBdoctor automatically labels the CPU exception problem range, and you can see that CPU exceptions are generated periodically.

- Place the mouse on the magnifying glass to view the root cause SQL that caused the CPU exception. From the figure, you can see that the root cause SQL is a SELECT statement.

- In SQL association analysis, for the CPU exception SQL, click View Execution Plan, find the table missing index, execute CREATE INDEX IX_tempdb_ProductID ON master.dbo.tempdb (ProductID) to add the index problem to solve.

SQL Server resource jitter problem, through the performance insight function only three steps to find the root cause and solve, so simple.