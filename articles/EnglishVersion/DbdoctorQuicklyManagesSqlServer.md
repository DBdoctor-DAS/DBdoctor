# DBdoctor快速纳管SQL Server

最近一段时间，我们经常会收到了许多用户的咨询，问我们何时能纳管SQLServer？耐不住小伙伴们的猛烈催促及热切期待，本不想纳管SQLServer的研发团队也抓紧将这项需求提上日程。并在DBdoctor v3.2.2版本中成功实现了对SQLServer的纳管支持。

## 一、针对SQLServer，DBdoctor提供哪些功能服务？
### 1.强大的SQL审核
SQLServer具备企业级的SQL审核能力，支持SQL文件上传批量审核，在开发阶段即可完成SQL审核，提前识别SQL问题，同时针对线上存量SQL也可以实时抓取进行审核。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicbz7V6SdbNZrRy9icibiaMeY5FeXFVxocVm8txe7n6zUYUume0KImibX5AA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.全面巡检与报表

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDic9WFAW3fG8Rue7UoYXmqZFsbnfjyiciaDiacSKicBldDDaYBicZL4zG4ahoA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**支持定时和手动两种巡检模式**，引擎巡检项达到五十多项，支持单实例或批量巡检，**并支持多个维度的报表展示**。可对巡检项进行自定义，也可随意选择巡检项组成巡检模板。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZl4TbSDAQg9tnvxOz1UP5IG9KJcvZjaTyYJI07GDVuYbjVPDz98sQKbBYvzG54lIGEy8kpOlylZPQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

### 3.精确的主动发现

能够自动分析监控趋势，当资源出现抖动超水位线时会**自动标记问题区间**，并进行**SQL根因分析**，把根因与资源匹配，做到**基于颜色便可分析慢SQL**。进一步根据相关性与推荐算法，给出根因SQL。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicuO6xElqro0jQia0EZTCjno0fOBRIpJibPvfprEq5exxABzdWNUxXpoYg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.精准的存储预测
支持**基于AI预测**的存储分析，基于已训练预测模型，实时展示未来一周的磁盘趋势，并根据数据、日志、临时表文件等的存储事件进行**存储分析**，得出存储未来趋势。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicWNF0EmiaL8UibIuvq5YDl282aydIWuhhJoxuquV7dsahGZ86PDzaHG5w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 5.丰富的监控指标

SQLServer引擎包含九大类监控指标，包含：**主机、实例**等多个维度，引擎的监控指标达三十多项，自动把数据库的指标按照资源类型进行分类，可灵活查看任意时间点的指标数据。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicp91YK99M3C6TREIzVyqBI7wQUFjgeqt01ibgID62FibpmVnoiczHzxz2Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 6.精细的用户权限

企业监控平台必须具备用户与权限管理，DBdoctor免费版全面开放平台管理的能力，基于租户与项目的两级管控体系可以灵活适配公司组织架构或项目架构。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZl4TbSDAQg9tnvxOz1UP5IGHxvI7wWJXb5uKcDu5JLmSv8YMLxvlcT95ppY1d5M6icbhcmww7wMZuQ/640?wx_fmt=other&from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

基于用户组可以快速把相同权限的用户组织在一起，无需为新用户配置繁琐的实例与功能权限。

## 二、如何快速纳管SQLServer？

下面将详细介绍如何使用DBdoctor纳管SQLServer并对其进行性能诊断。

### 1.SQLServer数据库纳管架构

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDic900UWDPjp2FF51628C6whHs1WxchfVJtmwFEicJic8BQ9YHaeTgqyFpA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

SQLServer数据库主备为两个节点，分别是Primary和Standby。DBdoctor Server建议选择独立ECS/主机部署安装，成功部署后打开浏览器访问实例列表页面，分别对Primary、Standby节点进行纳管。

实例纳管时默认自动部署Agent进行数据采集，Agent按照主机维度进行部署，单个Agent可接管主机上的所有实例节点。


### 2.一分钟零依赖DBdoctor Server安装

**环境要求**：4c8g（建议独立的资源部署，可以添加选项--unlimited忽略4c8g的限制）
**下载安装包**：https://www.hisensecloud.com/h-col-126.html

```
#解压安装包并执行一条命令即可部署完成
./dbd -I
```
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicibicOkV21UwWqeQtgZBa4m5pESUvfMrS36P54ZD0VFxesrCWZpricsJNA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**服务访问地址**：http://<部署服务的主机ip>:13000

**登录账号**：tester/Root2023!

**详细文档**：https://dbdoctor.hisensecloud.com/h-col-144.html
### 3.快速纳管SQLServer

#### a）创建访问账号
```SQL
CREATE LOGIN test WITH PASSWORD ='Root2023!';
GRANT VIEW SERVER STATE TO test;
```
#### b）页面纳管实例节点
- 点击“实例纳管”录入SQLServer实例节点访问连接串信息，并检测连通性。
- 录入主机账号信息，选择手动安装，并在实例所在windows服务器上使用PowerShell执行指定命令。
- 点击 "提交"即可。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicoyUO5p23nh2lCccEqJU9abkJRjicXKqvJyIrsn3skpkjux8bZicy0ppw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**备注**：当连通性检测通过后，恭喜你实例纳管成功，性能洞察已开启，开始体验DBdoctor的强大吧！

## 三、如何使用性能洞察功能？

### 1.开启性能分析

实例列表中找到已纳管的SQLServer节点，点击实例诊断。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicqKRuia5C1quicLZPzHmz2FbRImgheNia4wGzicQAicoFdNoTnl8M7ZoJNug/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.查看性能洞察

点击性能洞察透视SQLServer的节点性能。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZmlOcQluTicsyOryNXyFJRDicuO6xElqro0jQia0EZTCjno0fOBRIpJibPvfprEq5exxABzdWNUxXpoYg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 进入到性能洞察页面，可以看到DBdoctor自动标注CPU异常问题区间，并且看到周期性产生CPU异常。

- 将鼠标放在放大镜上，查看导致CPU异常的根因SQL。从图中可以看到根因SQL为SELECT语句。

- 在SQL关联分析中，针对该CPU异常SQL，点击查看执行计划，发现该表缺失索引，执行CREATE INDEX IX_tempdb_ProductID ON master.dbo.tempdb (ProductID) 添加索引问题解决。

SQLServer资源抖动问题，通过性能洞察功能只需要三步即可找到根因并解决，如此简单。